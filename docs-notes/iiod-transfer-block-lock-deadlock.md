# iiod deadlock #3 — `entry->lock` held across a blocking socket read in `handle_transfer_block`

_Date: 2026-06-26. Branch: `fix/iiod-deadlock`. Proven by gdb backtrace on the
ARM board._

## Verdict

This is a **real iiod bug** — a defect in the daemon, reachable through ordinary,
correct client API usage. It is independent of the stress test (the test is a
valid reproducer) and independent of the client-library fixes already made.

It is the **third distinct deadlock** found in this effort:

| # | Side | Location | Status |
|---|------|----------|--------|
| 1 | client lib | `stream.c` `iio_stream_destroy` did blocking dequeue before cancel | ✅ fixed |
| 2 | server | `iiod/responder.c` `handle_free_block` worker stuck in `poll()` on disabled buffer | ✅ fixed (cancel before `cancel_sync`) |
| 3 | server | `iiod/responder.c` `handle_transfer_block` holds `entry->lock` across a blocking network read | ⛔ **this doc** |

Fixes #1 and #2 are confirmed working by the backtrace (the `buffer-dequeue` /
`buffer-enqueue` workers are idle in `iio_cond_wait`, not stuck in `poll()`).
Removing those deadlocks let teardown run at full speed, which **exposed** this
pre-existing latent bug.

## The proof (gdb, server side, ARM board)

The hung thread (this was a hang; the `SIGINT` at the top of the dump was the
operator pressing Ctrl-C):

```
Thread 592 "reader-thd":
  #2  async_io (interpreter.c:42)              ← blocked in io_submit / poll
  #3  readfd_aio
  #4  read_all (rw.c:33)
  #5  iiod_read (responder.c:1230)
  #6  iiod_rw_all (iiod-responder.c:156)
  #7  iiod_command_data_read (iiod-responder.c:200)
  #8  handle_transfer_block (responder.c:828)  entry=0xb5606ac8 block_entry=0xb56071f0
                                               cmd = {op=23 TRANSFER_BLOCK, dev=4, code=131072}
```

`responder.c:814` takes `iio_mutex_lock(entry->lock)`; `responder.c:827` then does
the blocking `iiod_command_data_read` to read the 8-byte `bytes_used`. **The lock
is held across the read.** The read never completes because the client cancelled
the buffer connection mid-command (during `iio_stream_destroy`) and has not yet
closed the socket — it is blocked elsewhere, see below.

## The deadlock cycle

Two TCP connections per context (network backend):

```
connection A  = main context socket   (pdata->iiod_client / client_fb)
connection B  = per-buffer socket      (buf->iiod_client)   ← opened by network_open_buffer
```

Command routing (confirmed in iiod-client.c / network.c):

| Command | Travels on |
|---------|-----------|
| OPEN_BUFFER, CREATE_BLOCK, **TRANSFER_BLOCK**, DEQUEUE | connection B |
| **FREE_BLOCK**, CLOSE_BUFFER | connection A (`client_fb`) |

But the server's `bufferlist` is **global, keyed only by `(device, idx)`**
(`get_iio_buffer_entry_unlocked`, responder.c:128). So commands arriving on either
connection resolve to the **same `buffer_entry` and the same `entry->lock`**.

The cycle:

1. **Server, connection B reader** is in `handle_transfer_block`, holding
   `entry->lock`, blocked in `iiod_command_data_read` waiting for `bytes_used`
   bytes the client will never send.
2. **Client** is in `iio_stream_destroy` → `iio_block_destroy` →
   `iiod_client_free_block`, blocked in `iiod_io_wait_for_response` for the
   `FREE_BLOCK` reply on connection A. (Proven by the earlier client backtrace,
   Thread 6.)
3. **Server, connection A reader** runs `handle_free_block`, which calls
   `iio_mutex_lock(buf_entry->lock)` (responder.c:756) — **the lock held by
   connection B's reader.** It blocks.
4. The client won't close connection B (that happens later in `iio_buffer_close`),
   so connection B's read never gets EOF. Permanent deadlock.

## Why it sometimes segfaults instead of hanging

The naïve fix — "just release `entry->lock` during the read" — introduces a
**use-after-free**: while connection B is mid-read with the lock released,
connection A's `handle_free_block` (or `CLOSE_BUFFER` → `free_buffer_entry`) takes
the lock and frees the `block_entry` / `iio_block` (and for `CLOSE_BUFFER`, the
`buffer_entry` and `entry->lock` themselves). When the read returns and the
transfer handler dereferences the freed block / re-locks the freed mutex → crash.

So the **hang and the segfault are the same bug**: a lock that is simultaneously
(a) held across blocking I/O and (b) the only thing keeping the block/buffer
alive during that I/O. Depending on timing and build you get one or the other.
This is why the symptom kept changing.

## The fix (design)

Two invariants must hold at once:

1. **Never hold `entry->lock` across a blocking socket read.** Otherwise any
   control-plane op on the other connection deadlocks.
2. **The `block_entry` and `buffer_entry` must not be freed while a transfer
   handler is reading into / about to commit them.** Otherwise UAF.

Implementation (contained to `iiod/responder.c`):

- Add to `buffer_entry`: a `struct iio_cond *cond` and a counter of in-flight
  transfer handlers (`active_transfers`).
- Add to `block_entry`: a `bool busy` flag.
- `handle_transfer_block`:
  1. lock `entry->lock`, look up the block, set `block_entry->busy = true`,
     `active_transfers++`, **unlock**.
  2. do the blocking read(s) with the lock released.
  3. re-lock; if the block/buffer is being torn down, bail without committing;
     else commit (`bytes_used`, enqueue token); clear `busy`, `active_transfers--`,
     `iio_cond_signal`; unlock.
- `handle_free_block`: while the target block is `busy`, `iio_cond_wait` on
  `entry->cond` (releasing `entry->lock`) until it clears, then free as today.
- Buffer teardown (`handle_close_buffer` / `iiod_responder_free_resources` →
  `free_buffer_entry`): after removing the entry from `bufferlist` under
  `buflist_lock`, wait for `active_transfers == 0` before destroying the lock/cond
  and freeing the entry, so a mid-read transfer handler can safely re-lock.

This removes the lock-across-I/O (kills the deadlock) and makes the free paths
wait for in-flight transfers (kills the UAF).

## Validation plan (important — this is concurrency code)

This fix must be proven, not just observed:

1. **AddressSanitizer, locally:** build libiio + iiod with `-fsanitize=address,undefined`,
   run iiod over TCP loopback against an emulated/xml context, point
   `iio_stresstest` at `ip:127.0.0.1` with many threads. ASan will flag any
   remaining use-after-free with both the alloc/free and use stacks. A clean run
   under load is the acceptance bar.
2. **Board:** run iiod under `gdb` during a long multi-threaded stresstest; confirm
   no thread parks in `handle_transfer_block` holding `entry->lock`, and the test
   makes continuous progress.

## Scope note

The `EEXIST` spam seen with many threads is **expected and unrelated**: a hardware
buffer `(device, idx)` can only be opened by one client at a time, so concurrent
clients racing to open the same device buffer get `-EEXIST` and retry. It is not a
bug and needs no fix.
