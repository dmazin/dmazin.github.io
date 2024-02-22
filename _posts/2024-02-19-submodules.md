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
Submodules seem to cause universal grumbling. They were a major pain in my ass. So, I sat down and learned how they work. Now they are just a slight pain in the ass. Join me on a journey into git's internals, and they can be a slight pain in your ass too.

The implementation of submodules is fairly simple, so it will take only a bit of your time to prevent any further confusion about them.

I'm going to start by explaining some important concepts of submodules. After that, we'll dive into how they work, and what's really doing on during common confusing scenarios.

## What's a submodule?
A git submodule is a full repo that's been nested inside another repo. Any repo can be a submodule of another.


This will make more sense if we use examples, so let me describe a toy example that we'll be playing with.

## The lay of the land
Suppose you are working on a webapp. Call this repo **webapp**. Here's what the repo looks like.
```
$ ls -al
total 16
drwxr-xr-x   7 dmitry  staff   224 21 Feb 19:46 .
drwxr-xr-x  75 dmitry  staff  2400 20 Feb 20:07 ..
drwxr-xr-x  15 dmitry  staff   480 21 Feb 19:46 .git
-rw-r--r--   1 dmitry  staff     4 21 Feb 19:46 README.md
drwxr-xr-x   4 dmitry  staff   128 20 Feb 20:57 tests
```

Say you want to use some library. For example, maybe it defines some Python modules you can import. That stuff lives in a different git repo. Call this repo **library**. Here's what it looks like.
```
$ ls -al
total 16
drwxr-xr-x  5 dmitry  staff  160 21 Feb 19:49 .
drwxr-xr-x  6 dmitry  staff  192 20 Feb 20:12 ..
-rw-r--r--  1 dmitry  staff   32 20 Feb 20:06 .git
-rw-r--r--  1 dmitry  staff   27 21 Feb 19:18 README.md
-rw-r--r--  1 dmitry  staff    0 21 Feb 19:49 my_cool_functions.py
```

## How do I use "library" as a submodule?
To use a repo as a submodule, invoke `git submodule add <url> <name of directory to stick the repo into>`.

So, let's do that.
```
$ git submodule add https://github.com/dmazin/library.git library
Cloning into '/Users/dmitry/dev/webapp/library'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
```

This creates a (for now) empty directory named `library`, and a file called `.gitmodules`.

`.gitmodules` starts to explain how git tracks submodules.

## How does git know where to download a submodule from?
You only need to add a submodule to a repo once. If someone else then clones your repo, they don't have to add it. This is because of `.gitmodules`. It tells git where to download a submodule, and where to stick it.

```
$ cat .gitmodules
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```

The nice thing about `.gitmodules` is that it's a regular file, tracked the regular way in git. That makes it not confusing.

***here begins new stuff i added feb 23 that kind of throws a wrench in things**
This is not the end of the story of how git tracks submodules, though. Allow me to introduce the next piece by showing a common confusing scenario.

## A common confusing scenario
Have you ever pulled a branch, and seen this?

```
$ git st
## main...origin/main
 M library
```

Huh, the submodule is modified? So you check `git diff`.
```
$ git diff
diff --git a/library b/library
index aa695bb..9904693 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2
+Subproject commit 9904693adf8206f4c47bbc179bf5065cd99d7a53
```

What does that mean?

Perhaps, at this point, you `git add .`, because when you have uncommitted changes, the hunch is to commit them. However, if you create a PR, you'll see that now your PR wants to update a submodule, which is only more confusing.

## The source of the confusion
I think the source of this confusion is that git tracks two things about a submodule: the current commit of the actual repo of the submodule (that is, of the `library` repo), and the submodule commit that `webapp` desires.

**TODO: here I can first show what commit submodule has checked out, then show what commit webapp points to**
***end of stuff i added feb 23***

## A submodule is always pinned to a specific commit
You know how package managers let you be relatively fuzzy when specifying a package version ("get me any version of requests so long as it's 2.x.x"), or to pin an exact version ("use requests 2.31.0 exactly").

Submodules can _only_ be pinned to a specific commit. This is because a submodule isn't a package; it's code that you have embedded in another repo. Things would be pretty confusing if two people could check out the same commit of the webapp repo, yet on one person's computer the `library` directory had contents that differed from the contents of `library` on another computer.

