---
title: Liquid v0.20
published_date: "2020-03-16 09:00:30 -0500"
data:
  tags:
    - programming
    - oss
---

`liquid` v0.20 resolves several planned breaking changes we've been holding off on.  This doesn't make us ready for 1.0 yet but this closes the gap significantly.

*[`liquid-rust`][liquid-rust] is a rust re-implementation of the [liquid][liquid] template engine made popular by the [jekyll][jekyll] static site generator.*

### Highlights

#### Conformance improvements

{% raw %}

We're striving to match the liquid-ruby's behavior and this release gets us closer:
- `where` filter implemented by or17191
- Improvements to `sort`, `sort_natural`, `compact`, and other filters by or17191
- Support for `{{ var.size }}`
- Improved equality of values
- Split `{% include %}` from accepting both a bare word (`foo.txt`) and quoted words (`"foo.txt"`) into the Liquid version (`"foo.txt"`) and Jekyll version (`foo,.txt`).  For the Liquid version, it now also accepts variables.

{% endraw %}

In addition, we've made it more clear what filters, tags, and blocks are a part of core liquid, Jekyll's extensions, Shopify's extensions, or our own extensions.

#### Improved API stability for `liquid`

The `liquid` crate has been stripped down to what is needed for parsing and rendering a template.
- `liquid_core` was created as a convenience for plugin authors.  `error`, `value`, `compiler`, and `interpreter` were merged in.
- `liquid_lib` was split out of `liquid` so we can evolve the non-stdlib plugins without breaking changes to `liquid`.

#### `render` can accept Rust-native types

Previously, you had to construct a `liquid::value::Object` (a newtype for a `HashMap`) to pass to `render`.  Now, you can create a `struct` that implements `ObjectView` and `ValueView` instead and pass it in:

```rust
#[derive(liquid::ObjectView, liquid::ValueView, serde::Serialize, serde::Deserialize, Debug)]
struct Data {
    foo: i32,
    bar: String,
}

let data = Data::default();
let template = todo!();
let s = template.render(&data)?;
```

In addition to the ergonomic improvements, this can help squeeze out the most performance:
* Can reuse borrowed data rather than having to switch everything to an owned type.
* Avoid allocating for the `HashMap` entries.

These improvements will be in the caller of `liquid` and don't show up in our benchmarks.

#### Other `render` ergonomic improvements

There multiple convenient ways to construct your `data`, depending on your application:
```rust
let template = todo!();

// `Object` is a newtype for `HashMap` and has a similar interface.
let object = liquid::Object::new();
let s = template.render(&object)?;

let object = liquid::object!({
    "foo" => 0,
    "bar" => "Hello World",
});
let s = template.render(&object)?;

// Requires your struct implements `serde::Serialize`
let data = todo!();
let object = liquid::to_object(&data)?;
let s = template.render(&object)?;

// Using the aforementioned derive.
let data = Data::default();
let s = template.render(&data)?;
```

#### String Optimizations

A core data type in liquid is an `Object`, a mapping of strings to `Value`s. Strings used as keys within a template engine are:
* Immutable, not needing separate `size` and `capacity` fields of a `String`. `Box<str>` is more appropriate.
* Generally short, gaining a lot from small-string optimizations
* Depending on the application, `'static`.  Something like a `Cow<'static, str>`. Even better if it can preserve `'static` getting a reference and going back to an owned value.

Combining these together gives us the new `kstring` crate.  Some quick benchmarking suggests
- Equality is faster than `String` (as a gauge of access time).
- Cloning takes 1/5 the time when using `'static` or small string optimization.

### Future work

There is still a lot of room for improvement and [contributors are most welcome][liquid-first]:

