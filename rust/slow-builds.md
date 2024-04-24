# Rust builds are slow

## Problem

Rust builds for large projects (including those with large dependencies) take a long time, and `target/` directories are enormous.
(I'm bundling the disk usage in to this document for now because many of the mitigations are the same for both build time and disk usage.)

This is particularly burdensome when

- a project depends on very large crates like [aws-sdk-ec2](https://crates.io/crates/aws-sdk-ec2).
- you build many different Rust projects on the same machine regularly. (This is particularly bad for disk usage. Most of my disk space is taken up by various `target/` directories.)
- automated dependency bumps invalidate build cache so you have to rebuild dependencies almost every time you build a project locally.
- many developers in a team are waiting for the same things to build.

## Mitigations available today

TODO: Flesh this out with links and details.

- Build everything on CI where possible.
- Use sccache locally.
- Use a shared sccache for your team.
- Use a shared build dir for all Rust projects for your user on your machine.
- Use shared libraries with a C ABI.
- Use Wasm Components.
- Use a better linker (mold).

## Future solutions

TODO: Flesh this out with links and details.

- Better sccache support
- First-class cargo hooks for whatever build sharing solutions you want. (I think there's an RFC for something like this, or comments on an issue somewhere.)
- Better support for dynamic linking:
    - crABI
    - `#[export]`
- Better cross-compilation
    - Hard to do because Apple doesn't want you to cross-compile to macOS.
- Wasm Components becoming more capable
