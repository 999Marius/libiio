# iiod v1 — `iio_stream_destroy` teardown deadlock: root cause & fix

_Investigation date: 2026-06-26. Branch: `fix/iiod-deadlock`._

## TL;DR

The permanent hang in `iio_stresstest` against real hardware is a **two-sided
deadlock** in the libiio v1 binary protocol teardown path. Both sides must be
fixed:

| Side | File / function | Problem | Fix |
|------|-----------------|---------|-----|
| **Client** | `stream.c` `iio_stream_destroy` | Does a *blocking* `iio_block_dequeue` before cancelling, so it waits forever for a response that never comes — and never reaches the code that sends `FREE_BLOCK`. | Call `iio_stream_cancel(stream)` first (✅ applied). |
| **Server** | `iiod/responder.c` `handle_free_block` | `iio_task_cancel_sync(dequeue_token)` is a no-op on an in-flight token, so it waits for a dequeue worker stuck in `poll()` on a disabled buffer. | Call `iio_buffer_stream_cancel(buf_entry->buf_stream)` before the `cancel_sync` (✅ already in working tree). |

The earlier attempts each fixed **one** side. The client-side fix is the piece
that was missing: without it, `FREE_BLOCK` is never even sent, so the
already-deployed server-side fix never gets a chance to fire.

The stress test itself is **correctly written** for the v1 protocol — the
deadlock is a library bug, not a test bug. See the last section.

---

## The architecture you need to hold in your head

For the **network backend**, a single `iio_context` owns **two kinds of socket**:

```
iio_context ─┬─ main socket  (pdata->iiod_client)      ← context-wide ops
             │     • OPEN_BUFFER / CLOSE_BUFFER
             │     • FREE_BLOCK            ← note this!
             │     • attrs, trigger, reg, NOP, events
             │
             └─ per-buffer socket (buf->iiod_client)   ← opened by network_open_buffer
                   • CREATE_BLOCK
                   • TRANSFER_BLOCK / ENQUEUE / RETRY_DEQUEUE
                   • the actual sample data stream
```

Confirmed in code:
- `network_open_buffer` (`network.c:444`) opens a **second** TCP connection via
  `network_setup_iiod_client`, stored in `buf->iiod_client` / `buf->io_ctx`.
- `iiod_client_open_buffer(client, client_fb, ...)` (`iiod-client.c`) stores the
  buffer socket as `pdata->client` and the **main** socket as `pdata->client_fb`.
- `iiod_client_free_block` (`iiod-client.c:1630`) sends `FREE_BLOCK` over
  `block->buffer->client_fb` — **the main socket**.
- `iiod_client_close_buffer` (`iiod-client.c:1506`) sends `CLOSE_BUFFER` over
  `client_fb` — **the main socket**.
- `iiod_client_dequeue_block` / `enqueue_block` use `block->io`, which lives on
  the **buffer socket** responder.

This split is the crux: **cancelling the buffer socket does not disturb the main
socket**, which is exactly why the fix is safe — `FREE_BLOCK`/`CLOSE_BUFFER` can
still complete on the healthy main socket after the buffer stream is cancelled.

---

## The hang chain (as proven by gdb), explained

### Client side — `iio_stream_destroy` (`stream.c:85`, before fix)

```c
for (i = 0; i < stream->nb_blocks; i++) {
    if (stream->blocks[i]) {
        iio_block_dequeue(stream->blocks[i], false);  // ← BLOCKS FOREVER
        iio_block_destroy(stream->blocks[i]);          //   never reached
    }
}
...
iio_buffer_close(stream->buf_stream);  // the only cancel — never reached
```

For a network block, `iio_block_dequeue` → `ops->dequeue_block` →
`iiod_client_dequeue_block` → `iiod_io_wait_for_response` (`iiod-responder.c:475`).
That waits on a condition variable for the server's dequeue response. The
context timeout was set to `IIO_TIMEOUT_INFINITE`, so `iiod_io_cond_wait` blocks
with `timeout_ms < 0` → `iio_cond_wait(..., -1)` — **no timeout**.

The response never comes (server is stuck — see below), so the loop never
advances. `iio_block_destroy` (which would send `FREE_BLOCK` via
`ops->free_block`) and `iio_buffer_close` (which would call `cancel_buffer`) are
both **after** the blocking dequeue and are never reached.

