# Cachew — Design Document

Cachew is a small Redis-compatible, in-memory key–value server written in C++23. It is a learning project. It speaks a subset of RESP2 over TCP, so `redis-cli` can talk to it.

## 1. Goals and non-goals

**Goals**
- A working single-node, in-memory KV server that `redis-cli` can use.
- Small, testable layers, each easy to explain.
- Safe behavior under malformed input, slow clients, and disconnects.

**Non-goals (v1)**
- Persistence, replication, clustering, authentication, TLS.
- Pub/sub, transactions.
- Lists, hashes, sets.
- Multithreaded command execution (execution is single-threaded by design; see section 5).
- Full Redis compatibility. The supported subset is documented instead.

## 2. Requirements

### 2.1 Functional

| Command | Behavior |
|---|---|
| `PING [msg]` | Replies `PONG`, or echoes `msg` as a bulk string |
| `ECHO msg` | Returns `msg` |
| `SET key value` | Stores the value, overwrites any existing one, clears any TTL. Replies `OK` |
| `GET key` | Returns the value, or null if missing or expired |
| `DEL key [key ...]` | Returns the number of keys removed |
| `EXISTS key [key ...]` | Returns how many of the given keys exist (duplicates counted) |
| `EXPIRE key seconds` | Sets a TTL. Returns 1 if the key exists, else 0 |
| `TTL key` | Returns the remaining seconds, `-1` if no TTL, `-2` if the key is missing |

- Command names are case-insensitive.
- Keys and values are binary-safe (may contain `\0` and `\r\n`).
- Wrong arity or a non-integer argument returns an error and keeps the connection open.
- Unknown commands return an error.
- Later additions: `SET` options `EX`, `PX`, `NX`, `XX`.

