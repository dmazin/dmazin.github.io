---
layout: post
title: "Demystifying git submodules"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-02-22
tags:
    - git
    - featured
description: TODO
---
# TODO
- [ ] Julia Evans found the fact that git doesn't automatically update submodules surprising. Like she says, it "breaks git's model". I should focus on that more at the beginning.
- [ ] One thing she suggested I expand on is what happens after you git pull. "After you git pull, git add used to be a no op" but (paraphrasing now) now you will undo the person's work.
- [ ] Julia suggested cutting discussion of what a git branch is.
- [ ] In fact, she suggested cutting all history and just focusing on a single commit.
    - [ ] If I don't cut that, Vaibhav suggested that a commit isn't necessarily a linked list; it's a graph.
- [ ] Question from Julia: "does `git config submodule.recurse true` make it automatically update submodules?"
- [ ] Julia mentioned an interesting post: https://diziet.dreamwidth.org/14666.html. I should see if I undertand the post. She mentioned that she still didn't understand some things after reading my psot
```
editing files and trying to commit them no longer reliably works
git ls-files can disagree with git log and git cat-file
" Some of the defects occur even if you don’t git submodule init"
```
---
Git submodules cause a fair amount of grumbling. They certainly have made me grumble, because I didn't understand them, and kept getting myself into confusing situations that I couldn't fix.

So, I finally sat down and learned how git tracks submodules. Turns out, it's not that bad. But also, the way git tracks submodules is definitely different from how it tracks other stuff, so it makes sense that it's confusing.

Anyway, it's pretty great to not be confused anymore, so read on.

## The lay of the land
This will make more sense if we use examples, so let me describe a toy example that we'll be playing with.

Suppose you are working on a webapp. Call this repo **webapp**. Here's what the repo looks like.
```
README.md
tests
```

Say you want to use some library. For example, maybe the library defines some Python modules you want to import. That stuff lives in a different git repo called **library**. Here's what it looks like.
```
README.md
my_cool_functions.py
```

With those examples, let's dive into submodules.

## What's a submodule?
A git submodule is a full repo that's been nested inside another repo. Any repo can be a submodule of another.

That doesn't seem so confusing, does it? However, there are two important, and tricky, facts about submodules.

### A submodule is always pinned to a specific commit
You know how package managers let you be relatively fuzzy when specifying a package version ("get me any version of requests so long as it's 2.x.x"), or to pin an exact version ("use requests 2.31.0 exactly")?

Submodules can _only_ be pinned to a specific commit. This is because a submodule isn't a package; it's code that you have embedded in another repo. Things would be pretty confusing if two people could check out the same commit of the webapp repo, yet on one person's computer the `library` directory had contents that differed from the contents of `library` on another computer.

In other words, every commit of the webapp repo specifies an exact commit of library to use.

The source of a whole lot of pain is that, when you check out a commit, git **does not** automatically update submodules for you. That is, it doesn't make sure the submodules are using the desired version. Why this is, I'm not exactly sure. But that's how it is.

### Git does not automatically update submodules
If you clone `webapp`, git _will not_ automatically download `library` for you.

Similarly, if a collaborator points webapp at a new commit of library, and you `git pull`, git _will not_ automatically update `library` for you.

I think it's fair to say that this is the single bigest cause of confusion: it breaks our mental model of how git works.

This confusion is worth dwelling on. I think I can best illustrate it with a dramatic re-enactment of something that has happened to me multiple times before I understood submodules.

## A day in the life of someone who uses submodules
Pretend this is my stream of consciousness.

Ah, 2013. What a time to be a "full-stack engineer"!

I wonder what contributions are awaiting me on the master branch?

```
$ git pull
<some output>
```

Oh, that's interesting. It seems like I have local changes. What are they?

```
$ git st
## main...origin/main
 M library
```

Hmmm... looks like `library` is a directory. What's in it?
```
$ ls library
README.md            my_cool_functions.py
```

OK, but why is `library` showing up as modified? Why isn't it saying that some specific file within library is modified? That's weird. Well, what does `bit diff` have to say?

```
$ git diff
diff --git a/library b/library
index 2fb3d7b..aa695bb 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef
+Subproject commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
```

Apparently, I have deleted `Subproject commit 2fb3d7` and added `Subproject commit aa695`? Surely I didn't do that. That's weird, let me do a hard reset.

```
$ git reset --hard origin/main
HEAD is now at 98fde8f point submodule to newest commit
```

Did it make it go away?

```
$ git st
## main...origin/main
 M library
```

No... that's so weird. Well, usually when I make local changes, I need to `git add` them. But it's so weird, I never have to `git add` after `git pull`, and certainly not after doing a full `git reset --hard`...

