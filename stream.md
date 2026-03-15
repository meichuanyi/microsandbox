# Streaming Architecture: SDK to Guest Agentd

## High-Level Overview

```
┌─────────────────────────────────────────────────────┐
│                    HOST PROCESS                      │
│                                                      │
│  ┌──────────┐    ┌─────────────┐    ┌────────────┐  │
│  │ User SDK │───▶│ AgentBridge │───▶│  FdWriter   │  │
│  │ (Sandbox)│◀───│  (dispatch) │◀───│  FdReader   │  │
│  └──────────┘    └─────────────┘    └─────┬──────┘  │
│                                           │         │
│                                    Unix Socket Pair  │
│                                    (AF_UNIX STREAM)  │
│                                           │         │
├───────────────────────────────────────────┼─────────┤
│                 SUPERVISOR (libkrun VM)    │         │
│                                           │         │
│                                    virtio-console    │
│                                           │         │
├───────────────────────────────────────────┼─────────┤
│                    GUEST VM               │         │
│                                           │         │
│  ┌────────────┐   ┌───────────┐    ┌─────┴──────┐  │
│  │ ExecSession│──▶│   Agent   │───▶│Serial Port │  │
│  │ (PTY/Pipe) │   │  main loop│◀───│(/dev/vportN)│  │
│  └────────────┘   └───────────┘    └────────────┘  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## Layer 1: Transport — The Physical Pipe

Everything flows through a single **Unix socket pair** (`AF_UNIX, SOCK_STREAM`), created in `spawn.rs:172-178`:

```rust
libc::socketpair(libc::AF_UNIX, libc::SOCK_STREAM, 0, fds.as_mut_ptr())
```

- **Host side**: keeps `host_fd`, wraps it in `AgentBridge`
- **Guest side**: `guest_fd` is inherited by the supervisor child process (CLOEXEC cleared), which passes it to libkrun as a virtio-console device
- **Inside the guest VM**: agentd discovers it as `/dev/vportNpM` by scanning sysfs for a port named `"agent"`

This is a **single bidirectional byte stream** — not HTTP, not WebSocket. Just raw bytes over a Unix socket that gets virtualized into a virtio-console serial port inside the VM.

## Layer 2: Framing — Length-Prefixed CBOR

Raw bytes on the wire use a simple frame format (`codec.rs`):

```
┌──────────────────────────────────────────┐
│  len (4 bytes, big-endian u32)           │
├──────────────────────────────────────────┤
│  CBOR payload (len bytes)                │
│  ┌────────────────────────────────────┐  │
│  │ Message {                         │  │
│  │   v: u8,          // version (1)  │  │
│  │   t: MessageType, // "core.exec…" │  │
│  │   id: u32,        // correlation  │  │
│  │   p: Vec<u8>,     // inner CBOR   │  │
│  │ }                                 │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

