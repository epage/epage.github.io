---
title: Code Reuse
---

<center>[<img src="https://assets.amuniversal.com/5d2d9c209fd5012f2fe600163e41dd5b" width=900m/>](https://dilbert.com/strip/1996-01-31)</center>

Several years ago, I was working with an engineer fresh out of college.  He was rightfully excited to start using all of the best practices he had learned in college, including Don't Repeat Yourself (DRY).  The problem is college is rarely a time where you see when best practices work and when they don't.  When does a college student ever get to reuse their code?  I tried to explain the dangers of DRY but I think instead I came across as the old curmudgeon who didn't want anything to do with these new fangled ways of making code better.

Over the years, I keep having to repeat myself on how best to reuse code, so I figured it was time practice DRY with my advice.

Guidelines for code reuse, in priority order:

- When reducing duplication, focus on reducing abstraction duplication and not implementation duplication.
  - The requirements that drove a shared implementation can evolve over time, causing the implementations to diverge.  When you share these kind of implementations, you either then need to undo your work or add "just one more" toggle to control things, leading to an unholy mess of spaghetti code
  - When focusing on the abstraction / requirements, you are identifying areas of shared code that is used together, or not. It is less likely to diverge.
  - My guidance on this is to wait until you have at least three implementations of something before trying to refactor for reuse because at that point you might be starting to understand the requirements (commonly called [The Rule of Three][I Dry-Ed Up My Code and Now It's Hard to Work With. What Happened].
- Composition over (implementation) inheritance
  - Inheritance is the second highest form of code coupling in C++, just behind "friend"
    - This makes it hard to ensure invariants are being satisfied in an objects API
    - This makes it hard to avoid spaghetti code where an implementation of the API jumps back and forth between a base class and the classes that inherit from it
- Weigh the value of cost of reuse
  - Just like computers have locality issues (instruction locality improves execution time, memory locality improves memory access), humans have code locality when reading / analyzing code.
    - If a minor section of code gets pulled away into a remote component, it will make it harder for people to read and reason about the code.
    - The better the abstraction, the less this is a problem.

[I Dry-Ed Up My Code and Now It's Hard to Work With. What Happened]: https://www.justinweiss.com/articles/i-dry-ed-up-my-code-and-now-its-hard-to-work-with-what-happened/
