# Cachew

Cachew is a small Redis-compatible, in-memory key–value server written in C++23. It is a learning project: the goal is to understand networking, protocol parsing, and server design by building each piece by hand.

It speaks a subset of RESP2 over TCP, so `redis-cli` can talk to it.

> **Status:** Design phase. No code exists yet. See [DESIGN.md](./DESIGN.md) for the full design and milestone plan.

## Supported commands

| Command | Behavior |
|---|---|
| `PING [msg]` | Replies `PONG`, or echoes `msg` |
| `ECHO msg` | Returns `msg` |
| `SET key value` | Stores the value, overwrites any existing one, clears any TTL |
| `GET key` | Returns the value, or nil if missing or expired |
| `DEL key [key ...]` | Returns how many keys were removed |
| `EXISTS key [key ...]` | Returns how many of the keys exist |
| `EXPIRE key seconds` | Sets a TTL. Returns 1 if the key exists, else 0 |
| `TTL key` | Remaining seconds, `-1` if no TTL, `-2` if the key is missing |

Command names are case-insensitive. Planned later: `SET` options `EX`, `PX`, `NX`, `XX`.

## Not supported

Persistence, replication, clustering, auth, TLS, pub/sub, transactions, lists/hashes/sets, inline (plain-text) commands, and multithreaded command execution. Cachew has no auth and binds to localhost only; do not expose it to a network.

## Build and run

Not available yet. Once Milestone 0 lands:

```sh
cmake --preset release && cmake --build --preset release
./build/release/cachew --port 5123   # default port 5123
redis-cli -p 5123 PING
```

Requires a C++23 compiler, CMake 3.25+, and `redis-cli` for testing.

## Docs

See [DESIGN.md](./DESIGN.md) for the architecture, testing strategy, and milestones.



