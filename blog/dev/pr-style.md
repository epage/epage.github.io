---
title: "Guideline: PR Style"
data:
    tags: ["guideline"]
---

# Checklist

Discussion
- Start with an Issue ([D-ISSUE](#d-issue))
- Avoid noise ([D-NOISE](#d-noise))
- Enable decisions ([D-SUMMARIZE](#d-summarize))
- Prepare the reviewer ([D-PREPARE](#d-prepare))
- These are two-way discussions ([D-DISCUSS](#d-discuss))
- Exercise caution in marking comments as resolved ([D-RESOLVE](#d-resolve))

Commit structure
- Atomic commits ([C-ATOMIC](#c-atomic))
- Reproduce the problem ([C-TEST](#c-test))
- Split out refactors ([C-SPLIT](#c-split))
- Be clear in a commit's intent ([C-INTENT](#c-intent))
- Isolate controversy ([C-ISOLATE](#c-isolate))

# Principles

Problems we are trying to avoid with these guidelines include:
- PRs immediately rejected
- PRs sitting in limbo for years
- Back and forth that burns you out from your contribution

Preparing a PR is an act of persuasive writing where you are trying to convince a reviewer to accept your contribution.
A contribution involves both the user impact and the architectural impact and the reviewer needs to be convinced on both fronts.

The [modes of persuasion](https://en.wikipedia.org/wiki/Modes_of_persuasion) include
- ethos, or an appeal to authority
- pathos, or an appeal to emotion
- logos, or an appeal to logic

Ethos can be dangerous.
If the reviewer does not the value the authority you are leveraging,
they will distrust you and your contribution.
Coercing through authority can lead to resentment or disengagement
(e.g. relying on management pressure to push your PRs through).
If the reviewer values your authority in any form,
they could be concerned for how a perceived slight might get amplified.
They may be more cautious in what they say, causing longer feedback loops.
A contributor can borrow authority by looking towards prior art but that can lead to cargo culting.
Prior art should be backed by analysis, making it part of your logos.

Contributors relying on pathos has led to many negative outcomes for reviewers.
For some, they take on a guilt that pressures them to do more and they burnout.
Some avoid the guilt by emotionally shutting themselves off from contributors.
A light touch is needed,
like providing anecdotes when getting buy-in on the problem being solved
which can help reviewers feel the impact the contribution will have on users
more than statistics would.

Logos is the primary mode when contributing.
Reviewers need to internalize your reasoning to help them make decisions in the future around your or related use cases.
Look to prior art and the motivations behind them (inferred if needed).
Provide guideposts within your contribution to step them through and understand your PR.

A contributor's audience also extends to future readers, including
- Users looking at PRs to better understand a CHANGELOG entry
- Contributors seeking to better understand the motives for the code they are changing
- Contributors root causing behavior, like with `git bisect`

# Discussion

<a id="d-issue"></a>

## Start with an Issue (D-ISSUE)

Each PR should link to an Issue with a reviewer's buy-in on the solution, either fixing it or being an incremental step (e.g. a refactor, see [C-ISOLATE](#c-isolate)).

When creating the Issue,
the focus should be on the relevant use cases and the challenges within them.
Provide anecdotes to add some ethos.
Potential solutions can and will be discussed but as exploration of options.
Do not be too attached to a specific solution.

Starting with a PR causes the discussion of use cases, challenges, solutions, and the implementation of one particular solution to all be mixed together.
Without discipline,
reviewers and contributors can needlessly iterate on the implementation in parallel to conversations that can lead to more fundamental changes, causing extra iterations and delay that can lead to frustration and burnout on both sides.

Discussing use cases, challenges, and solutions on a PR can fracture the conversation,
making it harder to follow,
making it take more work to come up to speed when looking at it,
likely leading the reviewer to avoid your PR.
Not all PRs get merged and a different PR might finally address the problem.
Some contributors are less familiar with git and create a new PR each time they try to update it.
Some will try out multiple solutions.
Some will abandon their work and another will pick it up.
The reviewer does not know whether the PR they are looking at will be one of these.

If someone does walk away from a PR,
the need is no longer being tracked in the reviewer's backlog.
Someone looking for existing conversations is likely to look among the Issues and not find it.

PRs also create social pressure on reviewers to quickly merge them.
The contributor is invested in the problem and there being a solution and presumably put effort into that particular solution.
Catering to these contributors can lead to burnout or emotional disengagement of reviewers.
Similarly, a large review backlog can burnout reviewers.
Contributors waiting on the resolution of an Issue gives reviewers the opportunity to evaluate their capacity for reviewing changes.

Exceptions:
- Obvious cases, on the order of typos.  In practice, many people over-estimate how obvious their case is, even for seemingly "simple" documentation changes.

Examples:
- [RFC 3537](https://rust-lang.github.io/rfcs/3537-msrv-resolver.html#status-quo) includes an anecdote in the collapsed text
- [winnow#131](https://github.com/winnow-rs/winnow/issues/131) had one contributor work through multiple solutions ([reference](https://github.com/winnow-rs/winnow/issues/131#issuecomment-2483705925)) and was resolved by another pair of contributors in [winnow#804](https://github.com/winnow-rs/winnow/pull/804)

<a id="d-noise"></a>

## Avoid noise (D-NOISE)

Do not make low effort posts,
e.g. "me too", "when will this be fixed?", "what is the status?".

These add nothing.

The Issue is the source of truth for a problem and should be updated if there is a status change.
Acting as a "forcing function" by posting notifies everyone, wasting there time.
If a reviewer decides to provide a personalized update for you,
it will cause a disruptive context switch and take from their total available time,
reducing the capacity to productively move that Issue forward.

"Me too"s are an ineffective way of communicating the scope of a problem.
When there is something new, describe your use case and the problem in a way to either convey the importance of the feature or additional requirements.

<a id="d-summarize"></a>

## Enable decisions (D-SUMMARIZE)

Summarizing the state of an Issue in terms that address reviewers needs is a powerful way to move an Issue forward.
Long threads take time for all participants to catch up on each time the topic is discussed
and making it likely that reviewers avoid the topic.

Reviewers need to consider:
- Alignment with the values of the project
- Alignment with other users needs
- Alignment with existing and expected features to make sure they are cohesive
- Long-term maintenance load, in particular the increase in expectations a change creates for users

Go beyond your personal need and look through the thread with that lens, covering topics like
- Use cases
- Requirements derived from those use cases
- Relevant requirements of the project that impact this Issue
- Relevant prior art from inside and outside of the project
- Solutions and their trade offs

It can also help to separately post a proposed solution derived from the above.

Be careful about too lightly dismissing roadblocks as irrelevant as that can erode trust in your summary with the reviewer.

Examples
- <https://github.com/rust-lang/cargo/issues/2644#issuecomment-1489371226>
- <https://github.com/rust-lang/cargo/issues/8728#issuecomment-1610265047>
- <https://github.com/rust-lang/cargo/issues/9930#issuecomment-1489089277>

<a id = "d-prepare"></a>

## Prepare the reviewer (D-PREPARE)

Set expectations for the reviewer through the PR description and inline comments.

Example topics to explore include:
- Where did you diverge from previous discussions and why?
- What is deferred and why?
- What decisions did you end up making along the way and why?
- What does not have automated testing and why?  What manual testing was done for it?

The answer to some of these questions may best be left attached to related lines of code as inline comments.

You may also want to add inline comments to acknowledge and start discussions on controversial topics.

<a id = "d-discuss"></a>

## These are two-way discussions (D-DISCUSS)

Issues and PRs are discussions and communication should go both ways.
As the user and/or the contributor,
you have a unique perspective that the reviewer lacks as they are viewing this from afar.
Don't blindly accept everything they say but share with them that personal knowledge.
However, recognize that in the end it is the reviewers decision and accept final decisions.

Similarly, respond to questions instead of making assumptions about their intent and pushing new code changes.
Even if you decide to act on a question,
still provide an answer and recognize that it might not be fully resolved.

<a id = "d-resolve"></a>

## Exercise caution in marking comments as resolved (D-RESOLVE)

Leave the resolving of threads to the reviewer unless you are extremely confident in your change and/or answer.

The reviewer will likely want to confirm how their feedback was resolved to make sure it was fully understood.
Leaving it open keeps it visible in their backlog.

# Commit structure

<a id="c-atomic"></a>

## Atomic commits (C-ATOMIC)

Each commit is minimal, yet complete.
It fulfills a single purpose.
It stands enough on its own to build, pass tests, pass formatters, etc.

Symptoms of a non-atomic commit:
- "and" is in the commit description: likely fulfilling two purposes and should be split
- contains dead code: not complete, making it harder to review the appropriateness without the caller
- common operations during a `git bisect` would fail (or mangle the commit like formatters)
- being followed by commits that address CI failures or review feedback

Commits within a PR serve as guideposts for explaining your change to the reviewer and future readers.
By focusing on atomic commits,
the reader is given a cohesive picture of what that commit is intended to do for properly evaluating all of the pieces.

Commits are the unit for root causing a problem with `git bisect`.
With atomic commits, the user running `git bisect` is likely to get a more precise result.

Exceptions:
- A "fix" to the main behavior-changing commit is purely additive and intentionally created as a separate commit more to break down the problem into smaller pieces for the reviewer to follow
- Pedantic linters may fail due to expected changes in upcoming commits within the PR
- If a commit is too large to easily understand, a well abstracted unused API may be a good place to split it to help guide the user through your changes

<a id="c-test"></a>

## Reproduce the problem (C-TEST)

Start the PR with one or more commits that introduce any additional tests needed for the PR,
with the tests reproducing the existing behavior.
Any feature or fix commits will then update the tests as behavior changes.

![test and fix commits in PR]({{site.base_url}}/talks/contrib-test-fix-split.png)

![impact of fix on tests]({{site.base_url}}/talks/contrib-test-after.png)

This applies the concept of "a picture is worth a thousand words" to your PR.
As future commits change the behavior,
this gets visualized in the diff of the test results,
clarifying the intent of your PR better than most PR descriptions both to the reviewer and any community members looking to better understand the intent of change.

This also helps you in making sure you are testing the right behavior
if you make a behavior change and the tests remain unchanged.

Depending on the type of change, the initial tests can take many shapes, including
- the fix commit just updates the expected output
- the feature commit adds an additional API call and updates the expected output
- the test commit uses an older API and the feature commit switches it to the new API call and updates the expected output

In determining how to structure the test commits,
the focus should be on ensuring the diff of the tests accentuates the key changes and minimizes noise.

Examples:
- [annotate-snippets-rs#380](https://github.com/rust-lang/annotate-snippets-rs/pull/380)
- [cargo#16265](https://github.com/rust-lang/cargo/pull/16265)

<a id="c-split"></a>

## Split out refactors (C-SPLIT)

Split refactor steps out of commits with behavior changes.

A commit may seem atomic because it has only one side effect but
the reviewer can still be flooded with a sea of green and red lines and
have to reverse engineer the changes to identify how the pieces fit together,
what changed behavior, etc.

Identifying and splitting out refactors can provide guideposts
to help the reviewer step through the change in smaller, faster to understand pieces.
Ideally, they would be broken down into individual refactor steps
(e.g. "rename a variable" or "extract a method").
It can even be helpful to split out the adding or removing of a scope.
For example, say a group of logic is going to be gated by an `if` condition,
you can have a refactor commit that moves that logic into an indented block
and then have the commit that adds the needed `if` before it.

If a lot of refactors touch the same section of code,
this means that a merge conflict on that section of code needs to be resolved multiple times.
However, resolving each conflict is much easier.
With [C-ISOLATE](#c-isolate),
the review can also go much faster, reducing the risk of conflicts.

For refactors that only make sense in the context of a feature or fix,
structuring all of this can be throwaway work if the PR is not accepted.
This is less likely to happen with [D-ISSUE](#d-issue).

How far to split up commits is subjective,
based on the complexity of the problem and
the familiarity of your reviewer with that section of code.

Examples:
- [annotate-snippets-rs#380](https://github.com/rust-lang/annotate-snippets-rs/pull/380)

<a id="c-intent"></a>

## Be clear in a commit's intent (C-INTENT)

The commit summary should clearly state the intent of the commit,
particularly the visibility of the impact.
This is where methods like [Conventional Commit](https://www.conventionalcommits.org/) help
which provides a way for describing the impact
(e.g. `feat`, `fix`, `refactor`)
and the scope of that impact.

This helps the reviewer know what to look for or not look for,
making it faster to scan for the commit and find ways that the commit didn't match the intent
(e.g. a refactor changing an error message).

Examples:
- [annotate-snippets-rs#380](https://github.com/rust-lang/annotate-snippets-rs/pull/380)

<a id="c-isolate"></a>

## Isolate controversy (C-ISOLATE)

Isolate changes that are likely to generate a lot of discussion to PRs with the fewest commits possible.
If refactors make sense independent of the controversial part of the PR,
split them out into a separate PR.

The most controversial part of a PR draws the most attention,
causing other parts of the PR to be neglected,
serializing the review feedback cycles.

It can also be difficult to track the state of many different threads and some things are likely to be forgotten until the very end, causing extra feedback cycles or PRs that aren't ready to be merged.

The longer the PR review takes,
the higher the chance of merge conflicts building up.
These are likely isolated to refactors commits,
thanks to [C-SPLIT](#c-split),
and a fairly good likelihood that they can be justified without the controversial part,
making them quick to merge when split out into a separate PR.

By splitting a PR up into smaller chunks,
any other work dependent on those earlier PRs is now unblocked.

If a change needs to be reverted,
you've made it clear what the minimally invasive set of commits is for reverting.

Commits should not be split out into a dependency PR if there is an assumption that the now dependent PR will be merged in the future.

You don't know what will be controversial before you post your PR.
Familiarity with the culture and practices of a project makes this easier to predict.

Examples:
- [cargo#13993](https://github.com/rust-lang/cargo/pull/13993) (refactor),
  [cargo#14169](https://github.com/rust-lang/cargo/pull/14169) (refactor),
  [cargo#14231](https://github.com/rust-lang/cargo/pull/14231) (refactor),
  [cargo#14234](https://github.com/rust-lang/cargo/pull/14234) (fix),
  [cargo#14239](https://github.com/rust-lang/cargo/pull/14239) (fix)
- [cargo#13589](https://github.com/rust-lang/cargo/pull/13589) (refactor),
  [cargo#13664](https://github.com/rust-lang/cargo/pull/13664) (fix),
  [cargo#13666](https://github.com/rust-lang/cargo/pull/13666) (refactor),
  [cargo#13693](https://github.com/rust-lang/cargo/pull/13693) (refactor),
  [cargo#13701](https://github.com/rust-lang/cargo/pull/13701) (refactor),
  [cargo#13713](https://github.com/rust-lang/cargo/pull/13713) (fix),
  [cargo#13849](https://github.com/rust-lang/cargo/pull/13849) (fix)
- [RFC 3502](https://github.com/rust-lang/rfcs/pull/3502) (T-cargo),
  [RFC 3503](https://github.com/rust-lang/rfcs/pull/3503) (T-lang)
