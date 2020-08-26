---
title: Testing
---

# Goals of Testing

Consider the [purpose of the test][Why Am I Writing This Test]:
- Using Tests to Specify the Requirements of the System
- Using Tests to Document the System
- Using Tests to Build Confidence in the System

## Specifying Requirements

> In most businesses, the only tests that have business value are those that are derived from business requirements. Most unit tests are derived from programmers' fantasies about how the function should work: their hopes, stereotypes, or sometimes wishes about how things should go. Those have no provable value. There were methodologies in the 1970s and 1980s based on traceability that tried to reduce system requirements all the way down to the unit level. In general, that's an NP-hard problem (unless you are doing pure procedural decomposition) so I'm very skeptical of anyone who says they can do that. So one question to ask about every test is: If this test fails, what business requirement is compromised? Most of the time, the answer is, "I don't know." If you don't know the value of the test, then the test theoretically could have zero business value. The test does have a cost: maintenance, computing time, administration, and so forth. That means the test could have net negative value

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)

> Consider whether the bulk of your unit tests should be those that test key algorithms for which there is a “third-party” oracle for success, rather than one created by the same team that writes the code. “Success” here should reflect a business mandate rather than, say, the opinion of a team member called “tester” whose opinion is valued only because it is independent. Of course, an independent evaluation perspective is also important. 

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)

A limitation with meeting this purpose is knowing your requirements.

In parsing a specified format, you know the quantitative requirements up front and you know what tests need to be written.

Most of the time, I find that my programming is exploratory.
- End-user requirements can be vague and require iteration to discover.
- Some requirements aren't real requirements.  How many times have the goal
  posts of Minimal Viable Product (MVP) shifted for you?  When
  implementing something you discover roadblocks and propose alternate
  "requirements" that meet still meet the user's need but with different trade
  offs.
- We need to be cautious of not conflating behavior with requirements through
  our testing.
- As a developer, we break end-user requirements down into unit requirements.
  These requirements are more fluid in details and location.

## Documenting the System

A benefit of tests over written documentation is it forces you to keep it up-to-date.

Some challenges:
- Tests need to be clean and to the point.  Too many test-focused abstractions
  or [DRY](code-reuse) can obscure communicating how the system behaves.
- Testing corner cases can get gnarly.  Consider organizing your tests so the
  most important details are immediately available and people can then dive
  deeper as needed.

A big help here is when the language supports "doc tests", verifiable example
code in documentation.

## Building Confidence

A consideration is how much confidence do you need in the application or a
specific section being developed. One-off scripts have a different confidence
bar than products like the one I worked on that one time shipped with CDs
warning that a known bug cause death or dismemberment.

## Regressions

> Note that there are some units and some tests for which there is a clear answer to the business value question. One such set of tests is regression tests; however, those rarely are written at the unit level but rather at the system level. We know what bug will come back if a regression test fails — by construction. Also, some systems have key algorithms — like network routing algorithms — that are testable against a single API. There is a formal oracle for deriving the tests for such APIs, as I said above. So those unit tests have value.

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)

# Trade-offs in Testing

Testing is a trade off.  Test coverage helps deliver a higher quality product
and helps in the future with understanding and safe evolution.  They come at a
cost though.
- Tests can increase the burden for making a change
  - The more tests, the more code you need to update when things change.
  - Tests more tightly coupled to code’s implementation (in contrast to
    requirements) means it is more likely you’ll break the tests as code
    evolves and need to update them.
  - Doesn't just make changes more expensive but has a psychological impact,
    making people less likely to change code when they see a problem.
- There is momentum to submitted tests.  It takes quite a bit of pain before
  one person feels comfortable removing the test, partially for justifying it
  to themself, partially because they feel the need to justify it to reviewers,
  and partially because they feel responsible for replacing it with the right
  tests.
- There is opportunity cost: what else could we have done rather than write the tests?

> The point is that code is part of your system architecture. Tests are modules. That one doesn’t deliver the tests doesn’t relieve one of the design and maintenance liabilities that come with more modules. 
> ...
> To make things worse, you’ve introduced coupling — coordinated change — between each module and the tests that go along with it. You need to thing of tests as system modules as well. That you remove them before you ship doesn’t change their maintenance behavior. 

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)

>  The tests are code. Developers write code. When developers write code they insert about three system-affecting bugs per thousand lines of code. If we randomly seed my client’s code base — which includes the tests — with such bugs, we find that the tests will hold the code to an incorrect result more often than a genuine bug will cause the code to fail!

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)


The trade off between these is not universal.  NASA has different needs than a
script I write for a one-time refactor of code.

A lot of this comes down to risk.

> If you cannot tell how a unit test failure contributes to product risk, you should evaluate whether to throw the test away. There are better techniques to attack quality lapses in the absence of formal correctness criteria, such as exploratory testing and Monte Carlo techniques. (Those are great and I view them as being in a category separate from what I am addressing here.) Don’t use unit tests for such validation.

