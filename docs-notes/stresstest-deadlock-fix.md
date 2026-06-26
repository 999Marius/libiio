# iio_stresstest — what it does and the v1 deadlock fix

## What iio_stresstest does

`iio_stresstest` is a multi-threaded stress test for libiio. It hammers context
creation, buffer streaming, and teardown in a tight loop to expose races and
resource leaks.

### Thread model

- Default thread count: `numCores × 4` (overridable with `-t N`).
- Each thread is independent: its own `iio_context`, its own TCP connection to
  the server, its own buffer stream.

### Inner loop (per thread)

```
while running:
    ctx = iio_create_context(uri)          # new TCP connection
    iio_context_set_timeout(ctx, INFINITE) # no timeout on any blocking call
    dev = get_device(ctx, name)
    mask = channels_mask(dev)

    while running:
        stream = iio_buffer_create_stream(buf, 4 blocks, 256 samples, mask)
        # ↑ for v1: opens a SECOND TCP connection for the buffer

        while running:
            block = iio_stream_get_next_block(stream)  # may block on server
            if error:
                threads_running = 0
                break
            if rand() % N == 0:
                break    # randomly break out of block loop

        iio_stream_destroy(stream)   # <-- BUG: can deadlock here
        if rand() % N == 0:
            break        # randomly break out of stream loop

    iio_context_destroy(ctx)
```

### What it measures

At the end of each outer cycle it prints:
- contexts created/s, buffers opened/s, block refills/s
- a histogram of inter-context timing (0 µs up to > 1 s buckets)

### Why single-thread works

With v1 + `-t 1` the emulator delivers data quickly. `iio_stream_get_next_block`
always completes before `iio_stream_destroy` is reached, so the two deadlocks
below are never triggered.

With v0 (text protocol), `iio_buffer_create_stream` fails immediately with
`-ENOSYS` (blocks are unsupported), so no teardown paths are exercised at all.

---

## The two deadlocks

### Bug 1 — `buffer.c` : `iio_buffer_stream_cancel` (wrong order)

**Location:** `buffer.c:55`

When a stream is torn down, `iio_buffer_stream_cancel` is called. Its job is to
stop the background worker task that drives `iio_block_io` → `ops->readbuf`.

Current code (broken):

```c
void iio_buffer_stream_cancel(struct iio_buffer_stream *buf_stream)
{
    const struct iio_backend_ops *ops = buf_stream->buf->dev->ctx->ops;

    iio_task_stop(buf_stream->worker);   // ← blocks if worker is inside readbuf
    if (ops->cancel_buffer)
        ops->cancel_buffer(buf_stream->pdata);  // ← never reached
    iio_task_flush(buf_stream->worker);
}
```

`iio_task_stop` waits (with no timeout) for the worker thread to become idle. But
the worker is blocked inside `ops->readbuf` waiting for the device to produce
data. Nothing will unblock `readbuf` until `ops->cancel_buffer` is called — but
that call comes *after* `iio_task_stop`, so it is never reached. Deadlock.

This affects both sides:
- **Client** via `iio_buffer_close` → `iio_buffer_stream_cancel`
- **Server** via `free_buffer_entry` in `iiod/responder.c` → `iio_buffer_stream_cancel`

**Fix:** call `ops->cancel_buffer` *before* `iio_task_stop`:

```c
void iio_buffer_stream_cancel(struct iio_buffer_stream *buf_stream)
{
    const struct iio_backend_ops *ops = buf_stream->buf->dev->ctx->ops;

    if (ops->cancel_buffer)
        ops->cancel_buffer(buf_stream->pdata);  // unblock readbuf first
    iio_task_stop(buf_stream->worker);           // worker can now exit cleanly
    iio_task_flush(buf_stream->worker);
}
```

---

### Bug 2 — `stream.c` : `iio_stream_destroy` (no cancel before dequeue)

**Location:** `stream.c:85`

When the inner block loop breaks early (due to random break or `threads_running`
going false), `iio_stream_destroy` is called with blocks still in flight.

Current code (broken):

```c
void iio_stream_destroy(struct iio_stream *stream)
{
    size_t i;

    for (i = 0; i < stream->nb_blocks; i++) {
        if (stream->blocks[i]) {
            iio_block_dequeue(stream->blocks[i], false);  // ← blocks forever
            iio_block_destroy(stream->blocks[i]);
        }
    }

    free(stream->blocks);
    iio_buffer_close(stream->buf_stream);  // ← cancel only happens here, too late
    free(stream);
}
```

For v1 network blocks, `iio_block_dequeue` calls `iiod_client_dequeue_block` →
`iiod_io_wait_for_response`, which waits on a condition variable for the server
to send a TRANSFER_BLOCK response. The context was set to `IIO_TIMEOUT_INFINITE`,
so this wait has no timeout. `iio_buffer_close` — which calls `cancel_buffer` and
would unblock the wait — is at the *end* of the function and is never reached.

`iio_stream_cancel` already exists and does exactly what's needed, but it is
never called from `iio_stream_destroy`.

**Fix:** call `iio_stream_cancel` at the top of `iio_stream_destroy`:

```c
void iio_stream_destroy(struct iio_stream *stream)
{
    size_t i;

    iio_stream_cancel(stream);  // unblock any in-flight block ops first

    for (i = 0; i < stream->nb_blocks; i++) {
        if (stream->blocks[i]) {
            iio_block_dequeue(stream->blocks[i], false);  // returns immediately with error
            iio_block_destroy(stream->blocks[i]);
        }
    }

    free(stream->blocks);
    iio_buffer_close(stream->buf_stream);
    free(stream);
}
```

`iio_buffer_close` will call `iio_buffer_stream_cancel` a second time. This is
safe: `network_cancel` guards against double-calls via `io_ctx->cancelled`, and
`iio_task_stop` on an already-idle task signals and returns immediately.

---

## Files changed

| File | Function | Change |
|------|----------|--------|
| `buffer.c` | `iio_buffer_stream_cancel` | Move `ops->cancel_buffer` before `iio_task_stop` |
| `stream.c` | `iio_stream_destroy` | Add `iio_stream_cancel(stream)` at the top |
