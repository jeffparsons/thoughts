# Rust build caching

[Rust builds are slow](slow-builds.md).

One way to mitigate this is through build caching.

The main solution that exists for this today is [sccache](https://github.com/mozilla/sccache), but it has limitations like [caching based on absolute paths](https://github.com/mozilla/sccache/issues/35) that make it ill-suited for use in development environments. My understanding is that these limitations exist primarily because sccache is used as a [`rustc` wrapper](https://doc.rust-lang.org/cargo/reference/config.html#buildrustc-wrapper) and therefore does not benefit from any of the higher-level project information that Cargo has available.

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
  - Output of "I built a thing" is... well... I guess nothing? Oh, maybe optionally "I relocated this to ______". That would be nifty. Write up a justification for this.
- Should "monolithic dependency blob dealie" be part of this? It would be good to at least consider it to make sure this design doesn't make it harder.