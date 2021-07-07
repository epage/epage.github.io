---
title: Experiments with `pushgen`
published_date: "2021-07-07 09:00:30 -0500"
data:
  tags:
    - programming
    - rust
---

Recently, there was an [announcement for
`pushgen`](https://www.reddit.com/r/rust/comments/oa6tcp/first_crate_pushgen_pushbased_approach_to/),
a port of C++ `transrangers` to Rust with a [follow up
post](https://github.com/joaquintides/transrangers/blob/master/rust.md) from
the author of `transrangers`.

Seeing the performance numbers, I was curious what the experience was like with
the different techniques compared in the followup and how the performance
worked out in a real world application.

I decided to port [`typos`](https://github.com/crate-ci/typos), a source code
spell checker, to
[`try_for_each`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_for_each)
and [`pushgen`](https://github.com/AndWass/pushgen).

For more on what `transrangers` and `pushgen` bring, see:
- [`transrangers`'s Intro](https://github.com/joaquintides/transrangers#intro)
- [Rust's iterators are inefficient, and here's what we can do about it.](https://medium.com/@veedrac/rust-is-slow-and-i-am-the-cure-32facc0fdcb)

## Introduction to `typos`

`typos` scans a git repo, spell checking file names and content.  It will
automatically ignore binary files.

Spell checking involves:
- Extracting text that looks like
  [identifiers](https://www.unicode.org/reports/tr31/) from the file.  To lower
  false positives, we try to detect and ignore emails, URLs, hashes, etc.
- Checking identifiers against a dictionary (atm has no content)
- If the dictionary could not confirm nor deny the spelling, we split the
  identifier into individual words (e.g. `PascalCase` -> `Pascal` and `Case`).
- Check each word against against a dictionary

## Architecture of `typos`

In pseudo-code, `typos` looks like:
```python
def walk_path(repo):
    for path in repo:
        check_file(path)


def check_file(path):
    for typo in check_str(path.name):
        report(typo)
    content = path.read_text()
    if not is_binary(content):
        for typo in check_bytes(content):
            report(typo)


def check_bytes(buffer):  # similar for `check_str`
    for identifier in parser.parse_bytes(buffer):
        if identifier in dictionary:
            yield dictionary[identifier]
        else:
            for word in identifier.split():
                if identifier in dictionary:
                    yield dictionary[identifier]


def parse_bytes(buffer):
    if is_ascii(buffer):
        for identifier in ascii_parser(buffer):
            yield identifier
    else:
        for text in utf8_chunks(buffer):
            for identifier in unicode_parser(buffer):
                yield identifier


def parse_str(buffer):
    if is_ascii(buffer):
        for identifier in ascii_parser(buffer):
            yield identifier
    else:
        for identifier in unicode_parser(buffer):
            yield identifier
```

Key takeaways:
- Starting with `check_bytes`, everything is composed of iterators nested in
  each other with transforms applied to them.
- Not as obvious in pseudo-code, extra layers are added to unify the iterator
  types between each branch.  In a language like Python, this is done by
  "boxing".  In `typos`, I chose
  [`Either`](https://docs.rs/itertools/0.10.1/itertools/enum.Either.html) and
  [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html).

## Performance of `try_for_each`

I need to first say, that I seem to have some jitter on my machine considering
[comparisons between `master` and `try_for_each`](https://github.com/crate-ci/typos/pull/310#issuecomment-875713350)
showed parsing taking 7% more time in some cases despite not changing.  I am
running in WSL on a laptop that is plugged in with most applications closed.  I
didn't spend much time making the environment less jittery.

With that said, `try_for_each` seems to have slowed down `check_file` more than that jitter:
```
check_file/Typos/code   time:   [147.52 us 150.23 us 153.51 us]
                        thrpt:  [1.8886 MiB/s 1.9298 MiB/s 1.9653 MiB/s]
                 change:
                        time:   [+26.226% +29.951% +34.477%] (p = 0.00 < 0.05)
                        thrpt:  [-25.638% -23.048% -20.777%]
                        Performance has regressed.
Found 8 outliers among 100 measurements (8.00%)
  5 (5.00%) high mild
  3 (3.00%) high severe

check_file/Typos/corpus time:   [50.001 ms 50.776 ms 51.638 ms]
                        thrpt:  [11.257 MiB/s 11.448 MiB/s 11.626 MiB/s]
                 change:
                        time:   [+21.952% +24.403% +26.950%] (p = 0.00 < 0.05)
                        thrpt:  [-21.229% -19.616% -18.001%]
                        Performance has regressed.
Found 10 outliers among 100 measurements (10.00%)
  6 (6.00%) high mild
  4 (4.00%) high severe
```
Remember: `check_file` is doing a lot more than running iterators, like IO.

One thing hurting `try_for_each` is that iterator authors can't provide
their own optimized implementation yet because the
[`Try` trait isn't stable](https://doc.rust-lang.org/std/ops/trait.Try.html).

## Porting to `try_for_each`

Since all iterators composed together and were only consumed in `check_file`,
this made it easy to port to `try_for_each`, see
[PR #310](https://github.com/crate-ci/typos/pull/310/files).

If specializing `try_for_each` (technically overriding the provided
`try_fold`) is going to be required to get the performance gains, that is
unfortunate.  When implementing `Iterator`, its already easy to overlook what
all other features we should implement for being a good citizen (e.g. size
hints).  Besides forgetting `try_for_each`, we'll also have to implement our
algorithms twice and keep them in sync.  A good test harness will help but all
of this increases the friction for taking advantage of `try_for_each`.

## Performance of `pushgen`

For `pushgen`, the improvements from lowering iteration overhead are not noticeable compared to everything else, like IO:
```
check_file/Typos/code   time:   [113.92 us 115.85 us 118.02 us]
                        thrpt:  [2.4566 MiB/s 2.5025 MiB/s 2.5449 MiB/s]
                 change:
                        time:   [-1.1247% +1.7826% +4.7350%] (p = 0.21 > 0.05)
                        thrpt:  [-4.5209% -1.7513% +1.1375%]
                        No change in performance detected.
Found 4 outliers among 100 measurements (4.00%)
  2 (2.00%) high mild
  2 (2.00%) high severe

check_file/Typos/corpus time:   [38.571 ms 39.072 ms 39.617 ms]
                        thrpt:  [14.673 MiB/s 14.877 MiB/s 15.071 MiB/s]
                 change:
                        time:   [-6.0200% -4.2708% -2.4899%] (p = 0.00 < 0.05)
                        thrpt:  [+2.5535% +4.4613% +6.4057%]
                        Performance has improved.
Found 9 outliers among 100 measurements (9.00%)
  5 (5.00%) high mild
  4 (4.00%) high severe
```

When breaking out just parsing, we do see decent improvements, for example:
```
parse_str/unicode/code  time:   [13.505 us 13.705 us 13.909 us]
                        thrpt:  [20.844 MiB/s 21.153 MiB/s 21.468 MiB/s]
                 change:
                        time:   [-19.030% -16.653% -14.273%] (p = 0.00 < 0.05)
                        thrpt:  [+16.650% +19.980% +23.503%]
                        Performance has improved.
Found 3 outliers among 100 measurements (3.00%)
  2 (2.00%) high mild
  1 (1.00%) high severe

parse_str/ascii/code    time:   [13.742 us 13.959 us 14.208 us]
                        thrpt:  [20.405 MiB/s 20.769 MiB/s 21.096 MiB/s]
                 change:
                        time:   [-12.491% -10.529% -8.5378%] (p = 0.00 < 0.05)
                        thrpt:  [+9.3348% +11.768% +14.274%]
                        Performance has improved.
Found 5 outliers among 100 measurements (5.00%)
  4 (4.00%) high mild
  1 (1.00%) high severe
```

And the overhead for adapting word-splitting into a `pushgen::Generator` was minimal:
```
split/words/code        time:   [1.4384 us 1.4654 us 1.4973 us]
                        thrpt:  [193.63 MiB/s 197.84 MiB/s 201.56 MiB/s]
                 change:
                        time:   [-1.9718% +2.1027% +5.9130%] (p = 0.29 > 0.05)
                        thrpt:  [-5.5829% -2.0594% +2.0114%]
                        No change in performance detected.
Found 3 outliers among 100 measurements (3.00%)
  3 (3.00%) high mild

split/words/corpus      time:   [2.7414 ms 2.7730 ms 2.8080 ms]
                        thrpt:  [207.02 MiB/s 209.63 MiB/s 212.05 MiB/s]
                 change:
                        time:   [+2.0584% +3.7444% +5.7863%] (p = 0.00 < 0.05)
                        thrpt:  [-5.4698% -3.6092% -2.0169%]
                        Performance has regressed.
Found 8 outliers among 100 measurements (8.00%)
  6 (6.00%) high mild
  2 (2.00%) high severe
```

`pushgen` won't be a magic bullet to improve performance but it might help with
specific hot-loops.  Let profilers and benchmarks be your guide.

## Porting to `pushgen`

`pushgen` is in its early stages, at v0.0.1.  I had to implement a lot of
features before porting `typos` to use `pushgen`.

Porting to `pushgen` mostly involved importing traits and swapping types.
- I did port my custom `Utf8Chunks` `Iterator` to being a `pushgen::Generator`.
- I adapted my word-splitting `Iterator` with `pushgen::from_iter` since I knew
  that would be messier and I wanted to know whether it was worth it first.
- Activating these `pushgen::Generator`s was taken care of by basing my branch
  off of my `try_for_each` branch.

See [PR #311](https://github.com/crate-ci/typos/pull/311/files).

There are still areas to explore to match the feature set of `Iterator`, like
[`Box<Generator>`](https://github.com/AndWass/pushgen/issues/35), double-ended
iterators, size hints, etc.

I imagine the name `Generator` will become confusing as Rust
[`std::ops::Generator`](https://doc.rust-lang.org/std/ops/trait.Generator.html)
becomes stable.
