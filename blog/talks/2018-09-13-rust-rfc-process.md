---
layout: remark.liquid
format: Raw
permalink: /{{parent}}/{{year}}/{{month}}/{{slug}}/
data:
  venue: Utah Rust
---
class: middle

# Designing Rust

The RFC Process

???

Takeaways
- I am aware of whats going into Rust
- I can provide feedback on Rust’s direction
- I can get my ideas into a Rust release

---
class: middle

# Typical Life of an RFC

---

![Pre-RFC]({{site.base_url}}/talks/rfc-pre.png)

???


Get feedback!

A lot of the initial feedback on RFCs come via posts to https://internals.rust-lang.org/

---

![RFC]({{site.base_url}}/talks/rfc-rfc.png)

???

Proposing the RFC happens via a PR with discussion being comment

---

![FCP]({{site.base_url}}/talks/rfc-fcp.png)

???

“At some point, a member of the subteam will propose a "motion for final comment period" (FCP), along with a disposition for the RFC (merge, close, or postpone). “
...
- The FCP lasts ten calendar days, so that it is open for at least 5 business days. It is also advertised widely, e.g. in This Week in Rust. This way all stakeholders have a chance to lodge any final objections before a decision is reached.
- In most cases, the FCP period is quiet, and the RFC is either merged or closed. However, sometimes substantial new arguments or ideas are raised, the FCP is canceled, and the RFC goes back into development mode.

---

![FCP Disposition]({{site.base_url}}/talks/rfc-disposition.png)

???

“At some point, a member of the subteam will propose a "motion for final comment period" (FCP), along with a disposition for the RFC (merge, close, or postpone). “
...
- The FCP lasts ten calendar days, so that it is open for at least 5 business days. It is also advertised widely, e.g. in This Week in Rust. This way all stakeholders have a chance to lodge any final objections before a decision is reached.
- In most cases, the FCP period is quiet, and the RFC is either merged or closed. However, sometimes substantial new arguments or ideas are raised, the FCP is canceled, and the RFC goes back into development mode.

---

![Tracking Issue]({{site.base_url}}/talks/rfc-tracking.png)

???

---

![Propose FCP]({{site.base_url}}/talks/rfc-tracking-propose-fcp.png)

???

---

![FCP]({{site.base_url}}/talks/rfc-tracking-fcp.png)

???

---

![Merge]({{site.base_url}}/talks/rfc-tracking-merge.png)

???

---
class: middle

# Entry Points

---

![Pre-RFC]({{site.base_url}}/talks/rfc-pre.png)

???


Get feedback!

A lot of the initial feedback on RFCs come via posts to https://internals.rust-lang.org/

---

![This Week In Rust]({{site.base_url}}/talks/rfc-twir.png)

![This Week In Rust: New]({{site.base_url}}/talks/rfc-twir-new.png)

???

---

![This Week In Rust]({{site.base_url}}/talks/rfc-twir.png)

![This Week In Rust: FCP]({{site.base_url}}/talks/rfc-twir-fcp.png)

???

---

![This Week In Rust]({{site.base_url}}/talks/rfc-twir.png)

![This Week In Rust: Approved]({{site.base_url}}/talks/rfc-twir-approved.png)

???

---

![This Week In Rust]({{site.base_url}}/talks/rfc-twir.png)

![This Week In Rust: Tracking]({{site.base_url}}/talks/rfc-twir-tracking.png)

???

---
class: middle

# Best Practices

???

Coming from someone who hasn’t even written an RFC :)

I have seen trends in what seems to work well or what is encouraged.

---

# Best Practices

Brainstorm

---

# Best Practices

Brainstorm

Focus on requirements, not features

---

# Best Practices

Brainstorm

Focus on requirements, not features

Push existing boundaries

???

Push existing boundaries
- Have a working implementation of your idea within the confines of the existing language, using nightly if needed
- Identifies strengths / weaknesses of the proposal
- Identifies importance of priority
- E.g. failure crate led to Error Trait work

---

# Best Practices

Brainstorm

Focus on requirements, not features

Push existing boundaries

Expect iteration

---

# Best Practices

Brainstorm

Focus on requirements, not features

Push existing boundaries

Expect iteration

Implement it!

???


You aren’t obligated to implement your RFC but sometimes we write more RFCs then there are people to implement.

In 2017, this led to “impl Period” where RFCs were put on hold and people focused on implementing them.

So if you want to ensure it gets in, lending a hand is the surest way to that.

---

# Best Practices

Brainstorm

Focus on requirements, not features

Push existing boundaries

Expect iteration

Implement it!

