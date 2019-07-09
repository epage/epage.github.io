---
title: Curating Commit History
---

Programming is often an exploratory exercise where, for example, one might not
know what a refactor should look like until the driving feature is done. This
leads to a messy commit history.

Git offers tools to create a clean history upfront or to fix it up later.

## What does an ideal history look like

- Every commit does exactly one thing.
- Every commit is buildable.
- Every commit message has a meaningful subject line
- Every commit message establishes context (what, why, and how)
- Commits are ordered to tell a story of how a feature was implemented.
- The commit graph is linear (and not [spaghetti][Bad commit graph]).

## Why keep a clean history

- Make it easy for someone to scan the history, parsing it for what they need.
- Help future developers know the design assumptions rather than them rediscovering them in a costly way.
- Avoid false positives with `git bisect`.
- Allow `git bisect` and `git blame` to more directly point to the desired change.

## What keeps us from the ideal history

> In theory, there is no difference between theory and practice. But, in practice, there is.

\- Jan L. A. van de Snepscheut

- [History should not be rewritten when the branch is being modified by more
  than one person.][Git series 2/3: rebase and the golden rule explained.]
- On some occasions, the cost of resolving conflicts from re-ordering commits
  outweighs the benefits of a clean history.
- CIs might not ensure that every commit is buildable either due to design
  limitations or resource constraints.

## How to keep a clean history

Git supports you with:

- Keeping history clean upfront
    - [The Index][Understanding Git - Index] (`git add <path>`, `git add -p <path>`)
    - [Stashing][Git Tools - Stashing] (`git stash`, `git stash pop`)
    - Branches (`git checkout -b <name>`)
- Fixing it up later
    - Rebasing (`git rebase`,`git rebase -i`, `git pull --rebase`)
    - Cherry-picking (`git cherry-pick <hash>`)
    - Universal undo (`git reflog`)

**Tips:**

- Again, [history should not be rewritten when the branch is being modified by
  more than one person.][Git series 2/3: rebase and the golden rule explained.]
- If you rewrote history after pushing to your remote branch, you will need to
  force-override it.  Use [`--force-with-lease`][force-with-lease] instead of
  `--force` as a safety catch in case someone else pushed to the branch.
- If you already published your PR, consider rewriting history after people
  have signed off on your PR so they can more easily see how you've
  responded to their feedback. This does mean you'll have to hold off on
  marking a PR for auto-completion until after you have rewritten history.

### Upfront: Distinct changes in distinct sections of code

When modifying one section of code, you might be reading another section and
notice something you want to change, like a typo.

When the changes are in separate files
```bash
git add <file1>
git commit
git add <file2>
git commit
```

When the changes are in the same file, you can select which parts to stage.
```bash
# Interactive staging let's you choose which changed hunks you want to stage
#
# Sometimes a hunk is bigger than you want and you can split it.
git add -p <file>
git commit
git add <file>
git commit
```

**Tip:** Make staging files your habit rather than `git commit --all` to save
yourself from having to fix things up later.

See also [Understanding Git - Index][Understanding Git - Index].

### Upfront: Partway through a change, realize you want to make a conflicting change

[Stashing][Git Tools - Stashing] is the go-to for quickly saving off your work to make another change

```bash
<change file>
git stash

<change file>
git add <file>
git commit

git stash pop
<resolve file>
<finish file>
git add <file>
git commit <file>
```

**Tip:** The stash is a stack of changes which can feel opaque to work with
which makes it harder to work with over periods of time or when multiple
changes are getting stashed.  It also can't be pushed for sharing / backup.  In
these cases, it might be better to use branches.

### Fixing: ctrl-z! ctrl-z! ctrl-z!

Before getting into how to rewrite history, its helpful to know how to recover
when you don't like where you ended up.  This is possible because git does not
automatically delete unreferenced commits in the commit graph (instead requires
calling `git gc`).  `git reflog` let's you browse your `HEAD`'s history, allowing
you to find commits from before rewriting history.

```bash
# Find hash from before rewrite
git reflog

# Update the branch and index to point to <hash>
# - `--soft` to not touch index
# - `--hard` to also update working directory
git reset <hash>
```

Lower impact commands:
```bash
# See differences
git diff <hash>
# Switch to <hash> as a detatched head (ie anonymous branch)
git checkout <hash>
# Update working directory, good if you want to revert your rebase via a commit.
git checkout <hash> -- .
```

See also [A Visual Git Reference][A Visual Git Reference] and [How to undo
(almost) anything with Git][How to undo (almost) anything with Git]

### "Fixing": Updating your branch to latest from `master`

Coming from other SCM's, `git merge` sounds familiar and might be what you
reach for by default.  The thing to remember is that git's commits are not
linear but a graph and `git merge` can easily lead to [complex commit
graphs][Bad commit graph].

