---
title: Minor Semver Issue
published_date: "2022-02-04 15:37:29 +0000"
is_draft: false
data:
  tags:
    - programming
    - rust
---
Is adding a function in a patch release a violation of
[semver](https://semver.org/)?  Technically, yes but technical answers aren't
always the right answers.

This came up in a recent discussion focused on the
relevant importance of setting the minimum patch version for a dependency.
Some crates go so far as to never bump their minor version, like
[serde](https://crates.io/crates/serde/versions).

<!-- more -->

Let's go to the semver spec, specifically [Item 7](https://semver.org/#spec-item-7):
1. Minor version Y (x.Y.z | x > 0) MUST be incremented if new, backwards compatible functionality is introduced to the public API.
2. It MUST be incremented if any public API functionality is marked as deprecated.
3. It MAY be incremented if substantial new functionality or improvements are introduced within the private code.
4. It MAY include patch level changes.
5. Patch version MUST be reset to 0 when minor version is incremented.

As I said earlier, adding a new function deviates from the spec because it is
"new, backwards compatible functionality ... introduced to the public API".

## Semantics of "Violation"

The original poster insisted on using the phrase "violation of semver" to
describe this scenario.  Let's start with the semantics of "violation".

While it can be easy to dismiss conversations around semantics, we need to keep
in mind that communication is about influencing other people.  If you use terms
in an unexpected way, it will create a barrier making it harder or outright
prevent you from influencing others and will likely add unnecessary frustration
to all involved.  We need to consider not just the literal meaning of a word,
or its denotation, but also its connotation.

I believe the author was intending to say that some crate authors are not
adhering to 100% of the semver spec.  Not all deviations from the spec are
equal though.  Removing a functional part of a Rust crate's API in a minor
release will break any dependent crate without a lockfile (tooling) and will
break the expectations of those with a lockfile that run `cargo update`
(cultural).  This deviation from tooling and cultural expectations is a source
of problems and ideally leads to the release being yanked.  I feel these high
impact problems are what people have in mind when people hear about a
"violation of semver".

So what is the level of impact of a patch with an API addition?

## Role of Minor and Patch Releases

My traditional understanding of patch releases is that they are low risk and
easy to audit.  Semver doesn't seem to actually care about these cases since
the spec only specifies "MAY" for updating the minor field on substantial new
functionality (when the API is the same).  It doesn't even acknowledge the risk
associated with large refactors with the spec being quiet on this case.  The
focus is on API additions and deprecations for a minor release, compared to a patch.

Even the value of that traditional understanding is diminished when tooling
isn't there to support it.  At least in Rust, doing a `cargo update` updates
freely across patch and minor releases.  You could constrain the version
requirement to only update within patches but that will break anyone relying on
your crate because `cargo` unifies version constraints within a major version,
meaning there will be at most one copy of a crate per major.  If a common
version within a major cannot be found, your build fails.  Overly constraining
is a problem today in Rust and the errors aren't all that helpful (yet).

Maybe I'm missing something but the only value I'm seeing in this semver rule
is to say to users "this version *might* contain changes of more interest than
patch releases".

## Release Early, Release Often

That a version might contain changes of interest breaks down depending on how
far you take "release early, release often".

People might hold off on releases because
- They want a splashy release with a lot of lot of headlining features
- They need more extensive testing than the CI provides
- They need to resolve quality regressions caused by new features
- They need to finish large features that were merged incrementally without a way to gate access to it
- The release process itself adds friction to discourage releasing often

For most of my projects, I release on every user-contributed user-facing PR.
Releasing this often makes my users more efficient because they had a need
driving their change and makes them happier not having to wait.  This works
because I aim for a high confidence in my architecture and tests to ensure
quality.  Even without when that is lacking, I know that the testing of
unreleased code will be nearly non-existent while smaller releases will act as
a form of rolling release and it gives users the granularity to isolate the
problem and get the most benefit while they wait for a fix.  I also try to
limit known regressions from new features and not expose features until they
are ready.  With [cargo release](https://github.com/sunng87/cargo-release),
there is little friction in publishing a new release.

I'll be honest, I do miss the splashy marketing of having larger releases.  If
I published every release, I would end up spamming reddit with multiple
releases each day.

With this continuous release model, what value am I providing users by bumping
minor on all API additions?  Most API additions end up being small and drawing
attention to them with a minor release creates just as much noise for the user
to sift through as patch releases.

Conversely, the bar set by semver spec would influence people to consolidate
changes and make bigger releases to avoid the minor release noise, losing the
benefits of continuously releasing.

## Minor Value

For me, the value I see in minor versions are:
- Drawing attention to larger impact features.  This can't be determined by any
  technical measure but by the maintainer's understanding of their user base.
  Two competent maintainers could come to different conclusions on what
  justifies a minor release or not, and that is fine.
- Calling attention to semi-breaking-changes.  Semver does not clarify what
  counts as a breaking change is and [any change can break
  someone](https://xkcd.com/1172/), from bug fixes to changes to your build
  requirements.
- Rate limiting deprecations.  You might have noticed I've not acknowledged
  this part of the spec yet.  Its hard not be be [nerd
  sniped](https://xkcd.com/356/) by deprecation warnings and they
  imply the existence of technical debt in the form of refactors to migrate
  to the new API before you are blocked on needing functionality only available
  in the new API or major version.

## Version Requirements

So getting back to the topic that led to this discussion, when you use a
["semver" version constraint
(`^`)](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#caret-requirements),
should you specify the patch field?   I think its preferred, with caveats.

If your version constraint is under-specified, a person pulling in your crate
could see a build failure because you needed a specific patch release but they
already are using a compatible version of the dependency and so cargo won't
upgrade it.  It doesn't matter if its an API addition or a bug fix, you can be
relying on it either way.

However, there are downsides to specifying a minimum minor or patch field.

If your version constraint is incorrectly specified (too low), people depending
on your crate fail as if you under-specified.

If you always specify the latest version, you are safe but at minimum you cause
unnecessary build churn for people depending on your crate.  Worst case,
someone needs a dependency held back, either because of a bug or supported
compiler versions and can't because your version requirement conflicts with
what the user needs.

It'd go a long way to solving this if we had a way to always test the version
requirements (cargo's unstable `-Z minimal-versions`) and if all your
dependencies tested it.  A stop gap measure is to always specify the full
minimum version when adding or upgrading a dependency.  That is what will
initially be in your lock file and what you and other active developers will
develop against so most likely it will be correct.
