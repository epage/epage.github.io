---
title: "Clap 3.2: Last Call Before 4.0"
published_date: "2022-06-13 15:21:28 +0000"
layout: default.liquid
is_draft: false
---
With excitement and trepidation, I'm announcing the release of
[clap 3.2](https://github.com/clap-rs/clap/blob/master/CHANGELOG.md).

With clap 3.1, we [discussed the need for a more open, extensible
API](https://epage.github.io/blog/2022/02/clap-31-a-step-towards-40/) and clap
3.2 represents one step in that direction.  With two new builder API concepts,
we are able to deprecate the following concepts:
- `Arg::allow_invalid_utf8`
- `Arg::validator`, `Arg::validator_os`
- `Arg::forbid_empty_values`
- `Arg::possible_values`
- `Arg::max_occurrences`
- `Arg::multiple_occurrences`
- `Command::args_override_self`
- `AppSettings::NoAutoVersion`
- `AppSettings::NoHelpVersion`

### Introducing `Arg::value_parser`

The derive API has had to deal with adapting raw `OsString`s into the user's
type.  We did this via the
[`#[clap(parse(...)]` attribute](https://github.com/clap-rs/clap/tree/master/examples/derive_ref#arg-types).

In place of the `parse` attribute, the Builder API had
- Native support for `str` (`value_of`) and `str` (`value_of_os`, `value_of_os_lossy`) with various built-in validators
- Support for `FromStr` types (`value_of_t`) which produced a lower quality error message than native validation would

`Arg::value_parser` brings the derive API's typing to the builder API.  We've
ported some of the built-in validators to this new API, added new ones, and
made it convenient to lookup the best value parser for a given type.  For
example:
```rust
let cmd = clap::Command::new("raw")
    .arg(
        clap::Arg::new("color")
            .long("color")
            .value_parser(["always", "auto", "never"])
            .default_value("auto")
    )
    .arg(
        clap::Arg::new("hostname")
            .long("hostname")
            .value_parser(clap::builder::NonEmptyStringValueParser::new())
            .takes_value(true)
            .required(true)
    )
    .arg(
        clap::Arg::new("port")
            .long("port")
            .value_parser(clap::value_parser!(u16).range(3000..))
            .takes_value(true)
            .required(true)
    );

let m = cmd.try_get_matches_from(
    ["cmd", "--hostname", "rust-lang.org", "--port", "3001"]
).unwrap();

let color: &String = m.get_one("color").expect("defaulted");
assert_eq!(color, "auto");

let hostname: &String = m.get_one("hostname").expect("required");
assert_eq!(hostname, "rust-lang.org");

let port: u16 = *m.get_one("port").expect("required");
assert_eq!(port, 3001);
```

While updating the `ArgMatches` API for this, we took the time to make other improvements:
- Not-present values are represented with an `Option` while before some calls
  used an `Option` while some used a `Result`
  ([#2505](https://github.com/clap-rs/clap/issues/2505))
- `ArgMatches` functions panic when the argument value access doesn't match the
  argument definition but now we offer non-panicking variants
  ([#3621](https://github.com/clap-rs/clap/issues/3621))

### Introducing `ArgAction`

Clap would infer how to process an argument when parsing.  There are cases
where it doesn't quite do what users want and we need to offer more explicit
control.  This is now done by specifying an `ArgAction`.  For now, this is a closed
API but we hope to allow users to provide their own actions in the future as we
firm up how it interacts with the parser.

As part of how clap processed arguments, it special-cased flags to not store any value but instead
users would just check if the flag was present or how many times the flag
appeared.  This makes it so flags don't interact well with other parts of the
API (e.g.
[`default_value_if`](https://docs.rs/clap/latest/clap/struct.Arg.html#method.default_value_if)).
In place of the old `ArgAction::IncOccurrences`, we are providing
`ArgAction::SetTrue` and `ArgAction::Count` which will store their results
as-if the user specified them on the command-line, allowing them to interact
with the rest of clap's API without special casing.

For example:
```rust
let cmd = Command::new("mycmd")
    .arg(
        Arg::new("quiet")
            .long("quiet")
            .action(clap::builder::ArgAction::SetTrue)
    )
    .arg(
        Arg::new("verbose")
            .long("verbose")
            .action(clap::builder::ArgAction::Count)
    );

let matches = cmd.try_get_matches_from(
    ["mycmd", "--quiet", "--quiet", "--verbose", "--verbose", "--verbose"]
).unwrap();
 assert_eq!(
     *matches.get_one::<bool>("quiet").expect("defaulted by clap"),
     Some(true)
 );
 assert_eq!(
     *matches.get_one::<u8>("verbose").expect("defaulted by clap"),
     Some(3)
 );
```

In working on actions, we realized we could simplify things if we made
`Command::args_override_self` the default and used actions to decide whether
each new occurrence overrode prior occurrences (`ArgAction::Set`) or appended
to them (`ArgAction::Append`).

### Parsers and Actions for the Derive API

While all of the above is a major departure from the builder API, this is less
impactful for the derive API because we know more about the user's intent. A
user can opt-in to the new behavior by providing a default action
(`#[clap(action)]`) or default value parser (`#[clap(value_parser)]`).  Default
actions will be inferred from the data type while the parser will defer to
`clap::value_parser!(T)` for the default.  This means users no longer need to
remember to special case the parse attribute for `PathBuf` and the value will
only be parsed once and stored rather than thrown away during validation.

This behavior will become the default in clap 4.0.

### Multicall and REPLs

On a completely unrelated note, we are stablizing `Command::multicall`.  This
new setting will cause the binary name (`argv[0]`) to be parsed as a
subcommand.  This allows for [busybox-like
binaries](https://github.com/clap-rs/clap/blob/master/examples/multicall-busybox.rs)
where multiple programs are shipped in one binary with the personality being
selected based on a hardlink name.  We also found this works quite well for
[embedded command
promots](https://github.com/clap-rs/clap/blob/master/examples/repl.rs).

### Resolving Deprecations

To help in resolving the deprecations, check out these example updates:
- [cargo](https://github.com/rust-lang/cargo/pull/10753) (Builder API)
- [cargo deny](https://github.com/EmbarkStudios/cargo-deny/pull/431) (Derive API)

However, we recommend people experiment with it and provide
[feedback](https://github.com/clap-rs/clap/issues) but only commit if you are
happy with the results.  We'll process the feedback and see how much we need to
pivot with the API design introduce in clap 3.2.  When clap 4 is released, if
we didn't depart much, we'll recommend upgrading to clap 3.2 to ease the
migration to 4.0.  Otherwise, we'll work to help people upgrade to whatever
clap 4.0 look like.

I mentioned at the beginning some trepidation with this
release.  It is major departure from how clap has previously worked.  We might
have made mistakes in the design.  People might not like it.  We want clap to
be moving forward to be ready for the needs of new and existing users but we
don't want to alienate the users we already have.

Normally, to deal with uncertainty, we start with feature-gating new work but
these changes were invasive and it is difficult to gauge how successful a
change is that isn't in a stable release.  However, we are coming up on when we
can start on the clap 4.0 release and we can address any design sins then.
This gives us the opportunity to collect more feedback with slightly lower
risk.

### Results

So did all of these changes help us meet our goal of improving compile times
and reducing binary size?  No,
[it made things worse](https://github.com/rust-cli/argparse-benchmarks-rs) but
that was expected as we just added more to the API without taking anything out.
We are hoping that in clap 4.0 things will improve as we
[remove deprecated features](https://github.com/clap-rs/clap/issues/3021).
Even then, we do not expect major results as I suspect a lot of the binary size
comes from a design philosophy and it will take time slowly chipping away at it
until we have a more open API design.

### Next Steps

Future areas of focus for improving out binary size and compile times include
[customizing error and help reporting and pluggable defaults and validation](https://github.com/clap-rs/clap/issues/3021).

Other focuses for clap 4.0 include:
- [Removing lifetimes from the API](https://github.com/clap-rs/clap/issues/1041)
- [Simplify how the number of values are specified](https://github.com/clap-rs/clap/issues/2688) and [be less magical about it](https://github.com/clap-rs/clap/issues/2687)
- And [several more things](https://github.com/clap-rs/clap/milestone/78)

We expect to let clap 3.2 settle for a month or so and then we'll get to work on clap 4.0.
