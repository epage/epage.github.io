---
title: "clap 3.0, a Rust CLI argument parser"
published_date: "2021-12-30 09:00:30 -0500"
data:
  tags:
    - programming
    - rust
---

I figured a great way to close out the year 2021 is to wrap up the long awaited
[clap 3.0 release](https://github.com/clap-rs/clap/blob/master/CHANGELOG.md#300-2021-12-31)!

Some major milestones along the way:
- Jan 24, 2018: The [first commit in the v3-dev branch](https://github.com/clap-rs/clap/commit/acdbd47152102607b7f4c6702cc076caca771280)
- Aug 30, 2019: [StructOpt 0.3 is released](https://github.com/TeXitoi/structopt/blob/master/CHANGELOG.md#v030-2019-08-30) with better clap integration
- May 03, 2021: [v3.0.0-beta.1 is released](https://github.com/clap-rs/clap/releases/tag/v3.0.0-beta.1)
- Dec 08, 2021: [3.0.0-rc.0 is released](https://github.com/clap-rs/clap/releases/tag/v3.0.0-rc.0)
- Dec 31, 2021: [3.0.0 is released](https://github.com/clap-rs/clap/releases/tag/v3.0.0)

Thanks to:
- kbknapp, pksunkara, dpc, killercup, spacekookie, yosh, ldm0, and any other
  maintainers or contributors along the way
- Our users, especially those providing feedback on beta and release-candidates
- Embark, Sentry, repi, and many other sponsors
- My employer, Futurewei, for giving me the opportunity to help wrap up clap 3.0

<!-- more -->

For users who helped us through testing, see our
[release-candidate changelog](https://github.com/clap-rs/clap/discussions/3110)
and
[beta changelog](https://github.com/clap-rs/clap/discussions/3100)

## Port Status

Ported
- [clap-cargp](https://crates.io/crates/clap-cargo)
- [clap-verbosity-flag](https://crates.io/crates/clap-verbosity-flag)
- [argparse-benchmarks-rs](https://github.com/rust-cli/argparse-benchmarks-rs)

Remaining:
- [The CLI book](https://rust-cli.github.io/book/index.html)

## Highlights

Everyone will have their own favorite aspect of the release.   For me, they include:

**Reducing Gotchas**

With `StructOpt`, I used a `arg: Vec<T>` and thought I got what I wanted:
collecting each flag's value into a `Vec` (`--arg alice --arg bob`).  What I
didn't expect is it also supported an unbounded number of arguments per flag
(`--arg alice bob`).  StructOpt's `Vec<T>` mapped to
[`Arg::multiple`](https://docs.rs/clap/2.33.3/clap/struct.Arg.html#method.multiple).
In clap 3, these have been split into separate concepts:
[multiple_occurrences](https://docs.rs/clap/3.0.0/clap/struct.Arg.html#method.multiple_occurrences) (what I wanted)
and
[multiple_values](https://docs.rs/clap/3.0.0/clap/struct.Arg.html#method.multiple_values)
(what surprised me).  Now, `arg: Vec<T>` maps to `multiple_occurrences` though
you can enable `multiple_values` if you want.

There are other examples I've seen as I've supported clap users but I don't remember enough to be able to enumerate them.

**[StructOpt](https://docs.rs/structopt/) Integration**

[StructOpt](https://docs.rs/structopt/) provides a serde-like declarative
approach to defining your parser.

As a user, I didn't mind using `structopt` as a separate crate.  As I've started
maintaining clap, I found the integration provided a missing feedback loop to
ensure new features weren't just capable of being exposed as a `derive` macro
but fit natural with how people used a `StructOpt`.  Custom help headings were
one particular area where there was a lot of iteration.

**Custom Help Headings**

Most of the CLIs I've made have been for work and have been in Python with
argparse.  One of the aspects I always missed was being able to categorize the
help so I can highlight the important arguments and shove into a corner the
unimportant.  clap finally has this feature!

```rust
#[derive(Debug, Clone, Parser)]
struct Cli {
    #[clap(long, help_heading = "CONFIG")]
    isolated: bool,

    #[clap(long, parse(from_os_str), help_heading = "CONFIG")]
    config: std::path::PathBuf,

    #[clap(short, long, help_heading = "DEBUG")]
    verbose: bool,
}
```
which produces
```bash
$ cargo run -- --help
test-clap 

USAGE:
    test-clap [OPTIONS] --config <CONFIG>

OPTIONS:
    -h, --help    Print help information

CONFIG:
        --config <CONFIG>
        --isolated

DEBUG:
    -v, --verbose
```

## Lessons Learned

I feel like a 4 year release cycle can't be passed up without talking about
what could be improved.  I can't speak for all of the other maintainer's along
the way, so this will be my personal experience and interpretation.  I'm sorry
if I misattribute anything.

**Maintainer Availability**

[![XKCD: Dependency](https://imgs.xkcd.com/comics/dependency.png)](https://xkcd.com/2347/)

For most, open source is done on the side and personal obligations take
priority.  I'm grateful that Foundation sponsors are hiring Rust developers to
keep things progressing and that
[the Foundation](https://foundation.rust-lang.org/news/2021-12-09-news-rust-foundation-to-launch-community-grants-program/)
and
[DevX](https://medium.com/concordium/the-devx-initiative-sponsorship-program-goals-and-principles-e640063eeaa7)
are exploring ways of more sustainable open source.

I'm also grateful the [WG-CLI](https://github.com/rust-cli/team/) stepped in
from time to time to help keep things moving forward.  My personal life took
over for me, so it originally prevented me from helping as part of these
efforts.  In working for Futurewei, its opened things up so I could step in and
help out while still maintaining my obligations in my personal life.
Unfortunately, WG-CLI has also mostly gone into maintenance mode.

Some ideas I want to play with for improving things further:
- Take a mentorship-first approach: Act as if the ideal state for any Issue is
  that it will be labeled `help wanted`, leaving the door open to contributors
  rather than taking a maintainer-first approach.
- Explore Bevy's concept of
  [maintainer focus](https://github.com/bevyengine/bevy/blob/main/CONTRIBUTING.md#what-were-trying-to-build)
  for reducing maintainer task-switching and risk of burnout.

**Avoiding Breaking Compatibility**

Earlier in the life of Rust's community, it felt like there was an aura around
maintaining compatibility.  When the next breaking release is an indefinite
time away, it puts pressure on the current release to be "perfect", to slip in
every breaking fix possible.  There is always something to improve though.

Thankfully we've come to better recognize when we need to avoid breaking
compatibility and when it is more acceptable (e.g. if types are so called
"vocabulary terms", being used for interop between crates).  Its ok to release
3.0 now and push off some of those improvements to 4.0 because its not going to
be that far away.

**Holding Onto Pets**

Even when being willing to break compatibility, it can be too easy to have a
pet improvement you want to make but that isn't critical to the release.

We need to be willing to say "not yet".  This can be hard.  One pet I gave into
before I hardened myself against them was getting `Arg::help_heading` to work
well with `derive`.  Once I recognized what I was doing and started watching
for it, I found many dear pets that I had to say "not yet" to or "only the
minimal amount to unblock this fix".  It hurt.  I hated seeing an area to
improve and passing it up at the risk of forgetting but I also knew that if I
gave in, I risked pushing back the 3.0 release further.

Another method for dealing with pets is to timebox them.  Its too easy for one
improvement to uncover another problem or to introduce a regression.  Setting a
time for how long you are willing to let this continue and then deciding how
much to leave in is an important tool.  A recent example of this is that clap
3.0 originally removed a hard-break token in help (`{n}`) because users of the
Builder API could just use `\n`.  The problem is the derive API has a
poor-man's markdown parser for detecting hard breaks and it doesn't work well,
so people have had to rely on `{n}`.  We looked into our options for a period
of time before decided to add `{n}` back in to give ourselves more time for
resolving it.

And finally, there is using feature flags to isolate your pets from impacting a
release.  We took several features that we felt weren't ready and put them
behind `unstable-` prefixed feature flags.

**Long Release Cycles**

Several of the others help lead to a long release cycle but having a long
release cycle is a problem in of itself.

People keep having ideas and want to keep contributing.  Halting development
for years would just kill any investment people have in contributing.  This can
introduce regressions and there isn't the forcing function of a release to make
sure these features are polished enough.

To help, we introduced our stablization process with `unstable-` feature flags
with
[stablization issues](https://github.com/clap-rs/clap/issues?q=is%3Aopen+is%3Aissue+label%3AC-tracking-issue).

We also had to take the hard stance of treating `master` is if we would release
it any day, rather than one day.  This raised the bar for what we'd accept for
contributions.

In the end, as we got to our release-candidate phase, we did introduce a
feature freeze but we knew it had a limited time frame (about a month).

## Plans for the Future

This will be different to each person though applying
[Bevy's "focus" goal](https://github.com/bevyengine/bevy/blob/main/CONTRIBUTING.md#what-were-trying-to-build)
should lead to some alignment over time.

For me, I feel like clap has
[let a 1,000 flowers bloom and is ready to rip 999 of them out by the roots](https://gigamonkeys.com/flowers/).
clap has grown organically and has a lot of built-in features.  Each new
feature makes it harder to discover every other feature (I never knew half of
what clap could do until I started contributing).  All of these features are
also built-in, controlled by runtime flags.  This makes it harder for the
compiler to identify dead code, requiring compiling and including nearly
everything into the final binary.  This slows down compile times and bloats
binary size.

This doesn't necessarily mean we'll be removing features.  There might be some
shifting of features where we keep the common case easy but still make less
common cases possible.  The main focus for this will instead be on making clap
more modular.  In part, this will be done by splitting out building blocks like
a
[lexer](https://github.com/clap-rs/clap/issues/2915) or
[help generation](https://github.com/clap-rs/clap/issues/2913).  These won't do
much on their own but enable changes down the road for customizing clap without
a flag for every detail like
[changing out the parser to support different CLI
conventions](https://github.com/clap-rs/clap/issues/1210) or allowing
[build-time help generation](https://github.com/clap-rs/clap/issues/2914).

These changes are a step towards making the API open ended; more of a library
of tools rather than a framework.  Moving more of the [validation logic out of
clap](https://github.com/clap-rs/clap/issues/3008) will also be a step in that
direction.
