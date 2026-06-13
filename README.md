# Fragment Reassembler

A **packet fragment reassembly library** that reconstructs original messages from ordered fragments arriving over unreliable channels, handling gaps, duplicates, and out-of-order delivery using offset-indexed buffering.

## Why It Matters

Network protocols (IP, UDP, SCTP) and message-oriented middleware frequently fragment payloads to fit MTU constraints. Without reassembly, the receiver sees disjointed chunks instead of coherent data. This library provides the reassembly primitive: given fragments tagged with their offset and length, it reconstructs the original contiguous buffer. The same problem appears in file systems (extents), video streaming (segment assembly), and distributed tracing (span reconstruction). Correct fragment reassembly must handle three failure modes: loss (missing fragments), duplication (retransmits), and reordering (out-of-order arrival).

## How It Works

Fragment reassembly is fundamentally a **constraint satisfaction** problem: given N fragments each defined by `(offset, length, data)`, reconstruct the original message of total length L such that every byte position [0, L) is covered exactly once.

**Naive approach** — sort fragments by offset and concatenate. This fails when fragments are missing or overlapping. Time: O(N log N) for the sort.

**Sliding-window approach** — maintain a receive buffer indexed by byte offset. For each incoming fragment, write data into the buffer at its offset. Track the contiguous prefix (the "leading edge") that has been fully received. When the leading edge reaches the total expected length, reassembly is complete.

```
Buffer:    [_ _ _ _ _ _ _ _ _ _ _ _]
Fragment 1 (offset=0, len=4):  [A B C D _ _ _ _ _ _ _ _]
Fragment 3 (offset=8, len=4):  [A B C D _ _ _ _ W X Y Z]
Fragment 2 (offset=4, len=4):  [A B C D E F G H W X Y Z]  → complete
```

**Complexity**: O(F + L) where F is total fragment bytes and L is reassembled length. Space: O(L) for the receive buffer.

**Duplicate detection**: If a fragment arrives whose offset range overlaps already-written data, the library overwrites (idempotent) or discards (deduplicating), depending on policy.

**Gap detection**: A bitmap tracks which byte ranges have been filled. A fragment is only released to the application when all preceding bytes are present.

## Quick Start

```rust
use fragment_reassembler::Reassembler;

let mut r = Reassembler::new(12); // expect 12-byte message
r.push(0, b"ABCD");
r.push(8, b"WXYZ");
assert!(!r.is_complete());
r.push(4, b"EFGH");
assert!(r.is_complete());
assert_eq!(r.assemble().unwrap(), b"ABCDEFGHIJKL"[..12]);
```

## API

| Type | Description |
|------|-------------|
| `Reassembler::new(expected_len)` | Create a reassembler for a message of `expected_len` bytes |
| `reassembler.push(offset, data)` | Insert a fragment at byte offset |
| `reassembler.is_complete()` | True when all bytes [0, expected_len) are filled |
| `reassembler.assemble()` | Return the reassembled buffer (or `None` if incomplete) |

## Architecture Notes

This library provides the reliability layer for the SuperInstance message bus. When agents communicate over UDP or unreliable transports, messages exceeding the MTU are fragmented; this library ensures transparent reconstruction. It contributes to **η** (reflex) in **γ + η = C** — reassembly is automatic, requiring zero coordination overhead from the application layer. See [Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

**Timeout and cleanup**: Fragments that never arrive (permanent gaps) must be garbage-collected. A common strategy is a timer per partial reassembly: if the timer expires before completion, the partial buffer is discarded to prevent memory leaks from malformed or malicious senders.

**Security**: Fragment reassembly is the target of several well-known attacks (teardrop, overlap attacks). A robust reassembler validates that fragment offsets don't overlap maliciously and rejects fragments claiming to extend beyond the expected total length.

## References

- Postel, J. RFC 791: "Internet Protocol — Fragmentation and Reassembly," 1981.
- Stewart, R. RFC 4960: "Stream Control Transmission Protocol," 2007.
- Wood, L. et al. "TCP and IP Fragment Reassembly," ACM SIGCOMM CCR (2002).
- Kent, C. & Mogul, J. "Fragmentation Considered Harmful," DEC WRL Tech Report (1987).

## License

MIT
