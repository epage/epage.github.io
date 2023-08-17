---
title: How to Maintain 100 Packages Without Going Insane
layout: default.liquid
is_draft: true
data:
  tags:
  - programming
  - rust
---

According to crates.io I have
[111 packages to my name](https://crates.io/users/epage?sort=recent-downloads)
as of writing this.
How can someone maintain so many packages?
The easy answer is "pad the numbers" though I'd like to think I'm not doing that.
Being paid to do it isn't necessarily an easy answer to this.
I've been iterating on what I do and figured I've gotten it refined to
a point others might find it useful.

Like questioning whether a serive needs to run at "google scale",
it can be worth asking if you need to maintain at "epage" scale.
I'd like to think the ideas here can help any aspiring package maintainer to
put more of their time towards what they want to do and less on the "work" side
of things.

<!-- more -->

## Purpose

An important question to ask yourself when starting a new project or helping
maintain someone else's is what is your purpose.

For the libraries I maintain, they are a means to an end, usually the programs
I help maintain like cargo.
In doing this, I recognize that I won't have the focus to make them the
absolute best so I have to consider what niche they will fill and whether that
niche is large enough to be worth it.
When I [forked `nom` to make `winnow`](https://epage.github.io/blog/2023/02/winnow-toml-edit-combine-nom/),
I had to decide whether the value gained out of forking would be worth the
liability of more code in the ecosystem to maintain.
I also knew that competing in areas like performance can take a lot of valuable
focus time, time I likely won't have to give and intentionally did not make
this a goal which is why I avoided talking numbers in the initial post.
Being the [fastest parser](https://epage.github.io/blog/2023/07/winnow-0-5-the-fastest-rust-parser-combinator-library/)
was a happy accident.

Others might want to maintain the fastest library or for it to be the best package in some other way.
That is great!
However, you need to recognize the focus time this will take and be realistic
about how many other things you can take on.

None of this is to say "you have to be X tall" to maintain a package.
Some might create a package to be more shortlived, whether as an experiment or
to fill a temporary gap, both roles my [`nom8`](https://crates.io/crates/nom8)
served before I decided to create `winnow`.
Others might create packages to learn.
Or you might have good intentions and have to leave things behind.
I know I have some packages I *want* to put more time towards but have not been
able to prioritize.

## Prioritize

I don't know about you but I will never get to all of the projects I want to work on.
Just some of the ideas I've not gotten to include
- [An updateable project template tool](https://github.com/epage/epage.github.io/issues/22)
- [A git-blame TUI](https://github.com/gitext-rs/git-dive/milestone/2) modeled off of [Perforce Time Lapse View](https://www.perforce.com/video-tutorials/vcs/using-time-lapse-view)
- [An ease of use, batteries included standard-library alternative](https://epage.github.io/blog/2021/09/learning-rust/)
- Figuring out [Nix](nixos.org/)
- Creating a [GUI for Nix](https://github.com/epage/epage.github.io/issues/8)

I repeatedly keep telling myself "not yet" because I want to work on them but I
know its better for me to put my time elsewhere.

## Focus Time

Some tasks just require sitting down and giving your undivided attention to it.