→ **The client never sends `FREE_BLOCK`.** This is why the server-side fix in
`handle_free_block` "still hangs" — the command that would trigger it never
arrives.

### Server side — why the dequeue worker is stuck

- The `buffer-dequeue` task (`buffer_dequeue_block`, `responder.c:332`) calls
  `iio_block_dequeue` → `local_dequeue_mmap_block` (`local-mmap.c:216`) →
  `buffer_check_ready` → `poll()` (`local.c:229`) on the DMA fd + `cancel_fd`.
- If the buffer was disabled, the ADC DMA never completes, so `poll()` blocks
  until `cancel_fd` is signalled.
- The reader thread, handling `FREE_BLOCK`, calls
  `iio_task_cancel_sync(dequeue_token, -1)` (`responder.c:773`).

`iio_task_cancel` (`task.c:359`) only sets `done` **if the token is still in the
task's pending list**. Once the worker has pulled the token off the list and is
executing `task->fn` (i.e. sitting in `poll()`), `iio_task_token_find` returns
false and **cancel does nothing** (`task.c:376-377`). So `cancel_sync` falls
through to `iio_task_sync_core`, which waits on `done_cond` for the worker to
finish — which it won't, because it's parked in `poll()`.

→ Classic "cancel is a no-op on an in-flight token" — the comment at
`task.c:376` documents exactly this.

### How the two sides lock together

```
client iio_stream_destroy
   └─ iio_block_dequeue (blocking, infinite) ──waits──┐
                                                       │ response that never comes
server buffer-dequeue worker                           │
   └─ poll() on disabled DMA fd ──────────────waits────┘ (no cancel_fd signal)

…and even if the client got past dequeue, FREE_BLOCK would hit:
server handle_free_block
   └─ iio_task_cancel_sync(dequeue_token) ──waits for the same stuck worker
```

---

## The fix

### Server side (already in the working tree — keep it)

`iiod/responder.c` `handle_free_block`, before the `cancel_sync` calls:

```c
/* Cancel any in-flight I/O before waiting on the tokens. */
iio_buffer_stream_cancel(buf_entry->buf_stream);

iio_task_cancel_sync(entry->enqueue_token, -1);
iio_task_cancel_sync(entry->dequeue_token, -1);
```

`iio_buffer_stream_cancel` (`buffer.c:55`) calls `ops->cancel_buffer`
(`local_cancel_buffer` → `local_signal_cancel_fd`, `local.c:1523`) which writes
to the `cancel_fd` eventfd. That makes the worker's `poll()` return, and
`buffer_check_ready` returns `-EBADF` (`local.c:232`). The worker fn returns, the
token's `done` is set, and `cancel_sync` unblocks. ✅

### Client side (the missing piece — applied in this session)

`stream.c` `iio_stream_destroy`:

```c
void iio_stream_destroy(struct iio_stream *stream)
{
    size_t i;

    /* Cancel any in-flight buffer I/O before draining the blocks below. */
    iio_stream_cancel(stream);          // ← ADDED

    for (i = 0; i < stream->nb_blocks; i++) {
        if (stream->blocks[i]) {
            iio_block_dequeue(stream->blocks[i], false);  // now returns promptly
            iio_block_destroy(stream->blocks[i]);          // sends FREE_BLOCK
        }
    }

    free(stream->blocks);
    iio_buffer_close(stream->buf_stream);
    free(stream);
}
```

`iio_stream_cancel` → `iio_buffer_stream_cancel(stream->buf_stream)` →
`network_cancel_buffer` → `network_cancel` → `do_cancel`, which signals the
**buffer socket's** `cancel_fd` (`network-unix.c:91`). The client's blocked
`iiod_io_wait_for_response` is woken because the buffer-socket reader thread's
`recv`/`poll` returns `-EBADF`, the reader worker breaks out
(`iiod-responder.c:268`), and `iiod_responder_cancel_responses` signals every
pending IO (`iiod-responder.c:225`). `iio_block_dequeue` then returns an error
immediately, the loop advances, and `iio_block_destroy` sends `FREE_BLOCK` over
the still-healthy **main socket**. ✅

### Why calling cancel twice is safe

