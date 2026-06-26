# How iiod works — and why v0 never deadlocked but v1 can

A deep but friendly tour of the IIO daemon (iiod), the two wire protocols it
speaks (v0 "ASCII" and v1 "binary"), and exactly why the teardown deadlock we
chased only exists in v1.

---

## 1. The 10,000-foot view

iiod is a **server** that lets a remote machine use IIO devices (ADCs, DACs,
sensors) attached to a board, as if they were local.

```
   ┌─────────────────────────┐                 ┌──────────────────────────┐
   │   Client machine        │                 │   Board (e.g. Pluto)     │
   │                         │   TCP/USB/UART  │                          │
   │  your app + libiio  ────┼────────────────►│  iiod ──► /dev/iio:device│
   │                         │◄────────────────┤        (kernel driver)   │
   └─────────────────────────┘                 └──────────────────────────┘
```

The client links **libiio**. libiio has several *backends*:

- `local`  — talk to `/sys/bus/iio` and `/dev/iio:deviceN` directly (no network)
- `network`— talk to a remote iiod over TCP
- `usb`, `serial` — same idea over other transports

On the **board**, iiod itself uses the **local** backend to reach the hardware.
So a remote capture is really:

```
app → libiio(network backend) → TCP → iiod → libiio(local backend) → kernel
```

iiod is just a translator: it receives protocol commands on a socket, performs
the matching local IIO operation, and ships results back.

---

## 2. Two protocols: v0 (ASCII) and v1 (binary)

iiod speaks two languages over the same socket. Which one is used is decided at
**connection time** by a handshake.

### The handshake

When libiio's network backend connects, it tries to upgrade to binary
(`iiod_client_enable_binary`, iiod-client.c:987):

```
client → "BINARY\r\n"
         ├─ server understands it  → reply 0  → switch to v1 binary protocol
         └─ server too old/!support → error    → stay on v0 ASCII protocol
```

```c
ret = iiod_client_exec_command(client, "BINARY\r\n");
if (ret != 0)
    return 0;                 /* stay ASCII (v0) */
client->responder = iiod_responder_create(...);   /* go binary (v1) */
```

So **v0 is the legacy text protocol**; **v1 is the modern binary protocol** built
on top of a component called the *responder*. From here on, almost everything
behaves differently.

### v0 — ASCII / text protocol (the old way)

v0 is line-oriented and human-readable. The client literally sends strings:

```
OPEN iio:device4 256 <mask>\r\n      ← open a capture, 256 samples, channel mask
READBUF iio:device4 1024\r\n         ← please send 1024 bytes of samples
CLOSE iio:device4\r\n                ← done
```

Key properties of v0:

- **One TCP connection** for everything.
- **Strictly synchronous, request→response, one at a time.** The client sends a
  command and reads the answer before sending the next. There is a single
  in-flight operation on the wire.
- **No "blocks", no streaming pipeline.** You ask for a chunk of samples
  (`READBUF`), you get a chunk back. Buffering is simple and server-driven.
- In libiio's modern API, the block/stream functions are *not even supported* on
  v0 — `iiod_client_create_block` returns `-ENOSYS` (iiod-client.c:1249, the
  "Only supported on the legacy interface" / binary split). The new streaming
  API simply fails fast on a v0 server.

```
        v0 (ASCII) — single connection, ping-pong
   client                                   iiod
     │ ── "OPEN ...\r\n" ───────────────────►│
     │ ◄──────────────── "0\r\n" ────────────│
     │ ── "READBUF 1024\r\n" ───────────────►│
     │ ◄──────── 1024 bytes of samples ──────│
     │ ── "CLOSE\r\n" ──────────────────────►│
     │ ◄──────────────── "0\r\n" ────────────│
```

Because it is one connection and one outstanding request, **there is no way for
two operations to race each other.** Hold that thought — it is the whole reason
v0 never deadlocks.

### v1 — binary protocol (the modern way)

v1 was built for **high-throughput, low-latency, asynchronous, pipelined**
streaming. It introduces several concepts v0 never had:

- **Commands** are fixed-size binary structs (`struct iiod_command`: an opcode, a
  device index, a code field, a client_id) instead of text lines.
