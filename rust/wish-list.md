# My Rust wish list

Extremely incomplete.

## Async

- Async drop
- `async fn` in `dyn Trait` (though [`async_trait`](https://docs.rs/async-trait/latest/async_trait/)) is a pretty good workaround for now)
- Async generators
- `AsyncIterator`

## Motivated by async, but useful more generally

- [`Move` trait](https://without.boats/blog/changing-the-rules-of-rust/)
- "Unforgettable" types

## Tools to manage build time and space

See also: [Rust builds are slow](slow-builds.md).

- [crABI](https://github.com/joshtriplett/rfcs/blob/crabi-v1/text/3470-crabi-v1.md) for (statically or dynamically) linking separately-compiled crates
- [`#[export]`](https://github.com/m-ou-se/rfcs/blob/export/text/0000-export.md) for dynamic linking
- [Cargo cache management](https://blog.rust-lang.org/2023/12/11/cargo-cache-cleaning.html) [[tracking issue]](https://github.com/rust-lang/cargo/issues/12633)
- [Shared local build cache](https://github.com/rust-lang/cargo/issues/5931)
- Hooks in Cargo for external build caches (let other people decide how, e.g., remote shared caches work)

## Misc language features

- Generators
- Arbitrary User-definable contracts/proofs
- Scoped reference sharing channel as a safe escape hatch for static borrow checking (TODO: this needs its own doc to explain)
- [`BTreeMap` cursors](https://github.com/rust-lang/rust/issues/107540) — would make [rangemap](https://docs.rs/rangemap/latest/rangemap/) simpler and more efficient, and make it easier to build new features

## Misc tooling features

- First-class support for Wasm Components
