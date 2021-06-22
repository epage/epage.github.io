---
title: Code Review
data:
    tags: ["warstory"]
---

A lot of this is lessons hard learned from being a reviewer that
unintentionally made people feel dejected.  Regardless of my intentions, I had
to own my impact.  For some, my reviews were insightful learning experiences
but for others, they were a never ending gauntlet that burned them out.  My
tone defaults to analytical which comes off as short and critical.  I'd also be
focused on "my main task" and only mentally "check-in" when replying to an
update (particularly when assisting other teams as a subject-matter-expert),
losing track of how long the review has been running, how discouraging it is
for the person, and how inefficient it is for them.

# Goals

- Reduce bus factor by keeping other people abreast of changes
- Share culture and best practices
- Limit the effect of blindspots with more eyes
- Maintaining, or raising, the quality bar

Non-goals

- Exhaustively catch bugs

# Code Author

The primary concern is "how do I make this easier on the reviewer, especially so they focus on the important details?"

- Consider how controversial the change will be and make sure it is discussed
  before posting a review.  Is this a change of practice or not align with the existing abstractions?
- Keep reviewed changes small
- Annotate the review with a guided tour, especially giving the "why" behind
  the decisions you made.
- Communicate why you think the change is valid (ideally with tests)
- Make it clear what you changed in response to feedback
- Negotiate: you likely have more background on the constraints and timelines.
  Speak up and negotiate whether feedback needs addressing (e.g. not for
  short-lived code), or if it needs to be addressed now.
  - A big part of negotiating is trust: if you ask to defer changes, be sure to
    follow up and let the person know you did so.  Right or wrong, reviewers
    are likely to hold their ground more if they feel its "now or never".

# Code Reviewer

As a reviewer, you have to balance meeting the needs of the organization (e.g.
quality) with the code author (e.g. feeling dejected and checking out).

- Automate the minutia.  Besides saving you time, this makes code authors
  happier.  People feel better when there are emotionally detached, immediate,
  objective measures they are held against.  This isn't just linting but also
  includes other analysis like making it easy to browse covered lines to
  determine what is ok vs too risky.
- Be prompt: long cycle times are discouraging for the code author.
- Avoid associating code and ideas with people.  Discuss "this change", "this
  PR", and not "your PR".  This keeps the discussion objective and gives code
  authors more space to detach from their code.
- Periodically, check-in on your tone.  Text feedback loses context and can be
  interpreted in multiple ways.  When giving feedback, it usually is received
  in the most negative way.
- Be sure to call out the good as well!  Just make sure you aren't patronizing.
- If there is a lot of feedback for a PR, that suggests there is a disconnect
  in expectations.  These are best resolved by discussing directly, privately,
  and real-time with the person.
- If there is back and forth on an item of feedback, it is inefficient
  resolving by asynchronous back and forths and there might be a disconnect in
  understanding.  Again, switch to direct, private, and real-time
  communication.
- Pick your battles and make them clear by communicating severity.  Is the
  feedback about a minor stylistic improvement?  A major deviation from team
  standards or custom-facing bug?  Does it need to fixed before merging or can
  it be postponed to another PR?  How do you ensure there is a follow-up?
  - Is it worth yet another review cycle and the inefficiencies each cycle introduces?
  - Is it worth the risk being introduced since updates to a PR are probably
    not manually tested as exhaustively?
- Be humble by focusing on questions rather than feedback.  This gives the code
  author room to re-evaluate on their own, reducing the effect of ego and helps
  them grow.  It also helps remind you that the code author is the
  boots-on-the-ground and might have reasons for going a different route than
  you might theorize.

# Improving

Learn from other areas with review cultures, like the game
[Go](https://us.pycon.org/2018/schedule/presentation/149/),
[art](https://us.pycon.org/2018/schedule/presentation/149/), etc.

For example, in the game Go, when discussing the game:

>  Use "Black" and "White" instead of naming the players or using pronouns. This creates an objective discussion which allows both players to learn without their ego getting in the way.

From [Sensei's Library: Analyze the Game](https://senseis.xmp.net/?AnalyseTheGame)

# Resources

- [How to Make Good Code Reviews Better](https://stackoverflow.blog/2019/09/30/how-to-make-good-code-reviews-better/)
