---
title: "Speeding Up Rust Builds: Code-Gen Edition"
published_date: "2019-10-10 03:30:17 +0000"
layout: default.liquid
is_draft: false
data:
  tags:
    - programming
    - rust
---
*tl;dr Cache your code-gen results with the [`codegenrs` crate](https://crates.io/crates/codegenrs).*

Lately, there has been talk talk about improving build times, with a focus on
reducing bloat like [regex breaking out logic into features that can be
disabled](https://www.reddit.com/r/rust/comments/cz7u0j/psa_regex_13_permits_disabling_unicodeperformance/?st=k1k1fzk7&sh=69a9ab53),
[cargo-bloat going on a
diet](https://www.reddit.com/r/rust/comments/cg3p5m/cargobloat_08_debloated_5x_smaller_10x_faster/?st=k1k1cfs7&sh=f42a89f0),
[new cargo features to identify slow-to-build
dependencies](https://internals.rust-lang.org/t/exploring-crate-graph-build-times-with-cargo-build-ztimings/10975?u=dtolnay).
The area that has been impacting me lately is `build.rs`.  I've been
code-generating compile-time hash tables ([phf](https://crates.io/crates/phf))
which has added several dependencies to my build and takes a while.

Let's use [`imperative`](https://crates.io/crates/imperative) as an example.
`imperative` is a simple way to check a word is in the imperative-mood.  The
logic was taken from [pydocstyle](https://github.com/pycqa/pydocstyle) where it
was used to ensure the subject-line for a doc-comment should start with an
imperative-mood verb.

`imperative` uses:
- A set of blacklisted words (73 words).
- A map of word-stems to acceptable full-words (227 full-words).

And relies on the following unique dependencies:
- `phf_codegen`
- `multimap`

What if instead of code-generating in `build.rs` as part of `imperative` and
all dependents' builds, we checked in the result? The main risks are:
- Contributors forgetting to re-run code-gen
- Contributors changing the code-gen output without modifying the code-generator.

We can mitigate these risks by having the CI run a `--check` mode in the
code-generator that ensures the output matches what should be generated.

To setup `imperative-codegen`:
- Set `package.publish` to `false`.
- Add a dependency on [`codegenrs`](https://crates.io/crates/codegenrs).
- Write the [code-generator](https://github.com/crate-ci/imperative/blob/master/codegen/src/main.rs).
- Update the [CI to check the generated output](https://github.com/crate-ci/imperative/blob/master/azure-pipelines.yml).

Now let's look at some very unscientific numbers for clean builds of `imperative` (a
`lib` crate):

| `imperative` | `cargo check` | `cargo build` |
|--------------|---------------|---------------|
| `build.rs`   | 39.94 s       | 35.65 s       |
| `codegenrs`  | 22.55 s       | 26.06 s       |

*Note that this technique might also help make crates work better with
[alternative build
systems](https://people.gnome.org/~federico/blog/rust-build-scripts.html).*
