# EMU Backend — File-Based Buffer I/O Implementation Plan

## Background

The emu backend currently supports context creation and attribute read/write from XML.
This plan adds buffer streaming support: reading sample data from a file (RX) and writing
sample data to a file (TX), using only the v1 block ops API.

### Reference: iio-emu

The existing `iio-emu` standalone server (`../iio-emu/iio-emu`) implements a similar
concept via `GenericRXDevice` / `GenericTXDevice`:

- RX: reads from a configured file, rotates remaining data through a temp file for
  wrap-around streaming
- TX: appends to a configured output file

Key differences from our approach:
- iio-emu is a C++ network server; emu.c is a libiio backend (C, in-process)
- iio-emu configures file paths externally; we derive them from a naming convention
- iio-emu uses its own two-phase protocol (`readbuf`/`writebuf`); we use block ops

iio-emu's file-rotation approach for RX is interesting but adds temp-file complexity.
For v1 we return `-ENODATA` at EOF (simpler, explicit, easier to test against).

---

## Investigation: Block ops vs. readbuf/writebuf

**Conclusion: block ops only. `readbuf`/`writebuf` are not needed.**

The framework's dispatch in `block.c:155`:

```c
if (ops->enqueue_block && block->pdata)
    return ops->enqueue_block(block->pdata, bytes_used, cyclic);
/* else: worker thread → readbuf/writebuf */
```

Both conditions hold when emu implements `create_block` (returns non-NULL pdata) and
`enqueue_block`. The worker-thread/readbuf fallback is only taken when these are absent —
it is the remote-transport path (network, USB, serial). The emu backend is local and
in-process; block ops are the correct fit.

The v1 public API (`iio_block_enqueue`, `iio_block_dequeue`, `iio_stream_get_next_block`)
all go through block ops. The v0 compat layer (`iio_buffer_refill`, `iio_buffer_push` in
`compat.c`) also routes through `iio_block_enqueue`/`iio_block_dequeue`, so it works too.

---

## File naming convention

Files are derived automatically — no XML changes, no URI changes required.

Given:
- XML at `/path/to/context.xml`
- Device ID `iio:device4`, buffer index `0`

Files:
```
/path/to/iio:device4_buf0.bin
```

- RX device: opened `"rb"` — must exist before streaming
- TX device: opened `"wb"` — created (or truncated) on `open_buffer`

Direction is determined at `open_buffer` time via `iio_device_is_tx(dev)`.

### Why per-device, not per-context

A single context can have multiple buffer-capable devices streaming simultaneously
(e.g., `cf-ad9361-lpc` RX + `cf-ad9361-dds-core-lpc` TX on PlutoSDR). Each
`emu_open_buffer` call gets its own independent `FILE *`. Using one file per context
would cause:

- Two devices racing on the same `FILE *` (interleaved reads, corrupted data)
- TX `"wb"` truncating the file the RX device is currently reading

Per-device files avoid both problems cleanly.

### Unit testing with this convention

```c
/* 1. write known fixture data */
FILE *f = fopen("/path/to/iio:device4_buf0.bin", "wb");
fwrite(expected_samples, 1, sizeof(expected_samples), f);
fclose(f);

/* 2. stream via emu */
ctx   = iio_create_context(NULL, "emu:/path/to/context.xml");
dev   = iio_context_find_device(ctx, "cf-ad9361-lpc");
buf   = iio_device_get_buffer(dev, 0);
bs    = iio_buffer_open(buf, mask);
block = iio_buffer_stream_create_block(bs, block_size);
iio_buffer_stream_start(bs);
iio_block_enqueue(block, 0, false);
iio_block_dequeue(block, false);

/* 3. verify */
assert(memcmp(iio_block_start(block), expected_samples, block_size) == 0);
```

---

## File to modify

**`emu.c` only.** No new files, no CMake changes.

---

## Struct definitions

