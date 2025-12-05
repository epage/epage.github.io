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

Like questioning whether a service needs to run at "google scale",
it can be worth asking if you need to maintain at "epage scale".
I'd like to think the ideas here can help any aspiring package maintainer to
put more of their time towards what they want to do and less on the "work" side
of things.

<!-- more -->

## Purpose

An important question to ask yourself when starting a new project or helping
maintain someone else's is what is your purpose.

For the libraries I maintain, they are a means to an end.
The ends are  usually the programs I help maintain like cargo.
In doing this, I recognize that I won't have the focus to make them the
absolute best so I have to consider what niche they will fill and whether that
niche is large enough to be worth it.
When I [forked `nom` to make `winnow`](https://epage.github.io/blog/2023/02/winnow-toml-edit-combine-nom/),
I had to decide whether the value gained out of forking would be worth the
liability of more code in the ecosystem to maintain.
I also knew that competing in areas like performance can take a lot of valuable
focus time, time I likely won't have to give.
I intentionally did not make this a goal which is why I avoided talking numbers
in the initial announcement post.
Being the [fastest parser](https://epage.github.io/blog/2023/07/winnow-0-5-the-fastest-rust-parser-combinator-library/)
was a happy accident.

Others might want to maintain the fastest library or for it to be the best package in some other way.
That is great!
However, you need to recognize the focus time this will take and be realistic
about how many other things you can take on.

None of this is to say maintaining a package means you have to commit upfront
to a certain level of involvement.
Some might create a package to be more short-lived, whether as an experiment or
to fill a temporary gap, both roles my [`nom8`](https://crates.io/crates/nom8)
package served before I decided to create `winnow`.
Others might create packages to learn.
Or you might have good intentions and have to leave things behind.
I know I have some packages I *want* to put more time towards but have not been
able to prioritize.

## Prioritizing

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

cart brought up the idea of [focus areas for bevy](https://bevyengine.org/news/scaling-bevy/#focus-focus-focus).

> These areas will receive priority for my time, and I will try my best to
> direct contributors to those areas. This doesn't mean others aren't free to
> explore other areas they are interested in. Just don't expect to get them
> merged quickly. Ideally you're building a standalone plugin that doesn't
> require being directly merged into the Bevy repo anyway.

For me, this meant
- If an issue or PR looks trivial to triage, do so
- Choose a focus area among all projects and double-down on it until done

The idea is to reduce context switching, especially when it requires a lot of
background to re-acquaint with.
This is why the next task should most
likely be in the same project until enough important tasks are done to move on to
a project with the highest priority task.

## Issue and PR Creation

Something to keep on an eye on is what causes you to feel pressure to step out
of a focus area when you shouldn't.
For me, I've found there is an implicit social pressure to merge PRs.
Someone had a need and devoted their time to resolve it.
Quickly reviewing and merging PRs encourages further contributions and use of your package.
PRs can take a lot of time whether from not having consensus on the problem or
requirements, to problems with the implementation, to just the work it takes to
re-acquainting with some intricacies.
I've seen some who dealt with this pressure by becoming apathetic to
these users.

I'm trying to find a middle ground for this.
So far, my approach is to [document](https://github.com/clap-rs/clap/blob/master/CONTRIBUTING.md#where-to-start)
that work should start in an issue and once there is agreement, then a PR is welcome.
The downside is a lot of people (myself included) overlook contributing guides.
So far I've been taking a hard-line and close PRs, redirecting people to issues.

Another benefit I've found to use Issues for discussing the problem, requirements, and the user-facing design is it is more likely to be in one place.
I've had enough cases where multiple PRs are used to solve a problem, both from multiple people taking a crack at it or from one person working out of several branches.
This spreads out the relevant information, making it harder to for myself and others to catch up on and evaluate a PR against.

For issues, its at least a little easier to guide users on the right track via
[issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository).
For me, the goals are:
- Collect enough information
- Encourage a focus on problems and requirements, trying to break filers from focusing on solutions
- Avoid being too burdensome to create issues.

You can checkout what I do for
[clap](https://github.com/clap-rs/clap/tree/master/.github/ISSUE_TEMPLATE)
for ideas.
Some highlights include:
- A link encouraging use of Discussions
- Separate out bugs and features
- Focus bugs on behavior differences
- Call out use cases for features

## Automating

### Multiplying effort across repos

For most of my projects, the differences in CI and other aspects is minimal.
I want it as easy as possible to update all of my projects when I discover a new best practice.
Originally, I did this by running a directory diff on all of my projects and trying to make sure I found everything unintended differences.
This required multiple passes as I found a tweak made to a project, I then had to go back through diffing them all.

Then one day someone pointed out the potential for `git merge --allow-unrelated-histories`:
create a template repo and after-the-fact add it to all projects
(I think they were quoting a [Jon Gjengset video](https://www.youtube.com/@jonhoo)).
This has been a huge improvement to my workflow.
Now, when I discover a new idea, I add it to my
[template repo](https://github.com/epage/_rust)
and I do an occasional `git merge _rust/main` in each repo.
Resolve merge conflicts and done.

Now, using a template doesn't helpful for boilerplate that lives outside of the repo.
For github repo settings, this is where [Settings](https://github.com/apps/settings)
comes into play.
This app/bot is a thin wrapper around Github's APIs, letting you control your repo's settings in a file.
The main downside I've seen to this is the error reporting; its common for it to seem to not work and I have no idea what I did wrong.

### Reducing work within a repo

- RenovateBot
- cargo release

https://www.jlowin.dev/blog/oss-maintainers-guide-to-saying-no