- James O Coplien, ["Why Most Unit Testing is Waste"](https://rbcs-us.com/documents/Why-Most-Unit-Testing-is-Waste.pdf)

For general development, I recommend reasonable coverage that instills
confidence but doesn't get in the way.  Testing beyond this is YAGNI and should
be handled as issues crop up.  This is vague and has wiggle room.  Experienced
engineers can look at two pieces of code and the associated tests and decide
differently on what is a sufficient level of testing.  However, examples of
cases that I feel hurt us more than help us
- Validating human-formatted output unless it's mission critical that it
  follows specific guidelines at which point you only validate those
  guidelines.
- Writing tests for tests.  If test logic is getting complicated enough to
  test, we should probably re-think things.  This is not excluding [mutation
  testing](https://en.wikipedia.org/wiki/Mutation_testing) which checks the
  quality of tests.
- Testing assertions.  This is a subcategory of testing tests.  Even sqlite,
  with some of the most stringent testing, [does not test
  asserts](https://www.sqlite.org/testing.html#coverage_testing_of_defensive_code).
- Testing bugs.  This is, by definition, a test we want to break.  Fixing bugs
  is also one of the unquestionably best places to do TDD because the
  requirements are most clearly scoped.  However, prematurely putting those
  tests into the code is a violation of YAGNI.  For example, not all bugs are
  prioritized for fixing.  It is a burden we are adding to the code base
  without benefit until the day we might intentionally fix it.  The burden
  includes both more tests to update, extra noise in using tests as
  documentation, and possibly people misunderstanding the intent of the test
  and thinking its a requirement.
- Monkey patching in mocks: In most cases, you are tying yourself to the
  implementation.  For example, to avoid the filesystem, you might monkeypatch
  “os.path.exists” but then the tests might start failing in weird ways (or
  worse, silently succeed with lost coverage) when switching to pathlib’s
  “path.exists()”.  Ideally, we decouple integration points by making a higher
  level abstraction over them that the user wires together.  Sometimes that
  isn’t ergonomic enough, so we compromise by making the dependency injection
  optional or go as far as monkey patching the dependency.

# Testing methodologies

There are many ways to meet the above stated purposes.  Experiment with them in
extreme scenarios to explore them and when they help and where they break down.

When experimenting, some common considerations include:
- How easy is it to pin point a problem?
- What problems do you have a hard time finding?
- How does it improve your code's design?
- When might it hurt you code's design?
- How well does the test methodology support you in meeting the purpose of the test you are writing?  i.e. some might work better than others.

## Manual tests

I've seen two approaches
- Scripted: basically "you haven't automated this yet"
- Free-form: requires more product knowledge which developers might not always
  have but stresses the system in unexpected ways, finding bugs. The cost is
  pretty high thought.

## End-to-End tests

The closer to the user you test, it is more likely you have testing
requirements that matter (rather than book keeping).  However, it becomes more
expensive and there can be an exponential number of combinations to test.
Testing in smaller chunks than End-to-End means you can test more combinations
without going exponential.

## Integration Tests

## Unit Tests

An important question to ask is "what is a unit?".

Designs to make code more testable:
- Functional-core, Imperative-shell: Functional code is easy to test because
  its purely about inputs/outputs so you use this for your algorithmic business
  logic while keeping the integrations in a more imperative style.
- Hexagonal architecture or Ports and Adapters:  Create abstractions around your external systems that are easier to mock.
  - Some APIs are hard to mock.  For example, file systems are really APIs around "slow", global state.
  - Some APIs have a large surface, like file systems, Redis, etc.
  - Applications rarely rely on the details of the interaction.  For example,
    mocking enough file system calls (correctly) to support a file-lock would
    be burdensome but a mock of a higher level abstraction, like a
    configuration API, could get away with a Mutex or no lock at all.

## Static typing

Marker types: some view "goto fail" as a failure of the mismatch between
language formatting and behavior (whether the answer is only-explicit blocks or
whitespace-blocks) but Joe Birr-Pixton [presented][rustls: modern, faster,
safer TLS: goto fail] on how absence of failure is a poor indication of
validity.  A lot of applications or APIs embed "create, validate, perform" in
them where you have to audit to make sure the "validate" step takes place.
Instead if you embed your state into the types themselves, the only way to
create a `ValidatedType` is to go through the `validate` constructor which
focuses your auditing on just that one point of failure and you trust in the
type system to enforce the policy everywhere else.

## Linters

Generally linters try to infer higher level intent from your code by analyzing
the source.  A lot of bugs will still get missed and a lot of valid code will
be flagged.

Strategies for adopting static analysis in an existing code base:
- Have a stricter IDE-detected configuration than the CI enforces. This pushes
  people in the right direction to reduce the backlog.
- Quarantine existing lints into your backlog and have the CI prevent new lints
  from being introduced.

For false-positives, your options are:
- Play whack-a-mole by adding "ignores directives" in the code
- Warn (in IDE) but don't error (in CI).

## Proof systems

Unfortunately, I do not yet have experience with these.

# TODO

To integrate:
- [073: Driven By Need, Guided By Example with Dan North](https://www.greaterthancode.com/driven-by-need-guided-by-example)

Property-based testing
- Great intro: [video](https://www.youtube.com/watch?v=MYucYon2-lk) [slides](https://speakerdeck.com/pycon2018/hillel-wayne-beyond-unit-tests-taking-your-testing-to-the-next-level?slide=41)
[ [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)

[Why Am I Writing This Test]: https://keyholesoftware.com/2018/04/16/why-am-i-writing-this-test/
[rustls: modern, faster, safer TLS: goto fail]: https://youtu.be/aHMRFZkXq4Y?list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1&t=1257
