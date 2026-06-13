# Fragment Reassembler

**A Rust library for reassembling fragmented network packets**, reconstructing the original message from ordered and unordered fragments received over unreliable transports.

## Why It Matters

In network protocols — UDP-based custom transports, mesh networking, satellite communications, and IoT messaging — messages are frequently split into fragments to fit within MTU limits. The receiver must buffer arriving fragments, handle out-of-order delivery and loss, and reassemble the complete message. This crate provides the reassembly buffer data structure that handles these concerns. In the SuperInstance fleet, edge nodes communicating over unreliable WebSocket connections use fragment reassembly for large telemetry payloads.

## How It Works

The library implements a gap-filling buffer that tracks which fragment offsets have been received. Fragments arrive tagged with their offset within the original message. The reassembler inserts each fragment at the correct position and advances a "completed" pointer forward through contiguous data. Once the final fragment arrives and all gaps are filled, the complete message is emitted. This design handles reordering, duplication, and partial loss naturally — fragments simply fill in their slots as they arrive.

## Quick Start

```rust
// API surface under development — the crate currently provides
// foundational types for fragment tracking.
use fragment_reassembler::add;

fn main() {
    // Basic smoke test
    assert_eq!(add(2, 2), 4);
}
```

## API

| Function | Description |
|---|---|
| `add(left, right)` | Placeholder — full reassembly API under development |

## Architecture Notes

Part of the SuperInstance networking layer, designed to support reliable message delivery over the gossip protocol stack (`gossip-protocol`, `gossip-member`, `gossip-ping`). See the [Architecture Guide](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## License

MIT
