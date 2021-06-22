---
title: Planning
---

# Burn-out

Software development is a marathon.  You can't repeatedly sprint to the finish
line or you burn out.  This is one thing I really liked about the place where
we did 6 month waterfalls was that the project planning phase was low pressure
and gave you time to recharge from the last cycle before pressure mounted for
the current one.

With sprints, the pressure was always on.  Sprints were filled to capacity and
you were accountable to what was accepted.  Now slowing down for recharge
projects, helping others, or resolving tech debt need to be negotiated against
the features in the backlog and require an ROI conversation to get in the
schedule.  There is no mental breathing room and instead is turning into
micro-management.

# Estimating

> It always takes longer than you expect, even when you take into account Hofstadter’s Law.

\- [Hofstadter's Law](https://en.wikipedia.org/wiki/Hofstadter%27s_law)

I've found estimates useful for:
- Quickly identifying when there are different expectations for a project (people with wildly different estimates).
- Identifying expensive solutions so you can evaluate alternative solutions or
  alternative implementation plans to get benefits incrementally rather than going "dark".
- Help with planning schedules.

In general, you don't need a high level of precision for purposes.  Some use
fibonacci numbers to discourage false precision.  The trap someone can fall
into is the idea "if large tasks have low precision, breaking it down into
small tasks will increase the precision!".  One problem with this is you start
adding up error
(see [significant figures](https://en.wikipedia.org/wiki/Significant_figures)).
The other is that you are unlikely to account for all of the unknowns.

I find that variability of precision is more relevant for time horizon (now vs
future) more than task size but estimating for the day or week has limited
value.

When estimating, precision is relative to whether the problem is
- Simple: Easy and repeatable
- Complicated: Hard but repeatable
- Complex: Hard and not repeatable

(paraphrased from "The Checklist Manifesto" by Atul Gawande)

# Schedules

I've found schedules good for:
- Provide check-in points to gauge the health of a project and re-evaluate, if needed.
- Only in rare circumstances, ensure you are driving to a specific deadline, external or internal.
  Where possible, find a way to deliver that minimizes the scope tied to deadlines.  See [Spotify's thoughts on
  Innovation vs Predictability](https://www.youtube.com/watch?v=vOt4BbWLWQw).


