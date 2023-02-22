---
title: '`toml` vs `toml_edit`'
published_date: 2023-01-23 20:09:26 +0000
layout: default.liquid
is_draft: false
data:
  tags:
    - programming
    - rust
---
[`toml` v0.6](https://github.com/toml-rs/toml/blob/main/crates/toml/CHANGELOG.md#060---2023-01-23)
is now out with a new parser and renderer, addressing several existing issues
and ensuring compliance with the TOML 1.0 compliance tests.  This was done by
leveraging the
[`toml_edit`](https://crates.io/crates/toml_edit) crate.

<!-- more -->

## Why `toml_edit`

Last March, alexcrichton put out [a call for a maintainer](https://github.com/toml-rs/toml-rs/issues/460) for the
[`toml`](https://crates.io/crates/toml) crate.  I had become a maintainer of
the `toml_edit` crate as part of my work
on `cargo add`, getting `toml_edit` in shape that it became the only TOML parser in
cargo (see
[rust-lang/cargo#10086](https://github.com/rust-lang/cargo/pull/10086)) as the
cargo team wanted consistent parse behavior.  I offered to take over
maintenance of `toml` with goal of migrating `toml` onto `toml_edit`.

`toml` fills a similar role as `serde_json`, providing serde support for the
[TOML format](https://toml.io/en/) and a default data structure to deserialize
into.  `toml_edit` is more complex and slower because it needs to preserve all
end-user formatting from parse to display.  As part of the cargo work, we got
[`toml_edit` a lot closer to `toml` in performance](/blog/2021/09/optimizing-toml-edit/)
and offered the `easy` module as a `toml` compatibility layer.

In *theory*, we could just migrate everyone from `toml` to `toml_edit` and be
done.

In *practice*, there is no way to help people through a crate rename.
With `structopt` being absorbed into `clap` in 2021, we are still seeing people
use `structopt` unaware of the change over.  Additionally, there was interest
in a more stable API than what `toml_edit` offers as we had a lot of churn due
to the low level details in the API and as we figure out how best to allow
editing of TOML documents.

So keeping the `toml` crate around was worthwhile and we could lighten the
community's overall maintenance burden by combining efforts and code.

## Impact on `toml` users

`toml` now passes all [compliance tests](https://github.com/BurntSushi/toml-test) for TOML 1.0, including
- [Not erroring when a table appends to dotted keys](https://github.com/toml-rs/toml/issues/439)
- [No more stray `,` when writing arrays of tables](https://github.com/toml-rs/toml/issues/483)
- [No more `ValueAfterTable` errors when writing top-level key-value pairs](https://github.com/toml-rs/toml/issues/403), requiring users to [opt-in to a fix](https://docs.rs/toml/0.5.11/toml/ser/fn.tables_last.html)

Error information also improved, most notably the error messages are changing from the old
```
invalid type: string "a", expected isize\nin `foo.bar`
```
to `toml_edit`'s
```
TOML parse error at line 2, column 7
  |
2 | bar = "a"
  |       ^^^
invalid type: string "a", expected isize
```

Callers can also render the errors as they wish, like with
[ariadne](https://crates.io/crates/ariadne).
We also improved the quality of the span information being reported.

Pulling in `toml_edit` also helped highlight some issues with `toml`s API.
[`toml::Value`](https://docs.rs/toml/latest/toml/value/enum.Value.html) would
allow you to `Display` anything.  If it looked like a document, it would be
rendered as such.  Otherwise, it would be rendered as a value.  This dynamic API
makes it easy to get things wrong.  Instead, `toml::Value` always `Display`s a
TOML value while `toml::Table` will `Display` a TOML document.  Similar for
parse.  A concrete example of what this allows is for
[unambiguously parsing `["a", "b"]` as either document or a value](https://github.com/toml-rs/toml/issues/363).

Users should also expect maintenance going forward to improve as the code base
is easier to support and not just because of the two-for-one maintenance.
Before, `toml` had a handwritten parser that had to deal the non-linear nature
of TOML.  Now, `toml` parses everything to an AST and then deserialzies to the
end-users data types.  Separating the steps of parsing and deserialize
simplifies them, making it easier to confidently make changes.  The parser is
also easier to update as it is higher level, using a parser combinator crate.

In rough terms, we expect compiles to be slightly slower as more code across
more dependencies is being built.  Parse time is also about twice what it
was before (62us to 110us for cargo's `Cargo.toml` on my machine).  We do have
ideas on how to further improve parse times.  We do not track `Display` time
for TOML, assuming it isn't in a critical path.

## Impact on `toml_edit` users

As already mentioned, `toml_edit` users now have a more stable subset of the API that shares behavior and compile-time.

Otherwise, the biggest gain is span support.  We now track the location of each
`Item` within the original document while parsing.  This allows you to
deserialize to
[`serde_spanned::Spanned<T>`](https://docs.rs/serde_spanned/latest/serde_spanned/struct.Spanned.html)
to capture that location.  Deserialize errors will now look more like parse
errors, showing the error location, and allow you to lookup the span
programmatically.

Unfortunately, span information is only exposed through serde and errors at
this time.  To [maintain performance](https://github.com/toml-rs/toml/pull/429),
we only capture spans while parsing rather than Strings for all of the
format-preserving information, avoiding allocations for serde support.
`Document::from_str` has to replace those spans with strings to allow editing.
We don't keep the spans around for the editing API to keep the size of each
`Item` in memory smaller.  The spans also present a lot of challenges in an
editing API as we can't guarantee what source they are associated with.

Also, a lot of small pieces of polish were found through `toml`s tests, e.g.
inconsistent casing and newlines in errors.

We did speed up `toml_edit::de`, with parsing cargo's `Cargo.toml` going from
115us to 111us.  However, `Document::from_str` slowed down, going from 85us to
103us.  We have hopes to recover the performance loss.

Long term, the `toml_edit::easy` API is going to go away in favor of `toml`.
