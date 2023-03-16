---
title: 'anstream: simplifying terminal styling'
published_date: 2023-03-16 23:08:16 +0000
layout: default.liquid
is_draft: false
data:
  tags:
  - programming
  - rust
---
`anstream` is a new take on terminal styling for Rust and will be used in the upcoming clap 4.2 release.

Quick links
- [docs.rs](https://docs.rs/anstream/latest/anstream/)
- [crates.io](https://crates.io/crates/anstream)
- [repo](https://github.com/epage/anstyle/tree/main/crates/anstream)

<!-- more -->

## Challenges with Terminal Styling

Much like with syntax highlighting with code, styling a CLI is a quick way to
make the output more approachable and faster to scan.  Despite this, it took me
several years to add color to my CLIs because of the initial hump to get
started, having to answer questions like "which styling crate should I use?".

Initially, I took the cheap way out and used
[env_logger](https://crates.io/crates/env_logger) for all of my output,
relying on the coloring of the log levels.  A problem I had is I wanted to
customize the way the log levels looked but then they were no longer colored.
Going back now, it appears that `env_logger` has a bespoke styling API but I
never found it back then.

Once my applications matured to the point where coloring was the next step, I
finally sat down and dug into this.

[termcolor](https://crates.io/crates/termcolor) is a safe choice, being used in
applications like ripgrep.  The big selling point was support for older Windows
versions but this came at a huge cost to ergonomics.  As is usual in open
source, I decided to shortchange some Windows users for my own convenience and
decided against `termcolor`.

I ended up settling on [yansi](https://crates.io/crates/yansi) because
- The documentation made me feel more comfortable that it would work well for
  [CLI stylesheets](https://rust-cli-recommendations.sunshowers.io/managing-colors-in-rust.html#the-stylesheet-approach)
- It worked with any `Display` type
- It handled global control over styling, meaning I didn't need to thread
  styling state through my application
  - While great, this is why I skipped
    [owo-colors](https://crates.io/crates/owo-colors) which requires annotating
    every call with a is-supported check.
- It supported nesting of styled values which seemed useful though I never ended up using this

`yansi`s global control didn't work as well as I had hoped and I ended up
doing the bookkeeping myself using an experimental crate,
[concolor](https://crates.io/crates/concolor)
- I wanted to initialize `yansi`'s global with various [best practices](https://github.com/SergioBenitez/yansi/issues/18)
- `yansi`'s global is a simple `bool`, disallowing the caller to [track stdout and stderr separately](https://github.com/SergioBenitez/yansi/issues/17) when manually implementing the above best practices.
- `yansi` requires manual initialization which [doesn't work well in testing libraries](https://github.com/SergioBenitez/yansi/issues/20)

As my use cases got more complicated, I found I wanted to abstract away the
rendering code to make it more compoasable but it still had to know about the
capabilities of what it was being written to (`stdout`, `stderr`, or a file),
requiring that I hard code that or thread it through.
This is a problem shared by `termcolor`, `yansi`, `owo-colors`, etc.

The next challenge I ran into was supporting styling in library APIs in a semver-stable
fashion.  For example, creating customized sections in clap's help output
[stands out like a sore thumb as they can't be styled like the rest of the output](https://github.com/clap-rs/clap/issues/1433).
The options for fixing this aren't great:
- Expose the internal styling library (`termcolor` in clap's case), tying our API version to theirs
- Wrapping the internal styling library with our own API, allowing us to change
  it as desired, at the cost of a bespoke API that won't ever be as good as a
  third-party because it isn't the focus of clap and increasing clap's API surface area dramatically.

## Breaking the Gordian Knot with `anstream`

Whenever I dig into a Rust project, the first thing I do is look for unfamiliar
dependencies to see if there are knew gems to discover.  When browsing cargo's
`Cargo.toml`, I found [fwdansi](https://docs.rs/fwdansi), a
crate that allows you to convert ANSI escape codes into `termcolor` API calls.

What if a terminal styling library's API was made of ANSI escape codes and the `std::io::Write` trait?
- Hiding stdout/stderr details being `std::io::Write` would allow the rendering code
  to be independent of what it is being output to
- ANSI escape codes are likely more stable than any crate API I'll come across

Of course, you still need *something* for generating the ANSI escape codes.  So far, I've been using a mix of
- [anstyle](https://crates.io/crates/anstyle) for a simple API without bells and whistles
- [owo-colors](https://crates.io/crates/owo-colors) for ergonomics and **because** any conditional styling is opt-in (the very reason I previously avoided it) allowing me to avoid it
- [color-print](https://crates.io/crates/color-print) for styling at compile-time

Now you can write code like:
```rust
use anstream::println;
use owo_colors::OwoColorize as _;

// Foreground colors
println!("My number is {:#x}!", 10.green());
// Background colors
println!("My number is not {}!", 4.on_red());
```
and `println` will handle
- Stripping ANSI escape codes when printing to a file, `TERM=dumb`, `NO_COLOR` is set, `CICOLOR=0` is set, or if disabled by the application like with `--color never`
- Adapting ANSI escape codes to the wincon API for older Windows versions that don't support opting in to ANSI escape code support
- Passing through ANSI escape codes otherwise

Of course, I thought this was ingenious of me but it turned out that I'm not the
first person to think of this.  The python library
[colorama](https://pypi.org/project/colorama/) does just this and
[burnntsushi just didn't want to put in the effort for parsing ANSI escape codes](https://internals.rust-lang.org/t/terminal-platform-abstraction/6746/8?u=epage)
(thankfully I was able to build on [the work](https://crates.io/crates/vte) from Alacritty which came much later than `termcolor`).

### Performance

Post-processing comes at a cost and if its too high, this whole effort is dead
in the water.   Ironically, rendering of ANSI escape codes is the case with the
lowest overhead (auto-detect and pass-through) but most likely it could take the
performance hit since its being rendered to the screen which is already a
relatively slow operation.

Piping to a file is usually where people will care most about performance so I
created a fast way to strip ANSI escape codes:

| `linux $ rg -i linus` | time |
|-----------------------|------|
| [strip-ansi-escapes](https://crates.io/crates/strip-ansi-escapes) | `[2.0897 ms 2.1009 ms 2.1129 ms]` |
| `anstream::adapter::strip_str` | `[210.38 µs 211.53 µs 213.04 µs]` |
| `ansdtream::adapter::strip_bytes` | `[238.22 µs 238.94 µs 239.66 µs]` |

*Note: `strip_bytes` is slower than `strip_str` because of some ambiguity
between UTF-8 characters and some escape codes that goes away when we know the
input is UTF-8*

I implemented support for `wincon` mostly because I felt it less effort than
deciding whether it was still worthwhile or not to support and then try to
convince any holdouts if it wasn't.  This was not the highest priority for
optimizations.

| `linux $ rg -i linus` | time |
|-----------------------|------|
| `anstream::adapter::wincon_bytes` | `[939.34 µs 942.79 µs 946.42 µs]` |

For a real-world benchmark, I decided to check how much this slows down my
source code spell checker, [typos](https://github.com/crate-ci/typos), on the
Linux kernel.  As of f76da4d5ad51, `typos` v1.13.24 reported (in color) 176,210 possible misspellings across 71,530 files:

| `linux $ typos > /dev/null` | time |
|-----------------------------|------|
| v1.13.23                    | 20.082 s ±  0.111 s |
| v1.13.24                    | 20.426 s ±  0.104 s |

*Note: see the [PR](https://github.com/crate-ci/typos/pull/688) for more details*

### `anstream` and `clap`

Supporting `anstream` in `clap` is
[ready to go](https://github.com/clap-rs/clap/pull/4765) but it is a big
commitment in a highly visible crate like `clap` to change from the
tried-and-true `termcolor` to a crate that has only existed for 9 days
(starting its life under the `anstyle-stream` name) and is reliant on complex
parsers doing the right thing, no matter how well tested.

My plan is to let the above PR sit for a little bit as I get more experience
with `anstream` in my other crates and in case any feedback from this post
becomes relevant to when/if clap switches to `anstream`.

> Aside: some clap users might be surprised by this effort as
> [theming support](https://github.com/clap-rs/clap/issues/3234) is the top priority.
> The plan for theming support is to use
> [anstyle](https://crates.io/crates/anstyle) in clap's API once it hits 1.0 but feedback on the
> `anstyle`s API has been slow.  Work on `anstream` was a way for me to get some more
> practical experience with `anstyle` to vet its API and make improvements.

I also see `clap` as a testing ground for further adoption in places like
`env_logger`, dropping the bespoke styling API, or `cargo` which would make it
easier nicer looking commands.
