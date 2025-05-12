---
title: Reflecting on Errors in 2019
published_date: "2019-11-26 19:09:27 +0000"
layout: default.liquid
is_draft: false
---
With people reflecting on Rust in 2019 and what they want to see in 2020, error handling has come up again:
- [Rust in 2020, one more thing](https://www.ncameron.org/blog/rust-in-2020-one-more-thing/)
- [Thoughts on Error Handling in Rust](https://lukaskalbertodt.github.io/2019/11/14/thoughts-on-error-handling-in-rust.html)
- [Error Handling Survey](https://blog.yoshuawuyts.com/error-handling-survey/)

<!-- more -->

It felt like there was interest in moving `anyhow` into the stdlib.  While I
feel that a lot of it is ready or near-ready (see below), I feel we
have gotten stuck in a rut with the `context` pattern.  I've been told that
[the pattern is derived from
cargo](https://github.com/withoutboats/fehler/issues/16) and it feels like
we've mostly been iterating on that same pattern rather than exploring
alternatives. After framing my concerns with error handling and where we are at
with existing solutions, I'll introduce [`Status`](https://docs.rs/status/) as a
radical alternative (from the Rust ecosystem perspective) for addressing these
problems. If you aren't interested in all the middle stuff, feel free to skip
to the end!

Update: I messed up my reading of docs and test cases and claimed that `Box<dyn
Error>` implemented `Error`. This is not the case and my post has been updated
to reflect that.

## Why are errors such a hot topic in Rust compared to other languages?

When I look at other languages, for the most part what error is returned is an
implementation detail unless stated otherwise in documentation. This can be
achieved with `Box<dyn Error>` but with trade-offs

- In Rust, errors types are statically constrained with traits, preventing a
  `Box<dyn Error>` from being cloneable, serialiable, etc.
- For the performance sensitive, `Box<dyn Error>` is 2 pointers-wide and risks
  changing how the compiler returns values (using the stack instead of
  registers).
- Backtraces are not automatically included with `Box<dyn Error>`.
- Not even  `Box<dyn Error>` is a good citizen ([`impl From<E: Error>` for `?`
  usage and `impl Error` for
  interop](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ff8d532f1e6cdb1cd635e32d6c0432d0)).
  Another spin on this is [doing something similar ourselves](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=268fdba6fb89f3ecc99f1c3d8e9f01d4).
  This can't work until specialization is stabilized.

Other challenges that are unique to Rust:

- In Rust, functions opt-in to being failable while in other languages, at
  best, you opt-out, requiring boilerplate to switch back and forth.
- Rust does not provide a "low-effort" error type and syntax for prototyping
  (`std::io::Error` seems the closest).
- When adding `impl From<dep::Error> for crate::Error` for use with `?`, you
  are exposing `dep` in your public API, making it a breaking change to remove
  it or upgrade major versions.
- `fn main() -> Result<(), Error>` will use `Debug` rather than `Display`.
- When wrapping errors, it is unclear which error in the chain is the "real
  error", an implementation detail, or just augmented information
- Supporting `no_std`.

Challenges that I feel don't just apply to Rust:

- Mapping errors to process exit codes.
- Providing a localizable, end-user message.

## Status of solutions

Spoiler alert: I started this post as an introduction for
[`Status`](https://docs.rs/status/) but it grew from there. What's worse, is I
felt it best to order things from more concrete plans to more wild ideas. If
you are interested in a Proof-of-Concept that tries to address (almost) all of
the above problems, I recommend skipping to the end.

### Concrete error types

[`thiserror`](https://github.com/dtolnay/thiserror) is a fairly mature solution
for reducing the boilerplate. However, I feel it should narrow its focus on just `trait
Error`. We should
instead have a generalized solution for deriving `Display` and `From`.
- Allow other types to use them
- For `Display`, allow alternative policies
  ([attributes](https://jeltef.github.io/derive_more/derive_more/display.html)
  vs [doc-comments](https://github.com/yaahc/displaydoc))
- For `From`, add friction to the process to raise awareness of the trade-offs.

With those features removed, I'd love to see this moved into the **stdlib**.

### `Box<dyn Error>`

Some might see [`anyhow`](https://github.com/dtolnay/anyhow) filling this role
but I feel we should decouple adhoc error handling from an abstract error
container.

There was a discussion about a [`BoxError` on
internals](https://internals.rust-lang.org/t/proposal-add-std-boxerror/10953/2)
and I feel this is the way to go. I feel we should have something that looks like:

```rust
#[derive(Debug. anyhow::Error, derive_more::Display, derive_more::From)]
struct BoxError(Box<Box<dync Error + Send + Sync + 'static>>);
```

This does not solve cloning, serialization, localization, or a host of other
issues, however.

For me, the main open questions before putting this in the **stdlib** are:
- Like `Box`, we are blocked on specialization to support both `impl Error` and `From<E: Error>`.
- Name bikeshedding.
- Whether the "thin pointer" (`Box<Box<dyn _>>`) should be generalized.
- Whether backtraces should be included (personally, I think that should be
  left to concrete and ad-hoc error types).

### Ad-hoc Error Types

I think `anyhow::Error` and `anyhow::anyhow!` are close to something we can standardize. My main concerns
- Couples the concept of `Box<dyn Error>` (`anyhow::new`) with ad-hoc errors (`anyhow::msg`).
  - Being distinct can reduce scope, hopefully speeding up moving into the **stdlib**
  - By being distinct, I feel we'll do a better job communicating the trade-offs of different patterns.
- It favors the `context` idiom (adding metadata by wrapping errors) which I
  think is an immature space that needs further exploration.

### `From` for `?`

The first problem is `From<dep::Errpr>` exposing implementation details.
- `snafu` experiments with the ideas of macro-generated "selectors" which are
  private types which would keep `From<dep::Error>` private. I find the
  approach interesting and an area we should do more experimentation (though
  from an approachability perspective I dislike derives generating anything
  from than the `impl` for a trait).
- Controlling visibility on trait impls.  There was an
  [RFC](https://github.com/rust-lang/rfcs/pull/2529) which got attention for
  other reasons.  It ended up stalling due to soundness concerns with
  specialization.

The second problem is `From<E: Error>`
- Ideally we get specialization to solve this.
- I think `snafu` selectors might also help here since they don't need to `impl Error`.

### Streamlined Syntax

This is a more diverse space
- [`anyhow::bail!`](https://github.com/dtolnay/anyhow): Seems like a mature area though `fehler` might replace some roles of it.
- [`anyhow::ensure!`](https://github.com/dtolnay/anyhow): Seems handy but if we go with `fehler`, the name might need to be bike-sheded.
- Reduced `Result` boilerplate with [`fehler`](https://github.com/withoutboats/fehler): I think this has a lot of
  potential with the biggest hindrance being people's initial
  impression. `fehler` originally also encompassed `anyhow` and provided a
  complete solution using exception nomenclature.  Like a lot of people, I was
  concerned about this. For me, I was concerned about how different exceptions
  and monadic errors are and how adopting exception language, while potentially
  more approachable, would lead people into making bad assumptions. Since then,
  `fehler` has [narrowed its focus on reducing `Result`
  boilerplate](https://github.com/withoutboats/fehler/issues/20), relying on
  crates like `anyhow` for everything else.
- [Try-expressions](https://github.com/rust-lang/rfcs/blob/master/text/2388-try-expr.md):
  While this is only at the stage of having the keyword reserved, I look
  forward to this. The alternatives are `map` (high boilerplate) and closures
  (low discverability).

### Everything else

Introducing my Proof-of-Concept, [`Status`](https://docs.rs/status/). I refer to
it as an error container because it provides a basic structure for the major
parts of an error which you then populate.

Unlike the error-wrapping pattern found in `cargo` and generalized in `anyhow`, the pattern
implemented in `Status` comes from projects I've worked on which try to address the
following requirements:
- Programmatically respond to both the `Kind` of status and the metadata, or `Context`, of
  the status.
- Dealing with error-sites not knowing enough to describe the error but allowing the
  `Context` to be built gradually when unwinding where there is relevant information to
  add.
- Localizing the rendered message.
- Allowing an application to make some phrasing native to its UX.
- Preserving all of this while passing through FFI, IPC, and RPC (TODO [#1](https://github.com/epage/status/issues/1) [#2](https://github.com/epage/status/issues/2)).

These requirements are addressed by trading off the usability provided by per-site custom messages with
messages built up from common building blocks.  The `Kind` serves as a static description of the
error that comes from a general, fixed collection.  Describing the exact problem and tailored
remediation is the responsibility of the `Context` which maps general, fixed keys with
runtime-generated data.

`Status` grows from your prototype to a mature library.

A prototype might look like:
```rust
use std::path::Path;
type Status = status::Status;
type Result<T, E = Status> = std::result::Result<T, E>;

fn read_file(path: &Path) -> Result<String> {
    std::fs::read_to_string(path)
        .map_err(|e| {
            Status::new("Failed to read file")
                .with_internal(e)
                .context_with(|c| c.insert("Expected value", 5))
        })
}

fn main() -> Result<(), status::TerminatingStatus> {
    let content = read_file(Path::new("Cargo.toml"))?;
    println!("{}", content);
    Ok(())
}
```

The `TerminatingStatus` provides a `Debug` that produces user-visible data and
will print chained error messages, so the output will looks something like:
```
Failed to read file

Expected value: 5
```

You can then customize the `Kind` and `Context` used.

```rust
use std::path::Path;
#[derive(Copy, Clone, Debug, derive_more::Display)]
enum ErrorKind {
  #[display(fmt = "Failed to read file")]
  Read,
  #[display(fmt = "Failed to parse")]
  Parse,
}
type Status = status::Status<ErrorKind>;
type Result<T, E = Status> = std::result::Result<T, E>;

fn read_file(path: &Path) -> Result<String, Status> {
    std::fs::read_to_string(path)
        .map_err(|e| {
            Status::new(ErrorKind::Read).with_internal(e)
                .context_with(|c| c.insert("Expected value", 5))
        })
}
```
*(custom `Context` not shown)*

For more, check out [the docs](https://docs.rs/status/) and [the
issues](https://github.com/epage/status/issues). Like I said, this is a
Proof-of-Concept.  Your help is needed, whether feedback, ideas, or code. Even
something as simple as encouragement that this is worth pursuing would be good
feedback into where I put my time.