- [Liquid compatibility][liquid-compat], especially [more extensive compatibility tests][liquid#396]
- [API Stability][liquid-api], including:
  - The parser.  While the new parser (see below) is a major improvement, we are exposing a thin wrapper around the implementation details of our parser.  This makes the code messier and still cannot easily be evolved especially without breaking changes.  Ideally, we'd find a way to take the parser out of `liquid`s public API.
  - Get `kstring` to 1.0
- Performance: The biggest bottleneck atm is `clone`-ing data for for-loops, see [#337][liquid#337]
- [Javascript API wrapping a wasm build][liquid#399]

I'm probalby going to slow down my rate of contributions as I shift my focus to
other projects that this work helped unblock.

### Highlights from past releases

Back in 2018 we had some major improvements that deserve to be called out, most especially work by [Goncalerta][Goncalerta], but I never finished my post.

#### New Parser

The parser in liquid-rust has been a hurdle for improvements.  It included a
regex-based lexer and a random assortment of functions to parse the tokens.
These functions were particularly a hurdle because logic was duplicated when it
shouldn't, shared when it shouldn't, etc making it difficult to grok, extend,
or refactor.

There are two big challenges with parsing in liquid:
- There is no official grammar or compliance tests
- Language plugins exist and are heavily used.

Liquid has the concept of filters, plugins to modify a value:
{% raw %}
```liquid
{{ data | uniq | first }}
```
{% endraw %}
These are relatively simple.

In contrast, tags and blocks declare their own grammar, for both parameters and a block's content.
{% raw %}
```liquid
Standard include: {% include variable %}
```
{% endraw %}
In contrast, a Jekyll-style include would pass the variable as {%raw%}```{{ variable }}```{%endraw%}.

[Goncalerta][Goncalerta] took on the monumental task of replacing all of this
with a [pest][pest]-based parser written from the ground up.

No matter the parser library, language plugins are a chalenge.  In Pest's case,
it has a grammar file that gets turned into Rust code.  For now, we've taken
the approach of expressing all the parts of the native grammar and where
plugins are involved, we expose more of a higher-level token stream than
before. This does present a challenge that our parser has to support every
feature of every plugin.

#### Conformance Improvements

What we did:
- Values
  - We improved [value coercion][liquid#158], including coercing [`now` and `today` to a date][liquid#182].
  - We added [whole number support to values][liquid#158].
  - Added [`.first` and `.last` to arrays][liquid#218].
  - Add support for indexing into a variable with a variable (e.g. `foo[bar]`) [(1)][liquid#214] [(2)][liquid#231].
- Fixed ranges (e.g. `1..4`) to be [inclusive][liquid#212].
- Indexing into variables
  - Support [this for filters][liquid#208]
- Filters
  - [`compact`][liquid#158]
  - [`at_least` and `at_most`][liquid#210]
- Blocks
  - Extend `if` blocks with [`and` and `or`][liquid#217]
  - Extend `for`-loops to support [iterating over `Object`s for
    `for`-loop][liquid#203] and accept [variables for `offset`,
    `limit`][liquid#213]
  - Support for [`ifchanged`][liquid#211]
  - Support `tablerow`[(1)][liquid#211] [(2)][liquid#213] [(3)][liquid#215]
- Tags
  - [`increment` and `decrement`][liquid#211]

Regarding the [jekyll library][jekyll-liquid]
- Filters
  - [`push`][liquid#220]
  - [`pop`][liquid#220]
  - [`unshift`][liquid#220]
  - [`shift`][liquid#220]
  - [`array_to_sentence_string`][liquid#220]

## Error Improvements

We've been iterating on the usability of template errors.  I blogged in the
past on [error handling in
Rust](https://epage.github.io/blog/2018/03/redefining-failure/) and am finding
the approach being taken here is offering a balance of good usability with
relatively low overhead.

Specifically, some things we've done:
- [Include a liquid backtrace][liquid#164]
- Include context, like variable values, in backtrace [(1)][liquid#164] [(2)][liquid#233]
- Fix up the tone and use more clear terms [(1)][liquid#231] [(2)][liquid#233]

## Performance

To help guide performance improvements, We [ported handlebar's and tera's
benchmarks][liquid#234] to Liquid:
- Save us from thinking about representative use cases
- Of all the arbitrary baselines, this seemed like one of the better ones.

We've made quite a good number of performance improvements
- Offer `Renderable` trait to [render to a `Write`][liquid#192], giving users
  the opportunity to control the backend and allocations. This might cause us
  problems with Liquid's [string trimming][liquid#244] but we have ideas on how
  to handle this.
- [Reduce allocations within `impl Display`][liquid#200].
- Have parallel code paths for `Option` and `Return`, optimizing for the
  failure case by avoiding expensive error reporting. [(1)][liquid#233]
- Play [whac-a-mole][whac-a-mole] with `clone()`s, finding ways to use
  references instead. [(1)][liquid#233]

These numbers were gathered on my laptop running under WSL.  To help show the
variability of my computer, we included the baseline from both the old and new
liquid runs.

- Liquid 0.13.7
- Liquid 0.18.0
- Handlebars 1.1.0
- Tera 0.11.20

Parsing Benchmarks

| Library                         | Liquid's `parse_text` |  Handlebar's `parse_template`  | Tera's `parsing_basic_template` |
|---------------------------------|---------------|----------------------------------------|-----------------------------|
| Baseline Run                    |               |    20,730 ns/iter (+/- 4,811)          | 24,612 ns/iter (+/- 8,620)  |
| Liquid v0.13.7 |     2,670 ns/iter (+/- 992)    |    26,518 ns/iter (+/- 14,115)         | 24,720 ns/iter (+/- 12,812) |
| Liquid v0.18.0 |     1,612 ns/iter (+/- 1,123)  |    24,812 ns/iter (+/- 13,214)         | 19,690 ns/iter (+/- 9,187)  |

Rendering Benchmarks

| Library        | Liquid's `render_text`  |  Handlebar's `render_template`  | Tera's `rendering_basic_template` | Tera's `rendering_only_variable` |
|----------------|-------------------------|----------------------------|---------------------------|---------------------------|
| Baseline Run   |                         | 35,703 ns/iter (+/- 4,859) | 6,687 ns/iter (+/- 1,886) | 2,345 ns/iter (+/- 1,678) |
| Liquid v0.13.7 | 2,379 ns/iter (+/- 610) | 16,500 ns/iter (+/- 6,603) | 5,677 ns/iter (+/- 2,408) | 3,285 ns/iter (+/- 1,401) |
| Liquid v0.18.0 |   280 ns/iter (+/- 60)  | 11,112 ns/iter (+/- 1,527) | 2,845 ns/iter (+/- 1,216) |   525 ns/iter (+/- 186)   |

Variable Access Benchmarks

| Library        | Tera's `access_deep_object` | Tera's `access_deep_object_with_literal` |
|----------------|----------------------------|----------------------------|
| Tera Run 1     |  8,626 ns/iter (+/- 5,309) | 10,212 ns/iter (+/- 4,452) |
| Tera Run 2     |  8,201 ns/iter (+/- 4,476) |  9,490 ns/iter (+/- 6,118) |
| Liquid v0.13.7 | 13,100 ns/iter (+/- 1,489) | **Unsupported**            |
| Liquid v0.18.0 |  6,863 ns/iter (+/- 1,252) |  8,225 ns/iter (+/- 3,567) |

[liquid#158]: https://github.com/cobalt-org/liquid-rust/pull/158
[liquid#164]: https://github.com/cobalt-org/liquid-rust/pull/164
[liquid#168]: https://github.com/cobalt-org/liquid-rust/pull/168
[liquid#182]: https://github.com/cobalt-org/liquid-rust/pull/182
[liquid#189]: https://github.com/cobalt-org/liquid-rust/pull/189
[liquid#192]: https://github.com/cobalt-org/liquid-rust/pull/192
[liquid#193]: https://github.com/cobalt-org/liquid-rust/pull/193
[liquid#195]: https://github.com/cobalt-org/liquid-rust/pull/195
[liquid#197]: https://github.com/cobalt-org/liquid-rust/pull/197
[liquid#199]: https://github.com/cobalt-org/liquid-rust/pull/199
[liquid#200]: https://github.com/cobalt-org/liquid-rust/pull/200
[liquid#203]: https://github.com/cobalt-org/liquid-rust/pull/203
[liquid#208]: https://github.com/cobalt-org/liquid-rust/pull/208
[liquid#210]: https://github.com/cobalt-org/liquid-rust/pull/210
[liquid#211]: https://github.com/cobalt-org/liquid-rust/pull/211
[liquid#212]: https://github.com/cobalt-org/liquid-rust/pull/212
[liquid#213]: https://github.com/cobalt-org/liquid-rust/pull/213
[liquid#214]: https://github.com/cobalt-org/liquid-rust/pull/214
[liquid#215]: https://github.com/cobalt-org/liquid-rust/pull/215
[liquid#217]: https://github.com/cobalt-org/liquid-rust/pull/217
[liquid#218]: https://github.com/cobalt-org/liquid-rust/pull/218
[liquid#220]: https://github.com/cobalt-org/liquid-rust/pull/220
[liquid#221]: https://github.com/cobalt-org/liquid-rust/pull/221
[liquid#231]: https://github.com/cobalt-org/liquid-rust/pull/231
[liquid#233]: https://github.com/cobalt-org/liquid-rust/pull/233
[liquid#234]: https://github.com/cobalt-org/liquid-rust/pull/234
[liquid#244]: https://github.com/cobalt-org/liquid-rust/pull/244
[liquid#337]: https://github.com/cobalt-org/liquid-rust/pull/337
[liquid#396]: https://github.com/cobalt-org/liquid-rust/pull/396
[liquid#399]: https://github.com/cobalt-org/liquid-rust/pull/399
[liquid-api]: https://github.com/cobalt-org/liquid-rust/labels/api-break
[liquid-compat]: https://github.com/cobalt-org/liquid-rust/issues?q=is%3Aissue+is%3Aopen+label%3Astd-compatibility
[liquid-first]: https://github.com/cobalt-org/liquid-rust/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22
[whac-a-mole]: https://en.wikipedia.org/wiki/Whac-A-Mole
[jekyll-liquid]: https://jekyllrb.com/docs/liquid/
[cobalt]: https://cobalt-org.github.io/
[Goncalerta]: https://github.com/Goncalerta
[pest]: https://pest.rs/
[liquid]: https://shopify.github.io/liquid/
[liquid-rust]: https://docs.rs/liquid
[jekyll]: https://jekyllrb.com
