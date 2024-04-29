# Rust builds are slow

## Problem

Rust builds for large projects (including those with large dependency trees) take a long time, and `target/` directories are enormous.
(I'm bundling the disk usage in to this document for now because many of the mitigations are the same for both build time and disk usage.)

This is particularly burdensome when

- a project depends on very large crates like [aws-sdk-ec2](https://crates.io/crates/aws-sdk-ec2).
- you build many different Rust projects on the same machine regularly. (This is particularly bad for disk usage. Most of my disk space is taken up by various `target/` directories.)
- automated dependency bumps invalidate build cache so you have to rebuild dependencies almost every time you build a project locally.
- many developers in a team are waiting for the same things to build.

## Mitigations available today

### Build things on CI where possible

If you have the luxury of building things in a CI environment, then you may be able to avoid building those things locally at all if you are not making changes.
But CI might not include builds for your platform, e.g., when developers are using MacBooks but macOS licensing makes doing Mac builds in CI prohibitively expensive.

This doesn't help much if you do need to make changes, because then you'll need to build the whole thing locally anyway. This could be mitigated by using dynamically linked libraries, but those are not very ergonomic in Rust. (See below.)

### Use [sccache](https://github.com/mozilla/sccache)

Developers could use sccache with a shared S3 bucket (or whatever).

This could be really useful, but sccache has some warts that are particularly troublesome on development machines like [caching based on absolute paths](https://github.com/mozilla/sccache/issues/35).

### Build multiple projects to one target directory

Get all Cargo projects to put their artifacts in a single shared directory, e.g., by setting the `CARGO_TARGET_DIR` environment variable. This could help reduce duplication of common crate builds across different projects.

You could do something similar with [sccache operating on a single machine](https://doc.rust-lang.org/cargo/guide/build-cache.html#shared-cache).

### Build dependencies as dylibs or rlibs

Per https://doc.rust-lang.org/reference/linkage.html.

This seems to be uncommon in the Rust community. But maybe that's just because when people have use cases for this they aren't the kind of use cases they talk about much publicly?

TODO: Look into whether dylibs or rlibs could be helpful when you know you're using the same compiler version on the same platform, etc. -- what is the experience actually like? I think Cargo doesn't actually have any support for linking to rlibs explicitly; it [expects to build everything from source](https://stackoverflow.com/questions/75295758/rust-libraries-with-cargo-rlib)?

TODO: Look into whether there's any reason to use a cdylib over a dylib if you know you're using the same compiler version.

### Build dependencies as Wasm Components

This has all the usual benefits of using Wasm Components, e.g., sandboxing, build-once-run-everywhere, etc.

However, it also means that if the dependency in question isn't natively available as a Wasm Component, then you'll need to wrap it up yourself. It's also hard to justify introducing Wasm Components into a codebase just to help wrangle dependencies if you didn't already have another compelling driver for it.

### Speed up linking using a better linker

Incremental builds can often spend a significant amount of time in linking. Using a better linker like [mold](https://github.com/rui314/mold) can help a lot.

## Future solutions

### Make sccache play better with dev environments

Make it understand relative paths.

### Shared compilation cache for Cargo

[Cargo issue 5931](https://github.com/rust-lang/cargo/issues/5931) has lots of thoughts about local shared compilation caches. Of particular interest:

- [epage's design sketch](https://github.com/rust-lang/cargo/issues/5931#issuecomment-1551478767), which includes consideration of garbage collection.
- [RobJellinghaus's comment](https://github.com/rust-lang/cargo/issues/5931#issuecomment-1947491412) about an internal Microsoft hackathon where they prototyped some of this stuff.

### Better support for dynamic linking in Rust

The [crABI RFC](https://github.com/joshtriplett/rfcs/blob/crabi-v1/text/3470-crabi-v1.md) describes an opt-in stable ABI.

The [`#[export]` attribute](https://github.com/m-ou-se/rfcs/blob/export/text/0000-export.md) RFC describes a mechanism to better support dynamic linking, and remarks that it would [allow not building the whole upstream crate](https://github.com/m-ou-se/rfcs/blob/export/text/0000-export.md#building-dynamic-dependencies).

### Better cross-compilation

It would be nice to be able to target macOS from Linux build hosts, but the current ways of doing this are not great. Current solutions require using bootlegged macOS SDKs (which I will not link here) and I have run into spooky crashes in the resulting binaries.

This is unlikely to improve because Apple doesn't want you to cross-compile to macOS.

### Wasm Components mature

Wait for Wasm Components to evolve and stabilise, and for more libraries to be available as Components. E.g. it seems likely that AWS will eventually publish Wasm Components for the AWS SDK, at which point it becomes unnecessary to compile the enormous `aws-sdk-ec2` crate yourself.
