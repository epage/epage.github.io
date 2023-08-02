---
title: Are we GUI (build) yet?
layout: default.liquid
is_draft: true
data:
  tags:
  - programming
  - rust
---

["Are we GUI yet?"](https://areweguiyet.com/) is a topic that frequently comes up within the Rust community.
There are a lot of hard problems related to the complexity of state management
to something as easy to overlook as
[smooth resize](https://raphlinus.github.io/rust/gui/2019/06/21/smooth-resize-test.html).

One problem I don't see discussed often is a development and release pipeline
that can handle the unique circumstances of each target platform.
I met up with a bunch of application framework developers the day after
[RustNL](https://2023.rustnl.org/) to explore this problem and what can be done
for it.

<!-- more -->

## Conditional Compilation



## Build orchestration

`cargo` workflow is designed around building a binary for a specific platform target.

Application developers need to deal with more than just a binary but assets as well.
For those I talked to, assets are typically processed in another part of the
development pipeline.
They tended to pull these in through `git-lfs` and only had to do minimal
processing, including de-duplication and conversion to the format needed for
the platform target.

For some applications, plugins are needed as well.
This means a single `cargo build` is not sufficient as multiple build artifacts
could be needed.
Similarly, on macOS, a universal binary is built of the crate being built for
multiple platform targets, like x64 and aarch64.

These binaries also need post-processing, like signing.

All of this, then needs to be packaged for distribution.

But build orchestration is not unique to application development.
The [post-build script](https://github.com/rust-lang/cargo/issues/545) feature request contains many other examples.

The needs of a platform also evolve and can be encumbered with legal requirements.

Related to the actual build is `cargo run` and `cargo test` which are dependent
on this plus can have special requirements to run the build artifacts.
We do have [a config for customizing the runner](https://doc.rust-lang.org/cargo/reference/config.html#targettriplerunner) but that is only for `cargo`s build artifact and is without any of these post-build steps.
It is also [an environment setting rather than a project setting](https://internals.rust-lang.org/t/proposal-move-some-cargo-config-settings-to-cargo-toml/13336).

### Potential solutions

As I referenced earlier, people have been requesting
[a post-build equivalent of `build.rs`](https://github.com/rust-lang/cargo/issues/545).
While this might fill some needs, I don't think it fills enough and will
instead lead to future frustrations when people run up to its limits.
At minimum, this wouldn't help when needing to integrate multiple build artifacts.
If this were to run during `cargo install`, then it would need a way to access
assets, some degree of control over their layout, and the binary would need to
know where all of that went.

One idea that is a little more out there is we could do all of this in a
`build.rs` once we have [artifact dependencies](https://github.com/rust-lang/cargo/issues/9096).
Your `build.rs` would pull in the relevant build artifacts, post-process them,
and output the final artifact format you need for distributing.

Something you can do today is to create third-party subcommands that can extend
cargo with the needed functionality,
like [cargo-apk](https://crates.io/crates/cargo-apk).
However, a project cannot automatically set these tools up for developers.
People have requested ways to do this and while
[cargo-xtask](https://github.com/matklad/cargo-xtask) is a way to emulate it,
it only supports running your own programs and not declaring dependencies on
external ones.
Maybe [artifact dependencies](https://github.com/rust-lang/cargo/issues/9096) could help.
However, there is a gap between creating the simplicity of an
[`[alias]`](https://doc.rust-lang.org/cargo/reference/config.html#alias)
and overhead of a cargo-xtask.
I appreciate what deno did with
[deno task](https://deno.land/manual@v1.35.0/tools/task_runner)
where they created a cross-platform shell language.
Personally, I wish there was a modern TCL but I could see something like
[nushell](https://www.nushell.sh/) or
[duckscript](https://sagiegurari.github.io/duckscript/)
filling a role like this.

There are also more general build orchestration tools, ranging from the venerable `make`,
and related tools like 
[cargo make](https://crates.io/crates/cargo-make) and 
[just](https://crates.io/crates/just),
to the all-in-one solutions like [Buck2](https://buck2.build/) and
[Bazel](https://bazel.build/)
(see also ["Scaling Rust builds with Bazel"](https://mmapped.blog/posts/17-scaling-rust-builds-with-bazel.html)).
Buck2 has a lot of really great things going for it but I can't endorse it
whole heartedly.
Last I looked at the project, `cargo` support was only for publishing packages,
creating a bifurcated ecosystem where you have to choose between Buck2 or
cargo.
It also loses out on a lot of the usability of cargo,
eschewing convention for explicit configuration both in defining your build
plan and in how you specify what to build.
Worst of all, it suffers from closed governance with public code dumps (via git
mirroring) where you have to hope our interests align with theirs.

Maybe it'd make sense to [create a new build orchestration tool](https://xkcd.com/927/).

My biggest care abouts are:
- This should feel cohesive rather than cobbled together
- It should be as easy to use as cargo, with a strong focus on convention
  - Low overhead for common tasks
  - Encourage a common vocabulary of tasks between projects
- You should be able to compose tasks out of build rules, both that you wrote and that others have published
- The default path should be for working with the native Rust ecosystem allowing users to build on their existing knowledge and resources, rather than re-inventing how to build

While some things I'm not quite sure of include:
- Whether this should use some scripting language for tasks and build rules (e.g. nushell or duckscript), Rust, or be like cargo-make and say "yes"
- Whether development should be decoupled from the Rust ecosystem.  I appreciate projects like [rhai](https://crates.io/crates/rhai) for Rust-native scripting but I wouldn't want my embedded scripting language choice to change based on the language I'm using for my core logic (while not a fan, this is one of the benefits of Lua).  Similarly, I worry about having build orchestration system that feels Rust-exclusive, even if it isn't.

While I've talked about build orchestration, we need to keep in mind the entire user workflow, including
- More complicated code-generation than `build.rs` does today, see [cargo px](https://crates.io/crates/cargo-px)
- `run` tasks that can deploy to web browsers, mobile devices, etc with hot reloading
- Testing that can handle more complicated scenarios like stress tests and end-to-end tests that involve scarce, non-deterministic resources like hardware.
