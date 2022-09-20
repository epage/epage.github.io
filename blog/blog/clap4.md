---
title: "clap 4.0, a Rust CLI argument parser"
published_date: "2022-09-16 09:00:30 -0500"
is_draft: true
data:
  tags:
    - programming
    - rust
---

We are excited to (pre-)announce clap 4.0!  This release focuses on removing
deprecated APIs and finishing what we couldn't do without breaking changes.

This release builds on the new APIs introduced during 3.x, see also:
- The [3.1 release](/blog/2022/02/clap-31-a-step-towards-40/)
- The [3.2 release](/blog/2022/06/clap-32-last-call-before-40/)
- Or the [CHANGELOG.md](https://github.com/clap-rs/clap/blob/v3-master/CHANGELOG.md) for the over 50 patch releases since 3.0.0

To put this into numbers:

|                       | Baseline  | 2.34.0    | 3.0.0     | 3.2.21    | 4.0.0     |
|-----------------------|-----------|-----------|-----------|-----------|-----------|
| Builder API Surface   |           | 174       | 245       | 282       | 165 [[1]](#removing-deprecated-features) [[2]](#num-args) |
| Lines of Code         | 6         | 13,462    | 17,308    | 24,044    | 20,653 [[1]](#removing-deprecated-features) [[3]](#removing-lifetimes) |
| Code size             | 218.2 KiB | 487.0 KiB | 609.3 KiB | 605.5 KiB | 544.3 KiB [[1]](#removing-deprecated-features) [[4]](#reducing-code-size) |
| Runtime               |           | 7.529 us  | 14.544 us | 14.657 us | 8.2478 us [[5]](#removing-implicit-version-help-behavior) |

*(see [Methodology](#methodology) for more details)*

> Aside: Yes, those clap v2 -> v3 numbers are not good.
>
> For Builder API Surface, it is understandable when you consider we mostly didn't
> remove functionality in v3 but deprecated it, removing it in v4.
>
> Lines of Code is mostly accounted for with the merge of `structopt` into
> `clap` as `clap_derive`.  We continued to have significant growth after
> that as we continued to develop replacement features for functionality in
> clap.  These more general solutions take up more lines though not more code
> size.
>
> For code size and runtime, one factor is that things fell through the cracks
> during clap v3's development.  clap's development went dark for an extended
> period of time and went through several maintainers.  This isn't to say one
> of the maintainers is at fault but that things get lost in hand offs.
> Towards the end, we double-downed on just getting out what we had and hadn't
> looked to see how we compared to v2.
>
> For code size, it looks like it was a lot of small changes that added up,
> like changing a `VecMap` to a `BTreeMap`.
>
> For runtime, it seems to mostly be a single feature that caused it which was
> removed in v4 [[5]](#removing-implicit-version-help-behavior).

Our plan is to give about a week window between the release-candidate and the
official release to allow for collecting and processing feedback.

<!-- more -->

## What Changed

As a caution, this will be a mix of high-level and low-level details as we try
to show how each change impacts not just functionality but several different
performance metrics to help understand the benefits of some features and the
costs of others.

Most people will care about:
- [Support Policy](#support-policy)
- [Polishing Help Output](#polishing-help-output)
- [More Specific Derive Attributes](#more-specific-derive-attributes)
- [`num_args`](#num-args)
- [Removing Deprecated Features](#removing-deprecated-features)

Some more specific changes:
- [Reducing Code Size](#reducing-code-size)
- [Removing Lifetimes](#removing-lifetimes)
- [Removing Implicit Version/Help Behavior](#removing-implicit-version-help-behavior)
- [Storing `Id`s for `ArgGroup`](#storing-ids-for-arggroup)
- [Introspecting on `ArgMatches`](#introspecting-on-argmatches)
- [Non-bool Flags](#non-bool-flags)
- [Fixing Parsing for Hyphenated Values](#fixing-parsing-for-hyphenated-values)

You can also skip ahead to
- [Looking Forward](#looking-forward)
- [How Can I Help](#how-can-i-help)

<a id="support-policy"></a>
### Support Policy

Before getting into the details, I want to assure you that we understand the
effort required to deal with breaking changes.  We both maintain CLIs using clap and have to
deal with updating 500+ tests for clap.  We are trying to balance the needs of
clap users for whom clap is already good enough with those who want to keep using clap
but are discouraged by the build times, binary size, or some missing features.
I'm hoping that the improvements we made will help justify the changes.

With clap 3.0, we adopted a deprecation policy to help developers migrate to new
versions.  After feedback from developers using clap, we adjusted that policy soon after clap
3.2 so that deprecations are opt-in (via the `deprecated` feature flag) so you
can choose the timetable of when to respond to them without extra noise from
the deprecations.  We've also improved `clap_derive` to avoid deprecations
being reported for code it generates on your behalf.

The next change is that we are officially supporting old major versions:

| Version                                              | Status        | Support |
|------------------------------------------------------|---------------|---------|
| [v4](https://github.com/clap-rs/clap/tree/master)    | active        | Features and bug fixes target `master` by default |
| [v3](https://github.com/clap-rs/clap/tree/v3-master) | maintenance   | Accepting trivial cherry-picks from `master` (i.e. minimal conflict resolution) by contributors |
| [v2](https://github.com/clap-rs/clap/tree/v2-master) | deprecated    | Only accepting fixes for ecosystem-wide critical bugs |
| v1                                                   | unsupported   | \- |

We have not decided on when we will end-of-life each version; this is an experiment
that we are iterating on as we go.  This policy is trying balance the need for
developers to upgrade at their own pace with the burden for maintainers to support
old versions.

See also our [CONTRIBUTING.md](https://github.com/clap-rs/clap/blob/master/CONTRIBUTING.md)

<a id="polishing-help-output"></a>
### Polishing Help Output

| Before | After |
|-|-|
| ![screenshot](/img/clap4/git-diff-3.2.21.svg) | ![screenshot](/img/clap4/git-diff-4.0.0.svg) |
| ![screenshot](/img/clap4/git-stash-3.2.21.svg) | ![screenshot](/img/clap4/git-stash-4.0.0.svg) |

During clap 3.0.0 development, we were looking at removing
`AppSettings::ColoredHelp`, making it always on.  This helped identify several
problems with our existing colored help and started a broader re-examination of
our `--help` output.  For more background on each decision, I
recommend you check out the
[parent issue](https://github.com/clap-rs/clap/issues/4132)

Problems with the colored help
- The choice of colors was controversial, some liked it but others hated it.
  We want safe out-of-the-box defaults
- Some applications have a specific branding to their color choices and clap's
  colors don't fit within that, so they would rather have none instead
- If the developer was providing additional ASCII-formatted content, the
  partial coloring is glaring

In the short term, `6079a871`, `8315ba3d`, `e02648e6` starts us off with a more neutral color palette by
focusing on bold/underline, rather than colors.  Longer term, we need to allow
users to customize the colors
([#3234](https://github.com/clap-rs/clap/issues/3234))
and provide their own colored output
([#3108](https://github.com/clap-rs/clap/issues/3108)).
We plan to focus on this during clap 4.x.

We then extended the styling to the usage line as well as the general structure
of the help output.  This required dropping `textwrap` and included clean up
that helped offset the extra code size from the extra formatting logic. This
happened in `2e2b63fa..a6cb2e65`
- Code size: 562.7KiB → 566.0 KiB (+3.3 KiB)
- Lines of Code: 19,078 → 19,468 <span style="color:red">(+390)</span>

A frequent point of confusion for users is that `-h` shows an abbreviated help
and `--help` shows a detailed help.  `rg` and `bat` include messages in the
output explaining the distinction.  I realized we could bake this directly into
the description for `-h` and `--help`. This happened in `c1c269b4`
- Code size: 567.1 KiB → 568.0 KiB (+0.9 KiB)
- Lines of Code: 19,693 → 19,708 (+15)

When getting user feedback on `--help`, I noticed that their usage statement
was meaningless, something like `prog [OPTIONS] [ARGS]`.  While showing
everything can be too much (see `git -h`), showing too little negates the
reason for the usage in the first place.  We've changed the usage to never
collapse positional arguments into `[ARGS]` as that seems to strike a nice
balance.  Longer term, we should provide ways for users to force certain args
to not be collapsed or to allow specialized usages per case in an `ArgGroup`.
This happened in `f3c4bfd9..a1256a6e`
- Code size: 570.4 KiB → 567.1 KiB (-3.3 KiB)
- Lines of Code: 19,631 → 19,628 (-3)

We also
- When looking at non-clap CLIs to avoid any [NIH syndrome](https://en.wikipedia.org/wiki/Not_invented_here), the ALL CAPS headers
 started bothering me more and more, so we switched to title case in `9b23a09f`.
 I know some users appreciated the man page style formatting but I'm hoping this
 will be a more polished look for people less familiar with man pages.
- When looking at the help after this change, it just felt off seeing
 "Subcommand".  "Subcommand" is just a more specific case for "Command" and its
 rare to see others do this, so I went ahead and switched to "Command" in
 `42c94384`.
- In looking at other CLIs, I also noticed they tend to put subcommands front and
 center.  This makes sense since the parent commands are generally not usable on
 their own.  We followed suit in `83d6add9`.  One problem with putting subcommands
 first is when the help output scrolls off the screen.  This highlights an
 existing problem with clap help output.  We have started [investigating adding
 a pager to help output](https://github.com/clap-rs/clap/issues/4201).
- Made usage / arg displays more consistent in `02db3043`, `df76168`, `76dd3a18`
- Removed name/version/author from the default `Command::help_template` as it is mostly redundant in `65b5b5f7`
- Further condensed `--help` output by switching the template to have Usage on one line in `9a645d2d`
- Even further condense `-h` with `next_line_help` by removing the blank lines between arguments in `bbb6c38b`
- Aim for horizontal density by changing indentation from the less common 4 spaces to the more common 2 spaces in `65c28978..c95c9f2f`
- Respect `COLUMNS` when we can't detect terminal width in `1466cd5f..79321f25`

<a id="more-specific-derive-attributes"></a>
### More Specific Derive Attributes

We've added support for new derive attributes, deprecating the old ones.

Before:
```rust
/// Simple program to greet a person
#[derive(Parser, Debug)]
#[clap(author, version, about, long_about = None)]
struct Args {
    /// Name of the person to greet
    #[clap(short, long, value_parser)]
    name: String,

    /// Number of times to greet
    #[clap(short, long, value_parser, default_value_t = 1)]
    count: u8,
}
```

After:
```rust
/// Simple program to greet a person
#[derive(Parser, Debug)]
#[command(author, version, about, long_about = None)]
struct Args {
    /// Name of the person to greet
    #[arg(short, long)]
    name: String,

    /// Number of times to greet
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}
```

As we look forward to some planned `clap_derive` features (e.g.
[#1807](https://github.com/clap-rs/clap/issues/1807),
[#1553](https://github.com/clap-rs/clap/issues/1553)
),
the use of a single `#[clap(...)]` attribute is limiting.  In addition,
we have seen users frequently confused by how the derive and builder APIs
relate (
[#4090](https://github.com/clap-rs/clap/discussions/4090)
).  We are hoping that by migrating users to `#[command(...)]`,
`#[arg(...)]`, and `#[value(...)]` attributes, code will be clearer, the derive
will be easier to use, and we can expand on the capabilities of the derive API.

This happened in `2f4e42f2..ba5eec31` and `20ba828f..f5138629` and as they are
focusing on a code generating, they should have negligible impact on code size
either way.

This also provided an opportunity for us to do some much needed cleanup of `clap_derive`, including
- Reducing duplicate parsing in `a20c67c8..3ec2f0f7`
- Improved error and deprecation reporting in `ebe181ac..20ba828f`, `f5138629..7ed7f71a`, `7ed7f71a..088b396d`

<a id="num-args"></a>
### `num_args`

The new `num_args` defines how many command-line arguments should be consumed as values, per occurrence:
```rust
et cmd = Command::new("prog")
   .arg(Arg::new("file")
       .action(ArgAction::Set)
       .num_args(2)  // single value
       .short('F'))
    .arg(Arg::new("mode")
        .long("mode")
        .num_args(0..=1))  // range of values
        .default_value("plaid")
        .default_missing_value("slow");
```

Previously, clap has had several ways for controlling how many values will be captured
without always being clear on how they interacted, including
- `Arg::multiple_values(true)`
- `Arg::number_of_values(4)`
- `Arg::min_values(2)`
- `Arg::max_values(20)`
- `Arg::takes_value(true)`

These have now all been collapsed into `Arg::num_args`.
Most changes happened between `ee06707c..179faa6b`
- Code size: 564.4 KiB → 563.5 KiB (-0.9 KiB)
- Lines of Code: 17,854 → 17,946 (+92)

See
[Issue 2688](https://github.com/clap-rs/clap/issues/2688) for more background.

<a id="removing-deprecated-features"></a>
### Removing Deprecated Features

A lot of deprecations were merely the removal of functions.  Others involved
changing defaults, percolating the effect throughout, and simplifying the code
as a result.

The most fundamental changes were the introduction of
- [ArgAction](https://docs.rs/clap/latest/clap/enum.ArgAction.html) which provided more transparency into clap's behavior
- [Arg::value_parser](https://docs.rs/clap/latest/clap/builder/struct.Arg.html#method.value_parser) which made clap's validation more open ended, allowing the deprecation of a lot of built-in validators

Removing deprecated APIs were done in `16bd7599..017b87ab` ([#3963](https://github.com/clap-rs/clap/pull/3963)) and `017b87ab..ee06707c`
- Code size: 604.6 KiB → 564.5 KiB <span style="color:green">(-40.1 KiB)</span>
- Lines of Code: 23,254 → 17,854 <span style="color:green">(-5,400)</span>

This is in addition to the Builder API Surface shrinking dramatically, making
it easier to discover and use the features that clap provides.

<a id="reducing-code-size"></a>
### Reducing Code Size

Along the way some more specific actions were taken to reduce code size.

The first was to move off of `IndexMap`, `IndexSet`,
`HashMap`, and `HashSet` to a custom built `FlatSet` and `FlatMap`.  clap does
not deal in big numbers and can get away with just using a `Vec` and manually
enforcing uniqueness by iterating over it.  `FlatSet` codifies that approach.
`FlatMap` takes the same approach but uses separate `Vec`s for the keys and
values.

Benefit
- Removed some dependencies (`indexmap`, `hashbrown`)
- Minimizing the number of containers means more reuse when they get monomorphized
- Building on `Vec`s allow even more monomorphized reuse as a `FlatStr<Str>` is
  really just a `Vec<Str>`.  This is more acute for `FlatMap` since we store
  the keys and values separately.  A `FlatMap<Str, Vec<Str>>` is really a
  `(Vec<Str>, Vec<Vec<Str>>)`, allowing reuse between the key and value and any
  `FlatSet<Str>` or `Vec<Str>` in the code base.

This happened in `b344f1cf..084a6dff`
- Code size: 561.2 KiB → 525.2 KiB  <span style="color:green">(-36 KiB)</span>
- Lines of Code: 17,789 → 18,075  (+286)

I tried doing the parallel-`Vec` approach for our formatted output
(`Vec<(Style, String)>`) but it wasn't an immediate win, so I dropped that
effort as it didn't look like there was enough to optimize about my
proof-of-concept to get a noticeable win.

Some of our increases in binary size this release came from being more detailed
in our [help formatting](#polishing-help-output).  Code that previously wrote
to a `String` now worked with our formatted output (`Vec<(Style, String)>`).
My hope is we can re-work this data structure in
[a future 4.x release](https://github.com/clap-rs/clap/issues?q=is%3Aopen+is%3Aissue+milestone%3A4.x).
To see how much this will help reduce binary size, I switched to a straight
`String` when the `color` feature flag is disabled.

This happened in `e75da2f8`
- Code size: 542.7 KiB → 542.4 KiB (-0.3 KiB)
- Code size without `color`: 533.6 KiB → 518.2 KiB <span style="color:green">(-15.4 KiB)</span>
- Lines of Code: 20,426 → 20,470  (+34)

Next, we split the formatting of errors out and wrapped it in a trait, allowing
more naive implementations or just not rendering any details at all.  This is
an example of using traits for reducing size rather than throwing in more
feature flags which can be harder to maintain and harder for a developer to discover
and use well.

This happened in `956789c3..d219c69c`.  An application that switches from the
"rich" to the "null" formatter dropped code size by <span style="color:green">6 KiB</span>.  This won't show up
in our usual metrics as this is opt-in and the example we are using for
measuring does not opt-in.

When adopting `StyledStr`, we also needed some behavior that `textwrap` didn't
seem to provide, so we re-implemented a subset that fit our needs (incremental
wrapping).

This happened in `37f2efb0`
- Code size: 577.6 KiB → 565.1 KiB <span style="color:green">(-12.5 KiB)</span>
- Lines of Code: 19,250 → 19,483  (+233)

In cleaning up `clap_derive`, we found some redundant generated code that we
cleaned up in `0b5f95e3`.  Our usual benchmark doesn't cover this case.  I ran
another one and didn't see any noticeable change, so congrats to the compiler
optimizing it?

Looking for another way to break down where all of the code size for clap comes
from, I split out auto-generated `help`, auto-generated `usage`, and `error-context` features.

This happened in `94c5802a..c26e7fd6`
- Code size: 542.9 KiB → 544.3 KiB (+2.6 KiB)
- Lines of Code: 20,312 → 20,653  (+341)

Breaking clap down by feature:
- Removing auto-generated `help`: 495.3 KiB <span style="color:green">(-49.0 KiB)</span>
- Removing auto-generated `usage`: 520.0 KiB <span style="color:green">(-24.3 KiB)</span>
- Removing `color`: 504.0 KiB <span style="color:green">(-40.3 KiB)</span>
- Removing `suggestions`: 509.1 KiB <span style="color:green">(-35.2 KiB)</span>
- Removing `suggestions`+`error-context`: 489.2 KiB <span style="color:green">(-55.1 KiB)</span>
- Removing all of the above: 422.6 KiB <span style="color:green">(-121.7 KiB)</span>

<a id="removing-lifetimes"></a>
### Removing Lifetimes

In the most common case, a developer never sees that clap borrows data or deal with
the lifetimes on the data types.  You just use the builder or derive and it is
all taken care of for you and is fairly fast with low code size.

However, when you do notice them they can be a pain to deal with.  Consider these use cases
- An application with a `--log <PATH>` parameter that defaults to a filename containing the 
  current timestamp.  With the builder API, this can be frustrating to return
  the `Command` from a function and more so with the derive API as it only
  supports `'static` lifetimes.  Your only options is to
  work around this with `once_cell` or leaking the memory
- An application that generates the `Command` from a file using
  [clap_serde](https://docs.rs/clap-serde/latest/clap_serde/)

When doing this, we wanted to use newtypes to allow
- Flexibility in changing how we optimize the strings (use `String`,
  `Cow<'static, str>`, etc), so we created `Str` and `OsStr`
- Clarify some distinctions in the API and offer more type safety between them,
  so we created `Id` for `Arg::new()` and `ArgGroup::new()`
- Allow developer-provided formatted content in
  [a future 4.x release](https://github.com/clap-rs/clap/issues?q=is%3Aopen+is%3Aissue+milestone%3A4.x),
  so we created `StyledStr`

To keep things convenient for the common case, we updated our APIs to accept
`impl Into<Str>` and `impl Into<OsStr>`.  This gave us the opportunity to
re-evaluate APIs that accepted a `&'static str` instead of a `&'static OsStr`
or had separate `str` and `OsStr` versions to clean this all up.

The use of `impl Into<Str>` did run into problems with the few places we had
`impl Into<Option<T>>` for resetting of fields because `None` was ambiguous on
what type it was coming from, so we created the `IntoResettable` trait to
resolve these ambiguities.

`a5494573..5950c4b9`
- Code size: 525.2 KiB → 553.4 KiB <span style="color:red">(+28.2 KiB)</span>
- Lines of Code: 18,075 → 18,460 <span style="color:red">(+385)</span>

`e81b1aac..c6b8a7ba`
- Code size: 553.4 KiB → 565.7 KiB <span style="color:red">(+12.3 KiB)</span>
- Lines of Code: 18,524 → 18,873 <span style="color:red">(+349)</span>

`d4ec9ca5..956789c3`
- Code size: 565.7 KiB → 563.6 KiB (-2.1 KiB)
- Lines of code: 18,876 → 18,883 (+7)

We originally accepted `String` in the API and had a `perf` feature flag to
control whether you prefer code size or runtime performance.  In looking back
at these numbers, the cost of accepting `String`s in clap's API seemed too high
for the how often this feature would be used.  We instead shifted it so clap
accepts only `&'static str` by default and the `string` feature flag changes
the API to accept `String`s as well, restoring most of the code size and runtime
performance from before we made any of these changes.

`1365b088..6dc8c994`
- Code size: 567.2 KiB → 541.1 KiB <span style="color:green">(+26.1 KiB)</span>
- Lines of code: 20,233 → 20,278 (+45)

As I mentioned, we have to support some fields being resettable.  We've made
this more consistent by allowing nearly all fields to be reset in
`ee387c66..7ee121a2`

<a id="removing-implicit-version-help-behavior"></a>
### Removing Implicit Version/Help Behavior

clap 3.0.0 tried to make `--version` and `--help` flags as easy to modify as possible
- You could use the built-in behavior
- You could tweak their behavior with `cmd.mut_arg("help", |arg| arg.help("Custom message"))` and have that propagated to subcommands
- You could provide your own, implicitly disabling the built-in help flag
- Along with providing your own, you can explicitly disable the built-in help flag

All of those automatically show help output when present which you could disable, making it a regular flag.

Magic, while making some cases easy, also makes it difficult to divine the
rules and force it to do what you want.  This had come up several times in
supporting developers but [Issue #3405](https://github.com/clap-rs/clap/issues/3405)
was the tipping point for re-thinking how we do this.

Now, things are simplified down to
- You can use the built-in behavior
- You can disable the built-in behavior explicitly with
  `cmd.disable_help_flag(true)` and provide your own flag.  For your custom
  help flag to show clap's help automatically, call `arg.action(ArgAction::Help)`.

Most changes happened between `0129d99f..6bc8dfd3`
- Code size: 563.5 KiB → 561.6 KiB (-1.9 KiB)
- Lines of Code: 17,944 → 17,789 (-155)

Specifically `f70ebe89` is where almost all of the parse speed gains happened
between clap v3 and clap v4, returning clap back to clap v2's performance.

<a id="storing-ids-for-arggroup"></a>
### Storing `Id`s for `ArgGroup`

Something that can easily be missed in clap is that when we are parsing and
store a value in `ArgMatches`, we also store that value in all `ArgGroup`s that
the arg is a part of.
- This can really balloon the  number of allocations performed.  Say you put a
 positional argument into a group and then pass hundreds of arguments into your
 command with `xargs`, we've now done a lot of extra clones that you likely
 didn't care about.
- We are also wanting to better integrated `ArgGroup` into `clap_derive` (
  [#3165](https://github.com/clap-rs/clap/issues/3165) 
  [#4211](https://github.com/clap-rs/clap/issues/4211),
  [#2621](https://github.com/clap-rs/clap/issues/2621)
  ).
  There isn't a way for `clap_derive` to know which variant
  of an enum it should match.

So we've changed things up and now for each occurrence (not value) of an
argument, we are storing its arg `Id` in the `ArgGroup`, reducing the number of
allocations involved (since an `Id` might not even be on the heap) and making
it easier to make programmatic decisions about the argument.

This happened in `41be1bed`
- Code size: 550.0 KiB → 550.7 KiB (+0.7 KiB)
- Lines of Code: 18,057 → 18,062 (+5)

<a id="introspecting-on-argmatches"></a>
### Introspecting on `ArgMatches`

A fairly regular request is for iterating over the arguments in `ArgMatches`,
whether for
[re-organizing the data by order of arguments for commands like `find`](https://github.com/clap-rs/clap/blob/master/examples/find.rs#L27)
or for
[layered configuration](https://github.com/clap-rs/clap/discussions/2763).
This was blocked on how we were storing argument `Id`s.  During the development
of clap 3.0, `Id`s were switched from a string to a hash of the string.  This
removed the need for a lifetime on `ArgMatches`, reduced memory overhead, and
opened the door for allowing alternative `Id` types in the future.  The problem
is that it is a one way process (can't get a string back from the hash) and the
way the traits were laid out it made it error prone (easy to accidentally
look up a hash of a hash).

After re-examining the alternative `Id` types feature request
([#1104](https://github.com/clap-rs/clap/issues/1104)),
we decided to not move forward with that route.  This meant we could go back to
strings, especially with our previous lifetime changes, and make it fairly
ergonomic to allow people to enumerate the arguments that were parsed.

This happened in `7486a0b4` though it was dependent on a batch of the lifetime
work mentioned earlier.  See the lifetime-removal work for metrics.

<a id="non-bool-flags"></a>
### Non-bool Flags

Sometimes you want a flag to convey something more specific than
`true`/`false`, like a `--https` flag tracking the relevant port:
```rust
let cmd = Command::new("mycmd")
    .arg(
        Arg::new("port")
            .long("https")
            .action(clap::ArgAction::SetTrue)
            .value_parser(
                value_parser!(bool)
                    .map(|b| -> usize {
                        if b { 443 } else { 80 }
                    })
            )
    );

let matches = cmd.get_matches_from(["mycmd", "--https"]);
let port = matches.get_one::<usize>("port").copied();
assert_eq!(port, Some(443));
```

Originally, this was possible in `clap_derive` but not with the builder API.
While the `value_parser` / `ArgAction` APIs brought a lot of features from
`clap_derive` to the builder API, this is one area where it regressed.

I had noticed we could re-work `SetTrue`s semantics to reuse existing machinery
within clap to accomplish this:
- Users can now override `SetTrue`s default `bool` value parser
- Developers can now rely on either 
  - Overriding `SetTrue` defaulting `arg.default_value("false")` (flag absent)
    and `arg.default_missing_value("true")` (flag present) with custom strings
    to be parsed by the value_parser
  - Adapt a `bool` value parser via `ValueParser::map`

This happened between `5950c4b9..e81b1aac`
- Code size: 553.4 KiB → 553.4 KiB (0)
- Lines of Code: 18,460 → 18,524 (+64)

<a id="fixing-parsing-for-hyphenated-values"></a>
### Fixing Parsing for Hyphenated Values

clap supports parsing values that look like flags, whether they are numbers or
flags to forward to another command.  This initially applied to all arguments
in a `Command` with `AppSettings::AllowLeadingHyphen`.  A more `Command`-wide specialization
was added with `AppSettings::AllowNegativeNumbers`.  Then we got an argument-specific
`ArgSettings::AllowHyphenValues`.  Unfortunately, the latter missed some cases in
`AppSettings::AllowLeadingHyphen`.  Developers were finding the `Arg`-specific one first
and running into road blocks.  We updated the now `Arg::allow_hyphen_values` to
operate like the `Command` version.

In working on this, it gave us an opportunity to re-evaluate this API.
Normally, a single `Arg` needs to accept hyphen values, especially numbers, and
it can lead to unexpected behavior to enable it on all `Arg`s via a `Command`
setting.  We've now mirrored `Command::allow_negative_numbers` to `Arg` and
deprecated all of the `Command` variants of these settings.  We then did
similar for a closely related setting, `Command::trailing_var_args` which
brings it inline with its related `Arg::last`.

This happened in `f731ce70..f97670ac`
- Code size: 568.4 KiB → 572.1 KiB (+3.7 KiB)
- Lines of Code: 19,900 → 20,041 (+241)

<a id="looking-forward"></a>
## Looking Forward

More immediately, I plan to focus on an experiment with how we style the
output.  clap has had several challenges in supporting more customization
- To customize the existing palette, we'd have to expose `termcolor` in the public API, requiring a breaking change to switch away
- To accept styled content, we've have to expose an entire styling API that delays applying the styles until rendering

For the first, I've started work on
[anstyle](https://docs.rs/anstyle/latest/anstyle/) (ANSI Styling) for providing
a common definition of ANSI styles with a minimal API so it is safe to put in
public APIs (once it hits 1.0).

The second is because of a fundamental design restriction in existing terminal
styling crates; they couple together styling with terminal capabilities.  I'm
going to experiment with an alternative route where the user styles their
output with their preferred ANSI styling crate and then the output stream
adapts it as needed, whether that means stripping the escape codes when piping
to a file or turning them into wincon API calls for older Windows versions.
The biggest risk is the performance hit when needing to adapt the output.  My
hope is that the cost will be acceptable for most applications.

In addition to this being a low maintenance way of accepting styled output, my
hope is that this will simplify clap's rendering code to get the same size
benefits as our [optimization for `StyledStr` without the `color`
feature](#reducing-code-size) which saved 15 KiB in code size.

Speaking of code size, while we've made reductions during clap 4.0, our work is still not done
(
[#2037](https://github.com/clap-rs/clap/issues/2037),
[#1365](https://github.com/clap-rs/clap/issues/1365)
).  Seeing the success of clap v4, we are optimistic that [a more open API
design](https://github.com/clap-rs/clap/discussions/3476) will continue to
reduce code size while making clap more flexible.

<a id="how-can-i-help"></a>
## How Can I Help

First, we welcome any feedback on what is going into clap 4.0.0 which is why we
are pre-announcing so we have opportunities to re-evaluate based on feedback.
It can be difficult to weigh feedback on Issues as the audience tends to be
small and self-selecting to those who agree, so we recognize the need for
finding ways to remove that bias and collect feedback from a wider audience.

Second, we would love for you to share what is great or what doesn't work well
with other CLI parsers so we can learn from all of the best and avoid
[NIH syndrome](https://en.wikipedia.org/wiki/Not_invented_here).  For example,
I've started
[digging into click's documentation](https://github.com/clap-rs/clap/discussions/3454)
to see what works well and how it might apply to clap.  This has already led to `value_parser!`s design being made extensible so we could support in it and `clap_derive` features like
[`FileReader` and `FileWriter`](https://github.com/clap-rs/clap/issues/4074).

Lastly, of our
[help-wanted Issues](https://github.com/clap-rs/clap/issues?q=is%3Aopen+is%3Aissue+label%3AE-help-wanted),
the one I'm most excited for is
[#3166](https://github.com/clap-rs/clap/issues/3166) as it will make
completions more featureful, less buggy, and more consistent across shells.

<a id="methodology"></a>
## Methodology

- `master` as of `16bd7599..c26e7fd6`
- Baseline is a simple program that reads all arguments and prints them back out
- API Surface:
  - `rg -U '\([\s]*(mut )?self,'` for `Command`, `Arg`, and `ArgGroup`
  - `rg '=> Flags::'` for `AppSettings` and `ArgSettings`
  - The 3.0.0 numbers are particularly bloated as setters were added for each `AppSettings` / `ArgSettings`
- Lines of Code: `tokei src clap_derive/src`
  - `clap_derive` didn't exist for clap 2 and accounts for 3,000-4,000 lines of code
  - While LoC is a flawed metric, we can get a rough feel for compile times with it
- Code size: `.text` from `cargo bloat --release --features cargo --example cargo-example`
  - This includes more than clap, like the example code itself, `std`, and any other dependencies
- Runtime: `cargo bench --bench 06_rustup -- parse_rustup_with_sc`
  - When removing lifetimes from the API, we traded off speed for binary size.
    Using the `--features clap/perf`, the time changes to 11.790us at the cost
    of 10 KiB in `.text` size (for `cargo-example`).

<!--
$ rg -U '\([\s]*(mut )?self,' src/app/mod.rs src/args/group.rs src/args/arg.rs --stats
112
$ rg '=> Flags::' src/*/settings.rs  --stats
62

$ rg -U '\([\s]*(mut )?self,' src/build/app/mod.rs src/build/arg/mod.rs src/build/arg_group.rs  --stats
172
$ rg '=> Flags::' src/build/*/*settings.rs  --stats
73

$ rg -U '\([\s]*(mut )?self,' src/build/app/mod.rs src/build/arg/mod.rs src/build/arg_group.rs  --stats
209
$ rg '=> Flags::' src/builder/*settings.rs  --stats
73

$ rg -U '\([\s]*(mut )?self,' src/builder/command.rs src/builder/arg.rs src/builder/arg_group.rs  --stats
165


fn main() {
    let args = std::env::args_os();
    for arg in args {
        println!("{:?}", arg);
    }
}
-->
