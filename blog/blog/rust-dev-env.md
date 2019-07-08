---
title: A Rust Development Environment
is_draft: true
data:
    tags: ["programming", "rust"]
---
Sadly, I’ve recently switched to Windows at home for some expediency reasons. I’ll call out the Windows-specific parts

## Install [rustup](https://www.rust-lang.org/en-US/install.html)

rustup is the toolchain manager. This helps you upgrade the compiler, cross compile, etc.
When you install on Windows, you need to decide [which ABI want to use](https://github.com/rust-lang-nursery/rustup.rs/blob/master/README.md#working-with-rust-on-windows). I already had Visual Studio installed, so I went with that.
Rust has several release channels you can follow to stay up to date: Nightly, Beta, and Stable. By default, you are on the Stable channel.
Some basic commands to know
'''sh
# Update rustup
rustup self update
# Update the default rustc
rustup update
# Switch to nightly
rustup default nightly
'''
https://gist.github.com/epage/cb521bb2dd4ea262d6355821d2c8eb42


Developing with nightly is a bit controversial. Some interesting aspects of Nightly
The Rust community keeps it fairly stable
Early access to compiler features
Early access to language or library features
Access to features with designs too controversial to make it into the language yet

The last is very controversial due to [concerns it might split the community](https://news.ycombinator.com/item?id=13251729). This is slowly going away as the most [commonly used feature](https://github.com/rust-lang/rfcs/blob/master/text/1681-macros-1.1.md) (needed for serialization) has been approved and is making its way to the [Stable channel](https://github.com/rust-lang/rust/blob/master/RELEASES.md).
I personally am sticking to stable features but taking advantage of improved error messages. At this point, most of those have already hit Stable.

## Setting up [stdx-dev](https://github.com/llogiq/stdx-dev)

Rust tries to not be opinionated but to leave that to the community. Being blessed as part of a language helps to raise awareness but it has also been shown to stagnate projects or limit evolution. stdlib’s are where libraries go to die.
The community is still trying to work out the best way to raise awareness. One experiment is github repos that represent what would be included if Rust were to be batteries included. [stdx-dev](https://github.com/llogiq/stdx-dev) is the developer tools version of that.
TK rustfmt and clippy
rustup bug

Clippy doesn't work with rustup 1.0 on Windows · Issue #876 · rust-lang-nursery/rustup.rs
https://github.com/rust-lang-nursery/rustup.rs/issues/876

Looks like the PATH / LD_LIBRARY_PATH issue is back, that used to happen here: #402 If you run clippy with the latest…github.com

'''bat
@ECHO OFF
setlocal

set PATH=%PATH%;C:\Users\epage.AMER\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\x86_64-pc-windows-msvc\lib\

%*

endlocal
'''
https://gist.github.com/epage/6a4c3397d9d2f12894a636799f41cd50




## Setting up VIM

### [rust.vim](https://github.com/rust-lang/rust.vim)
Be sure to install rustfmt first
let g:rustfmt_autosave = 1in your .vimrc file if you want
Can’t speak for syntastic and playpen integration since I’ve not used them

### [vim-toml](https://github.com/cespare/vim-toml)

### [vim-racer](https://github.com/racer-rust/vim-racer)

Out of frustration, I almost installed vim-racer for auto-completion but held back. At the moment, my primary goal is to learn Rust. I wanted rustfmt to auto-clean up my code to not get distracted by formatting my code while I focused on learning.

TK https://github.com/dan-t/rusty-tags
