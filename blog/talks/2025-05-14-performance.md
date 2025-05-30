---
layout: remark2.liquid
format: Raw
permalink: /{{parent}}/{{year}}/{{month}}/{{slug}}/
data:
  venue: RustWeek
---
class: title
name: title
# Performance

## We're In This Together

???

Hmmm, there is a lot of different kinds of performance, maybe I need to clarify

*Below is non-speaking:*

Goals:
- Solicit input on workflows
- Build excitement for the All Hands topic
- Seed the All Hands conversation so we're ready to go

Take-away:
- We need to look at user workflows, not tools
  - Keep in mind survivorship bias: what are people avoiding because its slow?
  - Impact on workflows is in buckets, not percents
- Improving build-time performance requires crossing teams and even out into the community
  - Requesting features that may not make sense in isolation
  - Both sides need to participate to get the benefit
  - Some approaches may negate each other

---
class: title
# **Build** Performance

## We're In This Together

???

As this is the Project track, we're talking about making the product we produce, Rust, faster.

---
## Build Performance

![rustc-perf graph]({{site.base_url}}/talks/perf-rustc-trend.png)

???

A lot of great work has been put into making Rust faster, including:
- Incremental improvements
- Incremental compilation
- Pipelined builds
- Parallel backend

*[Source](https://perf.rust-lang.org/dashboard.html)*

---
## On Roc's Zig migration

> ### Why Zig over Rust?
>
> **Rust's compile times are slow.** Zig's compile times are fast. This is not the only reason, but it's a big reason.
>
> Having a slow feedback loop has been **a major drain on** both our **productivity** and also our **enjoyment** when working on the code base. Waiting several seconds to build a single test, before it has even started running, is not enjoyable.

*\- Richard Feldman on migrating Roc from Rust to Zig*

???

Unfortunately, its not enough.

A factor in Roc moving away from Rust is the affect of build times.
This isn't just making things slower, its affecting how people feel when programming Rust.

People aren't needing 1% to 2% improvements, they are needing order of magnitude improvements.

There is one problem with this, according to a Zig developer, its actually slower than Rust.

*[Source](https://gist.github.com/rtfeldman/77fb430ee57b42f5f2ca973a3992532f)*

---
## On Roc's Zig migration

> **In contrast, there isn’t anything comparable on the Rust roadmap** in terms of performance improvements.
> The Cranelift backend has been WIP since 2019 when Roc had its first line of code written,
> so although I do expect it will land eventually,
> it’s not like Zig’s x86_64 backend which is (as I understand it)
> close enough to done that it may even land in the next release in a about a week,
> and which of course will be significantly faster than a Cranelift backend would be.

*\- Richard Feldman on migrating Roc from Rust to Zig*

???

Richard clarified that it was Zig's roadmap and trust in their ability to deliver that made the difference.

This frustration with build times is not unique to Roc.

While our LoC for LoC performance might be on par with C++,
- Non-C++ developers have **higher expectations**
- C++ developers don't rebuild the world.  Are game developers regularly **rebuilding Unreal from scratch?**

So how do we fix this?

*[Source](https://lobste.rs/s/0jknbl/roc_rewrites_compiler_zig)*

---
class: title

# Nature of Performance

???

We first need to look at performance from the user's perspective.

---
## Nature of Performance

![rustc-perf graph]({{site.base_url}}/talks/perf-rustc-trend.png)

???

Tracking performance is important but when looking at things from the users perspective,
a 10% performance improvement doesn't mean much.

---
## Nature of Performance

![performance buckets]({{site.base_url}}/talks/perf-buckets.png)

???

What matters if how people will **react to a delay**.
**If you lose their attention**, they will never notice a small improvement because they are off doing something else.
This makes the performance bars discrete with large jumps between them.

But reactions to what?

---
## Nature of Performance

Workflows involve iterating on:

- Errors and warnings
- Test failures
- Exploratory testing
- CI feedback
- Reviewer feedback

???

We can't stop here.  What people are doing within these workflows and problems they encounter in them are just as important.

---
## Nature of Performance

![cargo tree on zed]({{site.base_url}}/talks/perf-zed-workspace.png)

???

In the case of zed, there are over 150 workspace members with deep nesting (up-to 9 deep)
Iterating on compilation errors and tests in this kind of situation is dramatically
**different than working on leaf packages**.

This also isn't just about rustc or even cargo's performance but **about how they behave**.

Examples:

If Cargo and rust-analyzer are fighting over your build cache, you could end up doing full builds every time.

If you jump around your workspace to experiment with changing or testing different parts, the changing of activated features can cause a significant amount of rebuilding.

---
class: center,middle

# We need to talk to users

The bigger the better

???

We need to understand how they are interacting with Rust to identify what is impacting them

A major source of inspiration for this presentation was from Remy and I meeting with people from Zed and other projects.

---
## Nature of Performance

![tip: delete doctests]({{site.base_url}}/talks/perf-delete-doctests.png)

*\- matklad, Delete Cargo Integration Tests*

???

In particular, what are they working around or not using?
Could it be because of performance?

Here, we are seeing build-time advice to avoid doctests

*[Source](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html)*

---
## Nature of Performance

doctest execution time:
![merged doctests]({{site.base_url}}/talks/perf-merge-doctests.png)

???

These become rich areas to target with many positive side effects.

And in this case, it led to a significant reaction-changing improvement.

*[Source](https://blog.guillaume-gomez.fr/articles/2024-08-17+Doctests+-+How+were+they+improved%3F)*

---
## Nature of Performance

> ### Third-party crates
>
> Although the compiler improvements are nice, some declarative macros remain expensive, and rewriting or removing them altogether **from third-party crates is worthwhile.**

*\- nnethercote, How to speed up the Rust compiler in April 2022*

???

This can involve finding alternative ways around a problem, working cross-team, or even making changes to the ecosystem.

Nick was a great inspiration in this area.
When he hit the limit of optimizing declarative macros, he looked crates and found their macros' algorithmic complexity could be reduced.
He even updated the Little Book of Macros to talk about this.

*[Source](https://nnethercote.github.io/2022/04/12/how-to-speed-up-the-rust-compiler-in-april-2022.html)*

---
class: title

# Unexpected Improvements

???

When looking at why things are slow, the way to improve things can be **indirect**.
Work from one team might unexpectedly unblock improvements of interest to another team.

For example, what if I said that **T-lang moving forward with implicit `mod` statements** could speed up `cargo test`?

---
## Unexpected Improvements

![tip: merge itests]({{site.base_url}}/talks/perf-merge-itests.png)

*\- matklad, Delete Cargo Integration Tests*

???

Instead of generating an integration test binary per file, matklad recommended having a single integration test.

Granted, this is blurs the line of working around slow linking vs how things should be.

Managing this manually takes **more work and is error prone**, making it harder to recommend as the default choice for cargo users.

If we had "auto mod", this becomes a lot more effortless and we can consider making this the expected path for users.

Admittedly, this doesn't mean that there aren't complications or reasons *not* to do "auto mod"
but we should consider it and
it serves as an example of how changing our feature set can drive performance improvements without the slow slog of profiling and experimenting.
Granted, this instead involves the slog of an RFC.

*[Source](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html)*

---
## Unexpected Improvements

Feature Name: `cfg_version` and `cfg_accessible`

Permit users to `#[cfg(..)]` on whether:
- they have a certain minimum Rust version (`#[cfg(version(1.27.0))]`).
- a certain external path is accessible (`#[cfg(accessible(::std::mem::ManuallyDrop))]`).


???

Have you considered that these are a way to improve performance in Rust?

Looking at the depednencies for a "basic" web application and it had around 10 build scripts dealing with version detection with 2 proc macros built on top of that.

That requires us to build, link, and run each of these.

Its a similar story for other features like `cfg_alias`

---
class: title

# Competing Improvements

???

Another area that we need to talk through is when there are multiple major routes for improving bjild performance to understand which we should put our focus towards.

---
## Competing Improvements: macros

.column-1-3[

Ways to help proc-macros:
- Cache expansion
- Include expansion in `.crate` files
- Stabilize more features

]

???

There are several things we could explore for improving build times with proc-macros

Granted, proc-macros have other problems, like
- Build input tracking for caching
- Running of arbitrary code
- Lack of `$crate`

---
## Competing Improvements: macros

.column-1-3[

Ways to help proc-macros:
- Cache expansion
- Include expansion in `.crate` files
- Stabilize more features

]

.column-2-3[

Replace with declarative macros:
- Add `derive` and attribute macros
- Add function, struct, and enum patterns

]

???

Alternatively, we could focus on improving declarative macros. 
They can't replace every proc-macro but they can replace most to the point that people could more easily have a feature rich application without a single proc-macro.

This also means we avoid some of the other problems with proc-macros

---
## Competing Improvements: macros

.column-1-3[

Ways to help proc-macros:
- Cache expansion
- Include expansion in `.crate` files
- Stabilize more features

]

.column-2-3[

Replace with declarative macros:
- Add `derive` and attribute macros
- Add function, struct, and enum patterns

]

.column-3-3[

Replace most derives with reflection:
- Experiment with `facet`

]

???

I'm also really curious where Amos investigation into `facet` will lead.
This is still very young and there are areas that need further exploration and refinement.

The main limitations:
- Doesn't help with attribute macros
- Can't do compile-time transforms, like case conversions

But for a lot of derive use cases, we could see a lot of benefit.

So this is an example where there are multiple avenues for improvement within one or two teams and all are legitimate but if we spread thin our attention,
it can greatly limit our ability to make progress and reach *an* improvement.

*Self-notes: My main concerns are*
- `unsafe` (mostly an issue with this as a library)
- Is field visibility respected?
- How are attributes defined and stored?

---
## Competing Improvements: caching

.left-column[

cross-workspace caching
- Remote caching in future?

]

???

As another example, Cargo has been interested in cross-workspace caching.

Granted, build scripts and proc-macros are a problem for this and would be left out of the initial version.
You also would get unique cache entries if any dependency version is different.

To solve this, we also would likely to solve most of the problem of fine-grained locking of `target-dir`

However, the remote caching would not help with caching of workspace members

**Note that I'm not saying "pre-built packages".**
That would break Cargo's model where the end-application has full control over dependencies including
- Versions, including patching
- Profile
- RUSTFLAGS

We'd need to develop a whole new model of **"opaque dependencies"** to get that to work, much like how **`std`** is opaque.

---
## Competing Improvements: caching

.left-column[

cross-workspace caching
- Remote caching in future?

]


.right-column[

MIR-only rlibs
- Cross-crate on-demand compilation in the future?

]

???

There is also interest in MIR-only rlibs
- Avoid duplicate code-gen for generics
- Cross-crate dead code elimination before codegen

This would mean every artifact in your workspace has to **duplicate codegen**, reducing how much **caching we do within a project.**

If we can't reuse backend compilation between artifacts within a workspace, then **we also can't do it cross-workspace.**

So this means we could only **frontend caching**.
As extra **caching coordinating has its own cost**, does the frontend take long enough to overcome the caching costs?

A naive implementation would also lose out on **per-package profile overrides** which can be essential for test time.

All of this would be worse with **Zig-style** cross-crate on-demand compilation as that would happen earlier than MIR.
But, it also means that we wouldn't need to **use Cargo features as often** to allow people to opt-out of unused code which would also have usability improvements.
Imagine `windows-sys` without any features!

Maybe if MIR-only rlibs everywhere is far enough out that maybe caching could pay off in the **"short term"** (even though its far out as well).

I mentioned opaque dependencies.  Maybe we could use those as a deciding line for codegen.

---
## Competing Improvements: caching

.center[

![its complicated]({{site.base_url}}/talks/perf-conspiracy.jpg)

]

???

We need to unravel these threads which are cross-team to know what direction will pay off if we move forward.

---
class: center,middle

# We need to talk to each other

All Hands, Saturday at 9:30

???

So to move forward with these big performance improvements, we need to coordinate across the project,
whether its raising awareness of how other teams can help us with performance or to not negate each others work

---

## Ideas I'm aware of

.column-1-3[

- Alternative linker (almost there!)
- Parallel frontend
- Faster backend
- `cargo build` reusing `cargo check`
- "API" fingerprinting
- Cross-workspace caching`
- Remove caching
- MIR-only rlibs
- Zig-style compilation
- Fine-grained target-dir locking

]

.column-2-3[

- `cfg(version)`
- `cfg_value!()`
- `cfg_alias`
- Delegating build scripts
- System-deps
- Proc-macro expansion caching
- Replace proc-macros with declarative macros
- Reflection
- Faster `dev` profile
- `check` or `test`-specific opt-level

]

.column-3-3[

- Parallel `cargo fix`
- Introspection on rebuild reasons
- Port unit test tidy check to clippy
- "auto mod"
- Integrate doctests into Cargo's build process
- Coverage-driven test selection
- Stabilize cargo-workspace-hack
- Opaque dependencies

]

???

And there is a lot to talk about!
This is only what I'm aware of and would love to hear from you on what ideas you have as well.

I also can't do this on my own, even limiting this to the parts within Cargo.
We need help to deliver on this.