Each backend defines its own `iio_buffer_pdata` and `iio_block_pdata` in its own
translation unit (forward-declared in `iio-backend.h`, defined per-backend — see
`local.h:20-37` for the local backend pattern).

```c
struct iio_buffer_pdata {
    const struct iio_device *dev;
    unsigned int idx;
    FILE *file;     /* "rb" for RX, "wb" for TX; NULL if file could not be opened */
};

struct iio_block_pdata {
    struct iio_buffer_pdata *buf;
    void *data;
    size_t size;
};
```

No mutex or condition variable needed: `enqueue_block` and `dequeue_block` are
synchronous file I/O operations. There is no background thread to cancel, so
`cancel_buffer` is a no-op.

---

## Path construction helper

```c
/* Returns a heap-allocated path string; caller must free().
 * Returns NULL on allocation failure.
 * Cross-platform: handles both '/' and '\\' directory separators. */
static char *emu_make_data_path(const char *xml_path,
                                const char *dev_id,
                                unsigned int idx)
```

Steps:
1. `sep = strrchr(xml_path, '/')` — find last separator
2. On Windows also check `strrchr(xml_path, '\\')`, take whichever is later
3. Compute directory length: `dir_len = sep ? (sep - xml_path + 1) : 0`
4. `snprintf` into a `malloc`'d buffer: `{dir}{dev_id}_buf{idx}.bin`
5. Return the buffer

Uses only `strrchr`, `snprintf`, `malloc` — no platform-specific headers.

---

## Function implementations

### `emu_open_buffer`

```c
static struct iio_buffer_pdata *
emu_open_buffer(const struct iio_device *dev, unsigned int idx,
                struct iio_channels_mask *mask)
```

1. `pdata = zalloc(sizeof(*pdata))`; return `iio_ptr(-ENOMEM)` on failure
2. `pdata->dev = dev`, `pdata->idx = idx`
3. `xml_path = iio_context_get_pdata(dev->ctx)->xml_path`
4. `path = emu_make_data_path(xml_path, iio_device_get_id(dev), idx)`
5. `mode = iio_device_is_tx(dev) ? "wb" : "rb"`
6. `pdata->file = fopen(path, mode)` — NULL is **not** an error at open time:
   - TX NULL: permission error (rare); deferred to enqueue
   - RX NULL: file absent; deferred to enqueue with `-ENOENT`
7. `free(path)`, return `pdata`

### `emu_close_buffer`

```c
static void emu_close_buffer(struct iio_buffer_pdata *pdata)
```

1. `if (pdata->file) fclose(pdata->file)`
2. `free(pdata)`

### `emu_enable_buffer`

```c
static int emu_enable_buffer(struct iio_buffer_pdata *pdata,
                             size_t nb_samples, bool enable, bool cyclic)
```

Return `0`. No hardware to enable; no state to track.

### `emu_cancel_buffer`

```c
static void emu_cancel_buffer(struct iio_buffer_pdata *pdata)
```

No-op. File I/O in `enqueue_block` is synchronous and completes atomically from the
framework's perspective; there is nothing to interrupt.

### `emu_create_block`

```c
static struct iio_block_pdata *
emu_create_block(struct iio_buffer_pdata *pdata, size_t size, void **data)
```

1. `block = zalloc(sizeof(*block))`; return `iio_ptr(-ENOMEM)` on failure
2. `block->data = malloc(size)` — on failure: `free(block)`, return `iio_ptr(-ENOMEM)`
3. `block->buf = pdata`, `block->size = size`
4. `*data = block->data`
5. Return `block`

Setting `block->pdata` non-NULL (via this return value stored by the framework in
`block.c:52`) is what causes `iio_block_enqueue` to take the block ops path instead of
the worker-thread/readbuf fallback.

### `emu_free_block`

```c
static void emu_free_block(struct iio_block_pdata *pdata)
```

1. `free(pdata->data)`
2. `free(pdata)`

