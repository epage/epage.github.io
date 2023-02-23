---
title: winnow = toml_edit + combine + nom
published_date: 2023-02-22 18:35:24 +0000
layout: default.liquid
is_draft: false
data:
  tags:
  - programming
  - rust
---
It is finally time to take the wraps off where I disappeared to over the
last 6 months. Besides the family leave, I've mostly been chalking this up to
working on
[toml_edit](https://epage.github.io/blog/2023/01/toml-vs-toml-edit/) but one
particular building block took up most of that time.

I would like to introduce you to [winnow](https://crates.io/crates/winnow), a
fork of the venerable [nom](https://crates.io/crates/nom) parser combinator
library.  For those that want to skip all the details, you can checkout
[the documentation](https://docs.rs/winnow/latest/winnow/) and
[the migration guide and changelog](https://github.com/winnow-rs/winnow/blob/main/CHANGELOG.md#030---2023-02-22).

~~I would link to the docs but [docs.rs seems to be backed up this morning](https://docs.rs/releases/queue).~~

<!-- more -->

Probably the two most likely questions are:
- Why was this needed?
- Why not `<insert parsing approach here>`?

I think answering these is best done by covering the journey to winnow.

Steps:
1. [toml_edit and combine](#combine)
2. [toml_edit and Hand-written Parser](#hand)
3. [toml_edit and nom](#nom)
4. [A glance at chumsky](#chumsky)
5. [tonl_edit and winnow](#winnow)

<a id="combine"></a>
## 1. toml_edit and combine

When I got involved with toml_edit, it had been using
[combine](https://crates.io/crates/combine) for its parser combinator
library. While I was more familiar with nom, having used it in my own
projects, I adapted.

While working on getting toml_edit into cargo, the biggest problems I had were:

- Unlike nom, combine commits to a parse result by default and you have to
  opt-in to backtracking, making it harder to get the logic "just right"
- Performance and quality of error messages
- Difficulty to understand and debug what was going on due to the abstract
  nature of parsers describing how to parse, rather than directly parsing.

Thankfully [Marwes](https://github.com/Marwes) was responsive and helped me
through these challenges! I was able to improve toml_edit so it could be merged
into cargo, unblocking `cargo add` from being merged into cargo.

I thought things were fine and had moved on until I got a ping from [nnethercote](https://nnethercote.github.io/):

> nnethercote: Looking at <https://blog.rust-lang.org/images/2022-04-07-timing.html>, toml_edit is a monster
>
> nnethercote: Is it especially large, or use macros to generate a lot of code?

[Rust 1.60 was released](https://blog.rust-lang.org/2022/04/07/Rust-1.60.0.html)
with a new cargo feature to draw graphs of how long each crate took to compile
and toml_edit was front and center in the 1.60 announcement for being cargo's
slowest dependency! And I had nnethercote knocking at my door about it! I
needed to do something about this, but what?

I started with nnethercote's question: macros. The reason macros came up was
he was in the middle of
[investigating slow compiles due to macros](https://nnethercote.github.io/2022/04/12/how-to-speed-up-the-rust-compiler-in-april-2022.html)
due to a technique called
[tt-muncher](https://veykril.github.io/tlborm/decl-macros/patterns/tt-muncher.html)
which is quadratic relative to the lengnth of the input. *Every single parse
function in toml_edit (with some quite large) was wrapped in a
toml_edit-specific macro that wrapped a
[tt-muncher from combine](https://docs.rs/combine/4.6.6/combine/macro.parser.html).*

I first set out to test my theory that macros were the root cause by throwing
together
[parse-rosetta-rs](https://github.com/rosetta-rs/parse-rosetta-rs). This added
more evidence to the theory that it was the macros as the combines entry in the
shootout didn't use them and compiled in reasonable times. However, it wasn't
completely conclusive because (1) something else could be going on in either
parse-rosetta-rs or toml_edit and (2) parse-rosetta-rs was using an old
version of combine as that was the only json parser I found and I didn't want
to update it to the latest version.

<a id="hand"></a>
## 2. toml_edit and Hand-written Parsers

In theory, combine parsers can be written without these macros but it looked
to be an unwieldy undertaking on the level of switching to a different parser.
I have gotten opposing feedback that toml_edit should both use a
hand-written parser *and* it was a positive sign to be using a parsing library.

For myself, I felt hand-written parsers were harder to keep readable,
maintainable, and adaptable for bug fixes and new features. If this was a
project with a larger contributor base with people specializing on just
parsing, I could see this making sense. With this just being 1-2 people
occasionally dabbling with edits, this is less tenable.

In the end, if I did something hand-written, it would end up looking much like
a parser library. Whether to go this route then gets into the "Build vs Buy"
debate. Relying on an existing library is convenient as it distributes the
maintenance load, testing, bug finding, and fixing. However, parsing is
fundamental to toml_edit and depending on a parsing library ties my project
to theirs. For example, if we need an improvement or bug fix, we are beholden
to the maintainer of the parsing library for if/when the fix gets merged *and*
[released](https://github.com/crate-ci/cargo-release/).
If we were talking about ancillary functionality, becoming dependent on
another project isn't too big of an issue as we can more easily live without
improvements or change the dependency if needed. Thankfully, Marwes was great
to work with on combine though it was time to move on.

Or in other words, we do not want:

[![XKCD: Dependency](https://imgs.xkcd.com/comics/dependency.png)](https://xkcd.com/2347/)

<a id="nom"></a>
## 3. toml_edit and nom

nom is a parsing library that I'm both familiar with and is highly respected,
making it feel like a safe choice for replacing combine and its macros.
This is ironic because toml_edit originally
[switched to combine](https://github.com/toml-rs/toml/pull/26) to get rid of
nom's former macros.

I didn't get too far into the port before putting it on pause however, not from the amount of
work, but because the nom code was harder to read to my eyes which are not inducted
into the Lisp cult of parentheses.

[![XKCD: Lisp](https://imgs.xkcd.com/comics/lisp_cycles.png)](https://xkcd.com/297/)

In working on a large, readable parser, I felt switching to nom would be too
large of a regression. While I had been happy with nom before, my experience
was limited. My parsers had been small and I didn't have the point of
comparison of working with combine.  combine focused more on
trait functions (compare a standalone `map` vs `Iterator::map`) and had
niceties like implementing the `Parser` trait for tuples to sequence parsers.
I also assumed my difficulties were with my lack of experience writing parsers
and not with nom.

I decided to help improve nom as my route to unblock my work.  I started with issues and minor
contributions to feel out the process and priorities for nom with ideas like:
- [Issue #1408: Reduce Syntactic Noise](https://github.com/rust-bakery/nom/issues/1408)
  - [Issue #1393: Consolidate parser variants using ranges (e.g. many0, many_m_n) ](https://github.com/rust-bakery/nom/issues/1393)
  - [Issue #1417: impl Parser for tuples](https://github.com/rust-bakery/nom/issues/1417)
  - [Issue #1415: Update docs to point to Parser::map over nom::combinator::map. ](https://github.com/rust-bakery/nom/issues/1415)
- [Issue #1414: Swap module hierarchy so `streaming`/`complete` are in the root](https://github.com/rust-bakery/nom/issues/1414)
- [Issue #1416: Help people discover how to do take_while, et al, with parsers instead of predicates](https://github.com/rust-bakery/nom/issues/1416)
- [Issue #1409: Support custom containers for many functions](https://github.com/rust-bakery/nom/issues/1409)

An astute reader might notice that these predate nnethercote's message.  I
actually started on the port previously out of idle curiosity.

#### Responsiveness and Communication

My attempt to contribute to nom spanned over a year:
- **September 2021:** I attempted my port to nom out of idle curiosity.
  I opened issues, got a generally positive
  reception, and moved forward slowly, with
  [a simple prepartory refactor PR](https://github.com/rust-bakery/nom/pull/1412).
  After a couple of weeks, I got push back and then never heard back again when
  I asked a clarifying question. As this wasn't a priority, I moved on.
- **April 2022:** Rust 1.60 is released and nnethercote reached out to me. As this
  felt important to resolve and my previous approach floundered, I reached out
  directly to try to come to an understanding to allow my work on nom to
  progress. I was told this would align with nom v8 (despite few of my changes
  needing a breaking release) but that v8 was planned for May 2022 and should
  take about a week to do the release. May came and went.
- **July 2022:** I followed up on v8 which had been planned for May and am told
  that we can sync up in August 2022, after vacation is over. This never
  happened.
- **September 2022:** My priority for toml_edit's build performance went up
  as I became the maintainer of toml with the plan to rewrite it
  in terms of toml_edit and didn't want to negatively impact the ecosystem with
  the build times.  I reached out again and we finally synced up. Then nothing,
  again.
- **October 2022:** As users opened Issues and PRs for toml, I felt the pressure
  build as I didn't want to touch the existing code because of
  (1) all of this was throw-away work and would have to be re-implemented once
  toml used toml_edit
  and (2) the ramp up time on toml's hand-written parser would take me away from improving things.
  More desperate, I created
  [nom8](https://crates.io/crates/nom8), a short-lived fork to
  experiment with my proposals for nom v8 and to use them immediately,
  without being blocked on nom.
- **December 2022:** After some delay due to family leave, nom8 is finally ready.
  I received positive reception with the results! This is also the first time
  there was positive momentum on the nom repo; we were working to merge my
  PRs.

Regardless of how the merging goes (see below), nom was not living up to the
expectations for being a critical dependency. I have had reassurances that
this was a fluke and the maintainer will be more responsive going forward but
the inherent trust from nom's status has been lost and it is too late to
take the time to rebuild that trust; I need to move on.  It was too late a
while ago but I let myself be strung along with the promises that things would
progress "soon".

#### Ship of Theseus

While planning the merging of my work, things broke down. In large part this
was due to different priorities.

The first of these is that merging contributions is considered a low
priority for the project. An unspoken (as far as I can find) assumption in
nom is that it provides a "core" and people are to build or replace what they
need on top, like [nom_locate](https://crates.io/crates/nom_locate).

This helps explain some of the delay above but is its own problem: how many
dependencies do I need to pull in and how much do I need to directly replace to
make nom functional? At what point am I still really using nom vs
maintaining my own library with hints of nom? At what point is using nom
no longer pulling its weight and instead the dependencies and
re-implementations become a liability?

Each dependency I pull in needs a strong justification to keep the impact on my
users small and to avoid
[rust users picking apart my dependencies](https://www.reddit.com/r/rust/comments/uymdcp/introducing_wax_portable_and_opinionated_globs/).

Taking this to an extreme, to meet my parsing library ergonomic goals, I would
basically be replacing the top layer of nom, only being able to reuse
the core types.

Except even the core types might not be enough. When adding span support to
toml_edit, I found just
[increasing the size of my input-type made parsing take 30% longer](https://github.com/winnow-rs/winnow/issues/72).
For toml_edits use in cargo, these numbers take a barely-acceptably-slow TOML
parser over the edge to being too slow (which was mitigated for now due to
performance gains from a massive rework of the parser).

When I brought these numbers to nom's maintainer, they were brushed off
because they had a bad prior experience with the redesign work needed to fix
this (despite it working for combine and other parser library). Without the
maintainer onboard, *this requires me to throw out the core of nom and now I'm
left without anything to reuse.*

Now, before it would have come to this, I'm more likely to just move to a different
parser or hand write my parser but this illustrates the weakness of foisting
responsibilities onto users.

#### Existing Users vs Future Users

Another priority we conflicted on was which users to prioritze for:
- Existing parsers that have to be upgraded across breaking changes
- The parsers yet written

For me, the focus is on new users and the applications not-yet-written as I
feel the Rust community is at an inflection point (which
[matklad also spoke about](https://matklad.github.io/2023/01/25/next-rust-compiler.html)).

This doesn't mean there is nothing we can do for existing users. While things
haven't always gone smoothly on first release, I feel like with
[clap](https://crates.io/crates/clap), we've learned a lot about how to help
people through a changing API.

<a id="chumsky"></a>
## 4. A glance at chumsky

During this process, [chumksy](https://crates.io/crates/chumsky) has matured
some and I felt obligated to take a look. The two red flags for me were:
- They only compare performance with [pom](https://crates.io/crates/pom) and
  not nom and toml_edit can't take any more performance regressions
  ([they are improving this](https://github.com/zesterer/chumsky/pull/82)).
- They actively advertise the framework mindset that caused me problems with combine

I decided that it was not worth further investigation at this time.

<a id="winnow"></a>
## 5. toml_edit and winnow

Being unsatisfied with the options, it looks like I'm needing to go the "Build"
route. As I said, this doesn't mean I write something bespoke. Overall, I
find the approach nom takes to work well for me for writing parsers, so I
decided to take my nom8 fork and make that the base for my new parser, winnow.

> For those unfamiliar, winnowing is the process for separating chaff from grain
> by throwing them in the air and letting the breeze blow the chaff away, leaving
> the grain to fall and be collected. Thanks to
> [compenguy](https://github.com/compenguy) for the name!

My aim is for winnow to be your "do everything" parser, much like people treat
regular expressions. To this end, the project's priorities are:
1. Support writing parser declaratively while not getting in the way of imperative-style
   parsing when needed, working as an open-ended toolbox rather than a close-ended framework.
2. Flexible enough to be used for any application, including parsing binary data, strings, or
   separate lexing and parsing phases
3. Zero-cost abstractions, making it easy to write high performance parsers
4. Easy to use, making it trivial for one-off cases

While I dislike going dark for development, I wanted to get winnow far
enough along to clarify what my intention is with this project and my
commitment.  Its easy to have a big idea; its much better to show results.

So what does winnow bring to the table?
- [Clearer Usage](#usage)
- [Simpler, Easier API](#simple)
- [Easier Debugging](#debug)
- [`Located` and `Stateful`](#stream)
- [The Future](#future)

See also [the migration guide and changelog](https://github.com/winnow-rs/winnow/blob/main/CHANGELOG.md#030---2023-02-22).

<a id="usage"></a>
#### Clearer Usage

nom's documentation is scattered across various loose markdown files and
docs.rs, making it hard to find pertintent information. When exploring
documentation options in clap, we found:
- A lot of users want one place to go
- Github is a poor place to store documentation because people naturally look
  at `main` and not the version pertinent to them

Like with clap, I've moved all the documentation to docs.rs and organized it
along the [four types of documentation](https://documentation.divio.com/).  I
then integrated
[nominomicon](https://tfpk.github.io/nominomicon/introduction.html) as the
[tutorial](https://docs.rs/winnow/latest/winnow/_tutorial/chapter_0/index.html).

Narrowing in on the documentation, a basic question is buried: how do I
integrate this into my application. To answer this, I switched the
introductory example from calling the example parser as a building block to
integrating it into `FromStr`:

Before:
```rust
fn main() {
  assert_eq!(hex_color("#2F14DF"), Ok(("", Color {
    red: 47,
    green: 20,
    blue: 223,
  })));
}
```
After:
```rust
impl std::str::FromStr for Color {
    // The error must be owned
    type Err = winnow::error::Error<String>;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        hex_color(s)
            .finish()
            .map_err(winnow::error::Error::into_owned)
    }
}
```

While improving documentation is good, we need to head off problems before
people look to the documentation (if that even happens).

When dealing with custom error handling and application integration, users had
to deal with [`nom::Err`](https://docs.rs/nom/latest/nom/enum.Err.html) with
its `Error` and `Fatal` variants.  Why this exists and how it works isn't too
clear from the names and you have to read and grok the documentation.
We've renamed `Err` to
[`ErrMode`](https://docs.rs/winnow/latest/winnow/error/enum.ErrMode.html)
with its `Backtrack` and `Cut` variants,
making it clear that this is adding parser modality information to your error, with one
variant used for trying alternative parsers and the other for short-circuiting
to the caller.

Looking back at that `FromStr` example above, can a reviewer tell whether it is safe
to discard the `remainder` (`""`) from `hex_color`?  I feel like its just as
bad if we use the trait method:
```rust
fn main() {
  assert_eq!(hex_color.parse("#2F14DF"), Ok(("", Color {
    red: 47,
    green: 20,
    blue: 223,
  })));
}
```
To draw attention to this, I've renamed `Parser::parse` to
`Parser::parse_next`, adding nuance to "this parsed" by conveying "this
parsed a portion".
```rust
fn main() {
  assert_eq!(hex_color.parse_next("#2F14DF"), Ok(("", Color {
    red: 47,
    green: 20,
    blue: 223,
  })));
}
```
`Parser::parse_next` will also be more visible as I've shifted some
combinators to be directly on the `Parser` trait and I am considering
[switching built-in parsers to return `impl Parser` instead of `impl
FnMut`](https://github.com/winnow-rs/winnow/issues/162), each forcing the use
of `Parser::parse_next` over direct function calls.

Consistent language can also prevent incorrect behavior.  Consider coming
across `take_while_m_n` (consume `m..=n` tokens) as a reviewer:
```rust
    match input.position(|c| !cond(c)) {
      Some(idx) => {
        if idx >= m {
          if idx <= n {
            let res: IResult<_, _, Error> = if let Ok(index) = input.slice_index(idx) {
              Ok(input.take_split(index))
```
And compare it to:
```rust
  match input.offset_for(|c| !list.contains_token(c)) {
    Some(offset) => {
      if offset >= m {
        if offset <= n {
          let res: IResult<_, _, Error> = if let Ok(offset) = input.offset_at(offset) {
            Ok(input.next_slice(offset))
```
Without any other context, `input.offset_at(offset)` is likely to stand out.
This would lead a reviewer to double check assumptions, the pertinent one being
that byte offsets are not the same as token counts for `&str` (`offset_at`
converts a token count to a byte offset).  Knowing this, another bug stands
out: we are comparing byte offsets with token counts in `offset >= m` and
`offset <= n`.  This isn't hypothetical but represents a
[current bug in nom](https://github.com/rust-bakery/nom/issues/1630).

<a id="simple"></a>
#### Simpler, Easier API

Looking back over my parsers, I see I'm using a fraction of what `nom` offered me because
- Available built-in parsers were lost in the noise
- Composing the pieces was not obvious

This matches my experience in using and now maintaining clap: the larger an API
is, the less people are likely to discover what is available, lowering the
value of each part of the API with each new API addition.

The first area I focused on was trying to remove the distinction between
"complete" and "streaming" parsers. `nom` has a lot of parsers behave somewhat
differently when the input isn't completely in-memory but is streaming in from a source. Currently, that is
handled by having parsers duplicated, like with
`nom::bytes::complete::take_while` vs `nom::bytes::streaming::take_while`.

The problems I ran into with this were:
- It is easy to unintentionally choose the wrong kind or mix them together, getting incorrect behavior that only shows with certain inputs
- The extra nesting makes it harder to browse the API and discover functionality

After [some experimentation](https://github.com/rust-bakery/nom/issues/1535), I
found that we could tag the input type as being `Partial` and all of the
parsers could switch their behavior. The complete case (default) sees no
overhead from this switching. A user can implement a `Partial` tag that shows
no overhead but I decided to make the built-in `Partial` support being able to
switch to complete-parsing at runtime (e.g. the `Reader` reported the true
end-of-input). The overhead looks to mostly be from the larger input type
which should be reduced with
[Issue #72: Improving performance for `Located`](https://github.com/winnow-rs/winnow/issues/72).

Inspired by combine, I next looked to using Rust-native types for parsers, like:
- Tuples for sequencing parsers
- `&str` and `char` literals to act as `tag`s

For example, before:
```rust
delimited(char('('), number, char(')'))
```
After
```rust
delimited('(', number, ')')
```

We can also extend this to defining character classes with `one_of`:
- `&str`, `&[]`, and ranges (`0..=9`) as sets of characters
- Closures as predicates
- Tuples to compose these together

For example, before:
```rust
fn hexdigit(input: &str) -> IResult<&str, char> {
    one_of("0123456789abcdefgABCDEFG")
}
```
After
```rust
fn hexdigit(input: &str) -> IResult<&str, char> {
    one_of(('0'..='9', 'a'..='f', 'A'..='F'))
}
```

This let us consolidate variations of `one_of`, like `satisfy`. This also
applies to `take_while0` and related parsers, simplifying what we offer there
as well.

<a id="debug"></a>
#### Easier Debugging

I felt the [nom-tracable](https://crates.io/crates/nom-tracable) crate was a
great idea but never used it myself as I didn't want to pull in a dependency
just for debugging and instead made one-off tracing combinators. What if it
was built-in and all of the built-in parsers supported it?

This led to the `trace` combinator which does nothing in normal builds but will
dump the parse state when your parser is built with `--features winnow/debug`.

![Debug traces from a string parser](https://raw.githubusercontent.com/winnow-rs/winnow/main/assets/trace.svg)

*(benchmarks confirm this has no impact on performance when disabled)*

The trace can be severely hampered by the debug output of your input when
parsing bytes (`&[u8]` printing a list of numbers isn't too helpful) so I
created `winnow::Bytes` and `winnow::BStr` newtypes with debug output tailored
to different parsing applications.

<a id="stream"></a>
#### `Located` and `Stateful`

`toml` required `toml_edit` to track input spans of the TOML AST. In `nom`, this is generally done with 
[nom_locate](https://crates.io/crates/nom_locate) which never sat right with me due to:
- Reporting the column [without regard for the inherent complexity](https://manishearth.github.io/blog/2017/01/14/stop-ascribing-meaning-to-unicode-code-points/)
- Overhead from tracking the line, rather than just the span
- This mysterious `extra` field that would go unused in my applications

On a reddit thread, I came across [pori](https://crates.io/crates/pori) which improves on this:
- Separate `Stateful` and `Located` wrapper types
- `Located` only tracks the span, not lines or "columns"

In working with this, I realized that there was still some unnecesaarry
overhead from tracking the span as you parse. If you just add `Located`
without capturing any spans, your parser slows down from the bookkeeping to
track where the parser was currently at. I was able to rework this so there is
no active bookkeeping, making the overhead negligible compared to a
`Stateful<I, usize>`.

For my experimentation with different location tracking styles,
- [#61 Add Span Support](https://github.com/winnow-rs/winnow/pull/61)
- [#63 pori](https://github.com/winnow-rs/winnow/pull/63)
- [#64 nom_locate](https://github.com/winnow-rs/winnow/pull/64)

<a id="future"></a>
#### The Future

That said, the work isn't over yet.

I've started laying out [short term](https://github.com/winnow-rs/winnow/milestone/2) and [long term](https://github.com/winnow-rs/winnow/issues) plans and am particularly excited for:
- [Issue #72: Improving performance for `Located`](https://github.com/winnow-rs/winnow/issues/72)
- [Issue #96: Error recovery support](https://github.com/winnow-rs/winnow/issues/96)

Some of the work is a bit more nebulous and I would love your input on it!
- [Clearer names / module hierarchy](https://github.com/winnow-rs/winnow/discussions/95)
- [What is needed for binary parsing?](https://github.com/winnow-rs/winnow/discussions/85)
- [What is needed for streaming/partial/incomplete parsing?](https://github.com/winnow-rs/winnow/discussions/169)
- [Should built-in parsers return `impl Parser` rather than `impl FnMut`](https://github.com/winnow-rs/winnow/issues/162)
- [Should parsers use associated types rather than generics](https://github.com/winnow-rs/winnow/issues/163)

I also expect people to bring a lot of other ideas to the table and look forward to
[learning with you in how to further improve winnow](https://github.com/winnow-rs/winnow/discussions).

Discuss on
[reddit](https://www.reddit.com/r/rust/comments/1197es1/winnow_toml_edit_combine_nom/?),
[discourse](https://users.rust-lang.org/t/announcing-winnow-toml-edit-combine-nom/89746),
[mastadon](https://hachyderm.io/@epage/109909843656731429)

*Thanks to [Muscraft](https://github.com/Muscraft) for the proof-reading*
