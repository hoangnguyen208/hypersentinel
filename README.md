# HyperSentinel

A self-hosted, low-latency risk engine and pre-trade order gateway for Hyperliquid, written in Rust.

> Status: early development.

## Layout

| Crate | Responsibility |
|---|---|
| `crates/core` | Domain types, fixed-point arithmetic, ports (traits). No I/O. |

## Development

```sh
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Design documents live in [`docs/specs`](docs/specs); architecture decisions in [`docs/adr`](docs/adr).

## License

Licensed under either of Apache License 2.0 or MIT license, at your option.
