---
title: A Vision for Cargo
layout: default.liquid
is_draft: true
data:
  tags:
  - programming
  - rust
---

According to Stack Overflow, [cargo is the most desired development tool](https://survey.stackoverflow.co/2025/technology#2-cloud-development).
Many people attribute their choice of Rust to Cargo.
I've heard from many people that Cargo is already great and that they can't think of any way it could be improved.
I want to set our sights higher.

And I want to hear your thoughts on what an ideal workflow would look like and how we might get there.
This is just some things I've been thinking about.
I know I don't know or can't easily speak to [all of the workflows](#areas)
and others might have even better [ideas on how to solve them](#ideas).

We can also use your help in getting there.
These workflows are not isolated to just Cargo but also touch on Rust, crates.io, and more.
There are plenty of opprtunities to dig in and help.

<!-- more -->

<a id="areas"></a>

## Some workflows to improve

I want to frame this discussion around a high level view of workflows.
I'll cover specific problem areas (e.g. build scripts) as part of the [ideas on how to improve them](#ideas).

`#[non_exhaustive]`

- [Dependency management](#dependency-management)
- [Build performance](#build-performance)
- [Adaptability](#adaptability)
- [Cargo maintenance](#cargo-maintenance)

Also, [a word on agentic development](#agentic-development)

<a id="dependency-management"></a>

#### Dependency management

> “Always bet on the ecosystem”

\- [Battery packs: Let's talk about crates, baby](https://smallcultfollowing.com/babysteps/blog/2026/07/15/battery-packs/)

I feel strongly that the crates.io ecosystem has been one of Rust's superpowers,
enabling applications to be rapidly created with more capabilities and polish than would be practical without this ecosystem.

Applications can leverage not just the existing functionality but any future functionality and fixes.
For example, [`ripgrep`](https://github.com/BurntSushi/ripgrep/) is being optimized for large monorepos and [`typos`](https://github.com/crate-ci/typos) automatically benefits due to reusing [`ignore`](https://crates.io/crates/ignore).
A fresh implementation or a fork would not see benefits like this.
This also enables shared auditing.
In your application,
you can have one or two serialization frameworks also in use by others
or you can audit ten bespoke ones spread across config systems, network communication, etc.

Dependencies are not without their costs and risks which we should work to drive down.

**How do you discover a quality dependency?**

There are "known" dependencies to use,
assuming you are in the know.
Being productive shouldn't be gated on insider knowledge.

When there isn't a "known" dependency,
you are then left to find candidates which isn't always easy.
To compare them, you then look for a smattering of signals to see if it is safe, fit for your purposes, and can be depended on for the long term.
These may include download count, who depends on it, reviews performed, etc as well as trust carried over from other packages from the author.
Knowing and aggregating these can take a lot of work.

In the ideal world,
all dependencies would be reviewed.
We aren't in an ideal world but how do we get closer?
In Rust, we have `unsafe` as a way to isolate code that needs further inspection.
We need more types of audit points and to scale up their discoverability.
Even better if you didn't need to review these audit points
because they were opt-in where possible,
with a fallback that didn't need audits at the cost of performance or features.
Sometimes though,
we will still want to take shortcuts and trust in others for reviews
and tracking of our own or others' reviews should be integrated with using dependencies.

**How do we minimize upgrade overhead?**

First, we need to improve the state of linting and testing for maintainers so there are fewer bugs and unintended changes.

Assuming we are now talking about intended changes,
the easy answer would be to discourage breaking changes
but that comes at a cost to the maintainer in not being able to clear out some types of technical debt
and to users in the limits this places on features, fixes, and build performance.

We should encourage documenting changes and better surface that.
However, that is a stopgap that patches over the problem.
How can we make this better?

One source of inspiration is the Rust Language
which also has a form of opt-in breaking changes through Editions.
The Rust Project prepares for an edition with unstable features and by providing migrations for users.
That should be viewed as the minimum experience we offer maintainers and users for packages.
Going beyond that, we could better surface these changes to users,
inline to their workflows and directed to their specific circumstances.
This would especially be important for when migrations aren't possible.

As an experiment in what can be done today,
with clap I have adopted the practice of breaking changes being introduced as parallel features, deprecating the old behavior with a note in how to migrate, where possible.
The deprecations have been put behind a `deprecated` feature flag because many people turn warnings into errors, including deprecations, and having them on by default coupled upgrading with resolving deprecations increasing costs and risks.
This dramatically simplifies [migration guides](https://github.com/clap-rs/clap/blob/v4.6.4/CHANGELOG.md#migrating) to "enable a feature and do what it says".
Where a parallel feature isn't possible, we use an `unstable-vX` feature to not block development on an eventual breaking change and to make it easier to get feedback.

Problems I see with this:
- Awareness of the `deprecated` feature
- Discovering of the migration guide
- The migrations are still manual
- `unstable-vX` features can have surprising effects and need more guard rails to scale up their use
- These are just conventions I made up; sustaining a culture around these ideas needs a paved path to raise awareness and encourage their use

<a id="build-performance"></a>

#### Build performance

Improving build performance isn't about making things 2% faster
but about changing how users work and making Rust attractive for more users.
Users' points of comparison will range from C++,
which is more similar until you account for pre-built frameworks,
to scripting languages, where there are no builds.

We also need to look beyond the performance of `rustc` to the user's workflow.

Users are looking for instant feedback as they make small changes at the bottom of their stack
but that unfortunately causes the entire stack to rebuild,
taking some time even for `cargo check`.

They are iterating on test failures until they get the behavior just right,
either running all of `cargo test --workspace` in a loop ("because it's easy")
or hand constructing a command like `cargo test -p application-library --test area -- some::test::name` for each failure to give faster iteration time at the cost of a disruption.

Maybe while working through either of the above,
they rebase against `main` and now their whole stack is rebuilding because of a dependency update.
Or maybe they ran out of disk space and had to run `rm -rf target`.
Or maybe it rebuilt for no discernible reason.

Or maybe while working through either of the above,
Cargo blocks on a parallel invocation,
whether from rust-analyzer,
another worktree they want to share the cache with,
or from another project on their system that they are sharing the cache with.

Some users will task switch while they wait for CI to complete so they don't stack too many changes on top of unmerged code,
increasing the risk of costly merge conflicts.
CI systems have caches but caching too much can negate any of the benefits due to network transit and decompression times.
Defaults that work well for local debugging will increase cache sizes and slow down builds.
While build results get cached,
test results do not and the entire test suite re-runs.

See also my RustNL 2025 presentation:
[video](https://www.youtube.com/watch?v=-jy4HaNEJCo&t=678s&pp=ygUOcnVzdG5sIGVkIHBhZ2U%3D)
/
[slides](../../../../talks/2025/05/performance)

<a id="adaptability"></a>

#### Adaptability

Cargo is [intended to be opinionated](https://doc.crates.io/contrib/design.html)
but between that and backwards compatibility,
it can be hard to be adaptable to different needs,
whether that is [improved docker caching](https://github.com/rust-lang/cargo/issues/2644),
[complex application needs](../../../2023/08/are-we-gui-build-yet/),
or integrating into a larger corporate environment.

In git, they divide operations into [plumbing and porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).
Cargo is almost all porcelain commands with very limited programmatic use.
Adding plumbing commands and extending the programmatic output of the porcelain commands would go a long way to enabling people to adapt Cargo to meet their needs.

<a id="cargo-maintenance"></a>

#### Cargo maintenance

Underlying any talk of improving Cargo is that the Cargo team needs enough capacity to support these efforts on top of keeping the lights on.
Previously, Cargo went through a several year [feature freeze](https://blog.rust-lang.org/inside-rust/2022/03/31/cargo-team-changes/) due to a lack of capacity from the team.
I worry about how close we are to repeating that.

During that feature freeze and for a while after,
we had a focus on improving our processes and tooling to make us more effective in maintaining Cargo.
For example, we made it easier to make sweeping UX changes 
by developing and migrating to a snapshot testing library that met our needs
([snapbox](https://docs.rs/snapbox/), see [#14039](https://github.com/rust-lang/cargo/issues/14039)).

Now, the thing I see slowing us down the most and limiting what we change is Cargo's architecture.
Cargo is old and has bespoke frameworks,
constraints on concurrent operations,
uses C libraries when Rust ones can replace them,
and no clear boundaries that require global knowledge when making or reviewing a change.

Not everyone can fund work like that
and we can only build up new design and code reviewers so quickly.
Where outside contributors can most help the Cargo team is not with code but with thinking.
A major enabler for thinking is to have all the context in one place
by gathering all of the relevant information from the thread
and summarizing it for easy consumption.
For example, I have a PR on another project that slowly makes progress and when I go back to look at it,
I have to refresh myself on the behavior the PR is intended to implement by re-reading an entire Issue thread.
That is a lot of work and makes it so I need Dedicated Time® to review it,
putting it forever in that "someday" category.
That might sound like the ideal job for an agent.
It will still take time to vet the sourcing and think through any analysis it has made.
See also [D-SUMMARIZE](../../../../dev/pr-style/#d-summarize).

<a id="agentic-development"></a>

#### A word on agentic development

A large part of Rust's success is the effort in [talking to people as humans](https://hachyderm.io/@ekuber/116970121496881903).
This has incidentally benefited Rust among agentic development as many of the needs overlap.
I think continuing to focus on humans will be the best strategy in the long term,
with the assumption that agents will continue to improve.
Where examining the needs of agentic development can pay off
is by seeing the same problems faced by humans
but scaled to such a degree that it breaks us out of complacency with "good enough" 10% improvements.

As I see it, the three pillars where humans and agents overlap are:

**Signal quality:** One reason that Rust works well for agentic development is the feedback it provides.
We need to clarify Cargo's existing signals by not being so overly verbose.
We also need to provide more signals through tools like lints, audit points (think: `unsafe`), and so on.

**Dependency management:** We also need to consider signals for selecting and auditing of dependencies as well as reducing the costs of having dependencies.

**Build performance:** While a lot of agentic development is done in the background,
the performance of the commands that agents run matters,
especially if you want a tighter feedback loop
(e.g. [bun skipped `cargo check` in the first pass of its Rust port](https://bun.com/blog/bun-in-rust#false-starts)).
The faster the operations are, the closer to the inner loop they will be used, and the better quality of results will be.

<a id="ideas"></a>

## Potential steps to this future

While this list is long,
it is not intended to be exhaustive.
I got tired of writing.
I am also less qualified to speak on some of these subjects 
or I don't have enough experience with them to speak too much about them.

I frequently see the sentiment that nothing is being merged into Rust these days that justifies raising an MSRV.
Looking over this list,
I see a lot to get excited about.

`#[non_exhaustive]`

- Maintainability
  - [`async`-ify Cargo](#asyncify-cargo)
  - [Migrating away from `libgit2`](#libgit2)
  - [Resolving with PubGrub](#pubgrub)
  - [Generated shell completions](#generated-completions)
- [Plumbing commands](#plumbing-commands)
- [Structured logging](#structured-logging)
- [Open namespaces](#open-namespaces)
- [Registry namespaces](#registry-namespaces)
- [Public dependencies](#public-dependencies)
- Battery packs
  - [Package distributions](#package-distributions)
  - [Mod templates](#mod-templates)
- Auditing
  - [Crate provenance](#crate-provenance)
  - [Builtin dependencies](#builtin-dependencies)
  - [`lang:unsafe`](#lang-unsafe)
  - [Declarative derive macros](#declarative-derive-macros)
  - [Proc-macro and build script access controls](#access-controls)
  - [Reducing the need for build scripts](#remove-build-scripts)
  - [Reducing user build scripts](#build-script-delegation)
  - [Proc-macro and build script allow list](#build-script-allowlist)
  - [Cargo lints](#cargo-lints)
  - [Test coverage](#test-coverage)
- API evolution
  - [Feature lifecycle](#feature-lifecycle)
  - [API lifecycle](#api-lifecycle)
  - [Source inlining](#source-inlining)
  - [Auditing API changes](#auditing-api-changes)
- Performance
  - [Improved decomposition](#improved-decomposition)
  - [Removing unnecessary rebuilds](#removing-unnecessary-rebuilds)
  - [Shared caches](#shared-caches)
  - [Build cache GC](#build-cache-gc)
  - [Fine-grained cache locking](#fine-grained-cache-locking)
  - [Alternative compilation models](#compilation-model)
  - [Opaque dependencies](#opaque-dependencies)
  - [Test result caching](#test-caching)
  - [Iterating on test failures](#test-failures)

<a id="asyncify-cargo"></a>

#### `async`-ify Cargo

Cargo predates `async` / `await` and has its own bespoke async framework,
using `curl` as the executor.

This means Cargo can't leverage existing tooling around async executors,
the code base does not follow familiar patterns for contributors,
and takes a lot of work to then extend the code base to support running more things in parallel, like parsing of thousands of manifests.

We'd also want to switch from `curl` to a Rust native networking library but this is one of a couple of pieces holding us back.

Areas:
- [Cargo maintenance](#cargo-maintenance)
- [Build performance](#build-performance)

Tracking issue: [#16845](https://github.com/rust-lang/cargo/issues/16845)

<a id="libgit2"></a>

#### Migrating away from `libgit2`

When Cargo started, `git` was not deployed universally
yet the Cargo team chose to use it as the transport mechanism for the registry index.
`libgit2` helped to fill this gap and made it possible to have a custom UX for progress.
An escape hatch to use `git` was provided for any feature gaps users ran into.

Over 10 years later, `git` is everywhere and no longer used for most Cargo registries.
There is a high likelihood that those who have git dependencies will have `git` installed.
It seems the right time to make the `git` escape hatch the default,
improving fetch performance and network compatibility.

Maybe we eventually remove the `libgit2` network transport entirely.
This would remove another dependency on `curl`,
unblocking Cargo to switch to a Rust native networking library and a regular async executor.

Maybe we can even remove `libgit2` one day,
either replacing it with [gix](https://crates.io/crates/gix) or `git`,
removing yet another C component from the build of Cargo.
This will also reduce Cargo startup overhead.

Areas
- [Cargo maintenance](#cargo-maintenance)
- [Build performance](#build-performance)

Tracking issue: [#17227](https://github.com/rust-lang/cargo/issues/17227)

<a id="pubgrub"></a>

#### Resolving with PubGrub

Cargo's dependency resolver has bugs, poor errors, and missing features.
However, people don't feel confident making or reviewing changes
It can also be difficult to know how to update the reference SAT implementation in the
differential test suite.

PubGrub is a generalized dependency resolver.
We would be able to decouple the Cargo logic from the algorithm,
making it easier to change and removing our need to do differential testing.
The output is much richer, allowing for better errors.

PubGrub has been proven out by Astral with uv,
including the ease of making changes like [changing how prerelease works](https://github.com/astral-sh/uv/releases/tag/0.12.0).

Areas
- [Cargo maintenance](#cargo-maintenance)
- [Dependency management](#dependency-management)

Tracking issue: [#5284](https://github.com/rust-lang/cargo/issues/5284)

<a id="generated-completions"></a>

#### Generated shell completions

Cargo offers bash and zsh completions but these are written by hand with the expectation that any CLI changes are mirrored in these without a good testing story.
Contributors and reviewers should not be expected to be bash and zsh completion experts or have every shell installed on their system.

Cargo uses clap which offers generated completions but Cargo has some dynamically generated completions which clap only supports through an in-development feature.
Stabilizing that feature and switching Cargo's completions over to the new system will make them more maintainable and more feature rich as it is much easier to provide dynamic completions with the new system.

Areas:
- [Cargo maintenance](#cargo-maintenance)

Tracking issue: [#14520](https://github.com/rust-lang/cargo/issues/14520)

<a id="plumbing-commands"></a>

#### Plumbing commands

We want to break up `cargo check` into the smallest units of work possible and expose that as an API through a pipeline of plumbing commands.
This would put callers in charge of what Cargo is doing,
whether changing the data between steps to outright replacing steps with a custom implementation.

By having this pipeline of operations with clear inputs/outputs between them,
we'll by necessity need to do the same thing for the Cargo code base.
For example, feature resolution would be an isolated, pure transformation from inputs to outputs
This makes it so contributors and reviewers have a finite area of concern to work with,
making it easier to contribute and making reviewers more confident in accepting changes.

Maybe one day, this refactoring will lead to breaking up Cargo into smaller libraries
that people can use.
There are trade offs with this though as it constrains the design within `cargo`,
possibly makes it more complicated to abstract out CLI-specific assumptions,
and locks callers into one version of `cargo`.
I've maintained third-party tools that had called into `cargo`
and it was annoying to users to get "unknown key" warnings or "this feature is unstable" errors (despite being stable within their toolchain).
Plumbing commands should likely remain the assumed route for general purpose extensions.

Areas:
- [Cargo maintenance](#cargo-maintenance)
- [Adaptability](#adaptability)

Project Goal: [Prototype a new set of Cargo “plumbing” commands](https://rust-lang.github.io/rust-project-goals/2026/cargo-plumbing.html)

<a id="structured-logging"></a>

#### Structured logging

`cargo check --timings` can only show you how long that build took.
Similarly, `cargo check -v` can only tell you why something rebuilt in that build.
Many times, you don't know a build will be a problem until it is happening.
With structured logging,
you can get reports on previous builds to better diagnose problems.

There is a large overlap in our needs for structured logging and programmatic output for our porcelain commands.
In iterating on structured logging,
we are also exploring how we can improve our programmatic output.

Areas:
- [Build performance](#build-performance)
- [Adaptability](#adaptability)

Tracking issue: [#15844](https://github.com/rust-lang/cargo/issues/15844)

<a id="open-namespaces"></a>

#### Open namespaces

In other languages, namespaces are explicit and open to extension.
In Rust, they are coupled to the module system and closed to extension.

This feature is about making Rust's namespaces partially open to extension where a another package can look as if it is a mod in another package.

For example, take Clap:

| existing name   | potential name   | notes |
|-----------------|------------------|-------|
| `clap`          | `clap`           |       |
| `clap_derive`   | `clap::derive`   |       |
| `clap_complete` | `clap::complete` |       |
| `clap_mangen`   | `clap::man`      |       |
| `clap_lex`      | `clap_lex`       | first-party but a private dependency of clap |
| `clap-cargo`    | `clap-cargo`     | third-party though an extension of clap's API |

This is more of a language feature impacting API design.
This bleeds into dependency management in terms of providing an alternative to compose a cohesive API besides features,
which have their drawbacks,
and without coupling together the major version of all of the pieces,
allowing for breaking changes with a smaller impact.

This can further help dependency management if we extend the package UX in places like crates.io and docs.rs to make these navigable between them,
extending the cohesion to our UX.

Areas
- [Dependency management](#dependency-management)

Tracking issue: [#13576](https://github.com/rust-lang/cargo/issues/13576)

<a id="registry-namespaces"></a>

#### Registry namespaces

There are many reasons people may want this.
Within the scope of this document,
the most relevant is knowing a package came from a trusted source
and finding other packages from that source, like disjoint APIs from a company.

Areas
- [Dependency management](#dependency-management)

Reference: [Survey of organizational ownership and registry namespace designs for Cargo and Crates.io](https://internals.rust-lang.org/t/survey-of-organizational-ownership-and-registry-namespace-designs-for-cargo-and-crates-io/24027)

<a id="public-dependencies"></a>

#### Public dependencies

Being able to mark that a dependency (and its public dependencies) are part of your public API feels like a small, incremental improvement.

At its most basic, it helps catch accidentally leaking dependencies in your public API.
The most innocent example is an `impl From<dep::Error> for Error`.

Tools, like [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks/),
can build on this to help identify unintended breaking changes by upgrading a dependency.
It can be understandable to make a mistake and it helps to have tools catch the problem.
Where this turns from a "you mistake" to a "systemic failure" is with
[`workspace.dependencies`](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-dependencies-table).
When changing them, it is easy to overlook where they are used.
When preparing a library for release with an inherited dependency,
you can no longer look at only the library to tell what all changed since the last release.
I hesitate in further ingraining `workspace.dependencies`, like through `cargo add` ([#10608](https://github.com/rust-lang/cargo/issues/10608)),
until this problem is solved as I fear we would be setting people up for failure.

This also affects performance.
If you want to generate documentation while developing,
the simplest thing to do is `cargo doc` but that generates documentation for all dependencies, including slow to build dependencies like `windows-sys`.
If you care to look, you may discover `cargo doc --no-deps` and maybe that will be good enough for you but happy paths shouldn't be gated by knowledge.
With public dependencies, Cargo knows what dependencies are reachable by your project and can instead only generate documentation for those dependencies,
speeding up your documentation builds.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issue: [#44663](https://github.com/rust-lang/rust/issues/44663)

<a id="package-distributions"></a>

#### Package distributions

Some packages need to be upgraded in lockstep and there isn't any tooling to assist with that today.
We can improve this by allowing users to specify that a dependency's source comes from another dependency, like:

```toml
[package]
name = "some-cli"

[dependencies]
clap = { from = ["clap_complete", "clap_mangen"] }
clap_complete = "4.6.0"
clap_mangen = "4.6.0"
```

That declaration is a bit backwards and could be improved with maintainers providing "distribution" packages:

```toml
[package]
name = "some-cli"

[dependencies]
clap_distribution = "4.6.0"
clap.from = "clap_distributiona"
clap_complete.from = "clap_distributiona"
clap_mangen.from = "clap_distributiona"
```

This is starting to look similar to [battery packs](https://battery-pack-rs.github.io/battery-pack/) and could possibly be a way of upstreaming the concept into Cargo.

Blocked on:
- [Public dependencies](#public-dependencies)

Areas
- [Dependency management](#dependency-management)

Referenced at: [RFC 3516: Future possibilities](https://rust-lang.github.io/rfcs/3516-public-private-dependencies.html#caller-declared-relations)

<a id="mod-templates"></a>

#### Mod templates

`cargo new` only does whole-package generation from a fixed template.

There are multiple kinds of templating though:
- Repo
- Package
- Crate
- Module

Typically, these all get bundled together into repo or package templates
(see also [#5151](https://github.com/rust-lang/cargo/issues/5151)),
keeping the design scope quite large,
making it harder to move a design forward for Cargo.

One particular problem with repo templates is that they try to cover the needs of all of the other kinds of templates,
either giving users a "full stack" application template that they then have to pare down to meet their needs or making the system complicated through templating logic and user prompts.

We could extend `cargo new` with crate/module templates,
leveraging examples as templates.

Say you had a `rust_cli` [package distribution](#package-distributions) with topical examples, you could startup a package with:
```console
$ cargo new --bin --from rust_cli:async_main
$ cargo new --mod --from rust_cli:args
$ cargo new --mod --from rust_cli:watch
$ cargo new --test --from rust_cli:cli_tests
$ fd
Cargo.toml
src/main.rs
src/args.rs
src/watch.rs
tests/cli_tests.rs
```

*Note: this is only a rough sketch to highlight the idea*

What you get:
- Package metadata is inherited from a workspace, if it exists (already present)
- Dependencies populated from the given examples
- A runnable `src/main.rs` from the `main` example
- Additional modules that need to be edited and `mod`ed into `src/main.rs`

You would still need to edit the various mods and `mod` them into `src/main.rs`

Areas
- [Dependency management](#dependency-management)

<a id="crate-provenance"></a>

#### Crate provenance

The [xz backdoor](https://en.wikipedia.org/wiki/XZ_Utils_backdoor) highlighted, among other problems, the danger of releases diverging from VCS.
We need an audit point for when a published `.crate` file has no associated git sha, when it diverges from the source for that sha, and when that sha is an [imposter commit](https://docs.zizmor.sh/audits/#impostor-commit).

Areas
- [Dependency management](#dependency-management)

Referenced at: [blog: 999 crates of Rust on the wall](https://lawngno.me/blog/2024/06/10/divine-provenance.html)

<a id="builtin-dependencies"></a>

#### Builtin dependencies

Identifying impure packages (those that can access files, the environment, networks, RNGs, etc) can serve as another audit point.
We can get this through the `build-std` projects efforts to integrate dependencies on `core`, `alloc`, and `std` into Cargo's `[dependencies]`.

This becomes especially important for proc-macros which your environment may run while you are reviewing a package and so Cargo can know if a run of a proc-macro is safe to cache, particularly if a caching system needs binary reproducibility which can be defeated by a simple `HashMap`.

Of course, an `alloc`-only package only tells us some of the story due to `unsafe`, FFI, and `asm!`.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issue: [#16960](https://github.com/rust-lang/cargo/issues/16960)

<a id="lang-unsafe"></a>

#### `lang:unsafe`

As just covered, even with `alloc`-only dependencies, there are still holes in our audit point coverage, like `unsafe`, FFI, and `asm!`.
The idea here is to integrate access to language features into Cargo.

One way we could do this is to migrate to where `[features]` could activate `"lang:unsafe"`, e.g.

```toml
[features]
unsafe = ["lang:unsafe"]
```

This wouldn't be to shame those packages
but to serve as a higher level `unsafe` audit point accessible from the package's metadata
and encourage providing safe-only alternative implementations for those who might not need some of the benefits gained by `unsafe` implementations.
Of course, not every package can make use of `unsafe` optional and we would need to provide a way to work use `unsafe` with `default-features = false`.

Maybe this could even be extended to language preview features (e.g. the proposed [RFC 3965](https://github.com/rust-lang/rfcs/pull/3965)).
While there is a lot of risk with allowing access to language features before stabilization,
it would help if the default was for them to be opt-in by callers and easily discovered by the Rust Project.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

<a id="declarative-derive-macros"></a>

#### Declarative derive macros

Even better would be declarative derive and attribute macros as the compiler will ensure they are pure.
While these may not be able to handle every pure proc-macro case,
every bit would help especially among foundational macros.

This has the added bonus that there won't be a proc-macro crate and its dependencies that need building and linking,
even in `cargo check`.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issue: [rust#143549](https://github.com/rust-lang/rust/issues/143549)

<a id="access-controls"></a>

#### Proc-macro and build script access controls

Not all proc-macros can be pure.
Build scripts can never be pure.
But any problem with that is not specialized to these cases.
You will eventually run the rest of a dependency's code with `cargo test`, `cargo run`, and with production deployment.
What is special about proc-macros and build scripts is they are required to run to build the code.
You might want to build as part of your auditing.
Their non-determinism may impact caching solutions.

We need to be able to have audit points for these impure operations.
Even better if it can be extended towards tests.
Even better if it also helps with regular dependencies.

One approach is an access control system like [Cackle](https://davidlattimore.github.io/posts/2023/10/09/making-supply-chain-attacks-harder.html).
This would work for your entire build.

Maybe Cargo can use OS features to restrict build script and proc-macros access as a way to highlight these audit points.
Cargo should support a wide variety of OSs and they have different capabilities which may change with time,
likely making enforcement be best-effort.
We will need to provide some way for build scripts and proc-macros to specify the resources they wish to access.
Maybe we could make this also work for tests and `cargo run` but it wouldn't help with deploying your production binary.

Maybe we go all the way to a wasm sandbox.
This is trivial for pure proc-macros though other ideas mentioned here will help.
The big challenge is supporting what is needed by impure proc-macros as well as build scripts.
Like with using OS enforcement,
we will need a way to specify what resources can be accessed.
Wasm sandboxing can likely lead to slower builds and not just because of having to run a wasm interpreter.
Cargo goes out of its way to try to allow proc-macros and build scripts to reuse intermediate build artifacts with a user's application.
When we've previously benchmarked diverging these builds more for "build performance",
the results ended up being a wash because of the extra builds that happen.
This is also limited in what it helps; 
it would not extend to your tests, `cargo run`, or your production binary.

Areas
- [Dependency management](#dependency-management)

References:
- [Cackle](https://davidlattimore.github.io/posts/2023/10/09/making-supply-chain-attacks-harder.html)
- [compiler-team#1017](https://github.com/rust-lang/compiler-team/issues/1017)

<a id="remove-build-scripts"></a>

#### Reducing the need for build scripts

Cargo has provided build scripts as an escape hatch to allow rapid iteration without requiring upfront design work for first-class features within Cargo.
Cargo would likely have failed without this.

With hindsight, we can look back and identify patterns so we can design features to replace them,
greatly reducing the use of build scripts.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issues:
- [#14948](https://github.com/rust-lang/cargo/issues/14948)

<a id="build-script-delegation"></a>

#### Reducing user build scripts

Even when build scripts are still needed,
it would help if we only had to audit one build script,
rather than 20 build scripts that end up being wrappers around a library.

This is the idea behind build script delegation.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issues:
- [#14903](https://github.com/rust-lang/cargo/issues/14903)

<a id="build-script-allowlist"></a>

#### Proc-macro and build script allow list

With enough of the above features,
it wouldn't be too painful to make proc-macros and build scripts opt-in.
For when a build script isn't opted into,
maintainers could provide a fallback of a static set of build script directives to fill in just enough to move along on the happy path.

I see two benefits here:
- Avoid accidentally running these before they are audited
- Faster builds for those who don't care about the benefits afforded by a build script

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Reference:
- [Build script allowlist](https://rust-lang.zulipchat.com/#narrow/channel/246057-t-cargo/topic/build.20script.20and.20proc.20macro.20allow.20list/near/595506317)

<a id="cargo-lints"></a>

#### Cargo lints

While clippy has a few lints for Cargo,
it is at a disadvantage in terms of what data it has access to.
Adding a [linting system to Cargo](#cargo-lints) would increase the number of quality signals users and agents can receive.

These signals will improve the quality and build performance of dependencies.
In particular, the linting system will come with built-in unused dependency detection.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issue: [#12235](https://github.com/rust-lang/cargo/issues/12235)

<a id="test-coverage"></a>

#### Test coverage

To help improve the quality of your dependencies,
we should have a paved path for test coverage reporting.

I am also curious what the impact to agentic development would be to have first-class coverage reporting.
I've observed that agent output can look good on the surface but it is in the small details where it is likely to fail.
I have a theory that tests would amplify absurd output,
making it easier to spot in review.
I am unsure where all this might lead as there are [problems with overly focusing on coverage](../../../../dev/testing#trade-offs-in-testing).

Areas
- [Dependency management](#dependency-management)

Tracking issue: [#13040](https://github.com/rust-lang/cargo/issues/13040)

<a id="feature-lifecycle"></a>

#### Feature lifecycle

Features exist or they don't.
The act of adding a feature for what already exists is a breaking change.
Maintainers need to be able to evolve the feature set of their library over time,
minimizing breaking changes and helping users through breaking changes.

We could even restrict language preview features (see [`lang:unsafe`](#lang-unsafe)) to only unstable features,
ensuring they remain opt-in.

Areas
- [Dependency management](#dependency-management)

Tracking issues:
- [#7130: Deprecating features](https://github.com/rust-lang/cargo/issues/7130)
- [#10881: Stable and unstable features](https://github.com/rust-lang/cargo/issues/10881)

<a id="api-lifecycle"></a>

#### API lifecycle

Today, the only stable API lifecycle attribute is `#[deprecated]`.
`#[stable]` and `#[unstable]` remain unstable,
limiting them to only the standard library.

Like with the idea of language preview features (see [Feature lifecycle](#feature-lifecycle)),
we could make `unstable` APIs enabled through unstable features in Cargo,
ensuring they remain opt-in.

Areas
- [Dependency management](#dependency-management)

Reference:
- [all-hands-2026#14](https://github.com/rust-lang/all-hands-2026/issues/14)

<a id="source-inlining"></a>

#### Source inlining

While a `#[deprecated(note)]` helps with migrating off of an API,
it is still a manual process.
Migrating away from deprecated APIs is a manual process.
The ideal would be like Editions where you run `cargo fix` and you are ready to go.

While we may want to eventually allow maintainers to specify larger source transformations with `#[deprecated]`,
like what [Coccinelle](https://rust-for-linux.com/coccinelle-for-rust) or [ast-grep](https://astgrep.com/guide/rewrite-code) allow,
Go's [source inlining](https://go.dev/blog/inliner) is likely to get us pretty far while being
intuitive,
easy to validate,
and a small surface area to stabilize.

Areas
- [Dependency management](#dependency-management)

Reference:
- [blog: //go:fix inline and the source-level inliner](https://go.dev/blog/inliner)

<a id="auditing-api-changes"></a>

#### Auditing API changes

When reviewing a PR or making a release,
it can be easy to overlook a breaking change.

Conversely,
we could do a better job exposing API changes to callers on upgrade to help them through migrations or to discover new features.

There are multiple routes for improving this that need further exploration.

Areas
- [Dependency management](#dependency-management)

Reference:
- [cargo-public-api](https://crates.io/crates/cargo-public-api)
- [cargo-semver-checks/](https://github.com/obi1kenobi/cargo-semver-checks/)

<a id="improved-decomposition"></a>

#### Improved decomposition

In C++, a `.cpp` file is the unit of compilation.
Rust crates are a unit of compilation but
the larger the compilation unit, the longer code-gen takes.
This makes it so splitting crates can speed up compilation time at the cost of less visibility for optimizations though there is LTO to help with that.

Splitting isn't without its costs:
- May run into limitations with the Orphan rules
- Loss of dead code detection
- More boilerplate to manage

We either need to find an alternative way to express compilation units
(e.g. ["inline crates"](https://blog.yoshuawuyts.com/inline-crates)) or
to remove the friction in managing an application split into crates.

Some progress has been made on this, including
- workspace publishing
- workspace inheritance

Areas
- [Build performance](#build-performance)

Reference:
- [hawk: A workspace-aware Cargo lint for unnecessary public Rust APIs.](https://github.com/astral-sh/hawk)
- [blog: Coherence and crate-level where-clauses](https://smallcultfollowing.com/babysteps/blog/2022/04/17/coherence-and-crate-level-where-clauses/)

<a id="removing-unnecessary-rebuilds"></a>

#### Removing unnecessary rebuilds

When you change a package in your dependency graph,
you might not need to rebuild all dependents.

In C++, the interface (stored in `.h` file) is kept separate from the implementation (in a `.cpp` file).
There is more to this but it is enough of an explanation to serve our purposes.
Each `.cpp` file gets its own intermediate build artifact.
Each of these artifacts only needs to rebuild if a `.h` they depend on changes.

We don't have this concept in Rust yet but maybe we can do better.
The compiler already generates an `.rmeta` file for a package which is everything that is needed to build dependent,
intermediate build artifacts without the need for the full `.rlib`.
Cargo uses this today for what we call "pipelined compilation",
where dependent builds can start in parallel to their dependency finalizing compilation.

Could we get the same benefits of the `.h` / `.cpp` split, automatically?
Yes, with a lot of a compiler work and some Cargo work.
A polyfill already exists that demonstrates a portion of the build time improvements by replacing `--workspace` with the specific packages final artifacts that would be impacted.
A real solution would be able to skip building some intermediate dependencies.

This is a form of build output hashing.
When applied more generally,
Cargo can skip building artifacts, even linking, of dependents if the code change didn't change the output.
However, this can only work for packages that are binary reproducible.
Build scripts and proc-macros add some uncertainty here.

Areas
- [Build performance](#build-performance)

Tracking issue: [#14604](https://github.com/rust-lang/cargo/issues/14604)

<a id="shared-caches"></a>

#### Shared caches

To share build caches across workspaces today, you need to share the `build-dir`.
This is not without its downsides:
- This will serialize builds due to the locks.
- If something poisons the cache, you need to remove the entire cache to fix it
- This can interfere with incremental compilation
- Some cache entries will collide, causing extra rebuilds

This is intentionally not documented by the Cargo team.

If Cargo provided an append-only local cache for [pure builds](#builtin-dependencies),
a lot of these problems would be resolved.

The main downside to a local cache is that you won't get much reuse.
Just one dependency change can cause all dependents to have new cache entries.
This isn't so much an issue for sharing a cache across multiple worktrees of the same repo.

This would also be improved by having a remote cache that you trust,
like the CI for the project you are contributing to.

Why not pre-built binaries stored on crates.io?
Besides issues of trust,
almost no one would be able to use it.
They have to be on the exact same version of every package in their dependency tree
with the exact same features enabled
with the exact same compiler flags
with the exact same compiler version.
More on this [later](#opaque-dependencies).

Areas
- [Build performance](#build-performance)

Tracking issue: [#5931](https://github.com/rust-lang/cargo/issues/5931)

<a id="build-cache-gc"></a>

#### Build cache GC

The more Cargo caches, the more disk space it eats up.
Cargo needs to be a better citizen with respect to disk space.

This becomes even more important in CI where any extra content cached with the
CI provider is extra compression, upload, download, and decompression time
wasted.

Areas
- [Build performance](#build-performance)

Tracking issue: [#5026](https://github.com/rust-lang/cargo/issues/5026)

<a id="fine-grained-cache-locking"></a>

#### Fine-grained cache locking

Locking of the build-dir is all-or-nothing,
causing contention between builds in a user's terminal and rust-analyzer
or between git worktrees.

Today, cache entries are mutable and read while still being written to (for pipelined compilation).

Mutable entries give Cargo a cheap form of cache GC on local edits;
without them Cargo would need a [build cache GC](#build-cache-gc) to replace it.
Solving that would help when switching between branches.
Mutable entries also mean that Cargo does not need to know all build inputs a priori;
we can look to other build systems that have already solved this problem.

Areas
- [Build performance](#build-performance)

Tracking issue: [#4282](https://github.com/rust-lang/cargo/issues/4282)

<a id="compilation-model"></a>

#### Alternative compilation models

Zig will parse to AST, identify the entry points, like `main`, and then recurse until done.
This is the ultimate form of dead code elimination where platform conditionals don't need to be viral.

We use features for this today.
They avoid needing to parse to AST but are difficult to work with and aren't as fine grained as Zig's builds.

We have a polyfill present today in the form of [`-Zhint-mostly-unused`](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#profile-hint-mostly-unused-option).
This only offers benefits in extreme cases (e.g. `windows-sys`) and still benefits from activating specific features.

Another polyfill, [`cargo-slicer`](https://github.com/yijunyu/cargo-slicer), has taken a couple different forms.
One more extreme version would vendor your dependencies and strip them down to only what you used.
Each time you change how you use a dependency, you must re-slice your dependencies which requires an initial build to identify what is used.

This isn't an automatic win.
The compiler would need to instead do the work of [reducing unnecessary rebuilds](#removing-unnecessary-rebuilds).
Cargo couldn't use [local or remote caches](#shared-caches).
Compilation steps would become redundant between different binaries (like all tests).

Areas
- [Build performance](#build-performance)

<a id="opaque-dependencies"></a>

#### Opaque dependencies

Line for line and feature for feature, C++ and Rust may have similar compile times
but you generally don't rebuild Unity from scratch while everyone does rebuild Bevy from scratch.
You are also generally relinking all of Bevy as well.
What Cargo is missing is a good story for working with dynamic libraries.
Or in their more general form, opaque dependencies.

Rust dependencies today are transparent like headers-only libraries in C++.
Every detail of the dependency tree affects a developer's build.

An opaque dependency appears to you as a single package, independent of how many packages it is made up of.
An opaque dependency has an independent set of dependency versions and feature activations.
No unification happens.

On its own, this provides us a story for reducing link times.
With a [shared cache](#shared-caches), this offers opportunities for more reuse across projects.
With [zig-style compilation](#compilation-model), maybe compilation starts with `main` and the public functions of each opaque dependency, making this a hybrid between our current eager and zig's lazy builds,
allowing for some artifacts to be cached.

This also provides a way to avoid unifying features and dependencies,
a long standing request.

Areas
- [Dependency management](#dependency-management)
- [Build performance](#build-performance)

Tracking issue: [#3573](https://github.com/rust-lang/cargo/issues/3573)

<a id="test-caching"></a>

#### Test result caching

Cargo already has caching, including output, for builds.
Cargo could do the same for tests so long as Cargo knew all of the inputs to a test binary.

Areas
- [Build performance](#build-performance)

<a id="test-failures"></a>

#### Iterating on test failures

If we are more directly integrating libtest with `cargo test`,
Cargo can do basic things like running multiple test binaries at once.

We can go a step further and define new workflows.
For myself, I frequently run `cargo test`, figure out how to call the failure I'm focusing on, iterate until it passes, and then work on the next.
If Cargo had a fail-fast / run-failures-first mode,
this would remove a lot of friction to people having a faster test loop.

Areas
- [Build performance](#build-performance)
