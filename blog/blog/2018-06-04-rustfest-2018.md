---
title: RustFest Parist Trip Report
published_date: "2018-06-04 09:00:30 -0500"
data:
  tags:
    - programming
    - rust
---
I've been involved in the Rust community for about a year and a half now.  What
attracted me to Rust is that is looks like the first viable replacement for
C++.  It offers similar (actually better) protections than GCed languages and
the full language is available in any environment, including exception-like
error handling in the Windows kernel.

<!-- more -->

1aim partially sponsored my attendance at RustFest.  At the start of the year,
the Rust community organized working groups for targeted improvements to the
language for [several use
cases](https://internals.rust-lang.org/t/announcing-the-2018-domain-working-groups/6737).
1aim was looking for working group members to sponsor and the WG-CLI leader
sponsored me due to my involvement with the working group where I've been
working to make my life easier for my maintaining my tools.

Sadly, I'm writing this up so late from the main event (the talks), that I'm
not going into as much detail as I'd like.

### Talks (Saturday)

The opening Keynote was on the [Feynman Method for
learning](https://www.youtube.com/watch?v=23lRkdDXqY0&index=1&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1).
Really enjoyed this.  Probably the most memorable part is dealing with jargon.
I've heard before that Elon Musk has a ban on acronyms and proprietary jargon

> "Don't use acronyms or nonsense words for objects, software, or processes at
> Tesla. In general, anything that requires an explanation inhibits
> communication. We don't want people to have to memorize a glossary just to
> function at Tesla."

Feynman articulately calls out the real problem with jargon in this quote that
Vaidehi Joshi shares:

> When we speak without jargon, it frees us from hiding behind knowledge we
> don't have.

Next was a talk on a Rust implementation of TLS / SSL.  Key parts of this talk

*   [Leveraging the type system to avoid GOTO FAIL](https://www.youtube.com/watch?v=aHMRFZkXq4Y&feature=youtu.be&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1&t=1250)
    *   While the focus is on Rust, languages with weaker type systems can
        still get some benefit from these practices.
*   Apparently, there is a OpenSSL wrapper around rustls.  OpenSSL is a big
    problem for making "portable" pythons for Linux for nibuild.  I wonder if
    we can leverage this.

Later was a talk on [writing reliable systems](https://www.youtube.com/watch?v=hMJEPWcSD8w&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1&index=8).  Highlights

*   Calls out challenges we have in manually identifying corners that need testing.
    *   Pesticide paradox: code becomes immune to tests
    *   Besides being good in general, provides good reason to be using property-based testing
*   Good to see more background on property-based testing
*   Model testing seems intriguing for complex driver interactions at my work.
    *   It randomly generates ops to perform on your code.  If it comes across a failure case, it will randomly drop ops to find the minimal reproducible situation.
    *   There was talk of linearizibility.  I don't remember the details on this but had a note that it could help with testing some of our concurrent code.

The [Monotron
talk](https://www.youtube.com/watch?v=pTEYqpcQ6lg&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1&index=11)
is a great showcase at the potential for Rust for firmware.

Misc talk notes

*   [RustFest Paris 2018 - Immutable Data Structures and why You want them by Bodil Stokke - YouTube](https://www.youtube.com/watch?v=Gfe-JKn7G0I&index=3&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1 "https://www.youtube.com/watch?v=Gfe-JKn7G0I&index=3&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1")  was enjoyable but not as much immediate use
*   [RustFest Paris 2018 - A Rust Crate that also quacks like a modern C++ Library by Henri Sivonen - YouTube](https://www.youtube.com/watch?v=Ct7jveV7j8g&index=5&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1 "https://www.youtube.com/watch?v=Ct7jveV7j8g&index=5&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1")  has potential value to us but I fell asleep in due to my hotel roommate coming in late the night before.
*   [RustFest Paris 2018 - Vector graphics rendering on the GPU in Rust with Lyon by Nicolas Silva - YouTube](https://www.youtube.com/watch?v=2Ng5kpDirDI&index=7&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1 "https://www.youtube.com/watch?v=2Ng5kpDirDI&index=7&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1")  shows some impressive performance work.
*   [RustFest Paris 2018 - Bringing Intel SGX to the Rust Ecosystem by Yu Ding - YouTube](https://www.youtube.com/watch?v=8IvWPeavjiQ&index=9&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1 "https://www.youtube.com/watch?v=8IvWPeavjiQ&index=9&list=PL85XCvVPmGQgdqz9kz6qH3SI_hp7Zb4s1")  was interesting but unsure how much SGX will be relevant to me or my coworkers.

### Workshops (Sunday)

I skipped the workshops.

### Impl Days (Monday, Tuesday)

This was a two day hack fest.  I liked this better than the PyCon one.  I'm
unsure if its because I'm more involved in the community or they handle it
better with

*   Clearer mentorship model
*   Use-case driven cross-ecosystem initiatives making it easier for people to
    find problems relevant to them which makes them more invested.

I worked on one of my WG-CLI projects, CLI testing.  The crates (packages) provide

*   ability to find and launch you executable
*   run asserts on the result of your executable
*   filesystem fixtures
*   filesystem assertions

### Visiting Paris (Thursday, Wednesday)

This was my first time to Paris.

Misc notes

*   Went on the Arc de'Triomphe
*   Enjoyed a walked along the Seine River
*   Visited [Pere Lachaise cemetery](https://photos.app.goo.gl/SOus5YTJySi0JSSO2)
*   While nothing to special about duck or escargot, the sauces they came in were great!
*   The city is dirty
*   Everything felt one or two turns away.  I'm unsure if this was because of where my hotel was or all the weird angles all the streets are in.
*   Every mode of transport is used (train, subway, car, motorcycle, bicycle, scooter, electric scooter, those one wheeled segways) and the roads / sidewalks are chaos.  Anyone who could was scooting around stopped cars in traffic.
*   Experienced the stereotype of french customer service
    *   Got the ire of a bar tender for being the sober person in a group
    *   Despite my flight being empty (as I found out later), they enforced
        unusual baggage restrictions.  After waiting in line a while for
        checking my carry on, all of the Air France staff disappeared (I flew
        KLM to Paris).  Eventually they came back and I was served.  The
        gentleman got very frustrated and deposited me in another line without
        telling me why I was in the wrong line.  The gatekeeper of this new
        line kept telling me I was in the wrong line and needed to go back.
        Thankfully ignoring him was the right thing to do.
*   Should have brought a better umbrella.  It rained nearly every day and now my passport is messed up.  The Air France guy sure made his thoughts on my passport known.
