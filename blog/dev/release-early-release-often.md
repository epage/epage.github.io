---
title: "War Story: Release Early, Release Often"
data:
    tags: ["warstory"]
---

The phrase
[release early, release often](http://www.catb.org/~esr/writings/cathedral-bazaar/cathedral-bazaar/ar01s04.html)
was popularized by the "The Cathedral and the Bazaar".

At a prior job, we did a hybrid of feature-driven and timed releases.
We'd release twice a year in a tic-toc fashion
(longer cycle for big changes, shorter cycle for fewer changes).

This led to death marches and burn out.
When Sales came to R&D with a users need, it was needed yesterday.
Let's look at the latency of different release cycles by looking at two extremes:
release after each PR (how I treat my open source projects) or release once every 3 years (the C++ language spec).
With instant release, if the latency is 0.
With 3 year release cycles, the latency is 0-3 years, depending on when my feature was merged.
When a feature is at risk of not making the merge window,
the choice is waiting 3 more years for it or have a death march to make sure the feature makes the release.
The shorter the release cycle, the less pressure there is to force work to make a release.
This applies to open source too but instead the contributor is filling the role
of sales, management, and developer and put themself through their own death
march.

Paradoxically, when you release less frequently,
the overhead of the release is likely to be the same or worse than multiple releases in the same time period.
The first factor is the [cost/benefit trade off of automation](https://xkcd.com/1205/).
The second factor is related to the concept that [10x Is Easier Than 2x](https://www.penguinrandomhouse.com/books/710816/10x-is-easier-than-2x-by-dan-sullivan-founder-of-strategic-coach-with-dr-benjamin-hardy/).
I won't be speaking to that book because I've not read it, but the title matches my experience.
When you focus on smaller, more incremental improvement (like 2x),
you focus on the smaller, more immediately beneficial improvements.
The [cost/benefit trade off](https://xkcd.com/1205/) is a part of this.
When you focus on dramatic improvement (like 10x),
you need to re-think fundamentals to find a solution.
My first exposure to this idea was at this company.
They decided to adjust to a strict 3-month potential-release cycle.
They wouldn't necessarily release to customers every 3 months but they would simulate
the process to highlight these more fundamental problems that needed fixing to reduce their release overhead.

I said the worst-case latency for a 3 year release cycle is 3 years but that is a lie and is more likely 3.5+ years.
Why is that?
When a release cycle is extremely short,
a develper can't half-implement something and say "I'll fix that in a follow up".
When the release cycle is anything but extremely short, this becomes much easier.
This usually starts with tests and documentation but quickly expands to
functionality and stability of the new feature
(without tests and a clear communication of what it is, how do you know its done?).
You go from just tracking features and bugs to tracking incomplete, unstable features that must be completed by the deadline.
Now you need feature-freezes before each release where people work on stabilizing the product.
In addition to affecting the latency for delivering a feature,
extended, uncontrolled periods of "stop the world and fix bugs" can be draining and lead to burn out.
As an aside, we are now veering into feature-driven releases, unless its cost effective to revert the feature.

The later a bug is discovered, the more expensive it is to fix.
A stabilization effort at the end of the release is more expensive than discovering and fixing it immediately.
This means the growth of overhead from a release stabilization effort is more
than the cumulation of finding and fixing the bugs during the initial
development.

However, your own testing will never be as exhaustive as users as they can be
[quite creative in how they use your software](https://xkcd.com/1172/).
The only true test is to put it into users hands, for them to do acceptance
testing, rolling back if there are problems.
The longer the release cycle, the higher the expectation of "perfection" because of the latency for getting users fixes
(unless its deemed worthy for a hot fix but those are expensive because releasing is expensive).
This "perfectionism" makes on-boarding new contributors more difficult and
frustrating and leads to existing contributors being more prickly and burning
out as they get frustrated in enforcing these standards of perfection.

With this being hybrid with feature-driven releases,
the schedule was flexible to accommodate the "committed features".
The idea was to be a minimal-viable product
but you didn't find out what was truly minimal until a week before the release.
Rather than directly addressing the problem with self-honesty and telling others "no",
they invested a new term, "key".
When that still didn't work, they invented the term "driven".
To be fair, the system was working against them due to latency from the long release cycles.

A side effect of this lack of self-honesty and unwillingness to say "no"
was each feature team's PM would assume the schedule would slip
because the other feature teams were going to be late
(because they always are),
so they wouldn't address the problem of self-honesty until its too late.
This was so rampant that we coined the term "schedule [chicken](https://en.wikipedia.org/wiki/Chicken_%28game%29)".
This meant every release was late **and** features would be cut.
This also meant managers were driving a death march even for features that
weren't critical enough to hold up the release.
This is still relevant for projects that exclusively use timed releases with long development cycles because
there is the temptation to let a feature influence the schedule to avoid the issue of latency.
For example, in Rust, there were people saying that the 2024 Edition (a
meta-release) should slip to allow some specific features to "make it".

I wasn't around to see how their efforts to improve their release process
worked out but I have been contributing to Rust which follows a 6 week release
cycle which has been great!
I rarely think about thw merge window for a given release.
I know if I miss the next beta branch, waiting 6 weeks isn't too big of a deal.
The main time I do pay attention is when I want to *add* latency by merging
just after the beta branch so it gets 12 weeks of pre-release testing, instead
of 6-12.
Features are merged as complete or feature gated and developed incrementally.
Only a couple of times have I intentionally staged a change into multiple PRs
that all had to be in the same merge window
(the PRs were posted in days of each other).
We don't "staff" "stabilization cycles" but occasionally cherry-pick changes to
a passively maintained beta branch if we feel we should reduce the latency for
delivering the fix.
