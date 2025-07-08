---
title: "toml v0.9"
published_date: "2025-07-08 11:50:30 -0500"
layout: default.liquid
is_draft: false
data:
  tags:
    - programming
    - rust
---

[toml v0.9](https://github.com/toml-rs/toml/blob/main/crates/toml/CHANGELOG.md#090---2025-07-08)
is a near-complete rewrite with dramatic performance improvements, `no_std` support, and many other improvements.

This is where I've disappeared to over the last 3 months, trying to wrap up a years-old experiment that has been "almost done" for these last 3 months, with use case after use case building up encouraging me to finish this up rather than my regularly scheduled Cargo work.

<!-- more -->

## Goal: performance

The primary goal of this effort was to experiment with a different parser architecture to see what the performance of it would be like.
People are not normally dependent on how long a TOML file takes to parse.
However, Cargo may parse a thousand TOML files on every invocation.
Most people don't notice the impact because compile times can dwarf the parsing of these files.
It's in the small things that this will be noticed, like

- `rustup` adding overhead to every the execution of every Rust toolchain command to parse `rust-toolchain.toml` ([rustup#2626](https://github.com/rust-lang/rustup/issues/2626))
- Higher latency when your shell looks up completions for `cargo` with Cargo's new completion system ([Unstable: native-completions](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#native-completions))
- Cargo's overhead when running a cargo-script ([Unstable: script](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#script))
- Other tools using cargo programmatically ([zulip: `cargo metadata` performance](https://rust-lang.zulipchat.com/#narrow/channel/246057-t-cargo/topic/.60cargo.20metadata.60.20performance/near/476523460))

Previously, I had proposed that `cargo publish` include a `Cargo.json` or `Cargo.cbor` inside of the `*.crate` file,
bypassing the overhead of parsing `Cargo.toml` for the majority of packages.
There was hesitance due to risks associated with having two files for the same content,
one viewed by users and the other viewed by machines.
This was a last ditch effort to prove whether `Cargo.toml` parsing could be sufficiently fast.

Comparing a [Cargo.toml](https://github.com/toml-rs/toml/blob/main/crates/benchmarks/src/Cargo.web-sys.toml) parsing across past, present (bold), future, and theoretical:

| library             | type | baseline | `preserve_order` | `preserve_order,fast_hash` |
|---------------------|----|----------|----------------|--------------------------|
| toml v0.5         | `Table` | 939us | 895us | \- |
| **toml_edit v0.22.27** | `Document` | \- | **542us** | \- |
| toml_edit v0.23.0 | `Document` | \- | 478us | \- |
| toml v0.9.0 | `DeTable<'static>` | 501us | 320us | 304us |
| toml v0.9.0 | `DeTable<'s>` | 398us | 283us | 267us |
| `serde_json` v1.0.140 | `Value` | 259us | 158us | \- |

For other and older views of the data, see
[#891](https://github.com/toml-rs/toml/pull/891),
[#922](https://github.com/toml-rs/toml/pull/922),
[#945](https://github.com/toml-rs/toml/pull/945)
[#956](https://github.com/toml-rs/toml/pull/956)
[#957](https://github.com/toml-rs/toml/pull/957)

The drop in parse times is sufficient to justify the more complex code.
This may even be enough to not need to add support for `Cargo.json`.

<!--

cargo bench --bench 0-cargo -- 0_cargo::serde_json::document::2-features
259us

cargo bench --bench 0-cargo -F preserve_order -- 0_cargo::serde_json::document::2-features
158us

cargo bench --bench 0-cargo -- 0_cargo::toml_v05::manifest::2-features
939us

cargo bench --bench 0-cargo -F preserve_order -- 0_cargo::toml_v05::manifest::2-features
895us


cargo bench --bench 0-cargo -- 0_cargo::toml_edit::document::2-features
0.22.27: 542us
main: 478us

cargo bench --bench 0-cargo -- 0_cargo::toml::detable_owned::2-features
501us

cargo bench --bench 0-cargo -F preserve_order -- 0_cargo::toml::detable_owned::2-features
320us

cargo bench --bench 0-cargo -F preserve_order,fast_hash -- 0_cargo::toml::detable_owned::2-features
304us

cargo bench --bench 0-cargo -- 0_cargo::toml::detable::2-features
398us

cargo bench --bench 0-cargo -F preserve_order -- 0_cargo::toml::detable::2-features
283us

cargo bench --bench 0-cargo -F preserve_order,fast_hash -- 0_cargo::toml::detable::2-features
267us

-->

### Format challenges

While JSON started as a subset of Javascript,
it has a fairly simple syntax and structure and it is well understood how to make it fast due to its heavy use in machine-to-machine communication.
On the other hand, TOML is more heavily oriented towards humans with less care for the impact on performance.

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
    "a": {},
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

[toml] v0.5 solved this problem by parsing tokens into an AST (a `Vec<Table>` where `Table` is not recursive) and generating lookup tables to make it easier to walk the logical structure among the syntactic structure.

[toml] v0.6 [delegated parsing to toml_edit]({{site.base_url}}/blog/2023/01/toml-vs-toml-edit/) which
parses directly into a [`Document`] representing the logical structure (a recursive [`Table`]).
[toml_edit] did two lookups for every table for simplicity of matching TOML's rules for merging of syntactic tables.
When starting to parse a syntactic table,
we remove the [`Table`] and directly insert into it.
When we find the next table header,
we then re-insert the [`Table`] we were just editing.

[toml] v0.9 continues with the approach of [toml_edit] by providing a logical [`DeTable`] that it parses into.

### Cargo requirements

Originally, `cargo` had been using [toml] v0.5 to parse `Cargo.toml` files.

This changed with the introduction of `cargo add` which used [toml_edit] for its ability to make changes to `Cargo.toml` without losing the users formatting.
When proposing to merge `cargo add`, the Cargo team was concerned about having two different TOML parsers inside of Cargo which may have slight incompatibilities and different errors.
I had proposed that we solve this problem by having one parser, [toml_edit], and extending it to support [toml]s API.
This brought on the requirement of "no loss in performance".

[toml_edit] was then relied on to parse `--config key=false` TOML expressions as it made it easy to validate that only a single key-value expression was present and not a full document, including comments and newlines.

Lately, the Cargo team is experimenting with using the richer information reported from [toml_edit] to provide rustc-like diagnostics for `Cargo.toml`.

To summarize, the traits we need to preserve while making [toml] faster include:
- Consistent parsing behavior between `serde`-like reading and writing of TOML and format-preserving edits of user TOML files
- Validation of `key=value` expressions
- Spans for keys and values

**To ensure consistent behavior,**
we cleaned up the tests in [toml] and [toml_edit],
isolating crate-specifics,
and synced them into each other.
This puts a burden on us to ensure that any changes to [toml] or [toml_edit]s shared tests get synced back to the other.

As a second line of defence, parsing of the AST is shared in the [toml_parser] crate.

As a third-line of defence, the conversion from the AST to the logical structure and the `serde` implementations are maintained in such a way that we can diff the implementations.
However, this is noisy and requires us to remember to sync even changes, much like the tests.

**Validating of `key=value` expressions can still be done with toml_edit.**

**As for providing Spans,**
we leveraged the fact that we need to provide a logical [`DeTable`] for `serde` and included Spans in it (i.e. `Spanned<DeTable>`).

Like [`toml_edit::Document`][`Document`], this requires [`DeTable`] to be an owned type so that `cargo` can keep it around until a diagnostic needs it.
However, it should be rare that `cargo` needs spans for dependencies.
To prepare for this possibility, we made [`DeTable<'s>`][`DeTable`] track all strings using
[`Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) to avoid allocations when escaping isn't used.
To make this owned, you can call [`DeTable::make_owned`] which switches all `Cow::Borrowed`s to `Cow::Owned`.
At that point, it is safe to
[transmute](https://doc.rust-lang.org/core/mem/fn.transmute.html)
a [`DeTable<'s>`][`DeTable`] to a [`DeTable<'static>`][`DeTable`] as lifetimes do not affect layout (or so I'm told).

### Implementation

Internally, [toml] v0.5 has a fallible tokenizer that iterated through `char`s in a `&str`, requiring UTF-8 decoding logic in the inner loop.
As mentioned before, it then parsed into a `Vec<Table>` (non-recursive) and created lookup tables to speed up access.

[toml] v0.6 (via [toml_edit]) directly parsed bytes, avoiding the UTF-8 decoding, directly into a [`Table`] (recursive).
For parsing bytes, it helps that TOML syntax only distinguishes 7-bit ASCII characters from each other and UTF-8 is left opaque.
`unsafe` was heavily used to avoid re-validating that the `&[u8]`s returned were actually `&str`.
The parser is written using parser combinators via [winnow].
This made it very easy to compare the implementation to the
[grammar](https://github.com/toml-lang/toml/blob/1.0.0/toml.abnf)
and got fairly good performance results compared to the old parser.
However, this required backtracking and carrying state in `Result`s which likely helped spill return values over from registers to the stack
(see [winnow#72](https://github.com/winnow-rs/winnow/issues/72)).
To preserve formatting,
the size of an [`Item`] returned by a parser is also quite large.
By doing direct parsing, we had deeper call stacks for this all to accumulate in and higher costs, both in terms of the call stack and parsed result, when backtracking.

[toml] v0.9 (via [toml_parser]) has an infallible [tokenizer][`Lexer`] that we collect into a [`Vec<Token>`][`Token`],
assuming the capacity based on the length of the `&str` we are tokenizing.
The tokenizer processes bytes within each individual step to calculate indices to take a sub-slice of a `&str`.
We made `unsafe` optional in toml_parser to avoid the UTF-8 boundary checks but found the overhead of the safe option to be low enough that we didn't use it.
We then report parse events and errors through callbacks,
with [toml] collecting them into a [`Vec<Event>`][`Event`] with the capacity inferred from [`Vec<Token>`][`Token`]s length.
In a couple of places (table headers, scalar values),
the parser does do some lookahead but only needs to compare [`TokenKind`]s and not iterate through a sequence of bytes.
The parser only looks at [`TokenKind`]s and not the `&str`, meaning that scalar values are not validated at this stage.
All structural syntactic errors are expected to be reported at this stage.
[toml] walks these events to build up a [`DeTable`], validating the logical structure and decoding and validating scalars along the way.

rustc's tokenizer tries to minimize the size of a token by only keeping the token length and storing it in a `u32`.
We were unsure how much value this would provide [toml] and didn't experiment with this.

## Secondary goals

While the focus was on performance for Cargo,
the architecture we were going with provided an opportunity to help with areas of interest.

### Error recovery

Like `rustc`, we wanted to allow [toml] to report as many errors as possible, not just the first.

The core of error recovery is that all errors reported through a [`&mut dyn ErrorSink`][`ErrorSink`] parameter.
The caller can choose whether they collected into a `()`, [`Option<ParseError>`][`ParseError`], [`Vec<ParseError>`][`ParseError`], or something else entirely.
This was another important reason to avoid backtracking because we didn't want to have to revert errors that had been reported.

To keep the logic in [toml] as simple as possible,
we also made the parser create synthetic [`Event`]s wherever possible
(e.g. reporting a [closed table header](https://docs.rs/toml_parser/latest/toml_parser/parser/trait.EventReceiver.html#method.std_table_close) with an empty span when a `]` was missing).

Currently, we only expose error recovery in [toml] through [`DeTable::parse_recoverable`].

### Bignum support

While I figure an `i64` is good enough for any config file,
that didn't appear to be the case.
While we previously parsed integers into an `i64`, [toml] now tracks [a validated, normalized string][`DeInteger`] that it can later parse into the type requested via `serde`.
Users can also directly parse into a [`DeTable`] and get the raw parts of a [`DeInteger`] to handle it themselves.

### `no_std`

While TOML is unlikely a good format for systems so constrained that they lack [alloc],
it could still work for systems that lack [std].
In fact, [`tomling`](https://crates.io/crates/tomling) was created for this purpose but the author would rather not maintain that.

In designing [toml_parser] for [alloc],
I found it wasn't too much harder to support [core],
so I did so as a challenge to myself.

For resource constrained systems or for general performance,
a caller may not care about parsing the entire document.
We made it so that an [`EventReceiver`] can say "don't care" when recursing into an array or inline table.
We also deferred scalar value validation to access by users, giving them control over which scalars they bother to validate.

In `no_std` environments, the size of the stack may further constrained.
Instead of offering built-in recursion protection to toml_parser,
it is built on top as an [`EventReceiver`] decorator that callers can control.

All of this is then collected up into [toml] which provides one way of handling all of these decisions.

### facet

[facet](https://facet.rs/) is an effort towards reflection in Rust,
including offering parallels to what is supported in serde.
[facet-toml](https://crates.io/crates/facet-toml) is currently built on top of [toml_edit], much like [toml] is.
They were considering writing their own but I encouraged them to wait on this effort ([facet#148](https://github.com/facet-rs/facet/issues/148)).
Our [`EventReceiver`] design was taking into account that they hope that they can parse TOML directly to the end-users type without an intermediate like [`Document`] or [`DeTable`].
We'll see how that goes.

To encourage sharing and best practices,
we also split out [toml_writer] to abstract away the lowest level details of writing out TOML.
They still have to deal with quirks of writing out tables correctly.
[toml] was updated to directly translate `serde` into toml_writer,
without an intermediate like [`DeTable`] or [`Document`],
completely dropping the dependency on [toml_edit].
Some extra buffers were used to avoid the `TableAfterValue` errors that users regularly saw with [toml] v0.5
([basic-toml#3](https://github.com/dtolnay/basic-toml/issues/3)).

### Reducing the proliferation of TOML parsers

Speaking of alternative TOML parsers, we have:
- [toml_edit] which is focused on format-preserving edits
- [toml] which is focused on `serde` compatibility
- [basic-toml](https://crates.io/crates/basic-toml) which is a deprecated, non-conformant fork of toml v0.5
- [toml-span](https://crates.io/crates/toml-span) which is a fork of `basic-toml` (inheriting its problems) but tracks spans
- [boml](https://crates.io/crates/boml), which is a dependency-light, non-conformant independent parser
- [taplo](https://github.com/tamasfe/taplo) which is a deprecated, non-conformant, LSP focused parser
- [tombi](https://github.com/tombi-toml/tombi) which is focused on being an LSP

*(for statements on conformance, see [toml-test-matrix](https://toml-lang.github.io/toml-test-matrix/))*

Based on the requirements, parser implementations, and lack of a maintainer for toml,
[transitioning toml to be on top of toml_edit]({{site.base_url}}/blog/2023/01/toml-vs-toml-edit/)
was the right call even if it was heavier than some people preferred.
With where we are at in testing, code structure, and parser implementations,
we can better fulfill some of the needs that led to these other parsers but it took a lot of work to get there.
Granted, we won't be "zero dependencies" but that is a proxy for other care abouts like compile times, binary size, web of trust, etc which were were either already doing well on or have improved with this release.

It might not make sense for everyone to switch to toml but maybe we can collaborate on some of the lower level layers in [toml_parser], like the parser or even the lexer.

## On parser combinators

In reflecting back on this rewriting a parser combinator-based parser into a hand written parser,
I still think the default choice should be parser combinators
([source](https://docs.rs/winnow/latest/winnow/_topic/why/index.html#hand-written-parsers)).
With parser combinators, there was generally a 1:1 mapping between code and  the
[ABNF grammar](https://github.com/toml-lang/toml/blob/1.0.0/toml.abnf).
Now, that relationship is less clear.
Each section of code is fuzzy in how it maps to the grammar and implements a subset of it,
either assuming a layer above or below takes care of it or there is not-quite duplicated code within the same layer that handles it.
Maintaining this in the future will be quite a burden, requiring knowledge of how the parts interact to make sure all of the pieces get updated correctly.

In this case, it was worth it because of user-facing goals it met.
In most cases, including [toml] in the past, the most important goal is being able to delivery something.

[`DeTable`]: https://docs.rs/toml/latest/toml/de/type.DeTable.html
[`DeInteger`]: https://docs.rs/toml/latest/toml/de/struct.DeInteger.html
[`DeTable::make_owned`]: https://docs.rs/toml/latest/toml/de/type.DeTable.html#method.make_owned
[`DeTable::parse_recoverable`]: https://docs.rs/toml/latest/toml/de/type.DeTable.html#method.parse_recoverable
[`Token`]: https://docs.rs/toml_parser/latest/toml_parser/lexer/struct.Token.html
[`TokenKind`]: https://docs.rs/toml_parser/latest/toml_parser/lexer/enum.TokenKind.html
[`Event`]: https://docs.rs/toml_parser/latest/toml_parser/parser/struct.Event.html
[`ErrorSink`]: https://docs.rs/toml_parser/latest/toml_parser/trait.ErrorSink.html
[`EventReceiver`]: https://docs.rs/toml_parser/latest/toml_parser/parser/trait.EventReceiver.html
[`ParseError`]: https://docs.rs/toml_parser/latest/toml_parser/struct.ParseError.html
[toml_parser]: https://docs.rs/toml_parser/latest/toml_parser
[toml_writer]: https://docs.rs/toml_writer/latest/toml_writer
[`Document`]: https://docs.rs/toml_edit/latest/toml_edit/struct.Document.html
[`Lexer`]: https://docs.rs/toml_parser/latest/toml_parser/lexer/struct.Lexer.html
[`Table`]: https://docs.rs/toml/latest/toml/type.Table.html
[winnow]: https://docs.rs/winnow/latest/winnow/
[toml_edit]: https://docs.rs/toml_edit/latest/toml_edit/
[toml]: https://docs.rs/toml/latest/toml/
[`Item`]: https://docs.rs/toml_edit/latest/toml_edit/enum.Item.html
[alloc]: https://doc.rust-lang.org/alloc
[std]: https://doc.rust-lang.org/stable/std/
[core]: https://doc.rust-lang.org/core/
