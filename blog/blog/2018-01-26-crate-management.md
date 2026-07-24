---
title: "Crate Management for #rust2018"
published_date: "2018-01-26 04:20:00 +0000"
is_draft: false
data:
  tags:
    - programming
    - rust
---
*Context: [Call for Community Posts](https://blog.rust-lang.org/2018/01/03/new-years-rust-a-call-for-community-blogposts.html) and [other posts](http://readrust.net/rust2018/)*

<!-- more -->

## Obligatory 2017 Reflection

When I started with Rust, I was planning on blogging about my experience. As I
learned though, I found most of the documentation I needed was in blog posts
that, though helpful, would give different suggestions without a hint as to why
they solved the problems differently.  This is frustrating for someone new
because you just want to get your task done and not have to first research a
whole other area.  I realized that the more valuable route for me would be in
improving documentation and tooling to serve as living documentation. I've even
taken this to try to find homes for some of the blog posts I bookmarked. So far
my only contribution has been to documenting [swapping `Option<Result<_>>` to
`Result<Option<_>>`](https://rustbyexample.com/error/multiple_error_types/option_result.html)
(which might get replaced with
[`transpose`](https://github.com/rust-lang/rust/issues/47338)) and documenting
[`Result`s strategies for
collections](https://rustbyexample.com/error/iter_result.html).

My area of biggest frustration has been in crate management. In a ecosystem
where small, single-purpose crates are encouraged, the need is even higher for
ensuring our crates are treated as cattle and not pets. Unfortunately, it
hasn't worked out that way for me. I've been trying to leverage my CI for
enforcing code quality, simplifying releases, etc but I've spent too much time
this year fighting my CI. In addition, the level of work to upgrade that parts
(like the CI config template or warning lists) is too high.

## Improving Crate Management

This is not as much a goal I'm wanting to suggest for the wider Rust ecosystem
in 2018 but more of a personal goal I have that I know I can't do alone.  It
involves cooperation from crate maintainers, people with more familiarity with
best practices than me, people to help refine my ideas, and people with skills
and resources I lack.

My goal is to make crate management passive so people can keep their crates
"modern" with little to no work.
- If it can be automated, it should
- Easy to update dependencies
- Easy to update CI tools
- Easy to modernize CI processes

I'm glad to see [I'm not the only
one](https://internals.rust-lang.org/t/the-libs-team-mission/6584/10) who sees
there is room for improvement here.

### Warning Epochs

Compiler and clippy warnings are a great way to maintain high quality code. The
challenge is in maintaining the set of enabled warnings.  You can
`#![deny(warnings)]` but that means your [build will break every time a new
warning is
introduced](https://github.com/rust-unofficial/patterns/blob/master/anti_patterns/deny-warnings.md).
So the recommendation is to instead explicitly list all of your warnings.  The
list of warnings is long and has to be listed in every `lib.rs`, `main.rs`, and
test `.rs` file.  You then have to poll the output of `rustc` to know when new
warnings are added, and update all of the relevant files in all of the crates
you maintain.

An easy way to handle this is for this to be a part of the [epochs
plan](https://github.com/rust-lang/rfcs/blob/master/text/2052-epochs.md).
Warning groups (including `warning`) could remain unchanged during epoch.  New
warnings could be added to `*-unstable` groups that will be stabilized at the
next epoch. This might seem to be an abuse of epochs because warnings are not
part of the language definition but I feel the benefit out weighs the possible
misuse.

This should also apply to clippy.

### Automate the Boring Stuff

The community already has [`bors`](https://bors.tech/) to ensure quality on
high traffic projects.  There is more we can do here though.

I've not written many web apps nor have the infrastructure to run such a bot.
I also know this is just the tip of the iceberg of licensing and people more
familiar with the topic (like distribution notices). This is definitely an area
where collaboration would be appreciated.

#### Dependency Management

One source of inspiration is [pyup](https://pyup.io/) for Python which will
create PRs to update your dependencies.  This will let you easily find the
changelog, review the test results of the upgrade, and easily select what to
do.  This would be quite trivial to adapt to Rust for projects that have a
`Cargo.lock` checked in.  It might even be safe to include all of the updates
in a single PR (with both commands to tweak things).

I could see our dependency bot going one further having PRs that offer to
update `Cargo.toml` dependencies across incompatible versions. Not every
compatibility break breaks every client.  Armed with a changelog and your CI,
it should be easy to decide whether to merge the PR.

If we're already writing a bot that watches dependencies, we could also make it
chime in on PRs that modify `Cargo.toml` to report on the licensing impact of
the change (e.g. new license introduced).

#### CI Tool Management

Beyond crate dependencies is CI versioning.
- Keeping [pinned tools](https://github.com/cobalt-org/cobalt.rs/blob/master/.travis.yml#L7) up to date
- Updating the version of rust used as part of the CI.  In basic cases, people
  just track stable, beta, and/or nightly.  People are exploring what should a
  libraries support policy be, whether its a sliding window of 2 versions back
  or not bumping minimum compiler versions on patch releases). I think as we
  coalesce on a best practice for this, we should explore how to automate this.

These are a little more challenging to automate because everyone can write
their `.travis.yml` et al any way they want.

### Centralized Documentation and Tooling

Right now, any documentation beyond trivial examples is spread across blog
posts.  The best resource is the great [trust
template](https://github.com/japaric/trust) but its hard to separate out the
complexity you don't need and your edits are mixed in with the template making
it hard to update.

My vision for this is a central repo that contains a crate CI mdBook, a
dedicated gitter channel for more targeted support, and commonly accepted CI
tooling.  This CI tooling includes trying to find ways to separate out trust's
mechanisms out from the user's policy so they can easily be modified
independent of each other and let the user more easily pick and choose what
parts they need.

I was concerned about centralization and the challenges with it but I was
encouraged at seeing killercup's [endorsement of the idea in
general](https://deterministic.space/rust-2018.html#aim-for-long-term-stability-of-the-library-ecosystem):

> There is no reason why so many popular crates should live in user-repos
> instead of in community-managed organizations (speaking in Github terms).
> Writing and then publishing a bunch of code as a crate one thing, but
> maintaining it, fixing bugs, replying to issues and pull requests, that takes
> up a lot of time as well. Time, that a lot of developers don't have, or don't
> want to invest. cargo-edit, for example, which lives under my Github
> username, has two wonderful maintainers, who are more active than I am. But
> should I create a cargo-edit organization and move the repo there? If there
> was a good and definitive answer, which would neither make me deal with the
> organizational aspects not result in accumulating lot of junk code, I'd be
> really happy.

(One challenge for a CI organization will in [coming up with a
name](https://www.reddit.com/r/rust/comments/7phnly/killercups_rust_2018/dshq71g/?st=jco1d8g4&sh=22389a92))

Specifically regarding CI tools, it'd be great to have best practices and be
able to more easily collaborate to bring them up to those best practices
(pre-build binaries for faster CIs, etc).

Some examples of tools I could see hosting in this repo include (if the authors agree):
- [`cargo-when`](https://github.com/starkat99/cargo-when) for conditional steps.
- [`cargo-kcov`](https://github.com/roblabla/cargo-travis) for coverage
  reporting (currently `cargo-coverage`; rename is suggested to distinguish
  between alternative coverage systems).
- [`cargo-coveralls`](https://github.com/roblabla/cargo-travis) for uploading to coveralls.
- Wrappers for generating compressed binaries for upload to github.
- Wrappers for generating commonly packages for major operating systems and distributions.

It might even be good to host tools not directly related to CI like
[`cargo-release`](https://github.com/sunng87/cargo-release) in an effort to
support users through the entire release process.

### Honorable Mention: rustfmt and clippy

The great stable/nightly divide for rustfmt and clippy being on nightly-only
introduce their own host of problems but I can skip all of that because they
are on their path for being integrated into the rust toolchain.