```
$ git add .
$ git commit
$ git push
```

Nice! `git st` now shows no changes.

(some time passes)

Oh no, I just put up a PR and my coworker is saying that I am undoing his changes! What are they talking about?

To understand what's going on, let's go into slow motion and explain what happened.

## What happens when a submodule gets updated
Let's say Lida wanted to add a function to `my_cool_functions.py` in `library`. How did she do that?

Updating library itself is straightforward: because `library` is a normal repo, she adds a new commit to the `main` branch of library.

Under the hood, while the latest commit of `main` in `library` was `2fb3d7`, now it is `aa695`.

How can the repos that import `library` as a submodule take advantage of the new function?

Here is where we must remember one of the tricky facts about submodule: each repo points to an exact commit of `library`.

So, because Lida also works on `webapp`, she updates webapp to point to `aa695` (the latest commit of `library`), commits it, and pushes to `main`.

Now, when you `git pull` the `main` branch of `webapp`, what happens?

Here we must remember the second tricky fact about submodules: git does not automatically update submodules when you `git pull`.

Let's illustrate this.

Recall the `git diff` that says that you are trying to change `Subproject commit 2fb3d7` to `Subproject commit aa695`.
```
$ git diff
diff --git a/library b/library
index 2fb3d7b..aa695bb 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef
+Subproject commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
```

Why is it saying that? Let's `cd library` and -- because it's a full repo -- see what the latest commit is.

```
$ GIT_PAGER=cat git log -1 | head -n 1
commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
```

Remember: `aa695` was the latest commit of `library` _before_ Lida added the new function. And this makes sense: _git does not automatically update submodules_, so `library` still points at the old commit.

But why is `git diff` saying that _I_ have modified something?

Let's think about the way `git` looks at things.

Each commit in git specifies exactly what the repo should look like. For example, what the contents of README.md should be. And, of course, what commit the submodule `library` should point at.

git compares that state against the _actual_ contents of your working tree -- that is, your local modifications. That is how it knows whether or not you have modified README.md (for example).

So, git knows that the latest commit of `webapp` specifies that `library` submodule should point at `2fb3d7`. And, yet, the actual latest commit of `library` is `aa695`. And, so, git treats that like any modification, and it pops up in your `git diff`.

This may still be confusing. Let's dive into exactly how git tracks submodules.

First, let's start with the basics.

## How does git know where to download a submodule from?
When someone first adds a submodule, git creates a file called `.gitmodules`.

This file tells git where to download submodules from.
```
$ cat .gitmodules
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```
The nice thing about `.gitmodules` is that it's a regular file, tracked the regular way in git. That makes it not confusing.

But, don't forget: git does basically nothing about submodules automatically. If you cloned repo, `git would not download library`. To download it, you would need to call `git submodule init`.

But wait... doesn't git specify an _exact_ commit of a submodule? And yet that is nowhere to be found in `.gitmodules`. Then how does git track it?

To understand that, let's do a deep dive on the commit object.

## Recap
Before diving on the commit, let me recap where we are in our scenario.

You have woken up in 2013, the future is bright, and you have `git pulled` `master` of the `webapp` repo. Now, `master` points at `98fde8f68459b439e59b77aa528a9d7f39af12a2`. This is the commit we are going to analyze.

Unknown to you, last night Lida updated the `master` branch of the `webapp` repo to point to `2fb3d7`, and now `git diff` says something weird.
```
$ git diff
diff --git a/library b/library
index 2fb3d7b..aa695bb 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef
+Subproject commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
```

So, let's dive into `98fde8f68459b439e59b77aa528a9d7f39af12a2` and see what's going on, and how git tracks the `library` submodule.

## Scrutinizing a submodule
A commit is just a file on disk. However, it's optimized/compressed, so we use a built-in utility to view it. Here's what the commit stores.
```
$ git cat-file -p 98fde8f68459b439e59b77aa528a9d7f39af12a2
tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
parent 04d79eb516d4b421e13f779951497d46319020c2
author Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000

point submodule to newest commit
```

What we want to focus on is the _tree_. A tree describes the contents of your repo.
Recall the contents of `webapp`:
```
$ ls -a
.git        .gitmodules README.md   library     tests
```

Now, let's take a look at the tree from above. Compare it to the output of `ls`.
```
$ git cat-file -p 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
100644 blob 6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
160000 commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef  library
040000 tree a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
```

Note how `README.md` and `.gitmodules` both make an appearance. But, look, what else do you see?
```
160000 commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef  library
```

There it is -- library! 