Wait, why am I saying "check out a commit"? We don't usually check out commits, we check out branches. We need to take a brief detour so that we can be precise with our language.

## Detour: a branch is just a pointer to a commit
We think of a branch as a set of commits, or even as the state of a repo's files. Internally, though, *commits* store the state of the git repo. On the other hand, a branch is an extremely light-weight thing: it's a pointer to a commit.

So if you make a commit to `main` and push, what you're really saying "here's a new commit, and, y'all's main branches should now point to my commit".

Let's prove this by looking at the tiny file that represents the `main` branch.

First, note I am currently on the `main` branch of `webapp`. Here's the latest commit:
```
$ git log -1
commit 41fd61ed3249a93434fb1926d5879142e63f96dd (HEAD -> main, origin/main, origin/HEAD)
Author: Dmitry Mazin <dm@cyberdemon.org>
Date:   Wed Feb 21 19:46:53 2024 +0000

    add readme
```

See for yourself that the contents of `main` on disk is simply the identifier of that same commit:
```
$ cat .git/refs/heads/main
41fd61ed3249a93434fb1926d5879142e63f96dd
```

This is why I say that, when you check out a branch, you're really checking out a commit. Commits store the state of your repo, not branches.

Back to submodules.

## How does git track which commit of a submodule we're supposed to use?
Wait... look back at `.gitmodules`. It doesn't specify a commit. Then how does git track which commit of a submodule we're supposed to use?

To answer that, let's briefly introduce how git tracks anything. In git, there are 3 main objects: **commits**, **trees**, and **blobs**.

Roughly, **commits** represent history, **trees** represent directory structure, and **blobs** hold actual file contents.

Every commit points to a tree. Basically, this is akin to saying "if you check out commit 41fd, when you `ls` you should see such-and-such files".

### What's in a commit?
Let's see for ourselves by looking at the commit we mentioned above, `41fd61ed3249a93434fb1926d5879142e63f96dd`.

```
$ cat .git/objects/41/fd61ed3249a93434fb1926d5879142e63f96dd
x;0
   @s
HI|$X9c!-PNOx{:wpzŜ>)uEb)4XK#[VʓNS      I2j`lFvFC\Ny+oEh]8
                                                          z8a\C"Д
                                                                 GT%
```

Oh no! It's binary! That's fine. git isn't trying to hide anything. It comes out-of-the-box with tools for viewing objects: `git cat-file`..

```
$ git cat-file -p 41fd61ed3249a93434fb1926d5879142e63f96dd
tree 69d049cd037debf8c12bd7bb092e762584888ab1
parent 6c607c1cea9feffb63cad6ec0e6c38190c2d20a5
author Dmitry Mazin <dm@cyberdemon.org> 1708544813 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708544813 +0000

add readme
```

As you can see, a commit points to its parent commit (the commit that came before it), as well as to a tree.

### What's in a tree?
I said a tree represents directory structure, and let's make that concrete by looking at the tree referenced by the commit, `69d049cd037debf8c12bd7bb092e762584888ab1`.

```
$ git cat-file -p 69d049cd037debf8c12bd7bb092e762584888ab1
100644 blob 6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
160000 commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2  library
040000 tree a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
```

Ooh, so that's interesting. The tree points to another tree (`tests/`)! This makes sense. Think of the way that directories contain other directories.

You can also see that the tree points to a couple blobs, representing regular files -- `.gitmodules` and `README.md`.

But what's that? It also points to a `commit`.
```
160000 commit aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2  library
```

That's a little strange. But the fact that it's called `library` gives it away: this is our submodule. So this is how git tracks submodules: as a tree that points to a commit.

## Recap: a submodule is always pinned to a specific commit
This is why I say that a submodule is pinned to a specific commit.

The `main` branch of `webapp` points to commit `41fd61ed3249a93434fb1926d5879142e63f96dd`, which points to tree `69d049cd037debf8c12bd7bb092e762584888ab1`, which points to submodule commit `aa695bbda27b1023a35d8b0a8d1d937ce5d25ce2`.

So when you check out commit `41fd`, you accept that `library` should use commit `aa69`.
