title: Redefining Failure
layout: default.liquid
is_draft: true
---

I recently got the chance to redo the error handling in two different crates I
help maintain. For [`liquid`][liquid], I decided to write the error types by
hand rather than use something like [`error-chain`][error-chain]. In the case
of [`assert_cli`][assert_cli], I decided to finally give [`failure`][failure] a
try.

From `failure`'s [announcement][announcement]:

> Failure is a Rust library intended to make it easier to manage your error
> types. This library has been heavily influenced by learnings we gained from
> previous iterations in our error management story, especially the [`Error`
> trait][error-trait] and the [`error-chain`][error-chain] crate.

I think `failure` does a great job iterating on Rust's error management story,
fixing a lot of fundamental problems in the `Error` trait and making it easier
for new developers and prototypers to be successful. While a replacement for
the `Error` trait is a disruptive change to the ecosystem, I feel the reasons
are well justified and the crate looks like it does a great job bridging the
gap.

On the other hand, I do feel that there are parts of `failure` that are a bit
immature, particularly [`Context`][context]. The reason this is concerning is
the implementation policies related to `Context` are coupled to the general
`Fail` trait mechanism (see [Separation of mechanism and policy][mechanism]).
We cannot experiment and iterate on `Context` without breaking compatibility
with `Fail` and there is a reasonable [hesitance in breaking
compatibility][failure10].

