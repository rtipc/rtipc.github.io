# RTIPC: Real-Time Inter-Process Communication

An inter-process communication library with **near-zero overhead**, specifically engineered to fulfill **hard real-time constraints**.

[View C11 Repository](https://github.com/rtipc/rtipc.git) • [View Rust Repository](https://github.com/rtipc/rtipc-rust.git)

## Core Features

* **Zero-Copy and Syscall-Free:** Data transfers instantly with absolutely no memory copying or kernel overhead.
* **Deterministic Behavior:** Data updates do not impact the runtime or execution predictability.
* **Real-Time Message Handling:** Features a **"force push"** operation. If the queue is full, the producer automatically discards the oldest unread message to make room for new data. This ensures the consumer always accesses the absolute freshest state.
* **SMP Optimized:** Messages are strictly cacheline aligned to minimize unnecessary cache coherence traffic across multi core systems.
* **Event Notification:** Optional `eventfd` support for seamless integration with standard Linux event loops like `select`, `poll`, and `epoll`.
* **Multithreading:** Multiple threads can communicate concurrently over separate, isolated channels.

> **Limitation:** Both the message size and the maximum queue length are fixed at creation time to guarantee deterministic performance.

### Formally Verified Algorithm

Because the "force push" operation increases algorithmic complexity, the underlying queue algorithm has been formally verified using **[SPIN/Modex](https://spinroot.com/)** to ensure correctness.

* [Algorithm Verification Repository](https://github.com/rtipc/message-queue.git)

## Why RTIPC?

### The Architectural Problem

Modern software benefits heavily from the Unix philosophy: *"Do one thing and do it well."* Splitting an application into multiple processes improves isolation, fault tolerance, and security. It also provides a clean migration path from legacy C/C++ to Rust, shielding new Rust components from common C/C++ memory bugs like wild pointers or use after free errors.

### Why not standard Linux IPC?

While APIs like sockets, pipes, and System V/POSIX message queues exist, they introduce significant overhead:

* **Sockets, Pipes and Message Queues:** These require copying data from the producer to kernel memory, and then from the kernel to the consumer. (Even though pipes can use `vmsplice` to avoid copying into the kernel buffer, the consumer still must copy the data out).
* **Standard Shared Memory:** This eliminates copying but leaves the complex, error prone burden of process synchronization entirely to the developer.

**RTIPC bridges this gap.** It utilizes shared memory for zero-copy speed, backed by a **Single Producer Single Consumer (SPSC) wait-free algorithm** to handle synchronization safely, automatically, and deterministically.

## Data Format and Serialization

When communicating between processes written in different languages, you need a common data format. While popular solutions like Protocol Buffers, Cap'n Proto, or FlatBuffers exist, **none of them are recommended for this use case.** Those serialization tools solve cross architecture compatibility, which is a problem that simply doesn't exist in local IPC.

Because all your processes run on the same CPU architecture, **standard C structs are the ideal format**. Almost every general purpose language supports them natively:

* **Rust:** Simply decorate your struct with `#[repr(C)]`.
* **Python:** Access fields seamlessly via `ctypes`.

> **Caveat:** While it is technically possible to run a 32 bit process on a 64 bit CPU, attempting to communicate between a 32 bit process and a 64 bit process over shared memory will fail due to differing structure alignments.

### Ensuring Consistency: rtipc-compiler

While full serialization frameworks are overkill, they do solve one major problem: **schema-consistency**.

To maintain a single source of truth across languages without runtime overhead, the **rtipc-compiler** project is currently being developed. While it is currently a **proof of concept**, its goal is to provide an Interface Definition Language (IDL) based on a subset of Rust syntax to automatically generate C compatible structs and classes across different target languages.

* [rtipc-compiler Repository](https://github.com/rtipc/rtipc-compiler.git)

## How It Works (Usage Workflow)

![alt text](https://github.com/mausys/rtipc-rust/blob/main/doc/flow.png)

1. **Define Channels:** The client configures a channel vector detailing the queue length, message size, optional `eventfd`, and custom user metadata.
2. **Allocate Resources:** The library creates anonymous shared memory, initializes `eventfd` instances, and maps the queues.
3. **Bootstrap via Unix Sockets:** The configuration vector is serialized. The file descriptors (shared memory + `eventfd`s) are sent to the target process via a standard Unix domain socket using native file descriptor passing (`SCM_RIGHTS`).
4. **Establish SPSC Loop:** The receiving process deserializes the request, maps the memory, and both processes begin communicating directly via shared memory, bypassing the kernel entirely.

### Alternative Bootstrap Methods

You can also use existing D-Bus infrastructure to handle the initial file descriptor exchange:

* [C Library with systemd D-Bus](https://github.com/rticp/rtipc-dbus.git)
* [Rust Crate with zbus + Tokio](https://github.com/rtipc/rtipc-zbus.git)

## Ecosystem and Implementations

RTIPC is under active development. The ecosystem is designed to be lightweight, portable, and cross language ready.

| Language / Binding | Status | Repository |
| --- | --- | --- |
| **C11** | Active Development (Dependency Free) | [rtipc](https://github.com/mausys/rtipc.git) |
| **Rust** | Active Development (Pure Rust) | [rtipc-rust](https://github.com/mausys/rtipc-rust.git) |
| **Python** | Work In Progress (via Cython) | [pyrtipc](https://github.com/mausys/pyrtipc.git) |
| **C++** | Planned (C Header compatible, RAII wrappers) | *Coming Soon* |
| **Java / C# / Go** | Planned | *Coming Soon* |