**Networking**
- Default port is `5123` (not Redis's 6379, so it can run beside a real Redis). Configurable; port `0` picks a free one and prints it.
- Binds to `127.0.0.1` by default.
- Serves many clients at once.
- Shuts down cleanly on `SIGINT` and `SIGTERM`.

**Protocol**
- Parses RESP2 requests (arrays of bulk strings).
- Encodes RESP2 replies: simple string, error, integer, bulk string, null, array.
- Handles requests split across several reads.
- Handles several requests in one read (pipelining), with replies in request order.
- Requests must be RESP arrays. Inline (plain-text) commands are not supported; anything else is a protocol error.

**Expiration**
- An expired key behaves as missing for every command.
- Expired keys are physically removed over time.
- `SET` clears a TTL. `DEL` removes the key and its TTL.

### 2.2 Non-functional

**Correctness**
- Commands run one at a time, in order, and each is atomic.
- Replies match request order per connection.
- Compatible with `redis-cli` for the supported subset.

**Robustness**
- No crash, hang, or undefined behavior on any input.
- Hard, configurable limits:
  - Bulk string size: about 1–8 MB.
  - Arguments per command: about 1024.
  - Input buffer per client.
  - Output buffer per client: about 16 MB.
- A slow or non-reading client cannot block others or grow memory without bound.
- A client disconnect, even mid-command, frees its resources and affects no one else.

**Performance (modest targets)**
- Hundreds of concurrent clients on one thread.
- Tens of thousands of operations per second on a laptop, measured with `redis-benchmark`.
- Average O(1) lookup, insert, and delete.
- The expiry sweep is time-bounded per tick.

**Security**
- Localhost-only by default. There is no auth, so the server must not be exposed publicly.

**Maintainability and quality**
- Parser, store, and dispatcher have no socket dependencies.
- Warnings enabled. Resources owned through RAII.
- Unit and end-to-end tests pass. Builds are clean under ASan and UBSan.
- Tests use an injectable clock, so they never sleep for expiry.
- Builds on macOS and Linux.

## 3. Architecture

```text
 socket bytes
     │
 ┌───▼────────┐   ┌────────────┐   ┌───────────┐   ┌──────────┐
 │ Connection │──►│ RESP Parser│──►│ Dispatcher│──►│  Store   │
 │ in/out buf │   └────────────┘   └─────┬─────┘   │ + expiry │
 └───▲────────┘   ┌────────────┐         │         └──────────┘
     └────────────│  Encoder   │◄────────┘
                  └────────────┘
 EventLoop (poll) drives Connections and owns the expiry timer.
```

**Dependency rule:** the `resp`, `store`, and `commands` modules know nothing about sockets. Only `net` touches file descriptors. Everything else can be unit-tested with plain strings.

## 4. Components

### 4.1 Store

- Holds string keys mapped to an entry containing the value and an optional expiry time.
- Backed by `std::unordered_map` with transparent hashing, so lookups by `string_view` do not allocate.
- Operations: set, get, delete, exists, expire, remaining TTL, and a bounded sweep.
- Takes an **injectable clock**, so tests control time.
- Uses `steady_clock`, so wall-clock changes cannot break TTLs.
- A value returned by `get` is a view into the store. Any later mutation invalidates it, so command handlers copy it into the reply immediately.

### 4.2 Expiration

Two mechanisms work together:

1. **Lazy expiry.** Every access checks the key's deadline. An expired key is erased and treated as missing. This guarantees correctness.
2. **Active sweep.** The event loop periodically (about every 100 ms) removes expired keys. This guarantees memory is reclaimed.

Sweep design (Redis-style random sampling):
- Besides the main map, keep a dense list of the keys that currently have a TTL. Each entry in the map records its position in that list, so adding and removing a key is O(1) (swap the last element into the freed slot).
- Each tick, pick a small random sample of keys from the list (for example 20) and erase the ones that have expired.
- If a large share of the sample was expired (for example more than 25%), sample again, up to a fixed per-tick budget, so a tick cannot stall the loop.
- This is probabilistic: an expired key may linger until it is sampled or accessed. Lazy expiry keeps behavior correct in the meantime, and memory use stays bounded because expired keys make up a shrinking share of the list.
- The random source is injectable, so tests can make sampling deterministic.
- `SET` without a TTL, `DEL`, and expiry remove the key from the list. `EXPIRE` and `SET` with a TTL add it.

`TTL` rounds to the nearest second, as Redis does. This is covered by tests.

Sweep pseudocode:

```text
sweep(budget):
    while budget > 0 and ttl_keys not empty:
        sample = random_sample(ttl_keys, 20)
        expired = 0
        for key in sample:
            if map[key] is expired:
                erase key from map and ttl_keys
                expired += 1
        budget -= 1
        if expired <= 25% of sample size:
            break
```

### 4.3 RESP parser

- Handwritten, in the socket-free `resp` module, with no third-party parsing library.
- A request is an array of bulk strings, for example a `SET k v` array with three bulk strings.
- Parsing is **incremental**. Given the bytes buffered so far, it returns one of:
  - a complete command plus the number of bytes consumed,
  - "need more bytes" (nothing consumed; the parser looks at the buffer again when more data arrives),
  - an error (protocol violation or limit exceeded).
- A command is complete only when every counted element has been read, so the parser cannot just wait for a `\r\n`. It checks by parsing.
- v1 is **stateless**: each call starts from the beginning of the buffer. This is simple, and cheap for normal requests. Re-reading is wasteful only for a large request that trickles in over many reads, which the size limits bound. A resumable parser that remembers its position (as Redis does) is a later option.
- Lengths are parsed with `std::from_chars`. Negative, overflowing, and non-digit values are rejected.
- Errors use `std::expected`, not exceptions.
- On a protocol error, the server sends an error reply and closes that connection.

### 4.4 Response encoder

- Reply types: simple string, error, integer, bulk string, null bulk, array. Modeled as a `std::variant`.
- The encoder appends directly into the connection's output buffer, avoiding intermediate copies.

### 4.5 Command dispatcher

- A table maps each command name to its arity and handler.
- Arity follows the Redis convention: a non-negative value is an exact argument count, and a negative value means "at least that many".
- The name is uppercased once, then looked up.
- Handlers are pure functions: given the store and arguments, they return a reply. This makes them easy to unit-test.
- Error texts follow Redis wording, for example "unknown command", "wrong number of arguments", and "value is not an integer or out of range".

### 4.6 Connection

Each connection owns:
- its socket, wrapped in a move-only RAII type that closes the descriptor,
- an input buffer and an output buffer with a drain offset,
- a flag meaning "flush remaining output, then close".

Behavior:
- **Read path:** read until the socket would block, append to the input buffer, then repeat parse → dispatch → encode until the parser needs more data. Pipelining falls out of this loop. The consumed input prefix is erased once per batch.
- **Write path:** try writing immediately. If the write is partial or would block, wait for the socket to become writable and resume.
- **Backpressure:** if the output buffer exceeds its cap, stop reading from that client, or disconnect it.
- **SIGPIPE:** suppressed per platform (`MSG_NOSIGNAL` on Linux, `SO_NOSIGPIPE` or ignoring the signal on macOS). The difference lives in one small platform header.

### 4.7 Event loop

- The listener uses `SO_REUSEADDR`, binds to `127.0.0.1`, and is nonblocking.
- Each iteration builds the poll set, calls `poll()` with a timeout equal to the time until the next timer, handles ready sockets, and accepts new connections until none are pending.
- Hangup, error, or a zero-byte read closes and removes that connection only.
- When the sweep timer is due, the loop runs a bounded sweep.
- **Shutdown:** the signal handler sets an atomic flag and wakes the loop (self-pipe or `EINTR`). The loop then closes all connections and exits.
- Later option: replace `poll` with `kqueue` (macOS) or `epoll` (Linux) behind the same interface.

## 5. Concurrency model

**Decision: a single thread, like Redis.** All commands run on one thread driven by a nonblocking event loop (`poll`). Many clients are served concurrently by taking turns, but never in parallel.

How it works:
- The loop waits for ready sockets, then for each one reads, parses, runs commands, and queues replies. Sockets are nonblocking, so no client can stall the loop while waiting on the network.
- A command finishes before the next one starts (run-to-completion). Each command, including multi-key ones like `DEL a b`, is therefore atomic, and the store needs no locks.
- Replies for one client keep request order, which makes pipelining work.

Why this choice:
- Operations are in-memory and mostly O(1), so one core handles a high request rate. Redis shows this.
- No locks means no data races, deadlocks, or lock overhead, and behavior is deterministic and easy to test.
- Simpler to reason about and explain, which suits a learning project.

Costs:
- A slow command stalls every client. All MVP commands are cheap, and the expiry sweep is time-bounded.
- Command throughput is capped by one core. Scaling beyond that would mean running several instances and sharding keys, as Redis Cluster does.

Possible later extensions, out of scope now: I/O threads for socket reads and writes with a single executor thread (as in Redis 6+), and background threads for slow cleanup.

## 6. Error handling

| Situation | Response |
|---|---|
| Unknown command, wrong arity, bad integer | Error reply; connection stays open |
| Protocol violation or limit exceeded | Error reply, flush, close the connection |
| Fatal read or write error | Close that connection only |
| `accept` fails (e.g. out of descriptors) | Log it and keep the loop running |

Exceptions are reserved for startup failures such as a bind error. They never escape the event loop.

## 7. Testing strategy

1. **Store:** set, get, overwrite, TTL transitions, lazy expiry with a fake clock, sweep budget.
2. **Parser:**
   - Table-driven valid and invalid inputs.
   - Fragmentation: feed a valid request split at every possible point and expect identical results.
   - Concatenation of several commands in one buffer.
   - Limits and overflow, such as an enormous declared length.
3. **Dispatcher:** arity, case-insensitivity, error texts.
4. **End-to-end:** run the server on an ephemeral port and drive it with raw sockets. Cover:
   - pipelining and byte-at-a-time sends,
   - disconnect mid-command,
   - many parallel clients,
   - a client that never reads,
   - TTL expiry with a short real sleep.
5. **Sanitizers:** ASan and UBSan on all tests.
6. **Fuzzing (stretch):** libFuzzer on the parser.
7. **Manual:** `redis-cli`, and `redis-benchmark` with pipelining.

## 8. Build and tooling

- C++23, CMake 3.25 or newer, with presets for debug and release.
- Warnings: `-Wall -Wextra -Wpedantic -Wconversion -Wshadow`.
- A static library `cachew_core` (resp, store, commands) is linked by both the server binary (`cachew`) and the tests.
- Test framework: GoogleTest, fetched with CMake `FetchContent`.
- Check C++23 library support early: `std::expected` needs libc++ 17+ or GCC 12+, and `std::print` needs libc++ 17+ or GCC 14+. Fall back to `iostream` if needed.

### Layout

```text
cachew/
  CMakeLists.txt
  DESIGN.md
  src/
    resp/       parser, encoder
    store/      store, expiry
    commands/   dispatcher, handlers
    net/        unique_fd, connection, event_loop
    main.cpp
  tests/
```

## 9. Milestones

The order is network-first: a real client (`redis-cli`) talks to the server from Milestone 1, and each later milestone replaces a stub with a real component. The dependency rule in section 3 still holds, because `resp`, `store`, and `commands` stay socket-free and are plugged into `net` as they arrive.

| # | Theme | Done when |
|---|---|---|
| 0 | Project setup | Empty server builds; a trivial GoogleTest passes |
| 1 | Client-server connection | `redis-cli -p 5123 PING` returns `PONG` |
| 2 | RESP protocol | `redis-cli` `PING` and `ECHO hi` return correct replies; parser tests pass |
| 3 | Commands and store | `redis-cli` runs all MVP commands; unit tests pass with a fake clock |
| 4 | Many clients, one thread | Nonblocking `poll` loop; pipelining, 100 parallel clients, and the slow-reader test pass |
| 5 | Expiry and hardening | Active sweep, limits, and clean shutdown work; sanitizer-clean |
| 6 | Measure and document | Benchmark numbers recorded; README lists supported commands and known deviations from Redis |

### Milestone 0: Project setup
- CMake project with presets (debug, release), warnings, and GoogleTest via `FetchContent`.
- Targets: `cachew_core` library, the `cachew` server binary, and the test binary, all with trivial `main`s.
- Confirm C++23 library support (`std::expected`) on the chosen compiler.

### Milestone 1: Client-server connection
- `UniqueFd` RAII wrapper; blocking single-client server: `socket`, `bind`, `listen`, `accept`, `--port` (default 5123).
- Server reads whatever arrives and replies with a hardcoded `+PONG`; it keeps the connection open until the client disconnects, then accepts the next one.
- This blocking version is a deliberate stepping stone. Milestone 4 replaces its loop with the nonblocking one, and the `UniqueFd` and socket setup are reused.
- Verify with `redis-cli -p 5123 PING`.
- Learn: POSIX sockets, RAII, TCP as a byte stream.

### Milestone 2: RESP protocol
- Handwritten incremental request parser (complete, need more, error) with limits and `std::from_chars` validation.
- Reply encoder for all reply types.
- Server parses real requests and answers `PING` and `ECHO`; protocol errors reply and close.
- Unit tests: split at every byte, concatenated requests, malformed and oversized input.
- Learn: `std::expected`, `std::variant`, `string_view`, designing for partial input.

### Milestone 3: Commands and store
- Store with injectable clock and lazy expiry; dispatcher with arity checks and case-insensitive names.
- Commands: `SET`, `GET`, `DEL`, `EXISTS`, `EXPIRE`, `TTL`, plus the existing `PING` and `ECHO`.
- Unit tests for each command, error texts, and TTL transitions using a fake clock.
- Learn: containers, `std::chrono`, handler tables, lifetimes of views into the store.

### Milestone 4: Many clients, one thread
- Nonblocking sockets and a `poll()` loop on a single thread (section 5); a `Connection` with input and output buffers.
- Partial reads and writes, pipelined requests, output-buffer cap, cleanup on disconnect.
- End-to-end tests: byte-at-a-time sends, disconnect mid-command, 100 parallel clients, a client that never reads.
- Learn: event-driven design, buffering, backpressure.

### Milestone 5: Expiry and hardening
- A list of keys with TTLs and a bounded, random-sampling periodic sweep driven by the loop timer.
- Enforce every limit; signal-based clean shutdown (`SIGINT`, `SIGTERM`).
- Hostile-input and failure-path tests; ASan and UBSan runs.
- Learn: data structure trade-offs, resource limits, signals, sanitizers.

### Milestone 6: Measure and document
- Run `redis-benchmark` (with and without pipelining) and record results and the machine used.
- Write a README: how to build and run, the supported command list, and known deviations from Redis.
- Note any hot spots found, as input for future work.
- Learn: benchmarking, reading results honestly.

**Definition of done (MVP):** multiple `redis-cli` clients independently use the supported commands. Expired keys disappear correctly. Fragmented and pipelined requests get correct replies. Disconnects do not crash the server. Tests and sanitizers pass.

## 10. Risks and open decisions

**Risks**
- A large request that arrives in many small pieces makes the stateless parser re-read it on each arrival. The size limits bound the cost. A resumable parser is the fix if benchmarks show a problem.
- `string_view` lifetime bugs between the store and the encoder. Mitigated by copying into replies and by ASan.
- macOS and Linux differ in SIGPIPE handling and `poll` behavior. Isolated in a platform header.
- `unordered_map` memory overhead is significant. Out of scope, but worth measuring.

**Open decisions**
- None at the moment.

## 11. Why RESP

RESP gives cheap framing, binary-safe values, typed replies, and native pipelining. It is also what lets `redis-cli`, `redis-benchmark`, and existing client libraries work unchanged. HTTP and gRPC were considered. Both cost more per request and hide the networking work this project is meant to teach. They remain candidates for a future optional frontend placed in front of the same dispatcher.
