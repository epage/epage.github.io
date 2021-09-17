---
title: Learnability of Rust
published_date: "2021-09-15 09:00:30 -0500"
data:
  tags:
    - programming
    - oss
    - rust
---

##  tl;dr

As part of improving the learnability of Rust, I propose:
- The `.crs` file subset of [cargo-script](https://github.com/DanielKeep/cargo-script) be brought into `cargo`
- We support converting `.crs` to full cargo projects with `cargo init --from <script>.crs`
- We collaborate on an ergonomics-focused standard-library-alternative, like [eztd](https://docs.rs/eztd)

## Use Cases

Something I've really appreciated about Rust community is taking seemingly opposing principles
and finding a way to bring them in harmony, for example:
- Memory safety without garbage collection
- Abstraction without overhead
- Concurrency without data races
- Stability without stagnation

One I've been contemplating for a while is "Learnability with control".  We've made incremental
improvements to learnability, for example changes in the 2018 and 2021 editions, but I feel
like we've only been working to a local maxima.  I appreciate the work
[boats did in sketching out an ideal](https://without.boats/blog/revisiting-a-smaller-rust/)
which, even if incompatible with other goals in Rust, helps lift our sights
past our local maxima.  More recently,
[Niko encouraged us to be bold and creative](https://www.youtube.com/watch?v=ylOpCXI2EMM).

In considering "learnability", we should also keep in mind the related "expert" workflows:
- Prototyping
- One-off programs

I'll use Python as our point of comparison in exploring these ideals in Rust
due to its popularity and my familiarity with it.

## Goals

Some parts of Python that stick out to me include:

1. Low friction exploratory programming (e.g. REPL or notebooks)
2. Low friction to experimenting (e.g. just open a ".py" file)
3. Low friction discovery of "good enough" packages (i.e. batteries-included)
4. Reads as pseudo-code
5. Small, easy to remember surface area (e.g. just use `[]` instead of a collection of subset/indexing functions)
6. Design out error cases (e.g. no out of bounds errors on `[]`)

*(yes, there are counter-examples)*

For Rust, we also want: Scale to control, when needed.  We don't want a split ecosystem with a high porting cost between them.

## Breaking It Down

![MC hammer: break it down]({{ site.base_url }}/img/hammer-time.gifv)

**Exploratory programming** comes in two parts
- Discovering what the actual API is
- Verifying behavior of the API

For API discoveability, Python has to deal with dynamically generated APIs,
which is less of a concern for Rust.  Rust does have traits that need to be
pulled in and crate features that need to be enabled.  For those in an IDE, I'm
assuming [rust-analyzer](https://rust-analyzer.github.io/) handles these cases.
For the rest of us, we have `rustdoc` which has high friction in being adapted to your live code.
I'm assuming us non-IDE users are enough of a minority that we don't need to invest in alternatives.

This still leaves us verifying assumptions on behavior of an API.  We can
invest in a REPL or a Notebook just for this use case but I'm thinking we'll
get a "good enough" solution faster by first investing in the next part.

Python gets you **quickly experimenting** on your ideas by allowing you to have just a `.py`
file.  Python doesn't scale too well though, running into problems with
relative imports and requiring packages to be installed globally or learning
about venvs.

I think we can do better in Rust.  We already have
[cargo-script](https://github.com/DanielKeep/cargo-script) as a starting point
for writing Rust code in a single file, pull in third-party dependencies, and
have unit tests.  The one additional piece I think is needed is turning a
`.crs` file into a full project with `Cargo.toml` for those times throw-away
code becomes maintained code.

This helps us with **exploratory programming** by being able to quickly write
throw-away code.  I know `cargo-script` also has expression evaluation but I do
not have experience with that and assume its too limited to help with the REPL
case.

Another benefit to adopting `cargo-script` is it'd make it easy to share
reproduction cases in bug reports.

For **discovery of packages**, Python takes the approach of
"batteries-included", meaning nearly everything you need is a part of the
stdlib.  This comes at the cost of everything having to maintain the same
compatibility guarantees, release on the same cadence, and original authors
moving on from their package.  The community sometimes leaves behind the
standard library and you have to be in-the-know of what the replacement is.

With that said, a lot of the community still relies on the stdlib, whether because
- Its good enough
- Its discoverable
- It reduces stress by not having to research and make a decision
- Its universally available
- It is unlikely to break

In Rust, we've had debates in the past about batteries-included,
[batteries-adjacent](https://lib.rs/crates/stdx),
[documentation](https://rust-lang-nursery.github.io/rust-cookbook/),
or [algorithms](https://crates.io/crates?sort=recent-downloads).

I think batteries-adjacent gets us most of the way there.  It ticks almost all
the boxes of batteries included while not being limited in its evolution.  For
being universally available, this is important in python mostly due to the
dependency management pain in Python which we have solved with `cargo`.  This
just leaves availability from isolated networks which is a minority case
(though a big pain for those that do have to live with it).

For breaking changes, we can mitigate this by the batteries-adjacent library
being composed of crates, rather than having logic of its own,  This makes it
easier to narrow down what sections broke compatibility and allows people to
depend on an older version of that subset, just adding a dependency and
changing their `use` statements.  If we maintain a certain level of friction to
the process of crates being added (required level of maturity, RFC process), we
are also likely to reduce the number of breaking changes.

I think a lot of Rust is almost there for **reading as pseudo-code**.  At its
best, Rust almost feels like Python.  Lifetimes and complex `where` clauses
stand out to me as pain points.  I think a learning focused standard
library-alternative that limits the use of lifetimes would both be easier to
grok and help encourage writing code that borderlines on pseudo code.  When
needing generics, favoring `impl` would also help to reduce some of the noise,
despite its downsides.

For a **small surface area**, if we take this standard library alternative, and
apply Python's "there is only one way to do it" mantra, I think we can reduce
quite a few functions at the cost of some runtime performance (for non-optimal
choices) and occasional extra work by the developer.

This standard library-alternative could **design out errors** as we re-examine
why different APIs might error or panic and look to generalize it.  For
example, instead of `text[range]` panicing on out-of-bounds, we re-define the
behavior to
[return at most the start and end bound](https://github.com/epage/eztd/blob/main/crates/eztd-core/src/string/mod.rs#L666).

A sticky question is whether such a library's string type should stay focused
on bytes or be focused on chars, avoiding panics when not aligned with character
boundaries.  Working on chars is safer and more familiar but it will surprise
people when a developer uses "lower level" Rust.

With this learning-focused API, we can even leverage people's muscle memories
to smooth out the transition to rust by [providing familiar
functions](https://github.com/epage/eztd/blob/main/crates/eztd-core/src/string/mod.rs#L334).

With all of this, we can still **scale to give users control** by providing interop
with the standard library.  If an optimizer points out a hot loop involving it,
a developer can drop down to the standard library version and make things
faster.  This is also important for interop with the ecosystem and allowing
gradual migration away from this standard library-alternative.

## Performance

Being easy-to-use doesn't mean performance has to be abysmal as
[Lily Mara's touched at RustConf](https://www.youtube.com/watch?v=CV5CjUlcqsw&list=PL85XCvVPmGQgACNMZlhlRZ4zlKZG_iWH5&index=4).

For example, `cargo-script` re-uses dependency builds across scripts, speeding up builds at the risk of contention.

For `eztd`, we can make `eztd::String` moderately performant with a no-lifetime API with
- Immutability
- [Small-string optimization](https://github.com/epage/eztd/blob/main/crates/eztd-core/src/string/inline.rs)
- [Wrapping the heap allocation with `Arc`](https://github.com/epage/eztd/blob/main/crates/eztd-core/src/string/shared.rs#L8)
- [Sharing `Arc<str>`s across instances by also tracking the relevant subset of the `str`](https://github.com/epage/eztd/blob/main/crates/eztd-core/src/string/shared.rs#L9-L10)

We can also reduce the build time overhead of `eztd` by
- Considering it a bug to have different versions of a dependency existing
- Leveraging features to disable sections of the crate

To further speed up build times, maybe we should consider sharing dependency builds across all builds, not just `cargo-scrupt`.

## `eztd`

I've started [`eztd`](https://github.com/epage/eztd) to showcase the ideas in
this post and with the hope that it encourages others to help make this
possible.

Be sure to check out `eztd`'s [design
guidelines](https://github.com/epage/eztd/blob/main/CONTRIBUTING.md#design-guidelines)
for some additional details on designing for ergonomics.

Just as important are the goals are the non-goals:
- Being faster than Python
- Providing stable vocabulary terms (i.e. we'll bump major frequently)
