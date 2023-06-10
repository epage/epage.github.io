---
title: Iterating on Testing in Rust
published_date: 2023-06-09 19:41:08.801406524 +0000
layout: default.liquid
is_draft: false
data:
  tags:
  - programming
  - rust
---
With the release of rust 1.70, there was
[some surprise and frustation](https://www.reddit.com/r/rust/comments/13xqhbm/announcing_rust_1700/jmji422/)
that
[unstable `test` features now require nightly](https://blog.rust-lang.org/2023/06/01/Rust-1.70.0.html#enforced-stability-in-the-test-cli),
like all other unstable features in Rust.
One of the features most affected is `--format json` which has been in
[limbo for 5 years](https://github.com/rust-lang/rust/issues/49359).
This drew attention to a feeling I've had: the testing story in Rust has been
stagnating.
I've been gathering my thoughts on this for the last 3 months and recently had
some downtime between tasks so I've started to look further into this.

The tl;dr is to think of this as finding right abstractions to stabilize parts
of
[cargo_test_support](https://doc.crates.io/contrib/tests/writing.html#cargo_test-attribute)
and [`cargo nextest`](https://nexte.st/).

<!-- more -->

## Testing today

Running `cargo test` will build and run all test binaries
```console
$ cargo test
   Compiling cargo v0.72.0
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running  /home/epage/src/cargo/tests/testsuites/main.rs

  running 1 test
test some_case ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Test binaries are created from a package's `[lib]` and for each `.rs` file in
`tests/`.
[`test`](https://doc.rust-lang.org/stable/test/).
```console
awesomeness-rs/
   Cargo.toml
   src/          # whitebox/unit tests go here
    lib.rs
    submodule.rs
    submodule/
      tests.rs
   tests/        # blackbox/integration tests go here
     is_awesome.rs
```

A test is as simple as adding the `#[test]` attribute and doing something that
panics:
```rust
#[test]
fn some_case() {
    assert_eq!(1, 2);
}
```

And that is basically it.
There are a couple more details (doc tests, other attributes) but testing is
relatively simple in Rust.

## Strengths

Before anything else, we should recognize the strengths of the existing story
around testing; what we should make sure we protect.
Scouring forums, some points I saw called out include:
- Low friction for getting tests up and running
- Being the standard way to test makes it easy to jump between projects
- High value-to-ceremony ratio
- Exclusively running tests in parallel puts pressure on tests being scalable

## Problems

For some background, when you run `cargo test`, the logic is split between two
key pieces:
- `cargo test` command which enumerates, builds, and runs test binaries pretty
  much only caring about their exit code
- [libtest](https://doc.rust-lang.org/stable/test/) which is linked into each
  test binary and parses flags, enumerates tests, runs them, and prints out a
  report.

### Conditional ignores

libtest is static.
If you `#[ignore]` a test, that is it.
You can make a test conditioned on a platform or the presence of feature flags,
like `#[cfg_attr(windows, ignore)]`.
However, you can't ignore tests based on runtime conditions.

In cargo, we have tests that require installed software.
The naive approach is to return early:
```rust
#[test]
fn simple_hg() {
    if !has_command("hg") {
        return;
    }

    // ...
}
```
But that gives developers a misleading view of their test coverage:
```console
$ cargo test
   Compiling cargo v0.72.0
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
     Running  /home/epage/src/cargo/tests/testsuites/main.rs

  running 1 test
test new::simple_hg ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```
In cargo, we've worked around this by providing a custom test macro that *at compile
time* checks if `hg` is installed and adds an `#[ignore]` attribute:
```rust
#[cargo_test(requires_hr)]
fn simple_hg() {
    // ...
}
```
```console
$ cargo test
   Compiling cargo v0.72.0
    Finished test [unoptimized + debuginfo] target(s) in 16.49s
     Running  /home/epage/src/cargo/tests/testsuites/main.rs

running 1 tests
test init::simple_hg::case ... ignored, hg not installed

test result: ok. 0 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

Having to wrap `#[test]` isn't ideal and requires you to bake in every runtime
condition into your macro.
This also then doesn't compose with other solutions.

Cargo is also unlikely to be able to recognize that it needs to recompile tests
when the conditions change.

See also [rust-lang/rust#68007](https://github.com/rust-lang/rust/issues/68007).

### Test generation

Data driven tests are an easy way to cover a lot of cases (granted, property testing is even better).
The most trivial way of doing this is just looping over your cases, like this code from `toml_edit`:
```rust
#[test]
fn integers() {
    let cases = [
        ("+99", 99),
        ("42", 42),
        ("0", 0),
        ("-17", -17),
        ("1_2_3_4_5", 1_2_3_4_5),
        ("0xF", 15),
        ("0o0_755", 493),
        ("0b1_0_1", 5),
        (&std::i64::MIN.to_string()[..], std::i64::MIN),
        (&std::i64::MAX.to_string()[..], std::i64::MAX),
    ];
    for &(input, expected) in &cases {
        let parsed = integer.parse(new_input(input));
        assert_eq!(parsed, Ok(expected));
    }
}
```

However,
- You don't know which `input` was being processed on failure (without extra steps)
  - Any debug output from prior iterations will flood the display when analyzing a failure
- Its "fail-fast": a broken case prevents other cases from running, requiring careful ordering
  to ensure the more general case is first
- You don't get the bigger picture of whats working and or not by seeing all of the failures at once
- You can't select a specific case to run / debug

Some projects will create bespoke macros are created so you get a `#[test]` per
data point.
When this happens frequently enough across projects, people will write their
own libraries to automate this, including:
- [trybuild](https://docs.rs/trybuild/latest/trybuild/)
- [trycmd](https://docs.rs/trycmd/latest/trycmd/)
- [toml-test](https://docs.rs/toml-test/latest/toml_test/)
- [criterion](https://docs.rs/criterion/latest/criterion/)

Or [libtest-mimic](https://docs.rs/libtest-mimic/latest/libtest_mimic/) for the choose-your-own adventure route.

For example, with trybuild, you create a dedicated test binary with a test
function and everything is then delegated to trybuild:
```rust
#[test]
fn ui() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

And you get this output:
![trybuild output](https://user-images.githubusercontent.com/1940490/57186576-7b0b5200-6e96-11e9-8bfd-2de705125108.png)

Alternatively, some testing libraries replace libtest as the test harness
```toml
# Cargo.toml ...

[[bench]]
name = "my_benchmark"
harness = false
```
```rust
use std::iter;

use criterion::BenchmarkId;
use criterion::Criterion;
use criterion::Throughput;

fn from_elem(c: &mut Criterion) {
    static KB: usize = 1024;

    let mut group = c.benchmark_group("from_elem");
    for size in [KB, 2 * KB, 4 * KB, 8 * KB, 16 * KB].iter() {
        group.throughput(Throughput::Bytes(*size as u64));
        group.bench_with_input(BenchmarkId::from_parameter(size), size, |b, &size| {
            b.iter(|| iter::repeat(0u8).take(size).collect::<Vec<_>>());
        });
    }
    group.finish();
}

criterion_group!(benches, from_elem);
criterion_main!(benches);
```

Custom harnesses are a second class experience
- Require their own test binary, distinct from other tests
- Varying levels of support or extensions for how to interact with them
- The cost for writing your own is high

### Test Initialization and Cleanup

When talking about this, people generally think of the classic JUnit setup with its own downsides:
```java
public class JUnitTestCase extends TestCase {
    @Override
    protected void setUp() throws Exception {
        // ...
    }

    public void testSomeSituation() {
        // ...
    }

    @Override
    protected void tearDown() throws Exception {
        // ...
    }
}
```

In Rust, we generally solve this with RAII:
```rust
fn cargo_add_lockfile_updated() {
    let scratch = tempfile::tempdir::new().unwrap();

    // ...


}
```

This has its own limitations, like some teardown errors being ignored.
I've had bugs masked by this on Windows, requiring manual cleanup to catch them:
```rust
fn cargo_add_lockfile_updated() {
    let scratch = tempfile::tempdir::new().unwrap();

    // ...

    scratch.close().unwrap();
}
```

Sometimes generic libraries like `tempfile` aren't sufficient.
Within cargo, we intentionally leak the temp directories, only cleaning them
up on the next run so people can debug failures.
This is also provided by `#[cargo_test]`.
However, we have regularly hit CI storage limits and it would be a big help if
the fixture tracked the size of these directories, much like tracking test
times.

Cargo also has a lot of fixture initialization coupled to the directory managed
by `#[cargo_test]`, requiring a package to buy-in to the whole system
to just use a little portion of it.

Sometimes a fixture is expensive to create and you want to be able to share it.  For example in cargo,
[we sometimes put multiple "tests" in the same function to share the fixture](https://github.com/rust-lang/cargo/pull/11023/files#diff-9e3e31cc30eb728fcb75dcddf441dc127b0eb8cf764b4ce6cc260c3ec9487c26),
running into similar problems as we do with the lack of test generation.

The counter argument could be made that we just aren't composing things right.
That is likely the case but I feel this organic growth is the natural outcome
of not having better supporting tools and needing to prioritize our own
development.

Having composable fixtures would go a long way towards making test code more reusable.
Take for instance [pytest](https://pytest.org/).
In a previous part of my career, I made a Python API for hardware that
interacted with the
[CAN bus](https://en.wikipedia.org/wiki/CAN_bus).
This had to be tested at the system level and required access to hardware.
With pytest, I could specify that a
[test required a can_in_interface resource](https://github.com/ni/nixnet-python/blob/main/tests/test_session_j1939.py).
[can_in_interface is a fixture](https://github.com/ni/nixnet-python/blob/main/tests/conftest.py#L30)
that could be initialized from the command-line and skip all dependent tests, if not specified.

```python
def pytest_addoption(parser):
    parser.addoption(
        "--can-in-interface", default="None",
        action="store",
        help="The CAN interface to use with the tests")

@pytest.fixture
def can_in_interface(request):
    interface = request.config.getoption("--can-in-interface")
    if interface.lower() == "none":
        pytest.skip("Test requires a CAN board")
    return interface

def test_wait_for_intf_communicating(can_in_interface):
    # ...
```

We have crates like [rstest](https://crates.io/crates/rstest), but they are
like `#[cargo_test]` and build on top of libtest.
We can't extend the command-line, have fixtures skip tests, and so on

### Scaling up development

As we've worked around limitations, we've lost the strength of transferability.
These solutions are also not composable; if there isn't a custom test harness
that works for your case, you have to build everything up from scratch.
libtest-mimic reduces the work to build from scratch but it still requires you
to do this for each scenario, rather than having a way to compose testing
logic.
This takes a toll on those projects until they say enough is enough and build
something custom.

### Friction between `cargo test` and libtest

So far I've mostly talked about test writing.
There are also problems with test running and reporting.
I think [cargo nextest](https://nexte.st/) has helped highlight gaps in the
current workflow.
However, `cargo nextest` is working within the limitations of the existing system.
For example, what would normally be attributes on the test function in other language's test
libraries, you have to specify in a separate config file.
`cargo nextest` also does process isolation for tests.
While it has benefits, I'm concerned about what would lose by making this the default workflow.
For example, you can't run `cargo nextest` on cargo today because of shared
state between tests, in particular the creation of short identifiers for temp
directories which allows us to have a stable set of directories to use and
clean up from.
Process isolation also gets in the way of trying to support shared fixtures in
the future.

Going back over our backlog, we've problems related to `cargo test` and libtest's interactions include:
- [Wanting to run test binaries in parallel](https://github.com/rust-lang/cargo/issues/5609), like `cargo nextest`
- [Lack of summary across all binaries](https://github.com/rust-lang/cargo/issues/4324)
- [Noisy test output](https://github.com/rust-lang/cargo/issues/2832) (see also [#5089](https://github.com/rust-lang/cargo/issues/5089))
- [Confusing command-line interactions](https://github.com/rust-lang/cargo/issues/1983) (see also [#8903](https://github.com/rust-lang/cargo/issues/8903), [#10392](https://github.com/rust-lang/cargo/issues/10392))
- [Poor messaging when a filter doesn't match](https://github.com/rust-lang/cargo/issues/6151)
- [Smarter test execution order](https://github.com/rust-lang/cargo/issues/6266) (see also [#8685](https://github.com/rust-lang/cargo/issues/8685), [#10673](https://github.com/rust-lang/cargo/issues/10673))
- [JUnit output is incorrect when running multiple test binaries](https://github.com/rust-lang/rust/issues/85563)
- [Lack of failure when test binaries exit unexpectedly](https://github.com/rust-lang/rust/issues/87323)

## Solution

To avoid optimizing for a local maxima, I like to focus on the ideal case and
then step back to what is practical.

### My ideal scenario

Before, I made a reference to pytest.
That has been the best model for testing I've used so far.
It provides a shared, composable convention for extending testing capabilities
that I feel help in the scenarios I mapped out.

Runtime-conditional ignores:
```python
@pytest.mark.skipif(!has_command("hg"), reason="requires `hg` CLI")
def test_simple_hg():
    pass
```

Case generation
```python
@pytest.mark.parametrize("sample_rs", trybuild.find("tests/ui/*.rs"))
def ui(sample_rs):
    trybuild.verify(sample_rs)
```

Initialization and cleanup
```python
def cargo_add_lockfile_updated(tmpdir):
    # ...
```

As for  the UX, we can shift some of the responsibilities from libtest to
`cargo test` if we formalize their relationship.
Currently, `cargo test` hands off all responsibilities and makes no assumptions
about command-line arguments, output formats, whats safe to run in parallel,
etc.

Of course, this isn't all that simple or else it would have already been done.
For libtest, its difficult to get feedback on unstable features which is
one reason things have remained in limbo for so long.
This also extends to stabilizing the json output for allowing tighter
integration between `cargo test` and libtest.
A naive approach to tighter integration would also be a breaking change as it
changes expectations for custom test harnesses and even individual tests are
run.

### Prototype

I started [prototyping the libtest-side of my ideal scenario](https://github.com/epage/pytest-rs/)
while waiting for some FCPs to close out.
My thought was to start here, rather than on `cargo test` as this would let me
explore what the json output should look like before working to stabilize it
for `cargo test` to use.
I even went a step further and implemented all other output formats (pretty,
terse and, junit) on top of the structures used for json output, helping to
further refine its design and making sure its sufficient for `cargo test` to
create the desired UX.

This prototype is still fairly early; we don't even have full parity with
[libtest-mimic](https://docs.rs/libtest-mimic/0.6.0/libtest_mimic/).
For this reason, none of the crates have been published yet.

The basis for the design is "what if this could replace the original libtest?".
Even if we this does not become the basis for libtest, my hope is that core
parts of the code can be shared to ensure consistent behavior, like serde types and the CLI.

So this is why I made
[yet another argument parser](https://github.com/epage/pytest-rs/tree/main/crates/lexarg).
I enjoy [clap](https://docs.rs/clap) and generally recommend it to people (which is why I've taken on maintaining it).
When someone needs something more lightweight, I generally point them to
[lexopt](https://docs.rs/lexopt/latest/lexopt/)
due to the simpleness of the design.

But.

When designing this prototype, I wanted to design-in support
for users to extend the command-line like you can in pytest.
This means it needs to be pluggable.
If this were exposed in libtest's API, then it can't break compatibility.
The easiest way to do this is to have as simple of an API as possible.
Clap's API is too big.
I was concerned even about the amount of policy in lexopt.
I don't know if `lexarg` will go anywhere but it allowed me to get an idea of
how a perma-1.0 test library could possibly have an extensible CLI.

<a id="libs"></a>
### Libs team meeting

After presenting on this at [RustNL 2023](https://2023.rustnl.org)
([video](https://www.youtube.com/watch?v=3aLPewRSiK8), [slides](https://epage.github.io/talks/2023/05/test/#p1)),
I had the opportunity to attend an in-person libs team meeting to discuss parts
of this with them
(along with [OsStr](https://github.com/rust-lang/rust/pull/109698)).

**Question: how much of this work will make it into libtest?**

I went in with the audacious goal of "everything", initially working on
extension points to allow out-of-tree experiments on a "pytest"-like
API and then slowly pull pieces of that into libtest.

We also discussed experimenting with unstable features by publishing libtest to
crates.io where we can break compatibility until we are satisfied and then add
the `#[stable]` attribute to the version shipped with rust.
This idea
[isn't new](https://internals.rust-lang.org/t/a-path-forward-towards-re-usable-libtest-functionality-custom-test-frameworks-and-a-stable-bench-macro/9139).

The biggest concern was with the compatibility surface that would have to be
maintained and instead we went the opposite direction.
Our aim is to make custom test harnesses first-class and shift the focus
towards them rather than extending libtest.
Hopefully, we can consolidate down on just a couple of frameworks with libtest
providing us with the baseline API, reducing the chance that test writing
between projects is too disjoint.  I also hope we can share code between these
projects to improve consistency and make it easier to conform to expectations.

**Question: where do we draw the line for these custom test harnesses?**

Today, you either get libtest with test enumeration, a `main`, etc, or you
have to build it all up from scratch.
Previously
[eRFC #2318](https://github.com/rust-lang/rfcs/blob/master/text/2318-custom-test-frameworks.md)
laid out a plan for rust to still own the `#[test]` macro and test enumeration
but be accessible from custom test harnesses.
For the cases I've enumerated and my gut feeling when prototyping, my
suspicion is that I'll be wanting to allow custom `#[test]` macros so my test
harness can control what code gets generated and can have nested
attributes (like `#[test(exclusive)]`) rather than repeating our existing
pattern of separate macros (e.g. `#[ignore]`).

To get parity with libtest, we'll need stable support for
- Test enumeration (see below)
- Disabling of the existing `#[test]` macro (no plans yet)
- Custom preludes to pull in `#[test]` macro from a dependency (no plans yet)
- Pulling in `main` from a dependency (no plans yet)
- Capturing of `println` (see below)

Plus a low ceremony way to opt-in to all of this (like
[rust-lang/cargo#6945](https://github.com/rust-lang/cargo/issues/6945)
).

We didn't cover everything but we made enough progress to feel happy with this plan

**Question: how will we do test enumeration?**

I had hoped that
[inventory](https://docs.rs/inventory/latest/inventory/)
or
[linkme](https://docs.rs/linkme/latest/linkme/)
could be used for this but there was concern about supporting these across
platforms without hiccups, including from dtolnay who is the maintainer of
them.

Instead, we are looking at introducing a new language feature to replace
libtest's use of internal compiler features.
This would most likely be a
[`#[distributed_slice]`](https://internals.rust-lang.org/t/from-life-before-main-to-common-life-in-main/16006/25?u=dtolnay)
though we didn't hash out further details in the meeting.

**Question: how will we capture `println`?**

When a test fails, its great that the output is captured and reported back from
`println`, `dbg`, and panic messages (like from `assert`).

How does it work though?
I'm sorry you asked.
At the start of a test, a buffer is passed to
[set_output_capture](https://github.com/rust-lang/rust/blob/master/library/std/src/io/stdio.rs#L979)
which puts it into thread-local storage.
When you call [println, print, eprintln, or eprint](https://github.com/rust-lang/rust/blob/master/library/std/src/macros.rs#L132),
they call
[_print and _eprint](https://github.com/rust-lang/rust/blob/master/library/std/src/io/stdio.rs#L1094)
which calls [print_to](https://github.com/rust-lang/rust/blob/master/library/std/src/io/stdio.rs#L1009),
writing to the buffer.
At the end of the test, `set_output_capture` is called again to restore
printing to `stdout` / `stderr`.

This means
- If you write directly to `std::io::{stdout,stderr}`, use libc, or have C code using libc, it will not be captured
- If the test launches a thread, its output will not be captured (as far as I can tell)
- This API is only available in nightly which libtest has special access to on stable and custom test harnesses do not

Previously, this
[whole process was more more reusable but more complex](https://github.com/rust-lang/rust/pull/78714)
and there is hesitance to go back in this direction.
For this to be stabilized, this also needs to be more general
This needs to cover `std::io::stdout` / `std::io::stderr` and likely libc.
This needs to be more resilient, capturing across all threads or inherited when
new threads are spawned.
Then there is `async` where its not about what thread you are running on.
Implicit
[contexts](https://tmandry.gitlab.io/blog/posts/2021-12-21-context-capabilities/)
for `stdout` and `stderr` would cover most needs but that I'm doubt we'll get
that any time soon, if ever (no matter how much I love the *idea* of it).

We could workaround this by running each test in its own process but that comes
with its own downsides as already mentioned.

So we don't have a plan that can meet these needs yet.

See also [rust-lang/rust#90785](https://github.com/rust-lang/rust/issues/90785).

### Next Steps

Now is where you can help.

This, unfortunately, isn't my highest priority because I don't want want to
leave a trail of incomplete projects.
I've previously committed to
[MSRV](https://github.com/rust-lang/cargo/issues/9930),
[`[lints]`](https://github.com/rust-lang/cargo/issues/12115),
[`cargo script`](https://github.com/rust-lang/cargo/issues/12207),
and [keeping up with my crates](https://crates.io/users/epage?sort=recent-downloads).
Even if this was my highest priority, this is too much for one person and is
spread across rust language design, the rust compiler, the standard library, and
cargo.
This will also take time to go through the RFC process for each part so
the sooner we start on these, the better.

But I don't think that means we should give up.

For anyone who would like to help out, the parts I see that are unblocked include:
1. Preparing a
  [`#[distributed_slice]`](https://internals.rust-lang.org/t/from-life-before-main-to-common-life-in-main/16006/25?u=dtolnay)
  Pre-RFC and moving that forward so we have test enumeration
2. Finish up what can be done on the [prototype](https://github.com/epage/pytest-rs/issues) for further json output feedback
3. Design a low ceremony way to opt-in to all of this (like [rust-lang/cargo#6945](https://github.com/rust-lang/cargo/issues/6945))
4. Sketching out ideas how we might disable the existing `#[test]` macro
5. Researching where custom preludes are at and see what might be able to move forward so we can pull in the `#[test]` macro
6. Similarly, researching the possibility of pulling in `main` from a dependency

These are roughly in priority order based on a mixture of
- The time it will take before its usable
- How confident I am that it can be solved
- The pay off from solving it, whether because its more generally useful or a higher pain point to not have (e.g. `distributed_slice` in both cases)

Alternatively, if your interests align with one of my higher priorities, I
welcome the help with them so more of us can focus on this problem.

Discuss on
[reddit](https://www.reddit.com/r/rust/comments/145f9gi/iterating_on_testing_in_rust/?)
[mastadon](https://hachyderm.io/@epage/110516052624823067)
