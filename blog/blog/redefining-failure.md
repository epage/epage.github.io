title: Redefining Failure
layout: default.liquid
is_draft: true
---

[`failure`](https://github.com/withoutboats/failure) is a young Rust crate to
make error management easy that is on a path to [1.0 in about a
week](https://boats.gitlab.io/blog/post/2018-02-22-failure-1.0/).

`failure` pulls together a lot of great concepts that makes error management
easy.  It is a use boon to new users and rapid prototyping:
- Trivial to pass any kind of error through entire program, replacing `panic`, `Box<Error>`, etc.
- More transparent error API compared to [`error-chain`](https://github.com/rust-lang-nursery/error-chain).
- Fewer custom errors because of `Context`.

With that said, I feel `failure` has some weak points.  If `failure` was just a
handy helper like `error-chain` this wouldn't be so bad.  You can instead just
select another crate to help you make errors and move on. `failure` isn't just
a series of helpers but comes across as the next-generation of the [`Error`
trait](https://doc.rust-lang.org/std/error/trait.Error.html).  While I see
value in making such a eosystem disruptive change, I feel caution should be
exercised when saying this is "ready" (1.0) so we avoid having to go through
this again.

I'll go as far as saying that I feel there are some fundamental limitations in
`failure` that should be resolved before crate authors consider migrating to
it.

I know it is less than ideal to receive feedback like this "late" in the
process.  I didn't get have the time to play with it early on and was limited
in providing feedback from a more theoretical angle. The ability to create a
thorough analysis from just readin the docs / code is limited and the weight of
such feedback is reasonably lower.

## My History of Failure

For the last year, I've been involved in:
- [Cobalt](https://cobalt-org.github.io/) (static site generator)
- [rust implementation](https://github.com/cobalt-org/liquid-rust) of Shopify's [liquid template language](http://liquidmarkup.org/).
- [`assert_cli`](https://github.com/assert-rs/assert_cli)

In my proprietary day job, I spent the last 10 years maintaining a multi-million
LoC product that is 80% user-facing library and 20% user-facing application.

### Day Job

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
    value: Vec<(ContextKind, impl Display)>,
    backtrace: failure::Backtrace,
}

struct InternalError {
    kind: ErrorKind,
    context: Box<Context>,
}
```

Things of note:
- `InternalError` has a way of localizing and formatting `Context` into the `PublicError::context`.
- `PublicError` / `InternalError` have a way of localizing `ErrorKind`.
  - `ErrorKind` always includes "what", "why", and "how to fix it".
- OS and library errors are manually converted to `InternalError`.  Sometimes
  the root cause is captured in `ContextKind`s that we hide in release builds.
- Layers higher in the stack may add to `Context`. They can even conditionally
  add to `context.value` to avoid duplication.
- Performance is controlled by only heap allocating or capturing a back trace if context is added.

### Cobalt

This is an end-user application that is a perfect fit for `failure`.

### `liquid`

Recently I did a [major revamp of `liquid`s errors](https://github.com/cobalt-org/liquid-rust/pull/164).  My goals were:
- Low friction for detailed error reports.
- Provide context to users with liquid backtraces.

Backtrace example:
```
{% raw %}Error: liquid: Expected whole number, found `fractional number`
  with:
    end=10.5
from: {% for i in (1..max) reversed %}
from: {% if blue skies == var %}{% endraw %}
  with:
    var=10
```

When planning this, I considered giving `failure` but with the cost of API
breakages and the time to figure out how to map my goals onto `failure` I
figured it was quicker just to roll it by hand.

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

### `assert_cli`

This week, I started on a refactoring of `assert_cli` in an effort to move it
to "1.0" for the CLI working group.  I needed some new errors and rather than
continuing to extend the existing brittle system, I thought I'd give `failure`
a try.  I figured this was fine because 99% of `assert_cli` users are just
unwrapping the error in their tests rather than passing it along, making
breaking error changes of little impact.

I structued `assert_cli` `error-chain` errors to leverage chaining.  The chaining hierarchy is something like:
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

Most of these errors really just exist for the sake of adding context and simplifies greatly with `failure`.

Leveraging the [`ErrorKind`
idiom](https://boats.gitlab.io/failure/error-errorkind.html) with
`failure::Context`, I was able to simplify this down a lot.

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

## Feedback on `failure::Context`

`error-chain` and friends have been around for a while and given us different
takes on how to write and chain errors but there hasn't been too much
experimentation with adding context to errors. In `failure`s case, it is using
roughly the same approach from when it was first announced.

`Context` serves two purposes in `failure`.
- Wrap a `failure::Error`, adding a single `impl Display` of context.
- Quick and dirty way to create a new error.

### Decouple Roles

Coupling these roles is initially convenient.  A user can quickly create an
error of from any piece of data they have and a `struct` that looks and acts
similarly to `Context` doesn't have to be created.

The problem is in the details.

Say my errors are wrapped like this: `Context -> Context -> AssertionError -> io::Error`

Those `Context`s might be just context they might be a quick and dirty error.

Which `Fail`s `Termination` trait should be respected for [`?` in
`main`](https://github.com/rust-lang/rfcs/blob/master/text/1937-ques-in-main.md)
(when that becomes a thing)?

How can the user find the error for my library in case they need to
make a decision off of `AssertionKind`?  All of them are reported as a
`cause()` in `Fail`.  `io::Error` will be reported as the `root_cause`. The
user would have to iterate, looking for an `AssertionError` to do anything with
it.

A specific example of this is when rendering the error for the user.
Some developers might want to show the leaf error with context while others
might want to show the entire error chain.  How do you know what is a leaf error?

### Inverted Order

Because `Context` generically wraps a` `failure::Error`, the ordering is inverted.

If I were to naively switch `liquid` to `failure` my errors would change from
```
{% raw %}Error: liquid: Expected whole number, found `fractional number`
  with:
    end=10.5
from: {% for i in (1..max) reversed %}
from: {% if blue skies == var %}{% endraw %}
  with:
    var=10
```
to
```
end=10.5
cause: {% raw %}Error: liquid: Expected whole number, found `fractional number`
cause: {% for i in (1..max) reversed %}
cause: var=10
cause: {% if blue skies == var %}{% endraw %}
```

### Better support type contracts

`failure::Error` is a fancy boxed version of `failure::Fail`. This is a great
solution for prototyping and applications like Cobalt.

Libraries are a different beast.  In cases like `aasert_cli` and `liquid`, I want to ensure:
- User has a defined finite set of errors to programmatically deal with.
- The errors are useful.

Having a type-erased `failure::Fail` makes this impossible.  Any `?` could be
turning an error from a dependency into a `failure::Fail` and passing it right
back up to my user. The only way for me to know is to exercise every error path
and inspect the output.

Instead I prefer to return `Result<T, AssertionError>` instead of `Result<T, failure::Error>`.

That's fine, `failure::Error` is just a convenience that `failure` provides, like how `Box<Error>`.  Except.
- Except if I want to add `Context` to my errors, the only reasonable ergonomic way to do so is to return `Result<T, failure::Error>`.
- Except [`Context` only supports wrapping a `failure::Error`](https://github.com/rust-lang-nursery/failure/issues/60), making it so any error can escape through a `Context`.

The only alternative is to reimplement the `Context` machinery that is built-in to the `failure::Fail` and `failure::ResultExt` traits.

(remember, traits are most useful when `use failure::ResultExt;` removing any namespacing, so this effective *is* the `ResultExt` trait.)

### Support a `KeyedContext`

When converting `assert_cli` to `failure`, I found  `failure::Context` works
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
pub struct KeyedContext
{
    key: &'static str,
    context: impl Display,
}
```
which can be used like:
```rust
return Err(AssertionError::new(AssertionKind::OutputMismatch))
    .context("expected to contain")
    .keyed_context("needle", needle.to_owned())
    .keyed_context("haystack", haystack.to_owned());
```

I think providing something like `KeyedContext` is important for `failure` to
help developers fall into the [pit of
success](https://blog.codinghorror.com/falling-into-the-pit-of-success/) for
giving helper errors to users.

## Feedback on `failure::Error`

`failure::Error` is a fancy boxed version of `Fail`.  Their APIs generally
deviate in ways that make sense for their different feature sets.

A quick summary of their behavior:

&nbsp;       | `Fail`           | `Error`
-------------|------------------|--------
`cause`      | child item       | inner
`causes`     | starts with self | starts with inner
`root_cause` | includes self    | includes self
`downcast`   | acts on self     | acts on inner

### `failure::Error` is not a `Fail`

While `failure::Error` acts as a boxed `Failed`, it isn't a `Fail`.  This
requires specialization because `From<Fail> for Error` would conflict with
`From<T> for T`.

### `failure::Error::cause` is a foot-gun

As note above, `Error` and `Fail` have similar APIs and `failure::Error` mostly
behaves as a proxy for the inner `Fail` with [`cause` being the
exception](https://github.com/rust-lang-nursery/failure/issues/85).  Despite
the `cause`s having different signatures (`Fail` returns an `Option<&Fail>`
while cause returns `&Fail`>), I think this is going to trip up a lot of
people.

Instead, I feel that `Error::cause` should behave exactly like
`Fail::cause`.

## Feedback on `failure::Fail`

### `causes` and `root_cause` are foot-guns

If `cause` is the child error, then `causes` would start with that and iterate
beneath it, right?  Unfortunately, no.  The current `Fail` is the first cause.

Similarly, if your `Fail` has no `cause`, then it will be the `root_cause`.

I can understand the need for functions that behave like these but not with
names that imply they won't include your current `Fail`.