### `emu_enqueue_block`

```c
static int emu_enqueue_block(struct iio_block_pdata *pdata,
                             size_t bytes_used, bool cyclic)
```

**TX** (`iio_device_is_tx(pdata->buf->dev)`):
1. If `pdata->buf->file == NULL` → return `-EBADF`
2. `n = fwrite(pdata->data, 1, bytes_used, pdata->buf->file)`
3. If `n < bytes_used` → return `-EIO`
4. Return `0`

**RX**:
1. If `pdata->buf->file == NULL` → return `-ENOENT`
2. `n = fread(pdata->data, 1, bytes_used, pdata->buf->file)`
3. If `n < bytes_used` → return `-ENODATA` (EOF)
4. Return `0`

`bytes_used` is guaranteed non-zero by `iio_block_enqueue` (sets it to `block->size`
if caller passed 0 — see `block.c:152`).

### `emu_dequeue_block`

```c
static int emu_dequeue_block(struct iio_block_pdata *pdata, bool nonblock)
```

Return `0`. Data is already in `pdata->data` after `enqueue_block` completes;
dequeue is always immediately ready.

---

## `emu_ops` registration

```c
static const struct iio_backend_ops emu_ops = {
    .create        = emu_create_context,
    .read_attr     = emu_read_attr,
    .write_attr    = emu_write_attr,
    .shutdown      = emu_shutdown,

    .open_buffer   = emu_open_buffer,
    .close_buffer  = emu_close_buffer,
    .enable_buffer = emu_enable_buffer,
    .cancel_buffer = emu_cancel_buffer,

    .create_block  = emu_create_block,
    .free_block    = emu_free_block,
    .enqueue_block = emu_enqueue_block,
    .dequeue_block = emu_dequeue_block,
};
```

Omitted (NULL): `readbuf`, `writebuf`, `get_dmabuf_fd`, `disable_cpu_access`,
`open_ev`, `close_ev`, `read_ev` — not applicable to in-process emulation.

---

## Error propagation

| Situation | Error returned | Propagates to |
|---|---|---|
| RX file absent at enqueue | `-ENOENT` | `iio_block_enqueue()` caller |
| TX file not writable at enqueue | `-EBADF` | `iio_block_enqueue()` caller |
| EOF during RX read | `-ENODATA` | `iio_block_enqueue()` caller |
| Short write during TX | `-EIO` | `iio_block_enqueue()` caller |
| malloc failure in create_block | `-ENOMEM` | `iio_buffer_stream_create_block()` |

---

## Verification

### Build
```bash
cmake --build build_emu --target iio
```

### RX test
```bash
# prepare fixture: 1024 bytes of known data
dd if=/dev/urandom of=temp_emu/iio:device4_buf0.bin bs=1024 count=1

# stream 256 samples (4 bytes/sample → 1024 bytes)
LD_LIBRARY_PATH=build_emu build_emu/utils/iio_rwdev \
    -u "emu:temp_emu/pluto_error.xml" -b 256 -s 256 cf-ad9361-lpc \
    | cmp - temp_emu/iio:device4_buf0.bin && echo "RX OK"
```

### TX test
```bash
dd if=/dev/urandom bs=1024 count=1 \
    | LD_LIBRARY_PATH=build_emu build_emu/utils/iio_rwdev \
    -u "emu:temp_emu/pluto_error.xml" -b 256 -s 256 -w cf-ad9361-dds-core-lpc

ls -la temp_emu/iio:device3_buf0.bin  # must exist and be 1024 bytes
```

### EOF test
Create an RX file smaller than one block and verify the caller receives `-ENODATA`.

### No-file RX test
Remove the RX file and verify `iio_block_enqueue` returns `-ENOENT`.

### Unit test integration
```bash
TESTS_API_URI="emu:temp_emu/pluto_error.xml" \
    ./tests/api/run_all_api_tests
```