I'd recommend `failure` for anyone writing an application. I'd recommend
library authors exercise caution, particularly if you have richer requirements
for your errors. For the crate author, I'd suggest it is either not ready yet
for "1.0" or to be more open to breaking changes than ["the distant
future"][failure10]. Below I'll go into more specific areas I think `failure`
can be improved.

I know it is less than ideal to receive this type of feedback "late" in the
process ([1.0 is expected to release next week][failure10]). I was strapped for
time and only recently had solid use cases to push `failure`s limits.  I at
least tried to provide theoretical feedback early on based on reading the code
and docs out of an eagerness for `failure` to succeed.  The ability to create a
thorough analysis from just reading the docs / code is limited and the weight
of such feedback is reasonably lower.

## My Recent Failures

So let's step through my recent case studies in error management

### `liquid`

[`liquid` is a rust implementation][liquid] of Shopify's [liquid template language](http://liquidmarkup.org/).

As mentioned above, I recently did a [major revamp of `liquid`s
errors](https://github.com/cobalt-org/liquid-rust/pull/164). I ended up
skipping `failure`. It was easier for me to write my error types from scratch
than to take the time to figure out how to map my needs to `failure`. I was
also concerned about making an unstable crate such a fundamental part of my
API.

My goals were:
- Low friction for detailed error reports.
- Give the user context to the errors with template back traces.

An example of a template backtrace:
```
{% raw %}Error: liquid: Expected whole number, found `fractional number`
  with:
    end=10.5
from: {% for i in (1..max) reversed %}
from: {% if "blue skies" == var %}{% endraw %}
  with:
    var=10
```

Here is a rough sketch of `liquid`s errors:
```rust
#[derive(Fail, Debug)]
pub struct Error {
    inner: Box<InnerError>,
}

#[derive(Fail, Debug)]
struct InnerError {
    msg: borrow::Cow<'static, str>,
    user_backtrace: Vec<Trace>,
    #[cause] cause: Option<ErrorCause>,
}

#[derive(Clone, PartialEq, Eq, Debug, Default)]
pub struct Trace {
    trace: Option<String>,
    context: Vec<(borrow::Cow<'static, str>, String)>,
}
```

I've created various "ResultExt" traits to make it some-what ergonomic to create:
```rust
let value = value
    .as_scalar()
    .and_then(Scalar::to_integer)
    .ok_or_else(|| Error::with_msg(format!("Expected whole number, found `{}`", expected, value.type_name()))
    .context_with(|| (arg_name.to_string().into(), value.to_string()))?;
...
let mut range = self.range
    .evaluate(context)
    .trace_with(|| self.trace().into())?;
```

While the context API needs some work, the overall approach worked great and
provides very help error messages to the user.

### `assert_cli`

[`assert_cli`][assert_cli] provides assertions for program behavior to help
with testing.

This week, I started on a refactoring of `assert_cli` in an effort to move it
to "1.0" for the CLI working group. I needed some new errors and rather than
continuing to extend the existing brittle system based on `error-chain`, I
thought I'd give `failure` a try. I figured breaking changes in `failure` would
have minimal impact on my users because 99% of `assert_cli` users are just
unwrapping the error in their tests rather than passing it along.

I structured `assert_cli`'s `error-chain` errors to leverage chaining. The chaining hierarchy is something like:
- Spawn Failed
- Assertion Failed
  - Status (success / failure) Failed
  - Exit Code Failed
  - Output
    - Strings matched when shouldn't
    - Strings matched when should
    - Bytes matched when shouldn't
    - Bytes matched when should
    - Sub-strings matched when shouldn't
    - Sub-strings matched when should
    - Byte subset matched when shouldn't
    - Byte subset matched when should
    - Predicate failed

Most of these errors really just exist for the sake of adding context.  This is
greatly simplified by leveraging `failure::Context` and [`ErrorKind`
idiom](https://boats.gitlab.io/failure/error-errorkind.html).

This is what it looks like, in Rust pseudo-code:
```rust
#[derive(Copy, Clone, Eq, PartialEq, Debug, Fail)]
pub enum AssertionKind {
    #[fail(display = "Spawn failed.")] Spawn,
    #[fail(display = "Status mismatch.")] StatusMismatch,
    #[fail(display = "Exit code mismatch.")] ExitCodeMismatch,
    #[fail(display = "Output mismatch.")] OutputMismatch,
}

#[derive(Debug)]
pub struct AssertionError {
    inner: failure::Context<AssertionKind>,
}
```

Most of the rest of this post goes into my initial experience in converting over to this.

## Feedback on `failure::Context`

`error-chain` and friends have been around for a while and given us different
takes on how to write and chain errors but there hasn't been too much
experimentation with adding context to errors. In `failure`s case, it is using
roughly the same approach from when it was first announced.

`Context` serves two purposes in `failure`.
- Wrap a `failure::Error`, adding a single `impl Display` of context.
- Quick and dirty way to create a new error.

### Decouple Roles

Coupling these roles is initially convenient. A user can quickly create an
error of from any piece of data they have and there isn't need for another
`struct` that looks and acts similarly to `Context`.

The problem is in the details.

Say my errors are wrapped like this:
```rust
Context -> Context -> AssertionError -> io::Error
```
(remember: `Context` is a `Fail`)

- Which `Fail`s' `Termination` trait should be respected for [`?` in
`main`](https://github.com/rust-lang/rfcs/blob/master/text/1937-ques-in-main.md)
(when that becomes a thing)?
- How does a user of my library identify my documented error type for making
  programmatically handling the error?
- If an application only wants to show the leaf error and not the causes, how does it identify what is a leaf error?
- How should adding `Context` in `no_std` work?  With the current approach, the `Context` is passed back and the original error is dropped.

Suggestions:
- Separate the roles by providing an alternative "easy error".
- Separate the roles so a clearer `no_std` policy can exist.
- Remove `Context` from the error chain by moving the `Context` from an error
  decorator to an error member, like `backtrace` and `cause`.
- Experiment with ways to give `no_std` users more control over the `Context`
  policy like moving the `Context` from an error decorator to an error member,
  like `backtrace` and `cause`.

### Errors are `Display`ed in Inverted Order

Because `Context` generically wraps a` `failure::Error`, the ordering is inverted when rendering an error for a user.

If I were to naively switch `liquid` to `failure` my errors would change from
```
{% raw %}Error: liquid: Expected whole number, found `fractional number`
  with:
    end=10.5
from: {% for i in (1..max) reversed %}
from: {% if "blue skies" == var %}{% endraw %}
  with:
    var=10
```
to
```
end=10.5
cause: {% raw %}Error: liquid: Expected whole number, found `fractional number`
cause: {% for i in (1..max) reversed %}
cause: var=10
cause: {% if "blue skies" == var %}{% endraw %}
```

Suggestion:
- Associate contexts with an error by, again, moving the `Context` from an error
  decorator to an error member, like `backtrace` and `cause`.

### Better support type contracts

`failure::Error` is a fancy boxed version of `failure::Fail`. This is a great
solution for prototyping and applications like Cobalt.

Libraries can be a different beast. In cases like `aasert_cli` and `liquid`, I want to ensure:
- User has a backwards compatible, defined, finite set of errors to programmatically deal with.
- The errors are useful.

Having a type-erased `failure::Fail` makes this impossible. Any `?` could be
turning an error from a dependency into a `failure::Fail` and passing it right
back up to my user. The only way for me to know is to exercise every error path
and inspect the output.

Instead I prefer to return `Result<T, AssertionError>` instead of `Result<T, failure::Error>`.

That's fine, `failure::Error` is just a convenience that `failure` provides, like with `Box<Error>`, except:
- If I want to add `Context` to my errors, the only reasonable ergonomic way to do so is to return `Result<T, failure::Error>`.
- [`Context` only supports wrapping a `failure::Error`](https://github.com/rust-lang-nursery/failure/issues/60), making it so any error can escape through a `Context`.

The only alternative is to reimplement the `Context` machinery that is built-in to the `failure::Fail` and `failure::ResultExt` traits.

Suggestion:
- Allow using context without `failure::Error` by, yet again, moving the
  `Context` from an error decorator to an error member, like `backtrace` and
  `cause`.

### Support a `ContextPair`

When converting `assert_cli` to `failure`, I found `failure::Context` works
great when you want to dump strings but has limitations for my more common
cases:
```rust
// Good: failure::Context works well in this case
return Err(AssertionError::new(AssertionKind::OutputMismatch))
    .context("expected to contain")
// Bad: failure::Context loses the context in this case
return Err(AssertionError::new(AssertionKind::OutputMismatch))
    .context(needle)
    .context(haystack)
// Alternative: Works but is a bit verbose, especially considering this is going to be 90% of my Contexts
return Err(AssertionError::new(AssertionKind::OutputMismatch))
    .context_with(|| format!("needle: {}", needle))
    .context_with(|| format!("haystack: {}", haystack))
```

So I added this:
```rust
#[derive(Debug)]
pub struct ContextPair
{
    key: &'static str,
    context: impl Display,
}
```
which can be used like:
```rust
return Err(AssertionError::new(AssertionKind::OutputMismatch))
    .context("expected to contain")
    .context(ContextPair("needle", needle.to_owned()))
    .context(ContextPair("haystack", haystack.to_owned()));
```

Suggestion:
- Provide something *like* `ContextPair` so `failure` can help developers fall
  into the [pit of
  success](https://blog.codinghorror.com/falling-into-the-pit-of-success/) for
  giving helpful errors to users.

## Feedback on `failure::Error`

`failure::Error` is a fancy boxed version of `Fail`. Their APIs generally
deviate in ways that make sense for their different feature sets.

A quick summary of their behavior:

&nbsp;       | `Fail`           | `Error`
-------------|------------------|--------
`cause`      | child item       | inner
`causes`     | starts with self | starts with inner
`root_cause` | includes self    | includes self
`downcast`   | acts on self     | acts on inner

### `failure::Error` is not a `Fail`

While `failure::Error` acts as a boxed `Failed`, it isn't a `Fail`. This
isn't possible without specialization because `From<Fail> for Error` would conflict with
`From<T> for T`.

### `failure::Error::cause` is a foot-gun

As note above, `Error` and `Fail` have similar APIs and `failure::Error` mostly
behaves as a proxy for the inner `Fail` with [`cause` being the
exception](https://github.com/rust-lang-nursery/failure/issues/85). Despite
the `cause`s having different signatures (`Fail` returns an `Option<&Fail>`
while cause returns `&Fail`>), I think this is going to trip up a lot of
people.

Suggestion:
- `Error::cause` should behave exactly like `Fail::cause` to avoid surprising developers.

## Feedback on `failure::Fail`

### `causes` and `root_cause` are foot-guns

If `cause` is the child error, then `causes` would start with that and iterate
beneath it, right? Unfortunately, no. The current `Fail` is the first cause.

Similarly, if your `Fail` has no `cause`, then it will be the `root_cause`.

I can understand the need for functions that behave like these but not with
names that imply they won't include your current `Fail`.  Granted, I personally
would find `causes` not returning the top `Fail` more convinient.

Suggestion:
- Either find new names or change the behavior of the functions to avoid surprising developers.

# Addendum

## Additional error management case studies

### Cobalt

[Cobalt][cobalt] is a static site generator that I maintain. As an end-user application,
it is a perfect fit for `failure`. I haven't converted it over due to lack of a
pressing need compared to the desired features.

### Day Job

This is from a 15+ year old, multi-million LoC product that is 80% user-facing
library and 20% user-facing application targeted at non-programmers.

Putting a flexible product into the hands of non-programmers means you need to
provide a lot of guidance to help people get out of situations you couldn't
predict. We consider good errors essential for our users.

The user-form of our errors look like (translated to Rust pseudo-code):
```rust
#[derive(Fail, Debug, Copy, Clone, ...)]
#[repr(C)]
pub enum ErrorKind {
    ...
}

#[derive(Fail, Debug)]
pub struct PublicError {
    kind: ErrorKind,
    context: String,
}
```
and the internal-form of our errors look like (translated to Rust pseudo-code):
```rust
#[derive(Fail, Debug, Copy, Clone, ...)]
#[repr(C)]
enum ContextKind {
    ...
}

struct Context {
    value: Vec<(ContextKind, Box<Display>)>,
    backtrace: failure::Backtrace,
}

struct InternalError {
    kind: ErrorKind,
    context: Option<Box<Context>>,
}
```

Things of note:
- `InternalError` has a way of localizing and formatting `Context` into the `PublicError::context`.
- `PublicError` / `InternalError` have a way of localizing `ErrorKind`.
  - `ErrorKind` always includes "what", "why", and "how to fix it".
- OS and library errors are manually converted to `InternalError`. Sometimes
  the root cause is captured in `ContextKind`s that we hide in release builds.
- Layers higher in the stack may add to `Context`. They can even conditionally
  add to `context.value` to avoid duplication.
- Performance is controlled by only heap allocating or capturing a back trace if context is added.



[assert_cli]: https://github.com/assert-rs/assert_cli
[liquid]: https://github.com/cobalt-org/liquid-rust
[cobalt]: https://cobalt-org.github.io/
[error-chain]: https://crates.io/crates/error-chain
[failure]: https://github.com/withoutboats/failure
[announcement]: https://boats.gitlab.io/blog/post/2017-11-16-announcing-failure/
[error-trait]: https://doc.rust-lang.org/std/error/trait.Error.html
[context]: https://docs.rs/failure/0.1.1/failure/struct.Context.html
[failure10]: https://boats.gitlab.io/blog/post/2018-02-22-failure-1.0/
[mechanism]: https://en.wikipedia.org/wiki/Separation_of_mechanism_and_policy
