---
title: Monorepos
---

A monorepo is a SCM repo that contains multiple components.  Some famous
monorepos hold an entire company's code in it; not all do.  Google's monorepo
does not contain Android, Chrome, or several other major products.  A monorepo
can be as simple as ripgrep where [several dependencies are developed in the
same repo](https://github.com/BurntSushi/ripgrep/blob/master/Cargo.toml#L33).

Basic assumptions about a monorepo
- Everything is versioned together
- Third party dependencies are vendored
- Single version of a third party dependency across the monorepo
- Build and test system is in place to provide confidence when making monorepo-wide refactors

For more background, see [Trunk Based Development](https://trunkbaseddevelopment.com/).

Different scopes of monorepos can include:
- An entire company, including unrelated code
- Coupled products
- Individual products
- Independent sections of a product, like the core vs plugins
- Not using one at all

## Considerations for the right scope

### Release schedule

- For some products, the business impact of adjusting the release date is small
  and they can align on a coordinated monorepo release train if its frequent enough
  (e.g. once a quarter)
- Products that have limited purchasing windows, like consumer holidays or academia
  need exceptions to the release train.

### Release schedule with shared shipping code

- If A depends on B and B has a breaking change, great, A gets updated
  - Ideally everything is always shippable but that isn't always the case.
  - So what if A needs to ship but B isn't ready, then A needs to release with an old B

### Supporting older versions of dependencies

Let's say that product A interoperates with B but we don't want to force users to
upgrade B, then A might need to build against old versions of B.  This violates
the principle that monorepos should align in their dependencies, especially if
its something from within the monorepo.

### Protected IP

Not everyone in a company has the same access rights to all IP, whether out of
paranoia or due to export-restrictions.

- git repo access rights are all or nothing
- How do you handle protected IP?
- How do you handle IP that becomes protected?

### Conversion and maintenance cost

- How trivially fast is the build and tests?
- Is your company built on a cloud computing platform and have the resources to
  scale up your build and tests with it (like Google and Microft)?  Including:
  - Bubble up of mono repo
  - Ideally identify when cached versions can be reused
  - Does it need to support native builds for platforms that major monobuild tools don't support?
  - The cost of migrating to any new tools needed for using a monobuild.

### Dev Efficiency

- Monorepo's impact on culture
  - Some part of culture are embedded in a monorepo's policies
  - Culture alignment can improve efficiency
  - Are there negative aspects of a mono-culture though? Do all practices work for all teams?
- For software that can't always be shippable, submissions might get throttled during stablization periods but now everyone is on the same stablization schedule, causing other teams to hold your product hostage.
- Cost of change has an impact on culture, including autonomy and empowerment
  - e.g. With monorepo, can find and update all clients and fix them
    - But required to do it in one pass
    - But required to do it to code you don't know
    - But merge conflicts can become unwieldy
- Third party dependencies being kept in lock step increases cost of upgrading, lower the advantage gained by new features / capabilities
  - "For third-party dependencies, the same rule applies, everyone upgrades in lock-step. Problems can ensure, of course, if there are real reasons for team B to not upgrade and team A was insistent. Broken backward compatibility is one problem."
  - "In 2007, Google tried to upgrade their JUnit from 3.8.x to 4.x and struggled as there was a subtle backward incompatibility in a small percentage of their usages of it. The change-set became very large, and struggled to keep up with the rate developers were adding tests."
  - \- [Monorepos - Trunk Based Development](https://trunkbaseddevelopment.com/monorepos/#third-party-dependencies-1)
- Quality of validation vs turn around time
  - Ideal: full build + test suite run on every PR and commit
    - Would take too long for some code bases; we'd need to find ways to skip building parts and identify pertinent tests
    - Might still take too long
  - Different products might have different bars on quality
    - Either due to how mission critical the software is
    - Or due to the cost of running all tests
  - Important dev efficiency metrics:
    - Time from PR approved by human reviewers to different levels of verification
    - Time from PR approved by human reviewers to being available for hand-testing
  - Barriers
    - Scaling test suites involving hardware requires more hardware and more maintenance
- SCM's ability to have a performant interactive experience is important
  - Slow CLI tools can break developers out of the zone, making them more easily distracted
  - Need to ensure commands are fast, especially ones used in the background by text editors / IDEs
  - Network issues are a major contributing factor, especially for remote developers and remote branches.
