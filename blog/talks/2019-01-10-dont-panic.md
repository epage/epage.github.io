---
layout: remark.liquid
format: Raw
permalink: /{{parent}}/{{year}}/{{month}}/{{slug}}/
data:
  venue: Utah Rust
---
background-image: url({{ site.base_url }}/talks/dont-panic.jpg)
class: center, middle

<!-- Image source: https://medium.com/sunnya97/hitchhikers-guide-to-the-galaxy-ed4ba3ba16cd -->

---
name: topics

# Topics

[Basics]({{ site.base_url }}/{{page.permalink}}#exceptions)

[Benefits]({{ site.base_url }}/{{page.permalink}}#benefits)

[Quirks]({{ site.base_url }}/{{page.permalink}}#quirks)

[API Design]({{ site.base_url }}/{{page.permalink}}#api)

---
name: exceptions
class: middle

# Exceptions

---

# Exceptions

```cpp
class Error: public std::exception {};

class Data {};

Data do_something();

// ...

try {
    Data data = do_something();
}
catch(Error& e) {
    // ... Handle
}
catch(...) {
    // ... Handle
}
// ... Use data
```

???

We're generally familiar with the concept.
- Side-band flow of execution generally used for reporting errors
  - Why only generally?  See Python's use of `StopIteration`
- Handled by pattern matching on types.

Why useful:
- For prototyping, you don't really need to worry about error handling
- Separates error handling from business logic
- Only way to return errors from constructors

Some languages have checked exceptions but seems like they are avoided
- Requires catch and conversion boiler plate

---

# Exceptions

Without being given any additional information, how many execution paths could
there be in the following code?
```cpp
String EvaluateSalaryAndReturnName( Employee e )
{
  if( e.Title() == "CEO" || e.Salary() > 100000 )
  {
    cout << e.First() << " " << e.Last()
         << " is overpaid" << endl;
  }
  return e.First() + " " + e.Last();
}
```
&mdash; [Herb Sutter, GotW #20](http://gotw.ca/gotw/020.htm)

???

Answer: 23
- `||` is short-circuiting, or not, depending on if it was overloaded
- Implicit conversions can throw
- Operators could be overloaded and throw

---

# Exceptions

### Exception-safe code is difficult to write.

---

# Exceptions

### Exception-safe code is difficult to write.

### Can't cross binary boundaries.

???

I could be out of date on this but I remember there being challenges with
- Incompatible STL's
- Type identifiers being different making catches fail.

So to handle this, you have to somehow catch all exceptions, turn them into
another type, and then the caller needs to turn that back into an exception.

---

# Exceptions

### Exception-safe code is difficult to write.

### Can't cross binary boundaries.

### Doesn't work in some environments (Embedded, Kernel).

---

# Exceptions

### Exception-safe code is difficult to write.

### Can't cross binary boundaries.

### Doesn't work in some environments (Embedded, Kernel).

### Definition of "exceptional cases" is use-case specific.

???

At least in C++, "exceptions should only be used for exception cases" is a
common phrase. Except when it comes to constructors because
double-initialization (constructor + init) is frowned upon.

In addition, the non-exception story is just taking off.  Before recent
standards, there wasn't a standard non-exception way of reporting errors.

---
name: rust
class: middle

# Errors in Rust

---

# Errors in Rust

### `panic!`

### `Option<T>`

### `Result<T, E>`

---
name: panic

# `panic!`

```rust
fn find(needle: i32, haystack: &[i32]) -> usize {
    for (idx, value) in haystack.enumerate() {
        if value == needle {
            return idx;
        }
    }

*   panic!("Could not find {}", needle;
}
```

???

Occasionally people say "hey, panic does stack unwinding, its the exceptions
part of Rust"
- Unwinding is a build option
- Think of this more like `abort` to be used for asserts, debug-only or not.
  - Generally, not meant to be "caught"
- Also useful for prototyping

---
name: option

# `Option<T>`

```python



def find(needle: int, haystack: list) -> Union[int, None]
    for idx, value in haystack.enumerate():
        if value == needle:
*           return idx
    else:
*       return None
```

???

Some of you may be used to `Null`, `nil`, or `None` acting as a sentinel.

Let's build up the Rust equivalent.

---

# `Option<T>`

```rust



fn find(needle: i32, haystack: &[i32]) -> bool {
    for (idx, value) in haystack.enumerate() {
        if value == needle {
*           return true;
        }
    }
*   false
}
```

???

Problems
- What does `true` and `false` mean (more obvious in this case)?
- No longer reporting the `idx`.

---

# `Option<T>`

```rust
enum Search { Exists, Missing}


fn find(needle: i32, haystack: &[i32]) -> Search {
    for (idx, value) in haystack.enumerate() {
        if value == needle {
*           return Search::Exists;
        }
    }
*   Search::Missing
}
```

???

Intent is clearer but still no `idx`.

---

# `Option<T>`

```rust
*enum Search { Exists(usize), Missing}


fn find(needle: i32, haystack: &[i32]) -> Search {
    for (idx, value) in haystack.enumerate() {
        if value == needle {
*           return Search::Exists(idx);
        }
    }

    Search::Missing
}
```

???

"Tagged unions" mean they track which state is active.  Enums are a state. So
in Rust, the language constructs are combined.

---

# `Option<T>`

```rust
*// enum Option<T> { Some(T), None}
*// use Option::*;

fn find(needle: i32, haystack: &[i32]) -> Option<usize> {
    for (idx, value) in haystack.enumerate() {
        if value == needle {
            return Some(idx);
        }
    }

    None
}
```

???

We can use the std enum instead which has a lot of useful functionality we'll
get to.

---
name: result

# `Result<T, E>`

```python



def find(needle: int, haystack: list) -> int:
*   if not haystack:
*       raise ValueError("No data to search")
    for idx, value in haystack.enumerate():
        if value == needle:
            return idx
    else:
        raise ValueError("Needle doesn't exist", needle)
```

???

What if we want to

---

# `Result<T, E>`

```rust
// enum Option<T> { Some(T), None}
// use Option::*;

fn find(needle: i32, haystack: &[i32]) -> Option<usize> {
*   if haystack.is_empty() {
*       return None;
*   }
    for (idx, value) in haystack.enumerate() {
        if value == needle {
            return Some(idx);
        }
    }

    None
}
```

???

Let's build on the `Option` case. What if we made `None` carry data as well.

---

# `Result<T, E>`

```rust
*// enum Result<T, E> { Ok(T), Err<E>}
// use Result::*;

fn find(needle: i32, haystack: &[i32]) -> Result<usize, String> {
    if haystack.is_empty() {
*       Err("No data to search".to_owned())
    }
    for (idx, value) in haystack.enumerate() {
        if value == needle {
            return Ok(idx)
        }
    }

*   Err(format!("Needle doesn't exist: {}", needle))
}
```

???

Let's build on the `Option` case. What if we made `None` carry data as well.

---
name: benefits
class: middle

# Benefits

---

# Benefits: Compile-time validation

```rust
if Ok(value) = find(10, &[1, 2, 3]) {
    ...
}

match find(10, &[1, 2, 3]) {
    Ok(value) => ...,
    Err(err) => ...,
}
```

???

Compile error to access enum states outside of a conditional
- No NullPointerException (Java) or seg fault (C/C++)

---

# Benefits: Checked "Exceptions"

Without the boiler plate
```rust
fn do_something(value: i32) -> Result<(usize, usize), String> {
    let idx_1 = find(value, data_1)?;

    let idx_2 = find(value, data_1)?;

    (idx_1, idx_2)
}
```

???

- Errors are explicitly listed
- `?` auto-converts errors where supported (`.map_err` otherwise)
- `Box<Error>` for when you don't care.

---

# Benefits: Helpers

```rust
find(10, &[1, 2, 3])
*   .map(|i| i*2)
*   .unwrap_or(30)

find(10, &[1, 2, 3])
*   .and_then(|i| find(i, &[2, 3, 4]))

find(10, &[1, 2, 3])
*   .or_else(|| find(20, &[1, 2, 3]))
```

???

The functional people call these "combinators

I recommend using these to help collect a thought but not to write your entire program with them.

---

# Benefits: Error interoperability

|            | Option       |                | Result      | |
|------------|--------------|----------------|-------------|-|
| **panic!** | `.unwrap()`\*|                | `.unwrap()`\* | `.unwrap_err()`\* |
| **bool**   | `.is_some()` | `.is_none()`   | `.is_ok()`  | `.is_err()` |
| **Option** | `.map(...)`  |                | `.ok()`     | `.err()` |
| **Result** | `.ok_or(...)`|                | `.map(...)` | `.map_err(...)` |

\* See also `expect` and `expect_err`

???

Prefer `unwrap` for prototyping and when the invariant is obvious (close
proximity, obvious).

Prefer `expect` when the invariant is not obvious.

`boolinator` crate helps convert `bool` to `Option`.

---
name: quirks
class: middle

# Quirks

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());
let value_ref: &Option<String> = &value;

value_ref.map(|s| s.trim());
```

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());
let value_ref: &Option<String> = &value;

*value_ref.map(|s| s.trim());
```

```
error[E0507]: cannot move out of borrowed content
 --> src/main.rs:3:5
  |
3 |     value_ref.map(|s| s.trim());
  |     ^^^^^^^^^ cannot move out of borrowed content
```

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());
let value_ref: &Option<String> = &value;

*let internal_ref: Option<&String> = value_ref.as_ref();
internal_ref.map(|s| s.trim());
```

???

`as_ref` isn't sufficient for giving you a `&str` as needed by most functions.

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());

let value_ref: &Option<String> = &value;
let new_value: Option<String>  = value_ref.clone();

let internal_ref: Option<&String> = value.as_ref();
let new_value: Option<String> = internal_ref.clone();
```

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());

let value_ref: &Option<String> = &value;
let new_value: Option<String>  = value_ref.clone();

let internal_ref: Option<&String> = value.as_ref();
*let new_value: Option<String> = internal_ref.clone();
```

```
error[E0308]: mismatched types
 --> src/main.rs:7:33
  |7 | let new_value: Option<String> = internal_ref.clone();
  |                                 ^^^^^^^^^^^^^^^^^^^^ expected struct `std::string::String`, found reference
  |
  = note: expected type `std::option::Option<std::string::String>`
             found type `std::option::Option<&std::string::String>`
```

---

# Quirks: Ownership

Deviation: `Deref`
```rust
let value = 5;
let value_ref = &value;

// All following are equivalent
value.clone();
(*value_ref).clone();
*value_ref.clone(); // Auto-deref
```

???

We get used to auto-deref making our `clone` work but when the reference is
interior, there is nothing to auto-deref.

---

# Quirks: Ownership

```rust
let value: Option<String> = Some("foo".to_owned());

let value_ref: &Option<String> = &value;
let new_value: Option<String>  = value_ref.clone();

let internal_ref: Option<&String> = value.as_ref();
*let new_value: Option<String> = internal_ref.cloned();
```

???

`cloned` is a convention for one-level deep `clone` on iterators, Option, and
Result.

---

# Quirks: Recoverable Errors

```rust
let value_os: OsString = std::env::var_os("PATH");
let value: String = value_os.into_string()?;
```

???

By convention, `into` functions take ownership.

What if this fails (not UTF-8) but we want to handle the original value?

---

# Quirks: Recoverable Errors

```rust
let value_os: OsString = std::env::var_os("PATH");
let value: String = value_os.into_string()?;
```

```rust
*pub fn into_string(self) -> Result<String, OsString>
```

Converts the OsString into a String if it contains valid Unicode data.

On failure, ownership of the original OsString is returned.

&mdash; [`std::ffi::OsString`](https://doc.rust-lang.org/std/ffi/struct.OsString.html#method.into_string)

???

nom and futures have a similar problem.  They report the success data in both
`Ok` and `Err`.

---

# Quirks: Iterating over Results

```rust
let rel_paths: Vec<Result<_, _>> = paths
    .map(|p| p.strip_prefix(root))
    .collect();
```

Link: [`strip_prefix`](https://doc.rust-lang.org/std/path/struct.PathBuf.html#method.strip_prefix)

---

# Quirks: Iterating over Results

```rust
let rel_paths = paths
    .map(|p| p.strip_prefix(root));
for path in rel_paths {
*   let path = path?;
    // ...
}
```

---

# Quirks: Iterating over Results

```rust
let rel_paths: Vec<_> = paths
    .map(|p| p.strip_prefix(root))
*   .filter_map(Result::ok)
    .collect();
```

---

# Quirks: Iterating over Results

```rust
*let rel_paths: Result<Vec<_>, _> = paths
    .map(|p| p.strip_prefix(root))
    .collect();
```

---

# Quirks: Iterating over Results

```rust
let (rel_paths, errors): (Vec<_>, Vec<_>) = paths
    .map(|p| p.strip_prefix(root))
*   .partition(Result::is_ok);
let rel_paths: Vec<_> = rel_paths.into_iter().map(Result::unwrap).collect();
let errors: Vec<_> = errors.into_iter().map(Result::unwrap_err).collect();
```

???

This also serves as an example of where I prefer `unwrap` over `expect`.

---

# Quirks: Nesting

Particularly `Result<Option<_>, _>` and `Option<Result<_, _>>`

```rust
let data = matches
    .value_of("context")
    .map(|s| {
        let p = path::PathBuf::from(s);
        build_context(p.as_path())
    })
*   .map_or(Ok(None), |r| r.map(Some))?
    .unwrap_or_else(liquid::value::Object::new);
```

See [rust#47338](https://github.com/rust-lang/rust/issues/47338)

???

Making this easier is held back by figuring out a name.

---

# Quirks: "catching"

```rust
let rel_path = match path.strip_prefix(root) {
    Ok(rel_path) => rel_path,
    Err(_) => Path::new(""),
};
```

???

Can lead to the arrow anti-pattern.

---

# Quirks: "catching"

```rust
let rel_path = path
    .strip_prefix(root)
    .unwrap_or_else(|| Path::new(""));
```

???

Combinators get you most of the way

---

# Quirks: "catching"

```rust
let rel_path = || {
    let rel_path = path.strip_prefix(root)?;
    Ok(rel_path)
}().unwrap();
```

???

Ok, so this example is terrible but shows how you can use `?` to jump to
another part of the function rather than exit it.

---

# Quirks: "catching"

```rust
let rel_path = catch {
    let rel_path = path.strip_prefix(root)?;
    rel_path
}.unwrap();
```

See [RFC 243](https://github.com/rust-lang/rfcs/blob/master/text/0243-trait-based-exception-handling.md) and [RFC 2388](https://github.com/rust-lang/rfcs/blob/master/text/2388-try-expr.md)

???

Again, terrible example to show `?` jumping to another part of a function rather than exiting.

Unlike functions, `Ok` is not needed.

---
name: api
class: middle

# API Design

---

# API Design

Application (internal API)
- `Box<Error>`
- `failure::Error`

Library (public API)
- See ["Patterns and Guidance"](https://rust-lang-nursery.github.io/failure/guidance.html)

---

# API Design: Considerations

Exposing implementation details

???

For implementation details, there are two considerations
- if you implement `From` for a internal dependency for use with `?`, you are
  making that internal dependency public
- If you expose a public enum of the different error types you consume, your
  API is tied to your implementation

---

# API Design: Considerations

Exposing implementation details

Programmatically checking for certain errors

---

# API Design: Considerations

Exposing implementation details

Programmatically checking for certain errors

Helping the user

---

# API Design: Example

```rust
let mut f = File::open(file)
    .replace("Cannot open file")
    .context_key("path")
    .value_with(|| file.to_string_lossy().into_owned().into())?;

// ...

self.filter
    .filter(entry, &*arguments)
    .trace("Filter error")
    .context_key("filter")
    .value_with(|| format!("{}", self).into())
    .context_key("input")
    .value_with(|| format!("{}", entry.source()).into())
    .context_key("args")
    .value_with(|| itertools::join(arguments.iter().map(Value::source), ", ").into())

```

See [liquid-error](https://docs.rs/liquid-error/0.18.0/liquid_error/)
