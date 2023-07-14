---
title: 'Winnow 0.5: The Fastest Rust Parser-Combinator Library?'
published_date: 2023-07-13 20:18:49.003915401 +0000
layout: default.liquid
is_draft: false
data:
  tags:
  - programming
  - rust
---
[Winnow](https://crates.io/crates/winnow) is a parser-combinator library for Rust and
[0.5](https://github.com/winnow-rs/winnow/blob/main/CHANGELOG.md#050---2023-07-13)
is now out.
I [last wrote](/blog/2023/02/winnow-toml-edit-combine-nom/#winnow) about the
0.3.0 release, so I'll be covering all of the releases since then.

<!-- more -->

## What is Winnow?

To recap [my last post](/blog/2023/02/winnow-toml-edit-combine-nom/#winnow),
you can think of Winnow as a parser toolbox, making it easier to get up and
running with your parser without getting in the way of hand-writing the
trickier parts.
My hope is that this can serve as a "do anything parser" much like how people
treat regex.

Winnow started as a fork of [nom](https://crates.io/crates/nom) as I had found
its toolbox model of parsers worked much better for me than the framework model
other parser libraries used like [combine](https://docs.rs/combine/latest/combine/).
The original goals for the fork were to improve the developer experience and [to remove a
corner case that had a performance cliff you could fall off
of](https://github.com/winnow-rs/winnow/issues/72).

Being the fastest was a non-goal of the fork as that is a competition I didn't want to get distracted by
and felt there were some optimizations that ran counter to my goals.
Specifically, chumsky uses GATs to allow the calling parser to communicate to
sub-parsers if there were steps that could be discarded (see niko's post
["Many modes: a GATs pattern"](https://www.smallcultfollowing.com/babysteps/blog/2022/06/27/many-modes-a-gats-pattern/)).
This works well for the framework model as you put together the building blocks
and the framework takes care of the rest.
The problem when applying GATs to the toolbox model is that hand-written
parsers are a black box to the system and refactor a parser to be hand-written
could mysteriously lead to a dramatic drop in performance.
If/when a user is aware of this, to workaround it requires writing some fairly
advanced Rust code with a decent amount of boilerplate.

## What changed since 0.3.0...

Jump to the [bottom](#numbers) if you just want to see performance numbers.

And checkout the [changelog](https://github.com/winnow-rs/winnow/blob/main/CHANGELOG.md#050---2023-07-13)
for more details.

### Usability: Improving type inference

One area of weakness for Winnow was that type inference didn't work as often as I felt it should.
We had code that looked roughly like:
```rust
pub trait Parser<I, O, E> {
    fn parse_next(&self, input: I) -> IResult<I, V, E> {
        // ...
    }

    // ...
}

impl<E> Parser<&str, char, E> for char {
    fn parse_next(&self, input: I) -> IResult<I, O, E> {
        one_of(self).parse_next(input)
    }
}
```
This allows `delimited('(', expr, ')')` to be a short-hand for `delimited(one_of('('), expr, one_of(')'))`.

However, there are many parser combinators directly on the `Parser` trait, much
like `Iterator`, and type inference would fail for these, like
`'n'.value('\n')`.

I had hoped that
[using associated types](https://github.com/winnow-rs/winnow/issues/163) would
help but that had its own complexity and downsides.
In the end, I found that we were assuming Rust would be able to infer types
between the calls to `Value::parse_next` and `Parser::value` when it wouldn't.
Instead, we could use
[`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)
to do this bookkeeping for Rust to make type inference work.

[#222](https://github.com/winnow-rs/winnow/pull/222) was released in 0.4.0.

### Usability: Clearer top-level parser

In 0.3, we renamed nom's `Parser::parse` to `Parser::parse_next` to make it
more obvious that this is an incremental step and parsing isn't finished
(I've personally ran into problems with this in some of my projects).

`Parser::parse_next` left people with `(rest_input, output ErrMode<Error>)` and they needed to:
- Ensure `rest_input` is empty and discard it (easy to overlook)
- Unify `ErrMode::Backtrack` and `ErrMode::Cut` variants

In Winnow 0.3, we provided `FinishIResult::finish`:
```rust
impl std::str::FromStr for Color {
    // The error must be owned
    type Err = winnow::error::Error<String>;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        hex_color
            .parse_next(s)
            .finish()
            .map_err(winnow::error::Error::into_owned)
    }
}
```

The main problem with using an extension trait is discoverability.
Effectively, the only real option is hope people read the right part of the docs to discover it.

We moved the logic from `FinishIResultExt::finish` to `Parser::parse` now that the name was available:
```rust
impl std::str::FromStr for Color {
    // The error must be owned
    type Err = winnow::error::Error<String>;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        hex_color
            .parse(s)
            .map_err(winnow::error::Error::into_owned)
    }
}
```

[#221](https://github.com/winnow-rs/winnow/pull/2) was released in 0.4.0.

### Usability: Naming and organization

I've found that nom's naming was confusing and parsers were spread across so
may `mod`s that it was hard to find what you were looking for as you were
always looking through a pinhole view of the API
[#95](https://github.com/winnow-rs/winnow/discussions/95).

While I don't think we are at a perfect state, I think things are getting better.

For the module hierarchy, we decided to focus on the high level need, so instead of 
```rust
pub mod bits;
pub mod branch;
pub mod bytes;
pub mod character;
pub mod combinator;
pub mod multi;  // a couple of parsers are almost exclusively for binary data
pub mod number;  // a mix of ASCII and binary parsers
pub mod sequence;
pub mod trace;
```
we now have:
```rust
pub mod ascii;
pub mod binary;
pub mod combinator;
pub mod token;
pub mod trace;  // probably should just be folded into `combinator`
```

Always open to [people's ideas](https://github.com/winnow-rs/winnow/discussions/95) for how to further improve discoverability and clarity of parser code.

[#239](https://github.com/winnow-rs/winnow/pull/239),
[#240](https://github.com/winnow-rs/winnow/pull/240)
were released in 0.4.0.

### Usability: Ranged parsers

Are `take_while_m_n` and `repeat_m_n` inclusive or exclusive for `m` and `n`?
In Rust parlance, I *think* its `m..=n` but I don't even remember for sure.

We provide `repeat0`, `repeat1`, and `repeat_m_n`, so if you need `repeat5`,
might not immediately think to wrap `repeat_m_n`.

And for myself, I think it'd be nice to reduce the redundancy of those parsers
so we have a less overwhelming
[list of parsers](https://docs.rs/winnow/0.3.0/winnow/combinator/index.html).

So we switched from having separate `0..`, `1..`, and `m..=n` parsers to a single version that takes in an
[`impl Into<Range>`](https://docs.rs/winnow/latest/winnow/stream/struct.Range.html),
replacing
```rust
repeat0(parser::key_value).parse_next(i)
```
with
```rust
repeat(0.., parser::key_value).parse_next(i)
```

In contrast, chumsky takes the approach of:
```rust
key_value.repeated()
```
with [additional functions for controlling the range](https://docs.rs/chumsky/latest/chumsky/combinator/struct.Repeated.html).
I felt that within a parser's grammar, the fact that it is repeating is
important information that I wanted it more obvious by leading with it.

I had hoped that we could do
```rust
(Repeat(0..) * parser::key_value).parse_next(i)
```
but [I ran into issues](https://docs.rs/chumsky/latest/chumsky/combinator/struct.Repeated.html).

[#241](https://github.com/winnow-rs/winnow/pull/241) was released in 0.4.2
and [#247](https://github.com/winnow-rs/winnow/pull/247) was released in 0.4.6.

### Usability: Better out-of-the-box errors

`IResult` defaulted the `E` generic parameter to `InputError` but I found that was rarely useful
([#103](https://github.com/winnow-rs/winnow/issues/103)) with the output being:
```
error OneOf at:
name = "foo"
```
- This characterized the error by the `ErrorKind` which is debug information at best
- This dumped all of the output from where the failure started to occur (in this case, there was a missing `]` before a newline)
- We could not add additional context.  For example, with `combine` my errors would list what tokens were expected

So I took the error I made from `toml_edit` and pulled it in as `ContextError`:
```
parse error at line 1, column 9
  |
1 | [package
  |         ^
invalid table
expected ']'
```

In providing this new, rich error that is nearly as fast as no-op errors,
I went ahead and removed `VerboseError` which makes parsing json take twice as
long by itself.  This is a case where GATs would likely remove that overhead.

[#282](https://github.com/winnow-rs/winnow/pull/282) was released in 0.5.0.

### Performance: Inlining

As I mentioned, being the absolute fastest parser-combinator library is not our goal.
We fell into being one of the fastest by happenstance.
When working on API cleanups after the fork from nom, I sprinkled some
`#[inline]`s where they intuitively made sense and things got faster.
I initially didn't advertise this performance gain because I hadn't fully
characterized it and didn't know how much of a fluke it was, because my attention was elsewhere.

Since then, we've found some other places where sprinkling around `#[inline]`
made a big difference.
Rather than trying to take the time to characterize each set of changes we
made, I'll leave that for the performance numbers at [the end](#numbers).

### Performance: Direct people to the happy path

[`nom::one_of`](https://docs.rs/nom/latest/nom/character/complete/fn.one_of.html)
would match a token from a set:
```rust
fn hexadecimal(input: &str) -> IResult<&str, &str> {
  preceded(
    alt((tag("0x"), tag("0X"))),
    recognize(
      many1(
        terminated(one_of("0123456789abcdefABCDEF"), many0(char('_')))
      )
    )
  )(input)
}
```
We carried this behavior forward and generalized to also accepts ranges of
tokens (`'a'..='z'`) and tuples of token sets (`('a'..='z', 'A'..='Z')`).

However, when trying to find the root cause of a performance regression, we
found that using `&str` as a set of `char` tokens was 7x slower than other
methods in micro-benchmarks, likely due to unpacking variable-length UTF-8
bytes into a `char`
([#226](https://github.com/winnow-rs/winnow/issues/226)).

So we decided to remove the trait impl that allowed `&str` to serve as a token set
to avoid people accidentally taking this performance hit.

So instead our parser would look like:
```rust
fn hexadecimal(input: &str) -> IResult<&str, &str> {
  preceded(
    alt(("0x", "0X")),
    repeat(
      0..,
      terminated(one_of(('0'..='9', 'a'..='f', 'A'..='F')), repeat(0.., '_'))
    ).recognize()
  ).parse_next(input)
}
```

[#252](https://github.com/winnow-rs/winnow/pull/252) was released in 0.5.0.

### Performance: Imperative, rather than pure-functional parsing

We started with the parsing model of nom:
```rust
fn foo<I, O, E>(input: I) -> Result<(I, O), ErrMode<E>> {}
```
This works well!
As you parse `input`, you return the mutated version of it.
This makes it easy to reason about and test.

But there is a problem and, from this section, you can guess its performance.
The good news is that this isn't a general performance issue.
In fixing this, most benchmarks didn't see a change.
I suspect this is specifically when dealing with large `I` and `O` making `(I,
O)` in the return type to be very large and causing a performance loss.
In `toml_edit`, we hit this because we use `I=Located<&str>` which is `(&str,
&str)` or 4 pointers combined with large `O`s from tracking values and their
formatting.

We can instead switch the parser signature to:
```rust
fn foo<I, O, E>(input: &mut I) -> Result<O, ErrMode<E>> {}
```
and now the return type's size is independent of the size of `I`.

The primary technical challenge is dealing with the error case.
Translating `nom`s model to this, on any error, `input` should revert back to
its original position.
Instead, I found we could have error cases leave `input` as-is.
This doesn't just simplify and reduce overhead from making `input`
transactional but allows `E` to not track the original location where the error
occurred but to instead use the final location `input` was left at.

But do we lose the spirit of what we are trying to accomplish?

I'm still unsure.
I was pleasantly surprised to find that a lot of code was simplified because of this model.

However, there are cases where explicit lifetimes are required:
```rust
fn foo<'i>(input: &mut &'i str) -> Result<&'i str, ErrMode<E>> {}
```
I'm concerned this style of parser is not as Rust beginner friendly.
That needs to be counter-balanced with my suspicion that this new API will allow users to avoid `RefCell`.
Maybe its also ok because zero-copy parsing might already be more advanced?
At this point, I feel like the only way to know for sure is to give it a spin.

<a id="numbers"></a>
## Performance by the numbers

Using `chumsky`'s `json` benchmark:

| library             | version | time      | note |
|---------------------|---------|-----------|------|
| pom                 | 3.3.0   | 7.3021 ms |
| pest                | 2.7.0   | 1.2894 ms |
| nom                 | 7.1.3   | 341.28 µs |
| serde_json          | 1.0.102 | 317.31 µs |
| chumsky (zero copy) | 7a2e2e6 | 191.06 µs |
| sn                  | 0.1.2   | 154.84 µs |
| winnow              | 0.3.6   | 125.54 µs |
| winnow              | 0.4.0   | 138.71 µs |
| winnow              | 0.4.1   | 251.97 µs |
| winnow              | 0.4.2   | 242.56 µs |
| winnow              | 0.4.3   | 240.69 µs |
| winnow              | 0.4.4   | 230.01 µs |
| winnow              | 0.4.5   | 257.89 µs |
| winnow              | 0.4.6   | 262.45 µs |
| winnow              | 0.4.7   | 231.48 µs |
| winnow              | 0.4.8   | 234.59 µs |
| winnow              | 0.4.9   | 244.81 µs |
| winnow              | 0.4.9   | 245.02 µs | migrated bench to non-deprecated APIs |
| winnow              | 0.4.9   | 245.82 µs | made bench more declarative |
| winnow              | 0.4.9   | 96.408 µs | migrated bench away from slow API |
| winnow              | 0.5.0   | 97.328 µs |

Using `toml_edit`'s `cargo_manifest/toml_edit/medium`:

| version     | parser   | version   | time      | note |
|-------------|----------|-----------|-----------|------|
| 0.15.0      | combine  | 4.6.6     | 91.160 µs |
| **0.16.0**  | **nom8** | **0.1.0** | 79.140 µs |
| **0.16.1**  | nom8     | 0.1.0     | 79.813 µs |
| **0.16.2**  | nom8     | **0.2.0** | 79.971 µs |
| **0.19.3**  | nom8     | 0.2.0     | 96.821 µs |
| **0.19.4**  | winnow   | **0.3.0** | 78.559 µs |
| **0.19.7**  | winnow   | 0.3.0     | 78.502 µs |
| 0.19.7      | winnow   | **0.3.6** | 79.326 µs |
| **0.19.8**  | winnow   | **0.4.0** | 77.498 µs |
| 0.19.8      | winnow   | **0.4.1** | 76.795 µs |
| 0.19.8      | winnow   | **0.4.2** | 78.532 µs |
| 0.19.8      | winnow   | **0.4.3** | 78.682 µs |
| 0.19.8      | winnow   | **0.4.4** | 78.562 µs |
| 0.19.8      | winnow   | **0.4.5** | 78.572 µs |
| 0.19.8      | winnow   | **0.4.6** | 83.424 µs |
| **0.19.9**  | winnow   | 0.4.6     | 83.856 µs |
| 0.19.9      | winnow   | **0.4.7** | 73.803 µs |
| 0.19.9      | winnow   | **0.4.8** | 73.605 µs |
| 0.19.9      | winnow   | **0.4.9** | 74.002 µs |
| **0.19.12** | winnow   | 0.4.9     | 74.029 µs |
| **68be6ab** | winnow   | 0.4.9     | 74.085 µs | prep for winnow 0.5.0 |
| 68be6ab+    | winnow   | **0.5.0** | 63.075 µs |

- [winnow changelog](https://github.com/winnow-rs/winnow/blob/main/CHANGELOG.md)
- [toml_edit changelog](https://github.com/toml-rs/toml/blob/main/crates/toml_edit/CHANGELOG.md)

Methodology:
I ran `cargo bench` on the individual bench until I observed a second cluster
of results and picked the lowest. 
I've observed that on my machine, jitter is fairly tight around two clusters
and without waiting for that second cluster, the results can be misleading if the slower cluster manifested.
I would recommend not getting caught up in differences up to 10%, based on the variance I saw.
I should look into icount benchmarks...

Observations:
- The hassle of running benchmarks can lead to blindspots.
  I tended to focus on the performance impact to a Winnow benchmark and a `toml_edit` benchmark.
  Winnow uses lower level parsers to better line the code up with the TOML BNF
  grammar and I was able to catch most performance regressions there.
  Similarly, we fixed all reported regressions and yet they didn't affect the json bench list above.
  Even if I ran all of the benchmarks, I was unlikely to catch some of these problems.
  Switching to icount benchmarks with a benchmark CI service would be a big help.
- Seemingly minor refactors can have big affects.
  Some came from what was inlined or not (and adding more inlines helped) but I never figured it all out.
- The performance of some parsers seems dependent on the context its called in.
  In micro-benchmarks, I would not see a problem reproducible within a
  file-format parser using winnow.  Again, I suspect inlining differences to be
  at play.

So if Winnow is the fastest, why is the title of this post a question?
1. chumsky's benchmark is missing [yap](https://github.com/jsdw/yap) which seems promising on performance.
   For more parsers, see [parse-rosetta-rs](https://github.com/rosetta-rs/parse-rosetta-rs)
2. Performance will depend on your use case and needed feature set.
   I could make Winnow faster by changing the error type in the bench.
   The bench could also be slower if the parser was instrumented with span tracking or error recovery.

*Note: every change that has led to a performance improvement has been reported back to nom*

Discuss on
[lemmy](https://lemmy.ml/post/2014244)
[reddit](https://www.reddit.com/r/rust/comments/14yvfsy/winnow_05_parser_combinator_library_is_out_even/?)
[mastadon](https://hachyderm.io/@epage/110708637819903154)
