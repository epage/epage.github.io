extends: default.liquid

title: Using Cobalt for Blogging
date: 17 February 2017 09:00:30 -0500
path: /:year/:month
---

So I'm trying out [https://github.com/cobalt-org/cobalt.rs](Cobalt) and [https://pages.github.com/](github.io) for blogging.

## [https://pages.github.com/](Setup) my user github.io repo

Follow the github.io [https://pages.github.com/](steps).

## Initialize the repo

Instead of running
'''bash
cobalt new myBlog
'''

I copied from [https://github.com/amethyst/website](amethyst).

The main differences are
- Posts are kept in a "src" directory

### Tweak travis.yml

Enable syntax highlighting by replacing the install command with
'''bash
cargo install cobalt-bin --features="syntax-highlight"
'''

Set the email address

Went with practices recommended in llogiq's [https://llogiq.github.io/2016/07/05/travis.html](blogpost)
'''yaml
sudo: false
'''
