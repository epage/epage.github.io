title: Using Cobalt for Blogging
published_date: "2017-12-17 09:00:30 -0500"
layout: default.liquid
data:
    tags: ["oss", "programming"]
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


https://stackoverflow.com/questions/39978856/unable-to-change-source-branch-in-github-pages

Be sure to have "Build only if .travis.yml is present" set

Used bootstrap for a starting point in CSS
https://getbootstrap.com/css/


officially recognized attributes
- title
- extends
- date
- description (used for RSS)
- except (used for RSS)
- excerpt_separator
- is_post
- draft
- path (or specify it in .cobalt.yml)
 - Can't access the original template file's name though

 Liquid documentation https://shopify.github.io/liquid/

 Odd, kstep https://github.com/kstep/kstep.github.com  seems to be able to use post.excerpt without it falling back to post.content

 Tries to mirror Jekyll but missing features
 - blank excerpt separator causes no excerpts to be created (note: default separator is two \n's
 - :title reflecting file name
 - :slug for path
 - site.uri isn't available.  Maybe at least provide link?


https://domchristie.github.io/to-markdown/ to convert stuff to markdown
