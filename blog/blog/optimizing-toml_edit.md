---
title: A Journey in Optimizing `toml_edit`
published_date: "2021-09-30 09:00:30 -0500"
data:
  tags:
    - programming
    - oss
    - rust
---

##  tl;dr

[`toml_edit`](https://docs.rs/toml_edit) is a format preserving TOML crate,
allowing users to modify `.toml` files.

**Before:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml_edit       | 8.7us                   | 271us              |
| toml_edit::easy | 20.7us                  | 634us              |
| toml-rs         | 4.7us                   | 121us              |

**After:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml_edit       | 4.0us                   | 149us              |
| toml_edit::easy | 5.0us                   | 179us              |

**Target:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml-rs         | 4.7us                   | 121us              |

<!-- more -->

*Sorry for the formatting, I need to update my blog's theme*

## Goal

I am working on
[merging `cargo-add` into `cargo`](https://github.com/rust-lang/cargo/issues/5586).
Concerns raised as part of this:
- Preserving formatting when modifying a users `Cargo.toml` is required
- Consistent TOML behavior, including errors, between the parts of Cargo
  modifying TOMLs and the parts that don't need to

Rather than updating `toml_edit` to make it consistent with
[`toml-rs`](https://docs.rs/toml) (what Cargo uses today) and then keeping it
consistent over time, I proposed we see if we can make `toml_edit` serve all of
Cargo's needs.

Since then
- I've made `toml_edit` support the [TOML 1.0 spec](https://toml.io/en/),
  including passing the [unofficial compliance suite](https://github.com/BurntSushi/toml-test)
- Ported the `toml-rs` API on top of `toml_edit` as
  [`toml_edit::easy`](https://docs.rs/toml_edit/0.5.0/toml_edit/easy/index.html)
- [Removed support for invalid TOML files from Cargo](https://github.com/rust-lang/cargo/pull/9932)
  to avoid having to support them in `toml_edit`

While there is still
[some work to go](https://github.com/ordian/toml_edit/issues/133), we are
nearly ready for trying `toml_edit` in Cargo except one thing: performance.

## The Journey

### Know Your Target

When doing a drop-in replacement, it can be easy to assume the target is to
maintain performance but you can then spend a lot of time on something that
nobody will care about.  Unsure how much performance I could squeeze out of
`toml_edit`, I especially wanted to know what my leeway was.  Alex Crichton
was a big help in identifying our target, calling out specifically the Cargo's
resolver, the code that walks the entire dependency hierarchy of crates, as
being a user-impacting bottleneck, with `toml-rs` showing up when profiling.
So not only does it need to be at least as fast, they want to optimize it
further.

Ideally, I'd go and create
[a resolver benchmark](https://github.com/rust-lang/cargo/issues/9935) and
measure before/after to see how much of a real-world impact changes in TOML
parsing performance would be.  I decided to first see what low hanging fruit
existed in `toml_edit` to see if I could get it close to `toml-rs`.

### Choose Your Equipment

To verify my optimizations, I wrote up some benchmarks.  I used
[criterion](https://bheisler.github.io/criterion.rs/book/index.html) for the
feature set and being available on stable Rust.

We already know parsing is our main care about.  What should we parse?  I
generally like to see how my code scales over sample sizes since that
highlights different performance problems.  Since we were mainly focusing on
Cargo's resolver, I figured we'd want to focus on typical `Cargo.toml` files.
For a small file, I selected the output of `cargo init`.  For a larger file, I
chose Cargo's `Cargo.toml` file.

I relied on the
[performance book to help get started profiling](https://nnethercote.github.io/perf-book/profiling.html).
I chose `callgrind` with `kcachegrind` as my visualizer because I had been
meaning to try it out.

I broke out some of my benchmarks as `examples/` for easy profiling.  Maybe there is a way around this with criterion, \*shrug\*.

### The Downside to Format-Preserving

When parsing `key = 3.0e3  # hello!`, we need to break this down to:
- Key
  - The exact formatting used (e.g. `key` vs `"key"`)
  - A format-agnostic version for lookups
  - The trailing `" "`
- Value
  - The exact formatting used (e.g. `3.0e3` vs `3000.0`)
  - A format-agnostic version for processing
  - The leading `" "`
  - The trailing `" # hello!"`

We also duplicate some of these for user convenience in working with the API.

Compare this to `toml-rs` which only has to deal with `"key"` and `3.0e3`.

This puts us at a severe disadvantage due to the number of allocations we make.
Most keys are relatively short and people don't tend to use a lot of
whitespace.  We can use a technique called small-string optimization which
looks roughly like:
```rust
enum String {
    Inline([u8; 24]),
    Normal(String),
}
```
This removes the need for allocations and we avoid an indirection to reference the allocation.

I chose [`kstring`](https://docs.rs/kstring) over
- [`smartstring`](https://docs.rs/smartstring) because we rarely need mutability and the assumptions it
   makes scares me from using it in anything but a top-level `[[bin]]`
- [`compact_str`](https://docs.rs/compact_str) didn't seem to have benchmarks last I looked
- [`smol_str`](https://docs.rs/smol_str) last benchmarks I ran, it looked to be slower iirc

By default, `kstring`
- Inline strings store 15 bytes (16 bytes total with size)
  - `max_inline` feature switches to storing 22-bytes (23 bytes total) which can be slower for smaller strings
- Stores the heap string as `Box<str>`
  - `arc` feature instead uses `Arc<str>` which replaces the cost of allocations with the cost of ref-counting.

When benchmarking different combinations, I found `max_inline` by itself gave
the best results for our use cases.

I replaced all strings with newtype wrapper around `KString` (so we could swap
it out in the future) except the one in TOML values because
- Without munging up the grammar quite a bit, we need
  mutable strings to build up values or else we'd just be creating a `String`
  to convert it to a `KString`.
- Values are more in the user's face, so it felt more ergonomic to give them a familiar type.
- Allocations from values didn't seem to be too big of a deal in benchmarks

In the end, this dropped our larger benchmark from 250us to 203us.

*See the [PR](https://github.com/ordian/toml_edit/pull/211)*

### The Price of Convenience with Parsers

Parser combinators, like [nom](https://docs.rs/nom) or
[combine](https://docs.rs/combine), make it easy to convert a
[grammar](https://github.com/toml-lang/toml/blob/master/toml.abnf) to code but
can easily hide a lot of costs.

#### Unnecessary Allocations

The first is in allocations.  Say you want to parse a number, it could be easy to write
```rust
fn number(input: &str) -> IResult<&str, u32> {
    map_res(
        many0(digit),
        |c: Vec<char>| String::from_iter(c).parse(),
    )(input)
}
```
We are
- Allocating all `char`s into a `Vec`
- Allocating a `String` from the `Vec<char>` for using `parse`

While this example is artificial, cases like this existed in `toml_edit` and I
had to find them and replace them with operations like:
```rust
fn number(input: &str) -> IResult<&str, u32> {
    map_res(
        recognize(many0_count(digit)),
        |s: &str| s.parse(),
    )
    )(input)
}
```
This uses `many0_count` as a way of repeating a parser, without allocating, and
then uses `recognize` to take the parsed region and turn it into a `&str`.  No
allocations!

#### Lack of Batching

Sometimes, you really do need that allocation, like parsing a string where you
need to convert escape sequences (like `"\t"`) to their actual value.  For
example, a (simplified) TOML grammar might look like:
```rust
fn string_content(input: &str) -> IResult<&str, String> {
    map(
        many0(string_char),
        |c: Vec<char>| String::from_iter(c),
    )(input)
}

fn string_char(input: &str) -> IResult<&str, char> {
    alt((
        string_unescaped,
        string_escaped,
    ))(input)
}

fn string_unescaped(input: &str) -> IResult<&str, char> {
    ...
}

fn string_escaped(input: &str) -> IResult<&str, char> {
    ...
}
```
Rather than adding a `char` at a time to a `String`, we can do it in batches,
reducing the number of allocations and reducing the size of the hot loop:
```rust
fn string_content(input: &str) -> IResult<&str, String> {
    map(
        many0(
            alt((
                map(recognize(many1_count(string_char)), |s| String::from(s)),
                map(escaped, |c| String::from(c))
            ))
        ),
        |c: Vec<String>| c.join(""),
    )(input)
}

fn string_unescaped(input: &str) -> IResult<&str, char> {
    ...
}

fn string_escaped(input: &str) -> IResult<&str, char> {
    ...
}
```
This optimizes the common case (characters) at the cost of an uncommon case
(escape sequences).

It looks like this dropped parse time by about 2%.  Not as much of a win as I was hoping.

*See a [PR](https://github.com/ordian/toml_edit/pull/209)*

#### Bytes

Even though the parser receives a `&str` and gives out a `&str`, the inner loop
needs to decode the bytes into `char` to process the grammar's tokens.  Let's
see about bypassing that by parsing bytes instead of `char`.

All the horror stories of people treating UTF-8 as bytes had me cautious about
going down this route.  I first had to examine
[UTF-8](https://en.wikipedia.org/wiki/UTF-8#Encoding) to make sure we could
distinguish ASCII grammar rules from a user's UTF-8 text.  Thankfully, the left
most bit can ensures no ASCII byte can be mistaken for a later byte in a
multi-byte character.  It also looks like the grammar will never have us split
a multi-byte character, so we shouldn't corrupt already-valid UTF-8.

Ensuring valid UTF-8 can be costly though.  By default, we'd use
`std::from_utf8` to convert these bytes back to strings.  This will have to
verify the string is valid UTF-8.  Can we bypass this?

Most elements of TOML's gramamr are specify the exact characters supported
like numbers and dates.  In these cases, we know the validation isn't required.

So we can take
```rust
fn number(input: &str) -> IResult<&str, u32> {
    map_res(
        recognize(many0_count(digit)),
        |s: &str| s.parse(),
    )
    )(input)
}
```
and speed it up by switching to:
```rust
fn number(input: &[u8]) -> IResult<&[u8], u32> {
    map_res(
        recognize(many0_count(digit)),
        |b: &[u8]| {
            let s = unsafe { from_utf8_unchecked(b, "`digit` filters out non-ASCII") };
            s.parse(),
        }
    )
    )(input)
}

pub(crate) unsafe fn from_utf8_unchecked<'b>(
    bytes: &'b [u8],
    safety_justification: &'static str,
) -> &'b str {
    if cfg!(debug_assertions) {
        // Catch problems more quickly when testing
        std::str::from_utf8(bytes).expect(safety_justification)
    } else {
        std::str::from_utf8_unchecked(bytes)
    }
}
```

In switching to byte parsing, this still leaves us calling
`std::str::from_utf8` for comments and quoted strings.  If we only allow
end-users to pass in `&str`, we know the original string was valid.  Our
grammar *should* make sure we never split a multi-byte character.  We should
be able to get away with `from_utf8_unchecked` everywhere.

This seems iffy to me
- These `from_utf8_unchecked` calls would have a hidden dependence on the
  end-user API and we could add parsing of `&[u8]` without knowing we need to
  update code or where all to update.
- They also have hidden dependence on the grammar and its correctness to ensure
  no multi-byte character is split.

I checked in a profiler and `std::str::from_utf8` was only about 1% of the
instruction count, so I decided it wasn't worth it at this time.

This dropped our benchmarked parse times by 6-12%!

*See the [PR](https://github.com/ordian/toml_edit/pull/219)*

#### The Cost of a Good Error

While all of my examples have been using `nom`, `toml_edit` actually uses
`combine` (I'm just more familiar with `nom` and prefer the API).

One feature of `combine` is built-in support for creating great errors.  For example, when parsing a TOML value, the code might look like
```rust
// val = string / boolean / array / inline-table / date-time / float / integer
parse!(value() -> v::Value, {
    recognize_with_value(choice((
        string()
            .map(|s|
                v::Value::String(Formatted::new(
                    s,
                ))
            ),
        boolean()
            .map(v::Value::from),
        array()
            .map(v::Value::Array),
        inline_table()
            .map(v::Value::InlineTable),
        date_time()
            .map(v::Value::from),
        float()
            .map(v::Value::from),
        integer()
            .map(v::Value::from),
    ))).map(|(raw, value)| apply_raw(value, raw))
});
```
Say you have a syntax error, like
```toml
key = invalid-value
```
`combine` will check each `choice` and merge the errors.  It gives preference
to the error that occurs earliest in the string and combines errors that happen
at the same point.  For `invalid-value`, every value-type parser will fail at
the `i` and the error will list out what character is instead expected from
each type being parsed.

The problem with this is logic for your error case is now costing you even when no
user-facing errors happen as it falls through of the different
`choice`s.  `callgrind` said that `Errors::merge` was costing around 12% of
our total instruction count!

Our options around this were:
- Dropping `combine`, whether to use `nom` or to hand-write a parser
- Implementing our own error handling for `combine`, tweaking the errors until they are good enough
- Use a little documented macro `dispatch!` that acts more like a `match`
  statement than a series of `if-else`s, speeding up our initial choice and
  removing the merging of errors, at the risk of negatively impacting our
  errors and requiring duplicating logic from a grammar rule in its caller.

A holistic solution like dropping `combine` or changing the error handling
would offer the most performance (hitting every case) while `dispatch!` would
require playing whack-a-mole and could easily regress in the future.

With that said, I went ahead with `dispatch!` because it was quick to try:
```rust
// val = string / boolean / array / inline-table / date-time / float / integer
parse!(value() -> v::Value, {
    recognize_with_value(look_ahead(any()).then(|e| {
        dispatch!(e;
            crate::parser::strings::QUOTATION_MARK |
            crate::parser::strings::APOSTROPHE => string().map(|s| {
                v::Value::String(Formatted::new(
                    s,
                ))
            }),
            crate::parser::array::ARRAY_OPEN => array().map(v::Value::Array),
            crate::parser::inline_table::INLINE_TABLE_OPEN => inline_table().map(v::Value::InlineTable),
            // Uncommon enough not to be worth optimizing at this time
            _ => choice((
                boolean()
                    .map(v::Value::from),
                date_time()
                    .map(v::Value::from),
                float()
                    .map(v::Value::from),
                integer()
                    .map(v::Value::from),
            )),
        )
    })).and_then(|(raw, value)| apply_raw(value, raw))
});
```

Sprinkling a few `dispatch!`s through the code took our larger benchmark case
from 182us to 160us!  

*See the [PR](https://github.com/ordian/toml_edit/pull/222)*

### Hidden Costs in `serde`

While `toml_edit`s performance was now fairly close, Cargo will most care about
the parts that are a drop-in replacement for `toml-rs` and `toml_edit::easy`
was 3x slower than both `toml-rs` and `toml_edit`!

Like `toml-rs`, the layer above `toml_edit::Document` is a `serde` API followed by a `Value` type.

#### Deserialization Ambiguity

When profiling `toml_edit::easy`, `toml_edit::Datetime` was showing up despite the sample code not using it!  
When I first wrote `toml_edit::easy::de`, I serialized `toml_edit::Datetime` as
a string.  This requires serde to parse each string, to see if it might be a
date.  Files without a `Datetime` must pay the price for it existing.  On top
of that, we might misinterpret a user's string as a `Datetime`.

I decided to copy `toml-rs` and deserialize using a proprietary format sort-of like:
```rust
struct DatetimeSerde {
    field: String,
}
```

This cut our parse time from 602us to 484us!

*See the [PR](https://github.com/ordian/toml_edit/pull/226)*

#### Untagged Enums

When profiling, a lot of the remaining time was in error reporting, including
formatting of errors and allocations, particularly for `Value`.  This was
strange, because no user-facing errors were occurring.  Deja vu.

I decided to inspect the code that `#[derive(serde::Deserialize)]` was generating with `cargo expand`:
```rust
#[automatically_derived]
impl <'de> _serde::Deserialize<'de> for Value {
    fn deserialize<__D>(__deserializer: __D)
     -> _serde::__private::Result<Self, __D::Error> where
     __D: _serde::Deserializer<'de> {
        let __content =
            match <_serde::__private::de::Content as
                      _serde::Deserialize>::deserialize(__deserializer)
                {
                _serde::__private::Ok(__val) => __val,
                _serde::__private::Err(__err) => {
                    return _serde::__private::Err(__err);
                }
            };
        if let _serde::__private::Ok(__ok) =
               _serde::__private::Result::map(<i64 as
                                                  _serde::Deserialize>::deserialize(_serde::__private::de::ContentRefDeserializer::<__D::Error>::new(&__content)),
                                              Value::Integer)
           {
            return _serde::__private::Ok(__ok);
        }
        if let _serde::__private::Ok(__ok) =
               _serde::__private::Result::map(<f64 as
                                                  _serde::Deserialize>::deserialize(_serde::__private::de::ContentRefDeserializer::<__D::Error>::new(&__content)),
                                              Value::Float) {
            return _serde::__private::Ok(__ok);
        }
	// ...
        if let _serde::__private::Ok(__ok) =
               _serde::__private::Result::map(<Table as
                                                  _serde::Deserialize>::deserialize(_serde::__private::de::ContentRefDeserializer::<__D::Error>::new(&__content)),
                                              Value::Table) {
            return _serde::__private::Ok(__ok);
        }
        _serde::__private::Err(_serde::de::Error::custom("data did not match any variant of untagged enum Value"))
    }
}
```
It is trying to deserialize each variant of the `enum` one after the other.
Most values are `String`, `Table`, and `Array`, so it has to try `Integer`,
`Float`, `Boolean`, and `Datetime` first.  For each error, it calls:
```rust
#[cold]
fn invalid_type(unexp: Unexpected, exp: &Expected) -> Self {
    Error::custom(format_args!("invalid type: {}, expected {}", unexp, exp))
}
```

Since nearly every variant has a unique type, we can use serde's data model to dispatch for us:
```rust
impl<'de> serde::de::Deserialize<'de> for Value {
    fn deserialize<D>(deserializer: D) -> Result<Value, D::Error>
    where
        D: serde::de::Deserializer<'de>,
    {
        struct ValueVisitor;

        impl<'de> serde::de::Visitor<'de> for ValueVisitor {
            type Value = Value;

            fn expecting(&self, formatter: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                formatter.write_str("any valid TOML value")
            }

            fn visit_bool<E>(self, value: bool) -> Result<Value, E> {
                Ok(Value::Boolean(value))
            }

            fn visit_i64<E>(self, value: i64) -> Result<Value, E> {
                Ok(Value::Integer(value))
            }

            // ...
        }

        deserializer.deserialize_any(ValueVisitor)
    }
}
```

I feel like this happens enough, it'd be nice if `serde` offered an attribute
that let you tell it what type to deserialize as so it has smaller `if-else`
ladders

This dropped our benchmark from 484us to 181us!

*See the [PR](https://github.com/ordian/toml_edit/pull/227)*

### What Failed?

Not all fixes are successful.  Unfortunately, failed attempts are usually not
as well documented because they might not make it to being documented in a PR.

Some that I remember:
- I already covered the Batching which had smaller gains than I had hoped.
- I tried a couple more ideas to further optimize `KString`, but they instead slowed it down.
- I did attempt a `nom` port but decided the code illegibility wasn't worth it,
  though I am [working to improve that](https://github.com/Geal/nom/issues/1408)

### Re-cap of the Results

**Before:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml_edit       | 8.7us                   | 271us              |
| toml_edit::easy | 20.7us                  | 634us              |
| toml-rs         | 4.7us                   | 121us              |

**After:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml_edit       | 4.0us                   | 149us              |
| toml_edit::easy | 5.0us                   | 179us              |

**Target:**

|                 | `cargo init` Cargo.toml | cargo's Cargo.toml |
|-----------------|-------------------------|--------------------|
| toml-rs         | 4.7us                   | 121us              |

### Where We Are At

Unfortunately, for practical `Cargo.toml` files, we are still noticeably slower
in parsing.  Rather than invest a lot of time for small potential gains and risk
complicating the code, I figure its time to shift focus to `toml_edit`s impact on
[a resolver benchmark](https://github.com/rust-lang/cargo/issues/9935) and
evaluating where we an go from there.  Thanks to Eric Huss for their work on
this benchmark!  Perfect timing!

Thankfully, by having such a specific use case, we can explore a wider range of
options if `toml_edit` does end up having a noticeable impact.  One route I've
taken in other projects is looking to make up for the losses elsewhere in the
process.  I think Cargo has an even better option available, bypass TOML
parsing completely.  Most dependencies being walked, will be in crates.io
packages, which are immutable.  We could convert their `Cargo.toml` files into
a more optimizable format, giving gains well past what we'd get with any toml
library.  That seems like a lot of work, so hopefully we won't need it.

*Thanks to Futurewei for sponsoring this work*