`iio_buffer_close` (`buffer.c:184`) calls `iio_buffer_stream_cancel` again.
That's harmless:
- `network_cancel` guards on `io_ctx->cancelled` (`network.c:153`) — second call
  is a no-op.
- `iio_task_stop` on an already-idle worker just signals and returns.

---

## Note / latent issue worth flagging (not blocking)

The server-side `iio_buffer_stream_cancel` inside `handle_free_block` signals the
local `cancel_fd`, which is a **sticky eventfd that is never drained/reset**
(there is no `read()` of `cancel_fd` anywhere). Once signalled,
`buffer_check_ready` returns `-EBADF` **for every subsequent dequeue on that
buffer**. That is fine during teardown (all blocks are being freed and the
buffer closed right after), because:
- the client frees **all** blocks, then closes the buffer, and
- `free_buffer_entry` (`responder.c:58`) cancels the stream anyway.

But if `FREE_BLOCK` were ever issued for a single block while the buffer is meant
to **keep streaming**, this would wrongly poison the whole buffer. In the current
API/usage blocks are only freed at teardown, so this is latent, not active. If
you want to harden it, the buffer-wide cancel should move to
`handle_disable_buffer`/`handle_close_buffer` and `handle_free_block` should only
cancel the specific block's token — but that requires a per-block cancel
mechanism the local MMAP backend doesn't currently have (single `cancel_fd` per
buffer). Recommend leaving as-is for this fix and tracking separately.

---

## Is `iio_stresstest` written correctly for the v1 binary?

**Yes — the test is correct; the bug is in the library.** No changes are required
in `iio_stresstest.c`.

What the test does (per thread, `utils/iio_stresstest.c:315-362`):

```c
stream = iio_buffer_create_stream(buffer, 4, buffer_size, mask);
while (threads_running || i == 0) {
    block = iio_stream_get_next_block(stream);   // may block on server
    ...
    if (rand() % N == 0) break;                  // <-- breaks with blocks in flight
}
iio_stream_destroy(stream);                       // <-- teardown with in-flight blocks
```

This is a **legitimate and intended** API usage pattern: create a stream, pump
blocks, then tear it down — possibly while blocks are still enqueued on the
server. A correct library must handle `iio_stream_destroy` with in-flight blocks
without deadlocking. That's the whole point of `iio_stream_cancel` existing as a
public function.

Two test characteristics that *expose* (not cause) the bug:
1. **`iio_context_set_timeout(ctx, IIO_TIMEOUT_INFINITE)`** (`line 276`) removes
   the timeout safety net. With a finite timeout the blocked dequeue would
   eventually return `-ETIMEDOUT` and limp forward, masking the deadlock. Keeping
   `INFINITE` is the right choice for a stress test — it surfaces the real bug
   instead of hiding it.
2. **Multi-threaded with random early breaks** maximizes the chance of tearing
   down a stream exactly while a block is mid-flight on the server. With `-t 1`
   and a fast emulator, the dequeue almost always completes before destroy, which
   is why single-threaded runs don't reproduce it.

### Optional test-side improvements (quality-of-life, not correctness)

- Add a `-T <seconds>` global watchdog or an alarm that `abort()`s after N
  seconds of no progress, so a future regression produces a stack trace instead
  of a silent hang. (`quit_all` already exists; could be wired to `alarm()`.)
- Consider an explicit mode that runs with a *finite* timeout as a separate
  configuration, to differentiate "deadlock" from "slow".

These are enhancements; the test as written is a valid and valuable reproducer.

---

## Files changed in this fix

| File | Function | Change | Status |
|------|----------|--------|--------|
| `stream.c` | `iio_stream_destroy` | Add `iio_stream_cancel(stream)` at the top | ✅ applied this session |
| `iiod/responder.c` | `handle_free_block` | `iio_buffer_stream_cancel` before `cancel_sync` | ✅ already in working tree |

## Suggested validation

1. Rebuild client lib + iiod, deploy iiod to the board.
2. `iio_stresstest -u ip:<board> -t <numcores*4> <device>` — should run
   indefinitely without hanging; histogram should keep updating.
3. Stress with `-t 1` too (sanity — was already passing).
4. Re-attach gdb during a long run and confirm no threads are parked in
   `iio_block_dequeue` / `iiod_io_wait_for_response` / `local_dequeue_mmap_block`.
```