$ cd library
$ git log -1
commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2 (HEAD)
Author: Dmitry Mazin <dm@cyberdemon.org>
Date:   Tue Feb 20 20:05:29 2024 +0000

    README
```

That matches `aa695bb` above.

What about the other part of the sentence?


## How does git track which commit of a submodule we're supposed to use?
One potential answer is `.gitmodules`, but no. There is no mention of a commit in `.gitmodules`.
```
$ cat .gitmodules
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```

Then how does git track it?

To answer that, let's briefly introduce how git tracks anything.

## Detour: How git tracks stuff
How does git actually track the state of your files?

Let's say you have checked out the main branch. How does git know what the contents of your repo should become?

### branches
The truth about branches is that they are embarrassingly simple. We think of them as collections of commits, but internally a branch is just a pointer to a commit.

We can see for ourselves:
```
$ cat .git/refs/heads/main
98fde8f68459b439e59b77aa528a9d7f39af12a2
```

`98fde` is just the latest commit of the `main` branch. See for yourself:
```
$ git log -1 | head -n 1
commit 98fde8f68459b439e59b77aa528a9d7f39af12a2
```

So, when you check out the main branch, you're actually checking out a commit.

But what is a commit, really?

### commits
A commit is how git tracks history. I mean this literally: every commit references the previous commit, and so this linked list of commits is the data structure known as "git history".

Let's see what the commit `98fde` looks like[1]:
```
$ git cat-file -p 98fde8f68459b439e59b77aa528a9d7f39af12a2
tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
parent 04d79eb516d4b421e13f779951497d46319020c2
author Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000

point submodule to newest commit
```

See how it references a "parent" commit? That's the previous commit.

Note, also, that the commit references a "tree". What's a tree?

### trees, more trees, and blobs
A tree describes the actual contents of your repo.

Let's take a look at what `ls` says:
```
$ ls -a
.git        .gitmodules README.md   library     tests
```

Compare that to the tree referenced by the commit above:
```
$ git cat-file -p 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
100644 blob 6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
160000 commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef  library
040000 tree a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
```

This is worth picking apart a bit.

As you can see, the tree references yet another tree -- `tests`: 
```
040000 tree a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
```

This is why this object is called a "tree" -- it's very similar to directories. Just like a directory references other directories, a tree references other trees.

You can see that the tree also references "blobs":
```
100644 blob a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
```

blobs store actual file contents.
```
$ git cat-file -p 6feaf03c7a9c805ff734a90a245a417e6a6c099b
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```

But, finally, see that the tree references... a commit???
```
160000 commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef  library
```

The name -- `library` -- gives it away. This is how git tracks submodules.

### How git really tracks submodules
Let's recap: the main branch references a commit, which references a tree, which references a commit of a different repo. As far as I know, when a tree references a commit, it always means "submodule".

This proves that every commit of webapp specifies an _exact_ commit of library to use: each commit specifies exactly one tree, which specifies exactly one commit of library.

But what about our confusing scenario?

## How to update a submodule
Recall our git diff:
```
$ git diff
diff --git a/library b/library
index 2fb3d7b..aa695bb 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef
+Subproject commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
```

We can now use our knowledge to answer what's going on: one of our colleagues updated `main` to use a different commit of `library` than we are currently using. **When you pull/checkout branches, git does not automatically update submodules for you.** So, your repo state does not currently match the state indicated by the latest commit of main.

One way to rectify it would be to go into `library` and literally check out the referenced commit:
```
$ cd library

$ git checkout 2fb3d7b854c6779ebb8cfc9773470cd2bf4022ef
Previous HEAD position was aa695bb README
HEAD is now at 2fb3d7b add some cool functions

$ cd ..

$ git st
## main...origin/main

$ git diff
```

See? Now git says there are no changes.

You don't actually need to do that: `git submodule update` does the same thing.

Usually, I use `git submodule update --init --recursive`. `--init` downloads the submodule if it hasn't already been downloaded; and is a no-op otherwise, so it's safe to always pass. `--recursive` is used in cases the submodules themselves reference other submodules (!).

We've all learned something today. Let's recap.

## Recap
It's possible to embed a repo within another repo. This is called a submodule.

Each commit of the outer repo always specifies _exactly_ which commit of the submodule it wants by slightly obscure `commit->tree->commit` linkage.

When you check out commits, git doesn't automatically update submodules for you. You have to do that using `git submodule update`.

Hopefully, understanding how git tracks submodules makes them less confusing. It made them less confusing to me. Feel free to write me if anything still confuses you.

## Notes
[1] `git cat-file is a nifty utility that git ships with out-of-the-box. Objects are stored in binary and with some other special optimizations, so this lets us view them.
