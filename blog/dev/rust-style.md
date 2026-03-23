---
title: "Guideline: Rust Style"
data:
    tags: ["guideline"]
---

# Checklist

Project structure
- Prefer `mod.rs` over `name.rs` ([P-MOD](#p-mod))
- Directory-root modules only re-export ([P-DIR-MOD](#p-dir-mod))
- Prelude module only re-exports ([P-PRELUDE-MOD](#p-prelude-mod))
- Avoid `#[path]` ([P-PATH-MOD](#p-path-mod))
- API is a subset of file layout ([P-API](#p-api))
- Simple visibility ([P-VISIBILITY](#p-visibility))

File structure
- Private then public imports ([M-PRIV-PUB-USE](#m-priv-pub-use))
- Limit private imports ([M-PRIV-USE](#m-priv-use))
- Import items individually ([M-SINGLE-USE](#m-single-use))
- Central item first ([M-ITEM-TOC](#m-item-toc))
- Types then associated functions ([M-TYPE-ASSOC](#m-type-assoc))
- Associated functions then trait impls ([M-ASSOC-TRAIT](#m-assoc-trait))
- Caller then callee ([M-CALLER-CALLEE](#m-caller-callee))
- Public then private items ([M-PUB-PRIV](#m-pub-priv))
- Use your judgement on ambiguity ([M-AMBIGUITY](#m-ambiguity))

Function structure
- Group related logic ([F-GROUP](#f-group))
- Open with output variables ([F-OUT](#f-out))
- Blocks reflect business logic ([F-VISUAL](#f-visual))
- Pure combinators ([F-COMBINATOR](#f-combinator))

# Principles

Writing software is not just an act of writing for the compiler
but writing for your reviewer and those who will debug and change the code in the future.
It should be viewed as a form of technical writing and apply the same principles, including:
- [Inverted pyramid](https://en.wikipedia.org/wiki/Inverted_pyramid_%28journalism%29) or Progressive Disclosure:
  lead with or highlight the most salient details, letting readers decide how much they want to dig into the low-level details
- Guide the user through the structure:
  with writing, you guide the user through organization including topical essays, table of contents, sections, paragraphs, sentences, fragments, and words.

Unlike writing, code is heavily cross-referenced (i.e. functions)
and your reader has a limited memory shared with a lot of other context.
- Strong abstractions obviate the need to digging into details
- The weaker the abstraction, the closer it needs to be to the use or not even exist
  - When optimizing for CPU caches, we consider temporal and spatial locality. Let's offer our reader the same benefit.

# Project structure

<a id="p-mod"></a>

## Prefer `mod.rs` over `name.rs` (P-MOD)

When splitting a file into a directory,
prefer `mod.rs` for the root module.

This ensures the module is an atomic unit for
reading (e.g. browsing, searching) or operating on (e.g. moving, renaming).
For example, when browsing on GitHub, it can be easy to overlook the existence of a `name/` when seeing a `name.rs`.
This is also consistent with `lib.rs` and `main.rs`.

Automation: enable [`clippy.self_named_module_files = "warn"`](https://rust-lang.github.io/rust-clippy/master/index.html?search=self_#self_named_module_files)

Example:
```
src/
  stuff/
    stuff_files.rs
  stuff.rs
  lib.rs
```
Use instead:
```
src/
  stuff/
    stuff_files.rs
    mod.rs
  lib.rs
```

<a id="p-dir-mod"></a>

## Directory-root modules only re-export (P-DIR-MOD)

When splitting a file into a directory,
the root of the directory (`mod.rs`, `lib.rs`)
should only contain re-exports.
All definitions should live in topically named files with `mod.rs` acting as a Table of Contents.

It can be confusing for a reader to figure out where to look when logic is
split between topics and the root.
It is better to consistently limit it to re-exporting.

Possible exceptions:
- An inline `mod prelude` in `lib.rs`
- The titular definition for the module (e.g. a `fn compile` inside a `mod compile`)

<a id="p-prelude-mod"></a>

## Prelude module only re-exports (P-PRELUDE-MOD)

If a prelude is desired,
they are intended as an import convenience.

Users would not expect to find original logic in them.

<a id="p-path-mod"></a>

## Avoid `#[path]` (P-PATH-MOD)

Prefer the standard module lookup rules over `#[path]`.

This makes the project structure more predictable.

Possible exceptions:
- Modding `build.rs` generated files

Example:
```rust
#[cfg(windows)]
#[path(foo_windows.rs)]
mod foo;
#[cfg(unix)]
#[path(foo_unix.rs)]
mod foo;
```
Use instead:
```rust
#[cfg(windows)]
#[path(foo_windows.rs)]
mod foo_windows;
#[cfg(windows)]
use foo_windows as foo;
#[cfg(unix)]
mod foo_unix;
#[cfg(unix)]
use foo_unix as foo;
```

<a id="p-api"></a>

## API is a subset of file layout (P-API)

Modules should only re-export items from child modules and from sibling modules.
Inline modules should be avoided.

A dependent of your package should be able to take what they know of the API to browse to the code in question,
or at least a parent directory of it.

Exceptions:
- Inline test modules ([maybe](https://github.com/rust-lang/rust-clippy/issues/13589))
- Inline preludes
- `build.rs` generated files

<a id="p-visibility"></a>

# Simple visibility (P-VISIBILITY)

Limit visibility to module-scope (no `pub`), crate-scope (`pub(crate)`), or dependent-scope (`pub`).

If an item is not fully abstracted within a module to have no `pub`,
the differences in which crate-level module can access it is small and `pub(crate)` should be used.
By using `pub(crate)`, it reduces the friction in refactors.
If there is a module from which it is fully abstracted,
consider whether that module should instead be a crate.

Exceptions:
- Test modules

# File structure

<a id="m-priv-pub-use"></a>

## Private then public imports (M-PRIV-PUB-USE)

Group public imports with the file's public API.

Private imports are a detail of the implementation that mostly matters when editing code.
A reader approaching the file is likely to skip over private imports to look at the public items to get the big picture.

Example:
```rust
use rand::RangeExt as _;
pub use regex::Regex;
use serde::Deserialize as _;
```
Use instead:
```rust
use rand::RangeExt as _;
use serde::Deserialize as _;

pub use regex::Regex;
```

<a id="m-priv-use"></a>

## Limit private imports (M-PRIV-USE)

Imports should be limited to where the meaning of the calling code remains obvious.

Uncommonly used imports can be frequently added and removed as the implementation changes,
causing merge conflicts.

Expected use of imports
- Traits (anonymous imports)
- Heavily used items where the intent is clear from the name.

Example:
```rust
use std::collections::HashMap;
use std::collections::hash_map::Entry;
use serde::Deserialize;
```
Use instead:
```rust
use std::collections::HashMap;
use serde::Deserialize as _;
```

<a id="m-single-use"></a>

## Import items individually (M-SINGLE-USE)

Compound imports increase the likelihood of merge conflicts.

Compound imports have higher friction for editing by hand,
particularly when using editors optimize for line editing like vim.

Example:
```rust
use std::collections::{HashMap, hash_map::Entry};
```
Use instead:
```rust
use std::collections::HashMap;
use std::collections::hash_map::Entry;
```

<a id="m-item-toc"></a>

## Central item first (M-ITEM-TOC)

When there is a titular or quintessential item for a module,
that should come first before any other items.

This is likely the item the reader is looking for when reading the module.

This item provides the context for understanding the rest of the module.
The item serves as a Table of Contents for the rest of the module,
providing jumping off points for what the reader might want to dig further into.

Note: this is a specialization of M-CALLER-CALLEE and M-PUB-PRIV.

<a id="m-type-assoc"></a>

## Types then associated functions (M-TYPE-ASSOC)

To understand the intent, role, and how to use a type,
you need to see the public interface for it
as that provides the abstraction over the fields or variants.

Exceptions:
- When a type is a newtype for a data-only inner type

Example:
```rust
pub struct Foo { ... }

pub struct Bar { inner: BarInner }

pub enum BarInner { ... }

impl Foo {}

impl Bar {}
```
Use instead:
```rust
pub struct Foo { ... }

impl Foo {}

pub struct Bar { inner: BarInner }

pub enum BarInner { ... }

impl Bar {}
```

<a id="m-assoc-trait"></a>

## Associated functions then trait impls (M-ASSOC-TRAIT)

Typically,
associated functions form the core API for a type and
trait implementations augment that API.

Example:
```rust
pub struct Foo { ... }

impl Display for Foo {}

impl Foo {}
```
Instead use:
```rust
pub struct Foo { ... }

impl Foo {}

impl Display for Foo {}
```

<a id="m-caller-callee"></a>

## Caller then callee (M-CALLER-CALLEE)

The caller provides context for the callees.
The caller provides the context for understanding the callees.
The caller serves as a Table of Contents,
providing jumping off points for what the reader might want to dig further into.

The weaker the abstraction of the callee,
the more immediately after the caller it should be.

Note: this is a generalization of M-ITEM-TOC.

Example:
```rust
const FOO: str = "...";

fn bar(s: &str) -> Bar { ... }

fn foo(arg: Arg) -> Foo {
    // ...
    let b = bar(FOO);
    // ...
}
```
Use instead:
```rust
fn foo(arg: Arg) -> Foo {
    // ...
    let b = bar(FOO);
    // ...
}

const FOO: &str = "...";

fn bar(s: &str) -> Bar { ... }
```

<a id="m-pub-priv"></a>

## Public then private items (M-PUB-PRIV)

Group public items before private items, whether in a module, struct, or an impl block.

These are likely the item the reader is looking for when reading the block.

Grouping public items provides the context for understanding the rest of the module.
Grouping public items serves as a Table of Contents for the rest of the module,
providing jumping off points for what the reader might want to dig further into.

Note: this is a generalization of M-ITEM-TOC.

<a id="m-ambiguity"></a>

## Use your judgement on ambiguity (M-AMBIGUITY)

The file structure guidelines above can be in tension with each other.
They are roughly ordered but use your judgement for how to apply them in any given situation.

# Function structure

<a id="f-group"></a>

## Group related logic (F-GROUP)

Use newlines, or the lack of them, to group logic that shares a purpose.
These are your "paragraphs" of your function.

Example:
```rust
fn report_warning_count(&self, ...) {
    let gctx = runner.bcx.gctx;
    runner.compilation.lint_warning_count += count.lints;
    let mut message = descriptive_pkg_name(&unit.pkg.name(), &unit.target, &unit.mode);
    message.push_str(" generated ");
    // ... builds up `message`
    gctx.shell().warn(message)
}
```
Use instead:
```rust
fn report_warning_count(&self, ...) {
    let gctx = runner.bcx.gctx;

    runner.compilation.lint_warning_count += count.lints;

    let mut message = descriptive_pkg_name(&unit.pkg.name(), &unit.target, &unit.mode);
    message.push_str(" generated ");
    // ... builds up `message`
    gctx.shell().warn(message)
}
```

<a id="f-out"></a>

## Open with output variables (F-OUT)

If a block exists to incrementally build up state,
open with the declaration of that state.

This will announce the intent of that group.

Example:
```rust
let sentinel = get_sentinel();
let items = get_items();
let mut message = String::new();
for item in items {
    // ...
}
```
Use instead:
```rust
let mut message = String::new();
let sentinel = get_sentinel();
let items = get_items();
for item in items {
    // ...
}
```

<a id="f-visual"></a>

## Blocks reflect business logic (F-VISUAL)

Emphasize the business logic using blocks, like the bodies of `if`s, `else`s, and `match`s.
Where business logic has mutually exclusive paths, prefer `if` and `else` or `match`  over early returns.
Instead, prefer early returns for non-business bookkeeping.

Also prefer combinators (e.g. `Iterator`, `Option`, and `Result` methods) for non-business transformations.

This draws the reader's attention to the details that matter most, allowing them to dig in further as needed.

Example:
```rust
if let Some(foo) = foo {
    // ...
    if case {
      return Ok(...);
    }

    Ok(...)
} else {
    Err(...)
}
```
Use instead:
```rust
let foo = foo.ok_or_else(|| ...)?;
if case {
    Ok(...)
} else {
    Ok(...)
}
```

<a id="f-combinator"></a>

## Pure combinators (F-COMBINATOR)

Closures passed to combinators (e.g. `Iterator`, `Option`, and `Result` methods) should not have side effects in the business logic.

When a reader is scanning the file, they are unlikely to parse through the details of the combinators
and so should not have any surprises.

Exceptions:
- "Invisible" side effects like caching, logging
