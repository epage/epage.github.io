---
title: "clap 3.1: A step towards 4.0"
published_date: "2022-02-16 18:15:04 +0000"
layout: default.liquid
is_draft: false
data:
  tags:
    - programming
    - rust
---
[clap 3.1](https://docs.rs/clap/3.1.0) is here!  Clap is a CLI argument parser
for Rust and the v3.1 releases focuses on API cleanup slated for clap 4.0.  See
the [CHANGELOG](https://github.com/clap-rs/clap/blob/master/CHANGELOG.md) for
details.

clap 3.0 was in development for 4 years and though we saw comparisons to
Half-life 3 in response to the release, we also saw people who cited the long
gaps between breaking releases as a motivation for using it.  For clap to stay
relevant we feel we need to avoid the stagnation of long release cycles while
keeping things smooth for the users where clap is already "good enough".  The
v3.1 release is a major step in trying to strike that balance.

<!-- more -->

### clap's Model of Evolution

Traditionally, all features have been self-contained within the `clap::App`.
Customization has been done through runtime flags and cargo feature flags.
When a breaking change is needed, we batched them up into a large release
with just a CHANGELOG to help you making it easy to miss the more subtle
changes among basic transformations of "X was renamed to Y".

This has led to an ever growing API, a bloated code base, and gated evolution
of user applications.  

For example, clap's `--help` flag can:
- Print auto-generated help
- Print help from a bespoke template language
- Print a hard coded string
- Be turned off completely

Its unlikely the compiler can detect which you are using and compile out the
other implementations.  So even though you will only ever use one of these, we
have to compile support for all of them and include all of them in the final
binary.  Its no wonder clap 
[takes ~3x as long to compile as other parsers and results in ~20x the binary size](https://github.com/rust-cli/argparse-benchmarks-rs).
**This frequently gets excused as "clap provides a lot for you" but I feel there
needs to be a better way, that a CLI parser can be as powerful as you need
without paying for what you don't need.**

### A better way?

We want to be able to continue to extend clap while:
- You pay for what you use
- Features are discoverable
- Clap's maintainers are not blocking meeting your needs or exploring new ideas
- Minimal upgrade impact

Towards the end of clap 3's development, we started to look into how to
minimize upgrade impact.  We decided to leverage the compiler's deprecation
messages to take care of the more mechanical changes so we can focus the
CHANGELOG on the more subtle changes that might be missed otherwise.  We spent
a lot of time adding back in removed APIs, providing a smooth transition path
from [structopt](https://github.com/TeXitoi/structopt), etc.  I feel like this
was an overall success, both from my own experience of upgrading my
applications to clap 3 and from feedback we got from users about how smooth the
process was.

As an example of new feature development, we had
[a request](https://github.com/clap-rs/clap/issues/1693)
for [response-file support](https://docs.microsoft.com/en-us/windows/win32/midl/response-files).
Following the existing patterns, this would have been built directly into
clap's parser.  We might satisfy the immediate request but it opens the door
for people wanting it to fill their needs.  Some prior art shows there are uses
for [customizing the prefix character](https://docs.python.org/3/library/argparse.html#fromfile-prefix-chars).
We'd also need to adapt to support different syntaxes in the wild, including whether:
- comments are supported (and what syntax)
- lines are treated literally or quoting and escaping is needed
- response files are processed recursively and how to resolve relative paths

Instead, we created the [argfile](https://github.com/rust-cli/argfile) crate.
- You are not restricted to using it with clap
- You can customize it how you wish
- You only pay for it if you need it
- We only need to document the need it fills instead of bloating `App`s documentation with all of customization people want

### clap Moving Forward

Our new guidelines in clap's [CONTRIBUTING.md](https://github.com/clap-rs/clap/blob/master/CONTRIBUTING.md):

> Our releases fall into one of:
> - Major releases which are reserved for breaking changes
>   - Aspire to at least 6-9 months between releases
>   - Remove all deprecated functionality
>   - Try to minimize new breaking changes to ease user transition and reduce time "we go dark" (unreleased feature-branch)
> - Minor releases which are for minor compatibility changes
>   - Aspire to at least 2 months between releases
>   - Changes to MSRV
>   - Deprecating existing functionality
>   - `#[doc(hidden)]` all deprecated items in the prior minor release
> - Patch releases
>   - One for every user-facing, user-contributed PR (i.e. release early, release often)
> 
> If your change does not fit within a "patch" release, please coordinate with the clap maintainers for how to handle the situation.
> 
> Some practices to avoid breaking changes
> - Duplicate functionality, with old functionality marked as "deprecated"
>   - Common documentation pattern: `/// Deprecated in [Issue #XXX](https://github.com/clap-rs/clap/issues/XXX), replaced with [intra-doc-link]`
>   - Common deprecation pattern: `#[deprecated(since = "X.Y.Z", note = "Replaced with `ITEM` in Issue #XXX")]`
>   - Please keep API addition and deprecation in separate commits in a PR to make it easier to review
> - Develop the feature behind an `unstable-<name>` feature flag with a stablization tracking issue (e.g. [Multicall Tracking issue](https://github.com/clap-rs/clap/issues/2861))

While we expect this to evolve over time, we feel this is a good start to helping meet the needs from earlier.

### What this means for clap 3.1

In v3.1, you'll find that we've changed a large swath of the API with
deprecations.  We looked to our
[breaking-change issues for v4.0](https://github.com/clap-rs/clap/issues?q=is%3Aopen+is%3Aissue+milestone%3A4.0+label%3AM-breaking-change)
and focused on a common theme of changes that can be implemented now by
introducing a new API and deprecating an old one.  This allows us to develop
iteratively and allow people to migrate in piecemeal rather than relying on
large flag-day upgrades.

One catch I've found with this is how I've been turning warnings into errors in CI.  I've been running something like:
```console
$ cargo clippy --workspace --all-features --all-targets -- -deny warnings
```
and didn't pay attention to the fact that my CI will start failing if a dependency marks
an item as deprecated.  Without a lock file, your PRs might see failures
unrelated to the change at hand.  With a lock file, you will be forced to
respond to all deprecations just to upgrade.  Some see this as a good thing but
I find the higher the cost of an upgrade, the more people avoid them and the
slower you move.  Being able to decouple the upgrade and the migration can be a
big help for evolving your code.

I'd recommend changing your CI to:
```console
$ cargo clippy --workspace --all-features --all-targets -- --deny warnings --allow deprecated
```

### Participating in the Conversation

If you'd like to look over what we are hoping to accomplish in the future,
check out our [milestones](https://github.com/clap-rs/clap/milestones).  They
represent our areas of focus and aren't exclusive of other work happening).
We are also doing a [cross-issue brainstorming](https://github.com/clap-rs/clap/discussions/3476)
of what clap's future API might look like.  **Please pipe in with any feedback; we
recognize that hearing your ideas is likely the only way to meet our ambitious
goals!**
