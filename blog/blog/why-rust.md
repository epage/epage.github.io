---
title: "Why Rust?"
is_draft: true
data:
    tags: ["programming"]
---
I’ve fallen into the pattern of using two kinds of languages: a rapid development language (Python) and a catch-all language (C++). I’ve long watched for something that would unseat C++ and I feel I finally found that in Rust.

A language for rapid development should allow me to quickly iterate on ideas even at the risk of there being bugs. In short, it shouldn’t be work.

My catch-all language needs to be usable within any environment I’m working on, included embedded applications, minimize defects, and give me control to get the maximum performance. I’m willing to sacrifice on the workflow to meet these needs.

For a long time, I’ve been an advocate for C++. I kept looking at new languages coming out that were suppose to save us from its flaws but they never quite accomplished the same level of expressiveness, usually due to their religious adherence to a single programming paradigm like OOP, or due to their lack of low level control.

- Java: OOP koolaide and protects the developer from themself too much
- C#: Initially a Java wannabe which means it copied a lot of the flaws though they try to patch over them now. Also maddeningly frustrating that they designed the language around using Visual Studio
- D: Initially the proprietary compiler limited things but then there was the standard library split. Also, I felt it was too inelegant. It felt like they kept bolting on features to the language itself when C++ could handle most or all of the cases through its expressiveness
- Go: Initially claiming to be a systems programming language but then they back off on it and it seems like it is being relegated to a backend language.

Rust is a language that Mozilla made to meet the needs of their research web browser, Servo, especially helping the developer handle insanely concurrent applications. Surprisingly, the language evolved into a systems programming language and is pretty exciting.

Some great things about rust:
- Full language, including exception-like error handling, is available in limited execution environments, unlike C++.
- A well designed package system (Cargo)
- A well designed toolchain manager (rustup)

TODO https://brson.github.io/fireflowers/
TODO https://withoutboats.github.io/blog/rust/2017/01/04/the-rust-module-system-is-too-confusing.html
TODO http://www.suspectsemantics.com/blog/2017/01/26/setting-expectations-for-rusts-difficulty/


https://vitiral.github.io/2017/02/25/one-year-with-rust.html
 Plus, it offered things which were completely new: concurrency without fear, safety without garbage collection and abstraction without cost.
 - change garbage collection to resource management