- **The responder** — a little engine on each side with a dedicated **reader
  thread** and **writer task** that multiplexes many logical I/O streams over one
  socket. Each logical stream has a `client_id`, so many requests can be
  *in flight at once* and responses are matched back by id.
- **Buffers and blocks** — instead of "give me 1024 bytes", the client creates a
  *buffer*, allocates several *blocks* of memory, and cycles them:
  enqueue a block (hand it to the server to fill), dequeue it (get it back full),
  reuse it. This is a pipeline that keeps the DMA busy.

```
        v1 (binary) — responder multiplexes, async, pipelined
   client                                   iiod
     │ ── CREATE_BLOCK   (id=1) ────────────►│
     │ ── TRANSFER_BLOCK (id=2) ────────────►│   many commands in flight,
     │ ── TRANSFER_BLOCK (id=3) ────────────►│   answered out of order,
     │ ◄── RESPONSE id=2 + sample data ──────│   matched by client_id
     │ ◄── RESPONSE id=3 + sample data ──────│
```

This is dramatically faster and is what real applications use today. But the
extra machinery — multiple in-flight operations, worker threads, and (crucially)
**a second TCP connection** — is also what creates room for deadlocks.

---

## 3. The v1 two-connection design (the important part)

This is the single most surprising fact, and the root of the whole saga.

When a v1 client opens a **buffer**, libiio's network backend does **not** reuse
the main connection. It opens a **brand-new TCP socket** dedicated to that buffer
(`network_open_buffer` → `network_setup_iiod_client`, network.c).

So one client ends up with **two connections** to iiod:

```
                         ┌────────────────────── Board: iiod ──────────────────────┐
   Client (one app)      │                                                          │
                         │   Connection A handler thread   Connection B handler     │
   ┌───────────────┐     │   ┌────────────────────┐        ┌────────────────────┐  │
   │ main client   │═════╪══►│ reader-thd (A)      │        │ reader-thd (B)      │  │
   │  (Connection A)│     │   │ attrs, trigger,     │        │ CREATE_BLOCK,       │  │
   └───────────────┘     │   │ OPEN/CLOSE_BUFFER,  │        │ TRANSFER_BLOCK,     │  │
                         │   │ FREE_BLOCK          │        │ DEQUEUE             │  │
   ┌───────────────┐     │   └─────────┬──────────┘        └─────────┬──────────┘  │
   │ buffer client │═════╪══════════════╪═══════════════════════════╪════════════►│
   │ (Connection B)│     │              │                            │              │
   └───────────────┘     │              ▼                            ▼              │
                         │        ┌───────────────────────────────────────┐        │
                         │        │  ONE global bufferlist, keyed by       │        │
                         │        │  (device, buffer index) — NOT by conn  │        │
                         │        │     → the SAME buffer_entry object     │        │
                         │        └───────────────────────────────────────┘        │
                         └──────────────────────────────────────────────────────────┘
```

Why two connections? So bulk sample streaming (Connection B) isn't serialized
behind control-plane chatter (Connection A). Great for throughput.

The catch:

- iiod gives **each connection its own reader thread**.
- But a buffer lives in **one global list**, keyed only by `(device, index)`.
- So **two threads, serving two connections of the same client, both operate on
  the same `buffer_entry`** — guarded by one mutex `entry->lock`.

| Command | Connection | Server thread |
|---|---|---|
| `CREATE_BLOCK`, `TRANSFER_BLOCK`, dequeue | B (buffer socket) | reader-thd B |
| `FREE_BLOCK`, `CLOSE_BUFFER`, attrs | A (main socket) | reader-thd A |

Two threads + one shared object + one lock = the necessary ingredients for a
deadlock. v0 has exactly one connection and one in-flight request, so it has
none of these ingredients.

---

## 4. The v1 streaming pipeline (what a capture actually looks like)

```
  iio_buffer_create_stream(buf, 4 blocks, 256 samples, mask)
        │
        ├─ opens Connection B  (the second socket)
        ├─ OPEN_BUFFER  on Connection A
        └─ CREATE_BLOCK ×4 on Connection B   (allocate 4 reusable buffers)

  loop:
     iio_stream_get_next_block(stream)
        ├─ ENQUEUE block N   (hand it to the server to fill via DMA)
        └─ DEQUEUE block N-? (wait until a filled block comes back)

  iio_stream_destroy(stream)
        ├─ (in flight blocks still enqueued!)
        ├─ FREE_BLOCK  ×4   on Connection A
        ├─ CLOSE_BUFFER     on Connection A
        └─ close Connection B
```

