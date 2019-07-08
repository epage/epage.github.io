---
title: PyCon 2018 Trip Report
published_date: "2018-05-18 09:00:30 -0500"
data:
  tags:
    - programming
    - python
---
I went to with a coworker to [PyCon](https://us.pycon.org/2018/) this year ([videos](https://www.youtube.com/channel/UCsX05-2sVSH7Nx3zuk3NYuQ/videos)).

We had several conversations about the value gained by going.  Looking back, I
think in aggregate it was worth it.  While I didn't always attend good talks, I
got value from a lot of the little things:

*   Getting a pulse on the community
*   Getting more out of the talks than I would on youtube
    *   I wouldn't give as much focus / attention
    *   I would likely skip some of the ones I did get stuff out of
    *   Getting a feel from the buzz on which talks to watch
*   A chance to synthesize and process all these different ideas
    *   Seeing XArray
    *   Seeing more about property-based testing
*   Collaboration, like
    *   Talking with other companies about how they deal with problems
    *   Talking with Conda to clarify its differentiating features to know
	whether it is worth distributing my company's packages via Conda.
    *   Talking with Bloomburg to see if they are interested in collaborating
	on [ni/hightime](https://github.com/ni/hightime
	"https://github.com/ni/hightime")

Why didn't I get more out of the talks?  For me, a lot of sessions didn't hit
the right sweet spot of being informative.  They either went straight into the
deep end even when describing basics or only hovered in the superficial.
I did get to talk to a large variety of people that were beneficial by
volunteering, the hallway track, the expo floor, and sprints.
Overall, I think the trip was worth it but the value proposition is close to
the edge of not being worth it.

I'll bold important highlights or value I felt I got out of the conference

### Thursday: Tutorial and Sponsor Day

I did not expect the [convention center to be underground](https://cdnassets.hw.net/d6/30/de360e2b45c6881219d45504445a/cleveland-civic-promo.jpg) and so I was late enough that I skipped my first sponsored workshop (on Data Science).

I went to a sponsored machine learning workshop.  I've not looked too much into
this topic but am interested for the potential to analyse a large test suite
backlog.  I guess bug reports are another good candidate.  It didn't really
explain much about machine learning, which made me wonder if it was just a
Watson Studio ad.  It looked like Watson Studio was just a rebranded Jupyter; I
couldn't tell what was uniquely "watson".

What I did get out of the workshop was that hearing about sklearn
(SciKit-Learn).  They recommended it for good support of pipelining processes
so you can reuse models.  A lot of the libraries do things for us, like turning
bodies of text into matrices to analyze.  We'll need to consider what is the
right way to do that for tests.  Probably data scraping and analysis are good
candidates to look into.

I was going to go to a session on Python inside Google but it went through a
subject change and I decided to skip it.  At this point I volunteered to help
stuff swag bags.  **Certain volunteer opportunities are highly recommended for
conferences so you can more easily network.**  This effectively was speed
dating on, well, speed.  We had an assembly line going where people picked up
and dropped items into the bags of the people walking by.  We had disjointed
conversations with the bag holders and less disjointed with the rotating set of
people in line.  I'm still in touch with at least one person who is now going
to try to convert his lab to Python from a lab-proprietary programming language
due to our time in line.

Next was the opening reception.  They had food and the expo floor was open.
**This gave me a chance to talk to the Anaconda folks to clarify the role of
conda vs pip / PyPI which I've not been able to find a good answer for my
company's python packages**.  Conda is cross-platform, cross-language packaging
system.  It effectively creates a chroot to more easily build non-python code.
The main area I see this being useful is if we were to port from ctypes to
[CFFI w/ API
mode](https://cffi.readthedocs.io/en/latest/overview.html#abi-versus-api) which
is now the recommended way of doing C bindings.  CFFI is faster, especially for
PyPy, and tries to be easier by [parsing C declarations rather than trying to
figure out how they map to the ctypes
API](https://dbader.org/blog/python-cffi#intro).  CFFI w/ API mode builds a
cpyext on-the-fly, requiring a compiler toolchain setup.  I didn't recommend
this for our product's python APIs out of usability concerns for Windows
non-programmers.  Conda opens the door for us to do this but we'd still want to
distribute via PyPI, so we'd have to support both CFFI and ctypes.  This only
makes sense if we ever setup a good code-gen system for the lower layers of our
Python APIs.

At the opening reception, I also talked to [Sentry](https://sentry.io/welcome/
"https://sentry.io/welcome/").  I've come across them recently in another space
and was able to get more background.  They are basically a one-stop solution
for
[Crashpad](https://chromium.googlesource.com/crashpad/crashpad/+/master/README.md
"https://chromium.googlesource.com/crashpad/crashpad/+/master/README.md") and
[Socorro](https://github.com/mozilla-services/socorro
"https://github.com/mozilla-services/socorro") for many different languages and
environments.  Their server code and language libraries are all open source.

### Friday: Talks

Dan Callahan gave the keynote today.  He is a developer advocate at Mozilla.
I thought his observations were on point and gave me hope for this "I hate the
web" developer

*   Your platform determines the tools you can use.
*   The web is becoming the universal platform.
*   WebAsm is opening up the platform to more tools.

I then went to a talk applying art critique to code reviews (see also
[codecrit.com](http://codecrit.com)).  Unfortunately it stayed too abstract to
be useful but I like the general idea.  **We train artists to give and receive
feedback but mostly ignore it within software development**.  At people's
request, she held an Open Space to get it more concrete but I missed it for
some reason.  **Open Spaces are attendee driven workshops.**  They post a blank
schedule for unused rooms and people put up stickies for what they want to
host.  I feel I didn't utilize the open spaces enough.

There were three really promising testing talks at the same time.  My coworker went to
the one on testing legacy code bases, so I went to the one on alternatives to
normal unit tests.  The description mentioned property-based testing which is
something I've heard about but has been abstract and the examples are never
deep enough for me to see how it can apply to real code bases.  **This speaker
did a good job showing how property-based testing can be used to as unit-tests
on steroids and adding contracts gives you integration tests on steroids**.  I
also appreciate that **he setup a website to host the results of hallway-track
discussions on the topic**, see
[https://hillelwayne.com/talks/beyond-unit-tests/](https://hillelwayne.com/talks/beyond-unit-tests/).
For those unfamiliar with them, the butchered description is that
property-based testing is where you write a [parameterized
test](https://docs.pytest.org/en/latest/parametrize.html) but only describe the
test data to use rather than enumerating specific cases.  The framework usually
sweeps basic cases (like -20 to 20 for numbers), known corner case values
(MAX_INT for numbers, unicode for strings), and any cases that have previously
failed.  That last part blew me away.  There are issues of how does your CI
"remember" that data but overall the idea is awesome.

Next was a session on API Design.  I was hoping it was going to be like last
year's talk that led to [PyDev/Python Project
Guidelines](https://niweb.natinst.com/confluence/display/PyDev/Python+Project+Guidelines
"https://niweb.natinst.com/confluence/display/PyDev/Python+Project+Guidelines")
.  Unfortunately it was very basic stuff that I was familiar with.  I did love
the shoutout for semver!

I again skipped a session and ended up talking to Dan Callahan and another
fellow I talked to several more times throughout the conference.

I went to a PyTorch talk.  Unfortunately, even its introductory material
quickly jumped into jargon I've never heard of before.

I had a goal of getting a group together for dinner through networking.  That
kind of stuff seems harder at these big conferences because you have large
company or community clicks.  We went to my favorite [Thai
place](http://phothangcafe.com/) (my wife and I went first came here a year ago
on vacation).  We ended up running into a stranger from the conference and ate
with her.  She does testing of algorithms for DNA sequencers.

I then went to Avengers 4 on Twilio's dime.  They posted on twitter about
having tickets.  They weren't even mentioning them at the booth, confusing me
at first, but low and behold, when I asked, I got a ticket from a fat stack.
**Twitter is essential at conferences for spontaneous opportunities to network
(and not just for the free swag)**.  There were people organizing groups for
dinner, bar hopping, cycling, rock climbing, etc.  Other people were giving
updates on talks in other rooms or highlighting amazing speakers you should go
see (like Raymond Hettinger who I sadly missed), etc.

### Saturday: Talks

We did lightning talks and then keynotes.  The keynotes were awesome,
especially Q who is a Detroit librarian who invests herself in what she gets
into.  She was asked to teach a programming class and that led to her down a
path of becoming a keynote speaker.  In general her talk was a great look at
tech education but it also dived into the problems in overcoming diversity.

First off, I went to a talk on distributed Jupyter applications.  I wanted to
go to this one since it was the type of data analysis our customers would be
doing.  It was doing data analysis on all of the data describing our oceans.
**For me, the key takeaways were that there is a library, XArray, that is meant
to work with correlated multi-dimensional data along with an associated file
format, netcdf.**  This seems suited to mixed measurement applications that
seem common in DAQmx and would be valuable to look at for future API features
for our DAQmx python package.

Dustin's "Inside the Cheese shop" talk was mostly a historical look at
packaging with little value for today.  I can understand why there is
[frustration](https://www.reddit.com/r/Python/comments/8jd6aq/why_is_pipenv_the_recommended_packaging_tool_by/?st=jh80j5mf&sh=4b172235
"https://www.reddit.com/r/Python/comments/8jd6aq/why_is_pipenv_the_recommended_packaging_tool_by/?st=jh80j5mf&sh=4b172235")
in the community over packaging when he came across as saying "look how much
better things are today, you should be grateful! Its easy! Look at our 20 page
long document on how to do it all!" and not really admitting to how bad the
ecosystem is. I'm still excited for [poetry](https://poetry.eustace.io/
"https://poetry.eustace.io/") as the solution for library development.

I went to the open space but it was too crowded; there wasn't much for me to
do.  I did remember I wanted to try to find out code-gen best practices in the
community.  Some projects do runtime codegen, creating classes and functions on
the fly.  I don't like that for our product's python packages because it makes the code
harder to read / follow.  So do we code-gen in `setup.py`?  Do we use an
external build tool like WAF to code-gen and package in one step?  Do we
code-gen and check the result into SCM, having a CI step ensure there aren't
hand edits?  Sadly, I couldn't find someone there who would know but I was
recommended to talk to the Ansible folks.  In a later talk, I also got the idea
to talk to Facebook.  Unfortunately, by the time I went to go do either, the
Expo floor was closed and its hard to pick people out of a crowd for what
company they work for.

With Lunch, I lost track of the VIM open space and was late enough to not get anything from it.

**Someone gave a postmortem on a major failure in the efforts in the transatlantic telegraph system.  This was a very good talk with a very engaging speaker.**

"You're an Expert, teach like one" was good.  There was another that summarized
lots of research on failure and resiliency ([bibliography](/blog/-
https:/www.zotero.org/groups/287407/failure/items)).  See
[twitter](https://twitter.com/vincesalvino/status/995389852395954176?s=15) for
a good quote on seniority from that.

I went to a talk from Facebook on the transition to Python 3.  Besides
realizing the speaker would be a good candidate to talk to about code-gen, I
got a great snippet of wisdom from it.  **"Educate for the future you want and
not the present you have"**.  Once he laid the foundation for Python3, they
switched the training course to Python 3 and that led to a major uptick in
their conversion.

### Sunday

I ended up skipping today.  It was mostly posters and job fair anyways.  My
coworker did report back about a really good talk about static analysis.  **It
seemed to get him interested in type annotations / mypy.**  Seeing the buzz and
Mark's excitement was a reassurance I made the right decision in leading the
charge to adopt mypy.  Back when we started on Python APIs, I saw the
development of mypy and felt that it was going to become something big and we
should consider it for the future if we don't start using it now.  This
confirms my prognostications.  Interesting enough, my interest wasn't as
strongly on the primary use case, code validation.  I don't see many of our
customers doing big engineer projects in Python where they'd know of and use
mypy.  My focus was on the improved value for IDEs to help our customers use
our APIs.  Recently, others in the Python community have started to see the
value of this, for example [The other (great) benefit of Python type
annotations](https://medium.com/@shamir.stav_83310/the-other-great-benefit-of-python-type-annotations-896c7d077c6b).

### Monday: Sprint

This week is a hackathon for various projects.  I sat with the PyLint folks and
looked at if the various forms of static analysis have programmatic APIs which
would be useful for a static analysis aggregator  I wrote have.  I want to
embed the tools rather than spawn them to make it easier to use with
PyInstaller.  PyLint seemed to be fine but I did create a [PR for
scspell](https://github.com/myint/scspell/pull/30).

Storms in Chicago caused my coworker problems on his way in; they caused me
problems on my way out.  My flight was canceled.  They were able to get me on
another one though.

### Misc

Recommended talks

*   [Dan Callahan - Keynote - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=ITksU31c1WY "https://www.youtube.com/watch?v=ITksU31c1WY")
*   [Saturday Morning Lightning Talks - PyCon 2018 - YouTube](https://youtu.be/VJ0vibC_Hl0 "https://youtu.be/VJ0vibC_Hl0")  especially Q's.
*   [Hillel Wayne - Beyond Unit Tests: Taking Your Testing to the Next Level - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=MYucYon2-lk "https://www.youtube.com/watch?v=MYucYon2-lk")
*   [Barry Warsaw - Get your resources faster, with importlib.resources - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=ZsGFU2qh73E "https://www.youtube.com/watch?v=ZsGFU2qh73E")
*   [VM (Vicky) Brasseur - The human nature of failure & resiliency - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=h-38HZqanJs "https://www.youtube.com/watch?v=h-38HZqanJs")
*   [Lilly Ryan - Don't Look Back in Anger: Wildman Whitehouse and the Great Failure of 1858 - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=GuNoaAFnTPg "https://www.youtube.com/watch?v=GuNoaAFnTPg")
*   [Shannon Turner - You're an expert. Here's how to teach like one. - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=3m555yWTaNI "https://www.youtube.com/watch?v=3m555yWTaNI")

Some I know I want to go back and watch

*   For the speaker:
    *   [Raymond Hettinger - Dataclasses: The code generator to end all code generators - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=T-TwcmT6Rcw "https://www.youtube.com/watch?v=T-TwcmT6Rcw")
    *   [David Beazley - Reinventing the Parser Generator - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=zJ9z6Ge-vXs "https://www.youtube.com/watch?v=zJ9z6Ge-vXs")
    *   [Ned Batchelder - Big-O: How Code Slows as Data Grows - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=duvZ-2UK0fc "https://www.youtube.com/watch?v=duvZ-2UK0fc")
*   For the buzz
    *   [Larry Hastings - Solve Your Problem With Sloppy Python - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=Jd8ulMb6_ls "https://www.youtube.com/watch?v=Jd8ulMb6_ls")
    *   [Mario Corchero - Effortless Logging: A deep dive into the logging module - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=Pbz1fo7KlGg "https://www.youtube.com/watch?v=Pbz1fo7KlGg")
    *   [Pieter Hooimeijer - Types, Deeper Static Analysis, and you - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=hWV8t494N88 "https://www.youtube.com/watch?v=hWV8t494N88")
        *   or was it  [Kyle Knapp - Automating Code Quality - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=G1lDk_WKXvY "https://www.youtube.com/watch?v=G1lDk_WKXvY")
    *   [Kenneth Reitz - Pipenv: The Future of Python Dependency Management - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=GBQAKldqgZs "https://www.youtube.com/watch?v=GBQAKldqgZs")
        *   Still fairly [controversial](https://www.reddit.com/r/Python/comments/8jd6aq/why_is_pipenv_the_recommended_packaging_tool_by/?st=jh80j5mf&sh=4b172235 "https://www.reddit.com/r/Python/comments/8jd6aq/why_is_pipenv_the_recommended_packaging_tool_by/?st=jh80j5mf&sh=4b172235")
    *   [Nathaniel J. Smith - Trio: Async concurrency for mere mortals - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=oLkfnc_UMcE "https://www.youtube.com/watch?v=oLkfnc_UMcE")
    *   [Code like an accountant: Designing data systems for accuracy, resilience and auditability - YouTube](https://www.youtube.com/watch?v=svcBO-OjYfM "https://www.youtube.com/watch?v=svcBO-OjYfM")
*   For possible customer impact
    *   [Jake VanderPlas - Exploratory Data Visualization with Vega, Vega-Lite, and Altair - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=ms29ZPUKxbU "https://www.youtube.com/watch?v=ms29ZPUKxbU")
*   Curiosity
    *   [Daniel Pyrathon - A practical guide to Singular Value Decomposition in Python - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=d7iIb_XVkZs "https://www.youtube.com/watch?v=d7iIb_XVkZs")
    *   [Sara Packman - The Journey Over the Intermediate Gap - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=49CIIu1XkIE "https://www.youtube.com/watch?v=49CIIu1XkIE")
    *   [Carol Willing - Practical Sphinx - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=0ROZRNZkPS8 "https://www.youtube.com/watch?v=0ROZRNZkPS8")
    *   [Valery Calderon - Reactive Programming with RxPy - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=I5AZ3rTR4Wc "https://www.youtube.com/watch?v=I5AZ3rTR4Wc")
    *   [Nina Zakharenko - Elegant Solutions For Everyday Python Problems - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=WiQqqB9MlkA "https://www.youtube.com/watch?v=WiQqqB9MlkA")
    *   [Joyce Jang - Build Teams as an Engineer - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=-9NpTeddWds "https://www.youtube.com/watch?v=-9NpTeddWds")
*   Because Rust: [vigneshwer dhinakaran - Pumping up Python modules using Rust - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=UYpWVfTng4s "https://www.youtube.com/watch?v=UYpWVfTng4s")
*   To get a better feel on the community and static types
    *   [Carl Meyer - Type-checked Python in the real world - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=pMgmKJyWKn8 "https://www.youtube.com/watch?v=pMgmKJyWKn8")
    *   [Greg Price - Clearer Code at Scale: Static Types at Zulip and Dropbox - PyCon 2018 - YouTube](https://www.youtube.com/watch?v=0c46YHS3RY8 "https://www.youtube.com/watch?v=0c46YHS3RY8")
