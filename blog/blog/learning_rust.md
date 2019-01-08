---
is_draft: true
---
anstall rust up
- Started at the perfect time since rustup hit 1.0

vim-rust
toml
let g:rustfmt_autosave = 1

rustup install nightly
rustup run nightly cargo install clippy
See https://github.com/rust-lang-nursery/rustup.rs/issues/876
cargo install rustfmt

Later switched to nightly
- Cargo check
- Better errors
- Any performance improvements

cargo init --bin --name relint --vcs git



cargo install cargo-outdated

Document checking in the lock file

Started with error_chain but taking it out because I feel like I need to start with more basics.

Using ripgrep as my template
- Originally meant to just modify ripgrep but realized I'd learn more by doing it on my own
- Liked the arg/flag functions for using clap.  Modified them to be arg/option/flag to match clap's terminology

Surprised I don't see people using (24 days, clap docs, ripgrep)
- crate_authors!()
- crate_version!()

Integrated rustfmt to let myself not worry about formatting
Tempted to get auto-completion going but, like not directly forking ripgrep, I felt it would teach me the library best if I kept having to scour the documentation


Useful documentation
- https://doc.rust-lang.org/std/index.html#modules
- https://doc.rust-lang.org/book/README.html


Odd that extern crate seems duplicate of my Cargo.toml

Resource https://github.com/ctjhoa/rust-learning/blob/master/README.md

practices https://pascalhertleif.de/artikel/good-practices-for-writing-rust-libraries/

TODO
- https://www.reddit.com/r/rust/comments/5jr30w/tokei_now_has_a_badge_service_now_you_can_show/
- https://rust-lang.github.io/book/ch11-03-test-organization.html


Common mistakes
- When doing a match on an enum, I expect to list the data types rather than the enum instances

Improve compile errors
- When a function is moving a mutable value out, you get an error about type mismatch (normal + &mut).  Suggest people use *
- When complaining about not specifying lifetimes for a struct, name what those lifetimes are



People to inform
https://www.reddit.com/r/rust/comments/5ldqa9/regex_02_is_out_precursor_to_10/dbwpwo8/?st=ixm3gpy5&sh=784fe31a



Option/Result

Iter<Result<_>> -> Result<Iter<_>>
- i.collect()
- Will stop the iterator early
Iter<Result<_>> -> (Vec<_>, Vec<Err>)
- let (oks, fails): (Vec<_>, Vec<_>) = results.partition(Result::is_ok);
- let values: Vec<_> = oks.into_iter().map(Result::unwrap).collect();
- let errors: Vec<_> = fails.into_iter().map(Result::unwrap_err).collect();

http://xion.io/post/code/rust-iter-patterns.html

Option<Result<_>> -> Result<Option<_>>
- opt.map_or(Ok(None), |r| r.map(Some))

Option<Option<_>> -> Option<_>
- opt.map_or(None, |o| o)
