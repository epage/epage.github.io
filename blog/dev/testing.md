---
title: Testing
---

Consider the [purpose of the test][Why Am I Writing This Test]:
- Using Tests to Specify the Requirements of the System
- Using Tests to Document the System
- Using Tests to Build Confidence in the System

To integrate:
- [073: Driven By Need, Guided By Example with Dan North](https://www.greaterthancode.com/driven-by-need-guided-by-example)

Property-based testing
- Great intro: [video](https://www.youtube.com/watch?v=MYucYon2-lk) [slides](https://speakerdeck.com/pycon2018/hillel-wayne-beyond-unit-tests-taking-your-testing-to-the-next-level?slide=41)
[ [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)

[Why Am I Writing This Test]: https://keyholesoftware.com/2018/04/16/why-am-i-writing-this-test/


Testing is a trade off.  Test coverage helps deliver a higher quality product and helps in the future with understanding and safe evolution.  They come at a cost though.
- Tests can increase the burden for making a change
  - The more tests, the more code you need to update when things change.
  - Tests more tightly coupled to code’s implementation (in contrast to requirements) means it is more likely you’ll break the tests as code evolves and need to update them.
  - Doesn’t just make changes more expensive but has a psychological impact, making people less likely to change code when they see a problem.
- There is momentum to submitted tests.  It takes quite a bit of pain before one person feels comfortable removing the test, partially for justifying it to themself, partially because they feel the need to justify it to reviewers, and partially because they feel responsible for replacing it with the right tests.  We’ve seen a recent example of this with revision-comment-hook’s itests.

The trade off between these is not universal.  NASA has different needs than a script I write for a one-time refactor of code.

For general development, I recommend reasonable coverage that instills confidence but doesn’t get in the way.  Testing beyond this is YAGNI and should be handled as issues crop up.  This is vague and has wiggle room.  Experienced engineers can look at two pieces of code and the associated tests and decide differently on what is a sufficient level of testing.  However, examples of cases that I feel hurt us more than help us
- Validating human-formatted output unless it's mission critical that it follows specific guidelines at which point you only validate those guidelines.
- Writing tests for tests.  If test logic is getting complicated enough to test, we should probably re-think things.  This is not excluding [mutation testing](https://en.wikipedia.org/wiki/Mutation_testing) which checks the quality of tests.
- Testing assertions.  This is a subcategory of testing tests.  Even sqlite, with some of the most stringent testing, [does not test asserts](https://www.sqlite.org/testing.html#coverage_testing_of_defensive_code).
- Testing bugs.  This is, by definition, a test we want to break.  Fixing bugs is also one of the unquestionably best places to do TDD because the requirements are most clearly scoped.  However, prematurely putting those tests into the code is a violation of YAGNI.  It is a burden we are adding to the code base without benefit until the day we might intentionally fix it.  The burden includes both more tests to update, extra noise in using tests as documentation, and possibly people misunderstanding the intent of the test and thinking its a requirement.
- Monkey patching in mocks: In most cases, you are tying yourself to the implementation.  For example, to avoid the filesystem, you might monkeypatch “os.path.exists” but then the tests might start failing in weird ways (or worse, silently succeed with lost coverage) when switching to pathlib’s “path.exists()”.  Ideally, we decouple integration points by making a higher level abstraction over them that the user wires together.  Sometimes that isn’t ergonomic enough, so we compromise by making the dependency injection optional or go as far as monkey patching the dependency.

