---
title: `toml` v0.9
published_date: "2021-12-30 09:00:30 -0500"
layout: default.liquid
is_draft: true
data:
  tags:
    - programming
    - rust
---

[`toml` v0.9][TODO] is a near-complete rewrite with dramatic performance improvements, `no_std` support, and many other improvements.

This is where I've disappeared to over the last 3 months, trying to wrap up a years-old experiment that has been "almost done" for these last 3 months, with use case after use case building up encouraging me to finish this up rather than my regularly scheduled Cargo work.

<!-- more -->

## Goal: performance

The primary goal of this effort was to experiment with a different parser architecture to see what the performance of it would be like.
People are not normally dependent on how long a TOML file takes to parse.
However, Cargo may parse a thousand TOML files on every invocation.
Most people don't notice the impact because compile times can dwarf the parsing of these files.
Its in the small things that this will be noticed, like

- `rustup` adding overhead to every the execution of every Rust toolchain command to parse `rust-toolchain.toml` ([rustup#2626](https://github.com/rust-lang/rustup/issues/2626))
- Higher latency when your shell looks up completions for `cargo` with our new completion system ([Unstable: native-completions](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#native-completions))
- Cargo's overhead when executing a fully built cargo-script ([Unstable: script](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#script))
- Other tools using cargo programmatically ([zulip: `cargo metadata` performance](https://rust-lang.zulipchat.com/#narrow/channel/246057-t-cargo/topic/.60cargo.20metadata.60.20performance/near/476523460))

I had proposed that `cargo publish` include a `Cargo.json` or `Cargo.cbor` inside of the `*.crate` file,
bypassing the overhead of parsing `Cargo.toml` for the majority of packages.
There was hesitance due to risks associated with having two for the same content,
one viewed by users and the other viewed by machines.
This was a last ditch effort to prove whether `Cargo.toml` parsing could be sufficiently fast.

### Format challenges

While JSON started as a subset of Javascript,
it has a fairly simple syntax and structure and it is well understood today how to make it fast due to its heavy use in machine-to-machine communication.
On the other hand, TOML is more heavily oriented towards human with less care for the impact on performance.

Some examples of this include:

TOML has order independent sub-tables which requires building up the result piecemeal which isn't compatible with models like `serde` and requires additional bookkeeping of some sort.

For example, the following are equivalent:
```toml
[one]

[two]

[one.a]

[three]

[one.b]
```
```json
{
  "one": {
    "a": {}
    "b": {}
  },
  "two": {},
  "three": {}
}
``` 
or
```toml
apple.type = "fruit"
orange.type = "fruit"
apple.skin = "thin"
orange.skin = "thick"
apple.color = "red"
orange.color = "orange"
```
```json
{
  "apple": {
    "type": "fruit",
    "skin": "thin",
    "color": "red"
  },
  "orange": {
    "type": "fruit",
    "skin": "thick",
    "color": "orange"
  }
}
```

TOML has very specific rules about how different table syntaxes can be combined (`[table]`, `dotted.key`, `{ "inline" = "table" }`).

For example,
```toml
[fruit]
apple.color = "red"
apple.taste.sweet = true

# [fruit.apple]  # INVALID
# [fruit.apple.taste]  # INVALID

[fruit.apple.texture]  # you can add sub-tables
smooth = true
```

`toml` v0.5 solved this problem by parsing tokens into an AST (a `Vec<Table>` where `Table` is not recursive) and generating lookup tables to make it easier to walk the logical structure among the syntactic structure.

`toml` v0.6 [delegated parsing to `toml_edit`]({{site.base_url}}/blog/2023/01/toml-vs-toml-edit/) which
parses directly into a [`Document`] representing the logical structure (a recursive [`Table`]).
`toml_edit` did two lookups for every table for simplicity of matching TOML's rules for merging of syntactic tables.
When starting to parse a syntactic table,
we remove the [`Table`] and directly insert into it.
When we find the next table header,
we then re-insert the [`Table`] we were just editing.

`toml` v0.8 continues with the approach of `toml_edit` by providing a logical [`DeTable`] that it parses into.

### Cargo requirements

Originally, `cargo` had been using `toml` to parse `Cargo.toml` files.

This changed with the introduction of `cargo add` which used `toml_edit` for its ability to make changes to `Cargo.toml` without losing the users formatting.
When proposing to merge `cargo add`, the Cargo team was concerned about having two different TOML parsers inside of Cargo which may have slightly incompatibilities and different errors.
I had proposed that we solve this problem by having one parser, `toml_edit`, and extending it to support `toml`s API.
This brought on the requirement of "no loss in performance".

`toml_edit` was then relied on to parse `--config key=false` TOML expressions, disallowing full documents from being present, including comments and newlines.

Lately, the Cargo team is experimenting with using the richer information reported from `toml_edit` to provide rustc-like diagnostics for `Cargo.toml`.

To summarize, the traits we need to preserve while making `toml` faster include:
- Consistent parsing behavior between `serde`-like reading and writing of TOML and format-preserving edits of user TOML files
- Validation of `key=value` expressions
- Spans for keys and values

**To ensure consistent behavior,**
we cleaned up the tests in `toml` and `toml_edit`,
isolating crate-specifics,
and synced them into each other.
This puts a burden on us to ensure that any changes to `toml` or `toml_edit`s shared tests get synced back to the other.

As a second line of defence, parsing of the AST is shared in the [`toml_parse`] crate.

As a third-line of defence, the conversion from the AST to the logical structure and the `serde` implementations are maintained in such a way that we can diff the implementations.
However, this is noisy and requires us to remember to sync even changes, much like the tests.

**Validating of `key=value` expressions can still be done with `toml_edit`.**

**As for providing Spans,**
we leveraged the fact that we need to provide a logical [`DeTable`] for `serde` and included Spans in it (i.e. `Spanned<DeTable>`).

Like [`toml_edit::Document`][`Document`], this requires [`DeTable`] to be an owned type so that `cargo` can keep it around until a diagnostic needs it.
However, it should be rare that `cargo` needs spans for dependencies.
To prepare for this possibility, we made [`DeTable<'s>`] track all strings using
[`Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) to avoid allocations when escaping isn't used.
To make this owned, you can call [`DeTable::make_owned`] which switches all `Cow::Borrowed`s to `Cow::Owned`.
At that point, it is safe to
[transmute](https://doc.rust-lang.org/core/mem/fn.transmute.html)
a [`DeTable<'s>`] to a [`DeTable<'static>`] as lifetimes do not affect layout (or so I'm told).

### Implementation

Internally, `toml` v0.5 has a fallible tokenizer that iterated through `char`s` in a `&str`, requiring UTF-8 decoding logic in the inner loop.
As mentioned before, it then parsed into a `Vec<Table>` (non-recursive) and created lookup tables to speed up access.

`toml` v0.6 (via `toml_edit`) directly parsed bytes, avoiding the UTF-8 decoding, directly into a [`Table`] (recursive).
For parsing bytes, it helps that TOML syntax only distinguishes 7-bit ASCII characters from each other and UTF-8 is left opaque.
`unsafe` was heavily used to avoid re-validating that the `&[u8]`s returned were actually `&str`.
The parser is written using parser combinators via [`winnow`].
This made it very easy to compare the implementation to the
[grammar](https://github.com/toml-lang/toml/blob/1.0.0/toml.abnf)
and got fairly good performance results compared to the old parser.
However, this required backtracking and carrying state in `Result`s which likely helped spill return values over from registers to the stack
(see [winnow#72](https://github.com/winnow-rs/winnow/issues/72)).
To preserve formatting,
the size of an [`Item`] returned by a parser is also quite large.
By doing direct parsing, we had deeper call stacks for this all to accumulate in and higher costs, both in terms of the call stack and parsed result, when backtracking.

`toml` v0.9 (via [`toml_parse`]) has an infallible tokenizer that we collect into a [`Vec<Token>`][`Token`],
assuming the capacity based on the length of the `&str` we are parsing.
The tokenizer processed bytes within each individual step but to calculate indices to take a sub-slice of a `&str`.
We made `unsafe` optional in `toml_parse` to avoid the UTF-8 boundary checks but found the overhead of the safe option to be low enough that we didn't use it.
We then report parse events and errors through callbacks,
with `toml` collecting them into `Vec`s with [`Vec<Event>`][`Event`]s capacity inferred from [`Vec<Token>`][Token]s length.
In a couple of places (table headers, scalar values),
the parser does do some lookahead but only needs to compare [`TokenKind`]s and not iterate through a sequence of bytes.
The parser only looks at [`TokenKind`]s and not the `&str`, meaning that scalar values are not validated at this stage.
All structural syntactic errors are expected to be reported at this stage.
`toml` walks these events to build up a [`DeTable`], validating the logical structure and decoding and validating scalars along the way.

## Secondary goals

While the focus was on performance for Cargo,
we wanted to experiment with some other improvements.

### Error recovery

Like `rustc`, we wanted to allow `toml` to report as many errors as possible, not just the first.

The core of error recovery is that all errors reported through a [`&mut dyn ErrorReport`] parameter.
The caller can choose whether they collected into a `()`, [`Option<ParseError>`], [`Vec<ParseError>`], or something else entirely.
This was another important reason to avoid backtracking because we didn't want to have to revert errors that had been reported.

To keep the logic in `toml` as simple as possible,
we also made the parser create synthetic [`Event`]s wherever possible
(e.g. reporting a closed table header with an empty span when a `]` was missing).

### Bignum support

While I figure an `i64` is good enough for any config file,
that didn't appear to be the case.
`toml` doesn't just delay scalar value decoding to the end,
it doesn't convert it into a native Rust type.
Instead it normalizes it so that the caller can parse the `&str` into the data type of choice.
We then created a [`DeInteger`] to carry this forward to the `serde` calls so that we could handle any size supported by `serde`.
Users can also directly parse into a [`DeTable`] and get the raw parts of a [`DeInteger`] to handle it themselves.

### `no_std`

If you are parsing in such a constrained environment that you lack [`alloc`],
it is understandable to be in an environment without [`std`].
In fact, [`tomling`](https://crates.io/crates/tomling) was created for this purpose but the author would rather not maintain that.

In designing [`toml_parse`] for [`alloc`],
I found it wasn't too much harder to support [`core`],
so I did so as a challenge to myself.

For resource constrained systems or for general performance,
a caller may not care about parsing the entire document.
We made it so that an [`EventReceiver`] can say "don't care" when recursing into an array or inline table.
We also deferred scalar value validation to access by users, giving them control over which scalars they bother to validate.

In `no_std`, the size of the stack by be different.
Instead of offering built-in recursion protection to `toml_parse`,
it is built on top as an [`EventReceiver`] decorator that callers can control.

All of this is then collected up into `toml` which provides one way of handling all of these decisions.

### `facet`

[facet](https://facet.rs/) is an effort towards reflection in Rust,
including offer parallels to what is supported in serde.
[`facet-toml`](https://crates.io/crates/facet-toml) is currently built on top of `toml_edit`, much like `toml` is.
They were considering writing their own but I encouraged them to wait on this effort ([facet#148]([https://github.com/facet-rs/facet/issues/148)).
Our [`EventReceiver`] design was taking into account that they hope that they can parse TOML directly to the end-users type without an intermediate like [`Document`] or [`DeTable`].
We'll see how that goes.

To encourage sharing and best practices,
we also split out [`toml_write`] to abstract away the lowest level details of writing out TOML.
They still have to deal with quirks of writing out tables correctly.
`toml` was updated to directly translate `serde` into `toml_write`,
without an intermediate like [`DeTable`] or [`Document`],
completely dropping the dependency on `toml_edit`.
Some extra buffers were used to avoid the `TableAfterValue` errors that users regularly saw with `toml` v0.5
([basic-toml#3](https://github.com/dtolnay/basic-toml/issues/3)).

### Reducing the proliferation of TOML parsers

Speaking of alternative TOML parsers, we have:
- `toml_edit` which is focused on format-preserving edits
- `toml` which is focused on `serde` compatibility
- [`basic-toml`](https://crates.io/crates/basic-toml) which is a deprecated, non-conformant fork of `toml` v0.5
- [`toml-span`](https://crates.io/crates/toml-span) which is a fork of `basic-toml` (inheriting its problems) but tracks spans
- [`boml`](https://crates.io/crates/boml), which is a dependency-light, non-conformant independent parser
- [`taplo`](https://github.com/tamasfe/taplo) which is a deprecated, non-conformant, LSP focused parser
- [`tombi`](https://github.com/tombi-toml/tombi) which is focused on being an LSP

*(for statements on conformance, see [toml-test-matrix](https://toml-lang.github.io/toml-test-matrix/))*

Based on the requirements, parser implementations, and lack of a maintainer for `toml`,
[transitioning `toml` to be on top of `toml_edit`]]({{site.base_url}}/blog/2023/01/toml-vs-toml-edit/)
was the right call even if it was heavier than some people preferred.
With where we are at in testing, code structure, and parser implementations,
we can better fulfill some of the needs that led to these other parsers but it took a lot of work to get there.
Granted, we won't be "zero dependencies" but that is a proxy for other care abouts like compile times, binary size, web of trust, etc which were were either already doing well on or have improved with this release.

It might not make sense for everyone to switch to `toml` but maybe we can collaborate on some of the lower level layers in [`toml_parse`], like the parser or even the lexer.

[`DeTable`]: TODO
[`DeInteger`]: TODO
[`DeTable::make_owned`]: TODO
[`Token`]: TODO
[`TokenKind`]: TODO
[`Event`]: TODO
[`ErrorReport`]: TODO
[`EventReceiver`]: TODO
[`ParseError`]: TODO
[`toml_parse`]: https://docs.rs/toml_parse/latest/toml_parse
[`toml_write`]: https://docs.rs/toml_write/latest/toml_write
[`Document`]: https://docs.rs/toml_edit/latest/toml_edit/struct.Document.html
[`Table`]: https://docs.rs/toml_edit/latest/toml_edit/struct.Table.html
[`winnow`]: https://docs.rs/winnow/latest/winnow/
[`Item`]: https://docs.rs/toml_edit/latest/toml_edit/enum.Item.html
[`alloc`]: https://doc.rust-lang.org/alloc
[`std`]: https://doc.rust-lang.org/stable/std/
[`core`]: https://doc.rust-lang.org/core/
