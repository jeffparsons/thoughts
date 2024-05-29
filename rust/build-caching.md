# Rust build caching

[Rust builds are slow](slow-builds.md), and they take up a lot of disk space.
One way to mitigate this is to share build artifacts between projects, either locally on one machine or even across multiple machines.

The main implementation of this idea that exists today is [sccache](https://github.com/mozilla/sccache), which wraps compilers like rustc to cache their outputs.
This approach is convenient in that it requires [very little cooperation from the Rust toolchain](https://doc.rust-lang.org/cargo/reference/config.html#buildrustc-wrapper).
However, for this same reason it is also limited in that it can not make use of the richer context that Cargo has about a build job.

There is appetite in the Cargo team and the Rust user community at large for [some kind of first party support for shared build caches in Cargo itself](https://github.com/rust-lang/cargo/issues/5931), but there are differing opinions on what shape that support should take -- in particular, whether Cargo should offer a self-contained solution, or whether it should instead define an interface for calling out to external caching implementations.

I will first explore some details that are relevant regardless of the specific approach taken.
Then I will attempt[^1] to impartially summarise my understanding of each of the main schools of thought I have seen in discussions (Zulip, GitHub issues) so far.
Finally, I will offer my opinion on what should be done next.

Please note that I do not actually have any experience working on Cargo, so the rest of this document likely contains errors and misconceptions.
I appreciate any corrections or comments.

## Fingerprints hash mtime, not source text

[Fingerprints](https://doc.rust-lang.org/nightly/nightly-rustc/cargo/core/compiler/fingerprint/index.html) are digests that include most of the inputs to a compilation unit.
To allow determining whether a compilation unit is _fresh_, fingerprints include the _mtime_ of source files, such that any modification of a source file will result in the new fingerprint no longer matching the compiled unit.

This formulation of fingerprints is suitable for their current purpose, but is inadequate for use with a _shared_ build cache.
If two copies of the same source exist on one machine, or many copies exist across many machines, then any hash that does not include file content will not be able to produce cache hits.

One answer to this is to (at least initially) only cache build artifacts for packages that come from immutable sources, e.g., crates.io.
A given version of a package from crates.io will always have the same source code, so there is no need to consider file content _or_ mtimes for these packages; a hash of the name, version, and origin is enough.

I do however think it is worth at least considering how to address this issue for _all_ units, even if an initial implementation embraces some pragmatic limitations.

TODO:

- algorithm (notes on this below) for generating a complete unique digest.
  - I'm not proposing that the other concepts should be thrown out -- just that we need a complete cache key for any compiled unit, and that it can not include mtime.
- mtime can still be used...
- todo: note that we can just ignore this problem for registry packages, because they don't change. but that it should be considered up-front if it affects any interface.

# General challenges

Regardless of the approach taken ("implementation first" or "interface first"), there are a few general challenges:

- TODO: discuss content hashing, input file discovery, mtimes.

## Future opportunities

The first camp, which includes Cargo team members (todo: which? is there actually consensus?) advocates for building first class support for build caching into Cargo itself, including tracking artifacts across multiple projects, cleaning up, and eventually interfacing with third party caches through some kind of plugin system. Some of the main arguments for this approach are that it will provide a comprehensive solution out of the box and therefore benefit more people, and that the hard parts will be needed to support `cargo script` anyway (todo: links) so we might as well.

The other camp advocates for defining a narrow interface and offloading the rest of the implementation work to the surrounding ecosystem. Benefits include getting something up and running quickly, and allowing for experimentation outside of Cargo itself, which is understaffed and is also for good reasons relatively conservative. Given that a plugin system is a fairly late priority (todo: citation needed) for the "first class support" option, it may be years before something is available to integrate with third party solutions.

I will explore both options here and describe my interpretation of what a solution might look like, because I am sympathetic to both approaches. I wonder if we could have our cake and eat it, too, without making a complicated mess. todo: substantiate this.

Todo: maybe insert a section here about general challenges before diving into specifics of each approach.

## Unordered thoughts

TODO: Turn these into prose.

- Cargo doesn't currently hash file content; it just uses mtimes.
  - This was a fine choice originally, and the tradeoffs described on {page} are valid. But shared build caches mandate file hashing.
  - Should still use mtime to avoid hashing things that haven't changed!
- Cargo also doesn't know for sure what content went into a build.
  - TODO: Find a reference.
  - TODO: Lay out an algorithm for "good enough" assurance that what you hashed is indeed what went into a build.
  - NOTE: Should it consider a directory's mtime? Folks are talking about implicit module definitions based on file names. At very least include an aside / footnote about this.
- Propose a concrete interface:
  - You can configure separate executables (because why not) to be run before building something, and after something is built.
  - They would usually be the same exe, but don't need to be, so that you can, e,g., inject filters along the way that only understand one side of the equation. Or... is that necessary? Maybe the filter should just pass through if it doesn't want to do anything to the other side. Yeah... that would probably be better.
  - If people want to compose stacks of filters, that should be handled outside of Cargo; Cargo should expose the MINIMAL interface to let other people sort this shit out themselves, and the filtering thing can absolutely be built on top of the minimal interface.
  - Input to "I want to build this" is all the file hashes. Or... a summary hash. Or something like that.
  - Output of "I want to build this" is an optional descriptor pointing to a file(s) (because the compiler might request multiple things) on disk that can be used to fulfil the request. It is only guaranteed to exist for... how long? Maybe the descriptor can hint whether you should make your own copy or prefer to reference it in-place. That would be cool.
  - Input to "I built a thing" is again the hashes (same as above), and map of file paths. They are only guaranteed to exist until your program exits!!!!!
  - Output of "I built a thing" is... well... I guess nothing? Oh, maybe optionally "I relocated this to **\_\_**". That would be nifty. Write up a justification for this.
- Should "monolithic dependency blob dealie" be part of this? It would be good to at least consider it to make sure this design doesn't make it harder.

[^1]: At the same time acknowledging that this is impossible, and that I am unavoidably biased.