Key constraints:
- **Max frame size**: 4 MiB (`MAX_FRAME_SIZE = 4 * 1024 * 1024`)
- **Serialization**: [CBOR](https://cbor.io) via the `ciborium` crate — binary, compact, schema-less
- **Double CBOR**: The outer `Message` is CBOR. The inner `p` field is *also* CBOR-encoded payload bytes. This allows the envelope to be parsed without knowing the payload type.

### Encoding path (sender):

```
Payload struct (e.g. ExecStdout { data })
    │
    ▼  ciborium::into_writer → inner CBOR bytes (msg.p)
Message { v: 1, t: "core.exec.stdout", id: 42, p: <inner> }
    │
    ▼  ciborium::into_writer → outer CBOR bytes
Frame: [u32 BE length][outer CBOR bytes]
    │
    ▼  write to socket/serial
```

### Decoding path (receiver):

```
Raw bytes from socket/serial
    │
    ▼  read 4 bytes → len
    ▼  read len bytes → outer CBOR
    ▼  ciborium::from_reader → Message { v, t, id, p }
    │
    ▼  msg.payload::<ExecStdout>() → ciborium::from_reader(p) → ExecStdout { data }
```

## Layer 3: Message Protocol — Types and Correlation IDs

### Message Types (`message.rs`)

```
                    ┌─────────────────┐
                    │   MessageType   │
                    └────────┬────────┘
          ┌─────────────────┼─────────────────┐
          │                 │                  │
    ┌─────┴─────┐    ┌─────┴─────┐    ┌──────┴──────┐
    │   Core    │    │   Exec    │    │     Fs      │
    ├───────────┤    ├───────────┤    ├─────────────┤
    │ Ready     │    │ Request   │    │ FsRequest   │
    │ Shutdown  │    │ Started   │    │ FsResponse  │
    └───────────┘    │ Stdin     │    │ FsData      │
                     │ Stdout    │    └─────────────┘
                     │ Stderr    │
                     │ Exited    │
                     │ Resize    │
                     │ Signal    │
                     └───────────┘
```

### Correlation ID (`msg.id`)

Every operation gets a unique 32-bit ID. This is the **multiplexing key** — it allows many concurrent exec sessions and file transfers over the single socket.

- **ID 0**: Reserved for `core.ready`
- **IDs 1+**: Allocated sequentially by `AgentBridge::next_id()` (atomic counter, skips 0 on wraparound)
- The same ID is used for the request and ALL associated response/streaming messages

```
Host                                Guest
  │                                   │
  │  ExecRequest     id=5             │
  │──────────────────────────────────▶│
  │                                   │
  │  ExecStarted     id=5             │
  │◀──────────────────────────────────│
  │  ExecStdout      id=5             │
  │◀──────────────────────────────────│  ← chunk 1
  │  ExecStdout      id=5             │
  │◀──────────────────────────────────│  ← chunk 2
  │  ExecStderr      id=5             │
  │◀──────────────────────────────────│  ← error output
  │  ExecStdout      id=5             │
  │◀──────────────────────────────────│  ← chunk 3
  │  ExecExited      id=5             │
  │◀──────────────────────────────────│  ← TERMINAL
  │                                   │
```

**Terminal messages** (`ExecExited`, `FsResponse`) trigger automatic cleanup — the subscription is removed from the dispatch table.

## Layer 4: Host Side — AgentBridge Dispatch

`AgentBridge` (`bridge.rs`) is the host-side message router.

### Structure

```
AgentBridge
├── writer: Arc<Mutex<FdWriter>>     ← shared async writer to socket
├── next_id: AtomicU32               ← correlation ID counter
├── pending: Arc<Mutex<HashMap<      ← dispatch table
│     u32,                              correlation ID
│     UnboundedSender<Message>          per-session channel
│   >>>
└── reader_handle: JoinHandle<()>    ← background reader task
```

### Background Reader Loop

A tokio task runs `reader_loop()` forever, doing:

```
loop {
    msg = codec::read_message(&mut reader)   ← blocks until frame available
    dispatch_id = if msg.t == Ready { 0 } else { msg.id }
    is_terminal = msg.t in {ExecExited, FsResponse}

    pending.lock()
    if let Some(tx) = pending[dispatch_id]:
        tx.send(msg)
        if is_terminal:
            pending.remove(dispatch_id)      ← auto-cleanup
}
```

### Subscription Model

- **`request()`**: Subscribe → send → receive ONE message → done. Used for simple request/response (stat, mkdir, etc.)
- **`subscribe()`**: Register channel BEFORE sending. Returns `UnboundedReceiver<Message>`. Used for streaming (exec, file read/write). The caller keeps calling `rx.recv()` until it gets the terminal message.

Critical ordering: `subscribe()` is called **before** `send()` to avoid a race condition where the response arrives before the handler is registered.

### FdReader / FdWriter (`stream.rs`)

The host-side FD is duplicated (`dup()`) into two independent halves:

```
host_fd ──dup──▶ read_owned  → AsyncFd → FdReader (impl AsyncRead)
         └────▶ write_owned → AsyncFd → FdWriter (impl AsyncWrite)
```

Both set to `O_NONBLOCK`. tokio's `AsyncFd` provides epoll-based readiness notification. The `FdReader` implements `poll_read` which loops on `poll_read_ready` → `nix::unistd::read()`, handling EAGAIN by clearing readiness and re-polling. Same pattern for `FdWriter`.

## Layer 5: SDK Surface — Sandbox.exec_stream()

### Exec Streaming Flow

When the user calls `sandbox.exec_stream("ls", ["-la"])`:

```
                         SDK (mod.rs)                      AgentBridge
                              │                                │
 1. Build ExecRequest         │                                │
    {cmd, args, env, cwd,     │                                │
     tty, rows, cols, rlimits}│                                │
                              │                                │
 2. id = bridge.next_id()     │                                │
                              │                                │
 3. rx = bridge.subscribe(id) │───subscribe(id)───────────────▶│
                              │                                │
 4. bridge.send(ExecRequest)  │───send(msg)───────────────────▶│──write──▶ socket
                              │                                │
 5. Spawn event_mapper_task   │                                │
    that reads from rx and    │                                │
    converts Messages to      │                                │
    ExecEvents                │                                │
                              │                                │
 6. Return ExecHandle {       │                                │
      events: event_rx,       │                                │
      stdin: Option<ExecSink>,│                                │
      bridge: Arc<bridge>,    │                                │
    }                         │                                │
```

The **event mapper task** (spawned as a tokio task at `mod.rs:666`) translates raw protocol messages into user-friendly events:

```
ExecStarted { pid }   →  ExecEvent::Started { pid }
ExecStdout { data }   →  ExecEvent::Stdout(Bytes::from(data))
ExecStderr { data }   →  ExecEvent::Stderr(Bytes::from(data))
ExecExited { code }   →  ExecEvent::Exited { code }
```

The user then consumes events:

```rust
let mut handle = sandbox.exec_stream("ls", ["-la"]).await?;
while let Some(event) = handle.recv().await {
    match event {
        ExecEvent::Stdout(data) => print!("{}", String::from_utf8_lossy(&data)),
        ExecEvent::Stderr(data) => eprint!("{}", String::from_utf8_lossy(&data)),
        ExecEvent::Exited { code } => break,
        _ => {}
    }
}
```

Or use convenience methods:
- `handle.collect()` — buffers all stdout/stderr until exit, returns `ExecOutput`
- `handle.wait()` — discards output, just returns `ExitStatus`

## Layer 6: Guest Side — Agentd Main Loop

`agentd` (`agent.rs`) runs a single-threaded tokio runtime inside the guest VM.

### Boot Sequence

```
1. Discover serial port: scan /sys/class/virtio-ports/*/name for "agent"
2. Open /dev/vportNpM with O_RDWR, set O_NONBLOCK
3. Wrap in AsyncFd for tokio
4. Send core.ready { boot_time_ns, init_time_ns, ready_time_ns }
5. Enter main select! loop
```

### Main Loop — Three Event Sources

```rust
loop {
    tokio::select! {
        // SOURCE 1: Bytes from serial port (host → guest)
        result = async_read_ready(&async_port) => {
            read_from_fd() into serial_in_buf
            while try_decode_from_buf(&serial_in_buf) yields Message:
                handle_message(msg)  // dispatch by type
            flush serial_out_buf to serial port
        }

        // SOURCE 2: Output from child processes/fs tasks
        Some((id, output)) = session_rx.recv() => {
            match output:
                Stdout(data) → encode ExecStdout → serial_out_buf
                Stderr(data) → encode ExecStderr → serial_out_buf
                Exited(code) → encode ExecExited → serial_out_buf; remove session
                Raw(bytes)   → append directly to serial_out_buf
            flush serial_out_buf to serial port
        }

        // SOURCE 3: Heartbeat (every 5 seconds)
        _ = heartbeat_timer.tick() => {
            write heartbeat file
        }
    }
}
```

### Buffer Management

```
serial_in_buf:  accumulates raw bytes from serial reads (max: MAX_FRAME_SIZE + 4)
                ← read in 64 KiB chunks
                → try_decode_from_buf drains complete frames

serial_out_buf: accumulates encoded frames to write
                ← encode_to_buf appends frames
                → flush_write_buf drains to serial port (handles short writes)
```

The output buffer is a **coalescing buffer**: multiple messages can be accumulated before a single `flush_write_buf()` call, which writes everything in one or more `write()` syscalls. This is efficient because it batches multiple small messages into fewer syscalls.

## Layer 7: Process Spawning and Output Capture (Guest Side)

### Two Modes: PTY vs Pipe

```
                    ExecRequest.tty?
                     ┌────┴────┐
                   true       false
                     │          │
              ┌──────┴──────┐  ┌┴───────────────┐
              │  spawn_pty  │  │  spawn_pipe    │
              │             │  │                │
              │ openpty()   │  │ Command::new() │
              │ fork()      │  │ .stdin(piped)  │
              │ setsid()    │  │ .stdout(piped) │
              │ TIOCSCTTY   │  │ .stderr(piped) │
              │ dup2 → 0,1,2│  │ .spawn()       │
              │ execvp()    │  │                │
              └──────┬──────┘  └┬───────────────┘
                     │          │
              pty_reader_task  pipe_reader_task
```

### PTY Mode — spawn_pty()

Uses raw `fork()` + `execvp()` (no tokio Command — because controlling terminal setup requires `setsid()` + `TIOCSCTTY` which can't be done with `pre_exec`):

```
Parent (agentd)                     Child
      │                               │
  openpty() → master_fd, slave_fd     │
  TIOCSWINSZ(master, rows, cols)      │
      │                               │
  fork() ─────────────────────────────┤
      │                               │
      │                           setsid()
      │                           ioctl(TIOCSCTTY)
      │                           dup2(slave→0,1,2)
      │                           close(slave)
      │                           setenv(...)
      │                           chdir(cwd)
      │                           setrlimit(...)
      │                           execvp(cmd, argv)
      │                               │
  drop(slave)                     ┌───┴───┐
  dup(master) → reader_fd         │Process│
      │                           │Running│
  spawn pty_reader_task(reader_fd)└───────┘
      │
  return ExecSession { pid, pty_master }
```

**pty_reader_task**: Background tokio task that polls the PTY master for output:

```rust
loop {
    guard = async_fd.readable().await      // wait for readiness
    n = libc::read(fd, buf, 4096)          // read up to 4096 bytes
    if n > 0:
        tx.send((id, SessionOutput::Stdout(buf[..n].to_vec())))
    else if n == 0:
        break  // EOF
    else if errno == EAGAIN:
        guard.clear_ready(); continue      // spurious wakeup
    else:
        break  // EIO = slave closed
}
code = waitpid(pid)
tx.send((id, SessionOutput::Exited(code)))
```

In PTY mode, stdout and stderr are **merged** into a single stream (the PTY master). There's no way to distinguish them — this is how real terminals work.

### Pipe Mode — spawn_pipe()

Uses tokio's `Command` with piped stdio:

```rust
let mut child = Command::new(&req.cmd)
    .args(&req.args)
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .stderr(Stdio::piped())
    .spawn()?;
```

**pipe_reader_task**: Reads stdout and stderr **concurrently** with `tokio::select!`:

```rust
while !stdout_eof || !stderr_eof {
    tokio::select! {
        result = stdout.read(&mut stdout_buf), if !stdout_eof => {
            match result {
                Ok(0) | Err(_) => stdout_eof = true,
                Ok(n) => tx.send((id, Stdout(stdout_buf[..n].to_vec()))),
            }
        }
        result = stderr.read(&mut stderr_buf), if !stderr_eof => {
            match result {
                Ok(0) | Err(_) => stderr_eof = true,
                Ok(n) => tx.send((id, Stderr(stderr_buf[..n].to_vec()))),
            }
        }
    }
}
code = child.wait().await
tx.send((id, Exited(code)))
```

Stdout and stderr are kept **separate** — each arrives as distinct `ExecEvent::Stdout` or `ExecEvent::Stderr` events.

## How Chunks Are Determined

### Exec Output Chunks

**There is no application-level chunking**. Chunks are determined entirely by the kernel:

```
Process writes "Hello, World!\n" to stdout
         │
         ▼
┌─────────────────────────┐
│ Kernel pipe/PTY buffer  │  (default: 64 KiB for pipes, 4 KiB for PTY)
│ "Hello, World!\n"       │
└────────────┬────────────┘
             │
    libc::read(fd, buf, 4096)  ← agentd reads up to 4096 bytes
             │
             ▼
    SessionOutput::Stdout(b"Hello, World!\n")
             │
             ▼
    ExecStdout { data: [72, 101, 108, ...] }  ← becomes one protocol message
```

- **Read buffer**: 4096 bytes (hardcoded in both `pty_reader_task` and `pipe_reader_task`)
- **Actual chunk size**: Whatever the kernel returns from `read()`, which is `min(available_bytes, 4096)`
- **No buffering/batching**: Each `read()` that returns data immediately becomes a protocol message
- **No coalescing**: If a process writes 100 bytes, then 200 bytes quickly, these may arrive as one 300-byte chunk or two separate chunks depending on kernel scheduling

This means:
- A process printing `"Hello\n"` produces one ~6 byte chunk
- A process dumping 10 KiB at once produces 2-3 chunks (4096 + 4096 + remainder)
- Output arrives at the SDK with the same granularity the kernel delivers it

### Filesystem Chunks

**Fixed-size chunking** at `FS_CHUNK_SIZE = 3 * 1024 * 1024` (3 MiB):

```
Guest reading a 7 MiB file:

File on disk: [███████████████████████████████████████████]  7 MiB
                    │                │              │    │
              ┌─────┘          ┌─────┘         ┌───┘    └──┐
              ▼                ▼               ▼           ▼
         FsData(3 MiB)   FsData(3 MiB)   FsData(1 MiB)  FsResponse{ok:true}
              │                │               │           │
              ▼                ▼               ▼           ▼
         Message id=7     Message id=7    Message id=7   Message id=7
```

Why 3 MiB and not 4 MiB? Because the CBOR envelope (Message struct fields, length prefix bytes, CBOR overhead) adds ~50-100 bytes. 3 MiB payload + overhead stays safely under the 4 MiB `MAX_FRAME_SIZE`.

**Host-to-guest writes** chunk at the same boundary:

```rust
// fs.rs:192 - host side
for chunk in data.chunks(FS_CHUNK_SIZE) {
    let fs_data = FsData { data: chunk.to_vec() };
    bridge.send(&msg).await?;
}
// Then send empty FsData as EOF
```

## Complete Exec Streaming Sequence Diagram

```
User Code            SDK/Sandbox         AgentBridge          Socket       Agentd            ExecSession
    │                     │                   │                  │            │                    │
    │ exec_stream("ls")   │                   │                  │            │                    │
    │────────────────────▶│                   │                  │            │                    │
    │                     │ id = next_id()    │                  │            │                    │
    │                     │ rx = subscribe(id)│                  │            │                    │
    │                     │──────────────────▶│                  │            │                    │
    │                     │                   │                  │            │                    │
    │                     │ send(ExecRequest) │                  │            │                    │
    │                     │──────────────────▶│                  │            │                    │
    │                     │                   │  [len][CBOR]     │            │                    │
    │                     │                   │─────────────────▶│            │                    │
    │                     │                   │                  │  serial rd │                    │
    │                     │                   │                  │───────────▶│                    │
    │                     │                   │                  │            │ decode ExecRequest │
    │                     │                   │                  │            │ spawn(id, req, tx) │
    │                     │                   │                  │            │───────────────────▶│
    │                     │                   │                  │            │                    │ fork()/spawn()
    │                     │                   │                  │            │                    │──▶ process
    │                     │                   │                  │            │   ExecStarted{pid} │
    │                     │                   │                  │            │◀───────────────────│
    │                     │                   │                  │  encode    │                    │
    │                     │                   │  [len][CBOR]     │◀───────────│                    │
    │                     │  dispatch(id)     │◀─────────────────│            │                    │
    │                     │◀─ Started{pid} ──│                  │            │                    │
    │ ExecEvent::Started  │                   │                  │            │                    │
    │◀────────────────────│                   │                  │            │                    │
    │                     │                   │                  │            │                    │
    │                     │                   │                  │            │  Stdout(bytes)     │
    │                     │                   │                  │            │◀──reader_task──────│
    │                     │                   │  [len][CBOR]     │            │                    │
    │                     │  dispatch(id)     │◀─────────────────│◀───────────│                    │
    │                     │◀─ Stdout(data) ──│                  │            │                    │
    │ ExecEvent::Stdout   │                   │                  │            │                    │
    │◀────────────────────│                   │                  │            │                    │
    │                     │                   │                  │            │                    │
    │    ... more chunks, interleaved stdout/stderr ...          │            │                    │
    │                     │                   │                  │            │                    │
    │                     │                   │                  │            │  Exited(code)      │
    │                     │                   │                  │            │◀──reader_task──────│
    │                     │                   │  [len][CBOR]     │            │ remove session     │
    │                     │  dispatch(id)     │◀─────────────────│◀───────────│                    │
    │                     │◀─ Exited{code} ──│ remove(id)       │            │                    │
    │ ExecEvent::Exited   │                   │                  │            │                    │
    │◀────────────────────│                   │                  │            │                    │
    │                     │                   │                  │            │                    │
    │ None (stream end)   │                   │                  │            │                    │
    │◀────────────────────│                   │                  │            │                    │
```

## Complete File Read Streaming Sequence Diagram

```
User Code        SDK/SandboxFs      AgentBridge      Socket     Agentd          fs::handle_read_stream
    │                  │                 │              │          │                      │
    │ read_stream(p)   │                 │              │          │                      │
    │─────────────────▶│                 │              │          │                      │
    │                  │ subscribe(id)   │              │          │                      │
    │                  │────────────────▶│              │          │                      │
    │                  │ send(FsRequest) │              │          │                      │
    │                  │────────────────▶│──[frame]────▶│─────────▶│                      │
    │                  │                 │              │          │ tokio::spawn          │
    │                  │                 │              │          │─────────────────────▶│
    │  FsReadStream    │                 │              │          │                      │
    │◀─────────────────│                 │              │          │                      │
    │                  │                 │              │          │    open file          │
    │                  │                 │              │          │    read ≤3MiB         │
    │                  │                 │              │          │                      │
    │                  │                 │              │  FsData  │◀─SessionOutput::Raw──│
    │                  │  dispatch(id)   │◀─[frame]────│◀─────────│                      │
    │                  │◀─ FsData ──────│              │          │                      │
    │  recv() → chunk  │                 │              │          │    read ≤3MiB         │
    │◀─────────────────│                 │              │          │                      │
    │                  │                 │              │  FsData  │◀─SessionOutput::Raw──│
    │                  │  dispatch(id)   │◀─[frame]────│◀─────────│                      │
    │  recv() → chunk  │                 │              │          │                      │
    │◀─────────────────│                 │              │          │                      │
    │                  │                 │              │FsResponse│◀─SessionOutput::Raw──│
    │                  │  dispatch(id)   │◀─[frame]────│◀─────────│  (ok: true)          │
    │                  │◀─ FsResponse ──│ remove(id)   │          │                      │
    │  recv() → None   │                 │              │          │                      │
    │◀─────────────────│                 │              │          │                      │
```

Note that file read is handled in a **background tokio task** on the guest, not inline in the main loop. This is because reading a large file could block the main loop. The task sends pre-encoded frames (`SessionOutput::Raw`) directly through the session channel, so the main loop just appends the bytes to `serial_out_buf` and flushes — no encoding needed.

## Multiplexing — Concurrent Operations

Because everything uses correlation IDs, you can have many operations in flight simultaneously:

```
Time  Socket bytes                          Active sessions
──────────────────────────────────────────────────────────
t0    → ExecRequest    id=1  (ls -la)       {1: exec}
t1    → ExecRequest    id=2  (cat file)     {1: exec, 2: exec}
t2    → FsRequest      id=3  (read /foo)    {1: exec, 2: exec, 3: fs_read}
t3    ← ExecStarted   id=1                  ...
t4    ← ExecStarted   id=2                  ...
t5    ← ExecStdout    id=1  "drwxr..."      ...
t6    ← FsData        id=3  [3 MiB]         ...
t7    ← ExecStdout    id=2  "file content"  ...
t8    ← ExecExited    id=1  code=0          {2: exec, 3: fs_read}
t9    ← FsData        id=3  [1.5 MiB]       ...
t10   ← FsResponse    id=3  ok=true         {2: exec}
t11   ← ExecExited    id=2  code=0          {}
```

Each SDK caller only sees messages for their correlation ID. The AgentBridge dispatch table routes everything transparently.

## Summary: What Determines Chunk Boundaries

| Operation | Chunk Size | Determined By |
|-----------|-----------|---------------|
| Exec stdout/stderr | 1-4096 bytes | Kernel pipe/PTY buffer + `read(buf, 4096)` |
| File read (guest→host) | 1-3 MiB | `FS_CHUNK_SIZE` constant + `read()` return |
| File write (host→guest) | 1-3 MiB | `data.chunks(FS_CHUNK_SIZE)` on host side |
| Stdin (host→guest) | Arbitrary | Whatever the user passes to `ExecSink::write()` |

The key insight: **exec streaming has no artificial buffering** — data flows the instant the kernel makes it available, through the entire pipeline (reader task → mpsc → main loop → serial → socket → bridge reader → event channel → user). This gives the lowest possible latency for interactive use cases.

## Key Source Files

| File | Role |
|------|------|
| `crates/protocol/lib/codec.rs` | Frame encoding/decoding (length-prefixed CBOR) |
| `crates/protocol/lib/message.rs` | Message envelope and MessageType definitions |
| `crates/protocol/lib/exec.rs` | Exec payload structs (ExecRequest, ExecStdout, etc.) |
| `crates/protocol/lib/fs.rs` | Filesystem payload structs (FsRequest, FsData, etc.) |
| `crates/agentd/lib/agent.rs` | Guest-side main select! loop |
| `crates/agentd/lib/session.rs` | PTY/pipe spawning and reader tasks |
| `crates/agentd/lib/fs.rs` | Guest-side filesystem handlers |
| `crates/microsandbox/lib/agent/bridge.rs` | Host-side message dispatch (AgentBridge) |
| `crates/microsandbox/lib/agent/stream.rs` | AsyncFd-based FdReader/FdWriter |
| `crates/microsandbox/lib/sandbox/mod.rs` | SDK exec_stream() and event_mapper_task |
| `crates/microsandbox/lib/sandbox/exec.rs` | ExecHandle, ExecEvent, ExecSink types |
| `crates/microsandbox/lib/sandbox/fs.rs` | SandboxFs, FsReadStream, FsWriteSink types |
| `crates/microsandbox/lib/runtime/spawn.rs` | Unix socket pair creation and supervisor spawn |
