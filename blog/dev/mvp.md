---
title: "Warstory: MVP (Minimum Viable Product)"
data:
    tags: ["warstory"]
---

I had a manager focused on shielding us, letting us focus on where we add value
while he took care of politics and unrealistic expectations.  Another team
would complain about us sandbagging our commitments though we delivered with
quality and on time while they were constantly on death marches.
Unfortunately, I had a project with a lot of overlap with this team.

We were releasing a new generation embedded hardware/OS combo.  Management was
pushing for us to support two embedded OSs on the hardware.  We ended up on a
death march to get this done in time for a company trade show.  Right at the
deadline, there was still some polish issues with the secondary OS (some were
outside our control).  Management decided to cut that second OS, saying it
would come back in the next release.

Narrator: it never did.

Later, we were releasing a new product based on this platform.  For faster time-to-market, it had ports on
it that were really soldiered plug-and-play sockets for a hardwired USB chassis
with a pretty front.  Future versions would not use this setup.  Management
wanted us to have the UX gloss over the physical hardware layout to show the
user the logical hardware layout.

The problem is last time we did this, it led to an unmaintainable mess.  The
expectation at the time was all hardware was supported for decades at a time.
This one-off piece of hardware was likely to make a significant impact to our
velocity for decades.  The sign off on this was owned by a management team that
played hot potato with their roles, never seeing the long term impact of their
decisions.

I fought this. I prepared presentations, talked to various people, and kept
telling management "no".  I had the support of other engineers, one-on-one, but
they weren't willing to stand up.  It finally came to the meeting where I was
going to present and give up.  I got approval to have the less-than-ideal UX on
the condition that the next generation UX would resolve this.

Narrator: it never did.

When planning our next generation UX, I brought up this UX problem and our
previous commitment to fix it, and all of the managers involved just shrugged
and ignored it.  No one cared anymore about this super important feature.

Turns out, in that meeting with the decision maker, he gave up at that point.
I barely outlasted him.

These taught me
- Only when being forced to make a delay/cut decision do we truly understand
  the MVP of a release.
- Marketing / sales can be capricious (turnover, lack of understanding of
  customer, overestimating the importance of deals) which makes committed feature
  sets within a deadline meaningless unless you can show lives are on the line
  (e.g. bug fixes that prevent death and dismemberment).
- Always ensure you have a feedback loop for the decisions you make.