```bash
# - Fetch origin/master
# - Take existing commits and append them to origin/master
git pull --rebase origin master
```

In case you want to explore the difference, like when dealing with conflicts,
it might be helpful to do this as separate commands:

```bash
git checkout master
git pull
git checkout - # `-` means "last branch"
git rebase master
```

See also [How to manage your Git history: Tips for keeping your commits
tidy][How to manage your Git history: Tips for keeping your commits tidy] and
[A tidy, linear Git history][A tidy, linear Git history]

### Fixing: Changing the most recent commit

```bash
<change file>
git add <file>
git commit --amend
```

**Tip**: If you've already pushed your PR, consider making a fixup commit (see
below) during the review and then cleaning them up later so reviewers can more
easily see how you've responded to their feedback.

See also [How to manage your Git history: Tips for keeping your commits
tidy][How to manage your Git history: Tips for keeping your commits tidy] and
[How (and why!) to keep your Git commit history clean][How (and why!) to keep
your Git commit history clean]

### Fixing: I need to change a specific commit

```bash
<change file>
git add <file>

# Find the commit hash your change fixes
git log
# Marks the commit as fixing up another commit
git commit --fixup <hash>

# Somtime later when you are ready to cleanup your history, this will
# automatically move your commit just after <hash> and squash it into <hash>
git rebase --autosquash master
```

See also [Auto-squashing Git Commits][Auto-squashing Git Commits] and [How (and
why!) to keep your Git commit history clean][How (and why!) to keep your Git
commit history clean]

**Tip:** You can turn on `--autosquash` by default with `git config --global rebase.autosquash true`

### Fixing: I need to add/remove/combine/split/re-order commits

`git rebase -i` let's you edit your commit history in your `$EDITOR` of choice

Note: things get funky when merge commits are used.  Another reason not to use them.

Just type:
```bash
git rebase -i master
```
And your commit history and instructions on editing it will pop up
```bash
pick 6e73d55c6 chore(nest: Baseline version of nipg.pl
pick 6aed59e08 chore(nest): Strip nipg down to essentials

# Rebase 1d9e1a08d..6aed59e08 onto 1d9e1a08d (2 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```
Once you've modified the "file", save and close and your history will be rewritten.

See also [How to manage your Git history: Tips for keeping your commits
tidy][How to manage your Git history: Tips for keeping your commits tidy] and
[How (and why!) to keep your Git commit history clean][How (and why!) to keep
your Git commit history clean]

**Tip:** By default, `$EDITOR` is VIM which works for me. For anyone else, you can change the editor, for example
to switch to VS Code, type `git config --global core.editor "code --wait"`.

## Resources

- [How to Write a Git Commit Message][How to Write a Git Commit Message]
- [How to manage your Git history: Tips for keeping your commits tidy][How to manage your Git history: Tips for keeping your commits tidy]
- [How (and why!) to keep your Git commit history clean][How (and why!) to keep your Git commit history clean]
- [Auto-squashing Git Commits][Auto-squashing Git Commits]
- [A Visual Git Reference][A Visual Git Reference]
- [Git series 2/3: rebase and the golden rule explained.][Git series 2/3: rebase and the golden rule explained.]

[How to Write a Git Commit Message]: https://chris.beams.io/posts/git-commit/
[How to manage your Git history: Tips for keeping your commits tidy]: https://ubuntu.com/blog/tricks-for-keeping-a-tidy-git-commit-history
[How (and why!) to keep your Git commit history clean]: https://about.gitlab.com/2018/06/07/keeping-git-commit-history-clean/
[Bad commit graph]: https://cdn-images-1.medium.com/max/800/1*e-tlWqLwbUd1UmZyC_KbGg.png
[Understanding Git - Index]: https://hackernoon.com/understanding-git-index-4821a0765cf
[Git Tools - Stashing]: https://git-scm.com/book/en/v1/Git-Tools-Stashing
[Auto-squashing Git Commits]: https://thoughtbot.com/blog/autosquashing-git-commits
[force-with-lease]: https://git-scm.com/docs/git-push#Documentation/git-push.txt---no-force-with-lease
[A Visual Git Reference]: https://marklodato.github.io/visual-git-guide/index-en.html?no-svg
[How to undo (almost) anything with Git]: https://github.blog/2015-06-08-how-to-undo-almost-anything-with-git/
[Git series 2/3: rebase and the golden rule explained.]: https://www.freecodecamp.org/news/git-rebase-and-the-golden-rule-explained-70715eccc372/
[A tidy, linear Git history]: http://www.bitsnbites.eu/a-tidy-linear-git-history/
