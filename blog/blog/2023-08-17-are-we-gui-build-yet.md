---
title: Are we GUI (build) yet?
published_date: 2023-08-17 17:32:22.05568602 +0000
layout: default.liquid
is_draft: false
data:
  tags:
  - programming
  - rust
---
["Are we GUI yet?"](https://areweguiyet.com/) is a topic that frequently comes
up within the Rust community.
There are a lot of hard problems to solve, from the complexity of state management
to something as easy to overlook as
[smooth resize](https://raphlinus.github.io/rust/gui/2019/06/21/smooth-resize-test.html).
One problem I don't see discussed as often is a development and release pipeline
that can handle the unique circumstances of each target platform.
I met up with a bunch of application framework developers the day after
[RustNL](https://2023.rustnl.org/) to explore this problem and what can be done
for it.

<!-- more -->

### Challenges

Cargo's workflow is designed around building a binary for a target platform.
This doesn't work for macOS as users would need x64 and aarch64 binaries built
and bundled as a universal binary.
For some applications, there is also the need for plugins into the main runtime
to run and test together.

These binaries also need post-processing, like signing.

After all of this, they then need to be packaged for distribution, including any assets.
For those I talked to, assets are typically processed in another part of the
development pipeline.
They tended to pull these in through `git-lfs` and only have to do minimal
processing, including de-duplication and conversion to the format needed for
the platform target.

Related to the actual build is `cargo run` and `cargo test` which are dependent
on this but add extra requirements for running the build artifacts, like
deploying to an emulator or an embedded device.
We do have [a config for customizing the runner](https://doc.rust-lang.org/cargo/reference/config.html#targettriplerunner)
but that is only for `cargo`s build artifact, absent of any of these post-build steps.
It is also [an environment setting rather than a project setting](https://internals.rust-lang.org/t/proposal-move-some-cargo-config-settings-to-cargo-toml/13336).

Each platform's needs also evolve in breaking ways and can be encumbered with
legal requirements, making it difficult for cargo to support natively

And these build orchestration challenges are not unique to application development.
The [post-build script feature request](https://github.com/rust-lang/cargo/issues/545)
contains many other examples.

### Potential solutions

Support for a [post-build equivalent of `build.rs`](https://github.com/rust-lang/cargo/issues/545)
might fill some of these needs but I don't think it fills enough and will
instead lead to future frustrations when people run up to its limits.
At minimum, this wouldn't help when needing to integrate multiple build artifacts.
If this were to run during `cargo install`, then it would need a way to access
assets, some degree of control over their layout, and the binary would need to
know where all of that went.

While it might be a bit of an abuse of the features,
we could do all of this in a
`build.rs` once we have [artifact dependencies](https://github.com/rust-lang/cargo/issues/9096).
The `build.rs` would pull in the relevant build artifacts, post-process them,
and output the final artifact format needed for distribution.

Something you can do today is to create third-party subcommands that can extend
cargo with the needed functionality,
like [cargo-apk](https://crates.io/crates/cargo-apk).
However, a project cannot automatically set these tools up for developers.
People have requested ways to do this and while
[cargo-xtask](https://github.com/matklad/cargo-xtask) is a way to emulate it,
it only supports running your own programs and not declaring dependencies on
external ones.
Maybe [artifact dependencies](https://github.com/rust-lang/cargo/issues/9096) could help.
However, there is a gap between the simplicity of automating with a simple
[`[alias]`](https://doc.rust-lang.org/cargo/reference/config.html#alias)
and a fully blown cargo-xtask with all of its complexity.
I appreciate what deno did with
[deno task](https://deno.land/manual@v1.35.0/tools/task_runner)
where they created a cross-platform shell language to for greater flexibility
without sacrificing cross-platform support.
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
[Last I looked at the project](https://www.reddit.com/r/rust/comments/136qs44/hello_rrust_we_are_meta_engineers_who_created_the/),
`cargo` support was only for publishing packages,
creating a bifurcated ecosystem where you have to choose between Buck2 or
cargo.
It also loses out on a lot of the usability of cargo,
eschewing convention for explicit configuration both in defining your build
plan and in how you specify what to build.
Worst of all, it suffers from closed governance with public code dumps (via git
mirroring) where you have to hope our interests align with theirs.
[pixi](https://prefix.dev/docs/pixi/overview) is intriguing as it looks like it
tries to sit between the lower-level `make` approach and the all encompassing
`buck2` approach.

Dare we consider
[creating a new build orchestration tool](https://xkcd.com/927/)?
Many already exist and provide a lot of compelling features.
I'm just not sure they provide the right blend of features.
The question is how much we can work with them to get there or if the effort is
better spent trying out new ideas.

### Next steps

... yes, thats it.

When we discussed this, I felt like we left with more questions than answers.
I'm hoping by sharing some of these thoughts
new overlooked options can be pointed out,
maybe some unexpected collaborations will open up,
or maybe we get the confirmation that something new really is needed.

To summarize, I feel like the biggest concerns were:
- Users need flexibility at every step of the way
- It should be as easy to use as cargo, with a strong focus on convention
  - Encourage a common vocabulary (e.g. `build`, `test`, `run`) of tasks between projects
  - Low ceremony for common tasks
  - i.e. this should feel cohesive rather than cobbled together
- You should be able to compose tasks out of build rules, both that you wrote
  and that others have published
- The default path should be for working with the native Rust ecosystem,
  including cargo, allowing users to build on their existing knowledge and
  resources, rather than re-inventing it.

### Aside: rust or general solutions?

Something I've been grappling with and would also appreciate input on is when
we should focus on language-specific solutions or cross-language solutions.

For a lot of these bigger application problems,
a lot is in common across languages for solving them.
Likely, people will also be needing to use more than just Rust.
I'm also not a fan of "lock in", that migrating to or from Rust also requires
an overhaul of everything else, both in processes and in knowledge.
I'd rather we not isolate ourselves in a corner with our own set of tools.

I think this applies more broadly than just build orchestration but also other areas like
- Templates for project scaffolding ([rust-lang/cargo#5151](https://github.com/rust-lang/cargo/issues/5151))
- Embedded scripting langues (i.e. lua vs Rust-specific solutions like [rhai](https://crates.io/crates/rhai))

That isn't to say that everything should be language agnostic.
Cargo is great!
Having a language-specific solution for building packages reduces a lot of overhead.

And a challenge with going for a wider audience is its harder to settle on some de
factor standard(s) because there are more inputs to decisions leading to more
fragmentation which requires more work for a smaller set of solutions to "win
out'.

So where should we be drawing the line?