Test it out

???

With the module re-work, people converted crates over to it to get a feel for what the code looked like and how it was to write code with that RFC.

This is possible because most features live behind a feature gate before hitting a stable release.

---
class: middle

# Challenges

---

![Undo]({{site.base_url}}/talks/rfc-undo.png)

???

Impl Trait involved:
- At least 5 RFCs, see https://github.com/rust-lang/rust/issues/34511
- Many internals.rust-lang.org threads
- Who knows how many IRC and offline conversations

The big ticket item was `impl Trait` in return position (existential types).  People lost track of the status of argument-position until the Rust release notes.

---
class: middle, center

![aturon]({{site.base_url}}/talks/rfc-aturon.png)

???

From https://www.reddit.com/r/rust/comments/8m3vlk/aturonlog_listening_and_trust_part_1/?st=jls9kk6h&sh=9c98c7d9

---

# Rust’s Design Values

Memory safety without garbage collection

Abstraction without overhead

Concurrency without data races

Stability without stagnation

???

From aturon’s blog series
- https://aturon.github.io/2018/05/25/listening-part-1/
- https://aturon.github.io/2018/06/02/listening-part-2/
- https://aturon.github.io/2018/06/18/listening-part-3/

---
class: middle, center

Of course, such reconciliations are not always possible, and certainly aren’t easy. It’s an aspiration, not an edict. But Rust’s culture and design process is engineered to produce such outcomes, by embracing pluralism and positive-sum thinking:

&mdash; Aaron Turon

???

- Pluralism is about who we target: Rust seeks to simultaneously appeal to die-hard C++ programmers and to empower dyed-in-the-wool JS devs, and to reach several other varied audiences. That’s uncomfortable! These audiences are very different, they have divergent needs and priorities, and the usual adage applies: if you try to please everyone, you won’t please anyone. But…
- Positive-sum thinking is how we embrace pluralism while retaining a coherent vision and set of values for the language. A zero-sum view would assume that apparent oppositions are fundamental, e.g., that appealing to the JS crowd inherently hurts the C++ one. A positive-sum view starts by seeing different perspectives and priorities as legitimate and worthwhile, with a faith that by respecting each other in this way, we can find strictly better solutions than had we optimized solely for one perspective.

---
class: middle, center


A high velocity thread often seems heated and “controversial”, even if the discussion is respectful and chock full of insights.

&mdash; Aaron Turon

???

- Pluralism is about who we target: Rust seeks to simultaneously appeal to die-hard C++ programmers and to empower dyed-in-the-wool JS devs, and to reach several other varied audiences. That’s uncomfortable! These audiences are very different, they have divergent needs and priorities, and the usual adage applies: if you try to please everyone, you won’t please anyone. But…
- Positive-sum thinking is how we embrace pluralism while retaining a coherent vision and set of values for the language. A zero-sum view would assume that apparent oppositions are fundamental, e.g., that appealing to the JS crowd inherently hurts the C++ one. A positive-sum view starts by seeing different perspectives and priorities as legitimate and worthwhile, with a faith that by respecting each other in this way, we can find strictly better solutions than had we optimized solely for one perspective.

---

# Root Cause

Momentum

Fatigue

Urgency

???


I personally don’t see high comment velocity as a root problem, but an issue that relates to deeper dynamics:
- Momentum. A comment thread has a kind of “momentum” of sentiment that can be hard to shift, and also hard to gauge as the thread gets long. If an RFC has an initial batch of negative (or positive) comments, it can be difficult to recover, in part because these are the first sentiments everyone sees.
- Urgency. Because comment threads are a major input into the decision-making process, there’s a sense of urgency to participate and keep the discussion “on course” from your personal perspective. This urgency is compounded when a thread is fast-moving or lengthy, or when a proposal originates from a team member (and thus is reasonably seen as having a higher chance of landing).
- Fatigue. Many people participate early on in an RFC thread, only to ultimately step away because the thread has too much traffic to keep up with or influence. There’s also sometimes a feeling of a topic getting discussed “to death”; many felt that way toward the end of the modules saga.

---

# RFC for RFCs

![Proposed Process]({{site.base_url}}/talks/rfc-process.png)

???

Iteration is happening but isn’t being applied yet.

http://smallcultfollowing.com/babysteps/blog/2018/06/20/proposal-for-a-staged-rfc-process/
https://aturon.github.io/blog/2016/07/05/rfc-refinement/

Some improvements have already been made, like to This Week in Rust.  Others I think are deferred past the 2018 Edition.

While there are still problems (like unannounced discussions in meetings without notes), things are improving.