On the server, blocks are pumped by worker threads (`buffer-enqueue`,
`buffer-dequeue`) that call into the local backend, which `poll()`s the kernel
DMA fd and an internal `cancel_fd`.

---

## 5. Why v0 "just works" and v1 can hang — side by side

The deadlock we found:

```
v1 teardown race (the bug)

  Connection B thread                  Connection A thread
  (handle_transfer_block)              (handle_free_block)
        │                                     │
   lock entry->lock                           │
        │                                     │
   blocking socket read  ◄── client cancelled │
   (waiting for bytes that                     │
    will never arrive)                         │
        │                              wants entry->lock
        │                                     │  (BLOCKED — B holds it)
        ▼                                     ▼
   ===========  DEADLOCK: B waits on network, A waits on B  ===========
```

Now place v0 next to it:

| | v0 (ASCII) | v1 (binary) |
|---|---|---|
| Connections per client | **1** | **2** (main + per-buffer) |
| In-flight operations | **1** (strict ping-pong) | **many** (responder multiplexes) |
| Server threads touching a buffer | 1 | **2** (one per connection) |
| Shared `buffer_entry` + lock | n/a | **yes** |
| Streaming/blocks | not supported (`-ENOSYS`) | core feature |
| Lock held across a blocking read | doesn't arise | **yes — the bug** |
| Can teardown race a transfer? | **No** | **Yes** |

The reasons v0 is immune, distilled:

1. **One connection, one outstanding request.** There is never a "second thread"
   doing a conflicting operation on the same buffer. The thing that deadlocks in
   v1 — two handlers fighting over `entry->lock` — simply cannot occur.
2. **No block pipeline.** v0 does `READBUF` (synchronous "send me N bytes"). There
   is no enqueue/dequeue/free lifecycle to get caught mid-flight during teardown.
3. **The new stream API isn't even wired to v0.** `iio_buffer_create_stream` on a
   v0 server fails immediately with `-ENOSYS`, so none of the fragile teardown
   paths are ever entered. (In the stresstest, v0 = instant fail = no deadlock.)

In short: **v0 is slow but bulletproof because it can only do one thing at a
time. v1 is fast because it does many things at once — and "many things at once,
sharing one lock" is exactly what makes concurrency bugs possible.** The
deadlock is the price of the pipeline.

---

## 6. Where the moving parts live (code map)

| Concept | File | Notes |
|---|---|---|
| Protocol handshake (BINARY) | `iiod-client.c:987` | chooses v0 vs v1 |
| v0 ASCII commands (OPEN/READBUF/CLOSE) | `iiod-client.c:1257+` | legacy path |
| v1 responder (reader thread, writer task, multiplexing) | `iiod-responder.c` | the engine |
| v1 command handlers on the server | `iiod/responder.c` | `handle_*` functions |
| Per-buffer second connection (client) | `network.c` `network_open_buffer` | opens socket B |
| Global buffer registry on server | `iiod/responder.c` `bufferlist` | keyed by (dev, idx) |
| Block lifecycle (client) | `block.c`, `stream.c` | enqueue/dequeue/destroy |
| Local backend DMA + cancel_fd | `local.c`, `local-mmap.c` | `buffer_check_ready` poll |
| Worker tasks | `task.c` | enqueue/dequeue worker threads |

---

## 7. Practical takeaways

- **v0 vs v1 is chosen automatically** by the `BINARY\r\n` handshake; old servers
  or tinyiiod stay on v0.
- The v1 **two-connection + shared-buffer** design is the source of the teardown
  race. It is real, but it needs an *aggressive* pattern to hit: tear down a
  stream mid-transfer, with an infinite timeout, repeatedly, multi-threaded —
  i.e. the stress test. Normal apps (open → stream → close once, finite timeout)
  rarely if ever trigger it.
- A **finite context timeout** turns the worst case from "permanent hang" into
  "brief stall + error", which is why production clients seldom see the hang.
- v0 is the right mental model for "what iiod fundamentally does"; v1 is that same
  job re-implemented for speed, with the concurrency cost that implies.
