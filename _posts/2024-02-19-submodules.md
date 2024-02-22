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
# How git submodules work
submodules, as a feature of git, seem to cause universal grumbling. They certainly made me grumble, until recently I finally sat down and read about how they work. It turns out that git submodules are simple. My personal experience is that learning how they work has made them a lot less painful.

First, let me discuss why they cause so much pain.

# Why git submodules are confusing
I think the reason submodules cause so much misery is because they violate our mental model of how git works.

If your local branch is behind its remote counterpart, git clearly indicates this by telling you how many commits behind you are, and you are only a `git diff` away from understanding which files are different and how the contents differ. If your working tree has unstaged changes (that is, you have edited files but have not `git add`ed the changes yet) you can similarly see what's changed and how, via `git diff`. 

submodules are different. For example, one common sight is to see that a submodule is shown as modified:
```bash
$ git st
## main...origin/main
 M inner-repo
```

When you `git diff`, instead of seeing changed files/contents, you see something else:
```diff
diff --git a/inner-repo b/inner-repo
index 25d49b4..2658bd0 160000
--- a/inner-repo
+++ b/inner-repo
@@ -1 +1 @@
-Subproject commit 25d49b40f09301aebc36c4e1a565e99095c3bfce
+Subproject commit 2658bd026ab4c6152961d43660d056448f776749
```

What does this mean? You're not sure, but, when you have uncommited changes, the urge is to stage and commit them.
```
You see a confusing untracked change to the submodule...
$ git st
## main...origin/main
 M inner-repo

So you stage it...
$ git add .

Now it's staged.
$ git st
## main...origin/main
M  inner-repo

You commit it and push, I guess?
$ git ci -am "something?"
[main 1145d81] something?
 1 file changed, 1 insertion(+), 1 deletion(-)

$ git push
```

You made the local branch look clean, but of course, when you open up your PR, you see that now, in the PR, you are suggesting a change to the submodule.

The right thing is to do something like `git submodule update --init --recursive`, but you have to be told to do those, the flags make little sense.

submodules need not be confusing. The way they are implemented is fairly straightforward, so let's demystify them.

# What is a submodule?
A submodule is a fully-featured git repo that's been nested in your outer repo. That means you can go in, make changes, push and pull, etc. -- it's a real-ass repo.

However, what's more interesting (and confusing) is how submodules are tracked in the outer repo.

# How does git track submodules?
Recall that we have created `inner-repo`, which has been pulled into `outer-repo` as a directory:
```
$ pwd
/Users/dmitry/dev/outer-repo

$ ls
inner-repo

$ file inner-repo
inner-repo: directory
```

`inner-repo` is a real git repo, but if you clone `outer-repo` afresh, you'll see that the `inner-repo` directory is empty.
```
$ ls -halt inner-repo
total 0
drwxr-xr-x  5 dmitry  staff   160B 19 Feb 20:33 ..
drwxr-xr-x  2 dmitry  staff    64B 19 Feb 20:28 .
```

**what follows is stuff i wrote feb 20**
So how does git know anything about `inner-repo`? How does it know where to download it from? This instroduces the first file used for tracking submoduiles: `.gitmodules`.

If you repo uses submodules, this file appears in the root directory of your repo (e.g. the same place you might put .gitignore).
```
$ cat .gitmodules
[submodule "inner-repo"]
        path = inner-repo
        url = https://github.com/dmazin/inner-repo.git
```

.gitmodules is a normal file. For example, if you want to change the remote URL of `inner-repo`, that change will be tracked in git like any other change. Because it works like normal files, it's not confusing, so that's nice. What _is_ confusing is that it does not mention which commit of the submodule the `outer-repo` wants. (TODO: this relies on me earlier explaining that a repo points at a specific comit) So where is _that_ stored?

# How does git know which commit of the submodule to point to?
Submodules become obvious once you understand how exactly git tracks them. Let’s peek inside .git, where git stores your repo’s state.

What follows is a brief primer on .git. If you are comfortable with the contents of .git, feel free to skip to [TODO]().

## What's inside .git?
First, recall the state of things: we have cloned `webapp`, but have not done anything to initialize the submodule `library`. Yet, somehow git knows what commit of `library` to use. We're firuging out how. (this last sentence may be unnecessary)

This is what `.git` looks like.
```bash
$ ls .git
HEAD        description index       logs        packed-refs
config      hooks       info        objects     refs
```

What you're looking at is a database that contains the entire state of your repo. The most interesting things to us are `refs` and `objects`: not only are they directories, but also git concepts. Let's introduce them.

### refs
You can think of a ref as a pointer or, of course, a reference. The only information they hold is the identifier of something else. The most famous example of a ref is a **branch**.

#### branches
A branch is merely a reference to a commit. Specifically, it's a reference to the latest commit of a list of commits. That means that a branch is a lightweight thing, and when you do something like `git checkout -b dmazin/scary-branch-backup` what you're really saying is "make it easy for me to remember the last commit before I did something scary".

Our only branch is `main`, and you can see it here:
```bash
$ ls .git/refs/heads
main
```

The entirety of that file is the latest commit of the branch.
```
$ cat .git/refs/heads/main
6c607c1cea9feffb63cad6ec0e6c38190c2d20a5
```

Let's cover the next most important kind of ref: HEAD.

#### HEAD
How does git track which branch you have actually checked out? You guessed it -- a ref. All it does it point to the branch you've checked out.

```
$ cat .git/HEAD
ref: refs/heads/main
```

Have you ever seen something like this?
```
You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.
```

You get there by checking out a specific commit:
```
$ git co 49f1966cc55fd39c23d7d5d44625dd520818d030
HEAD is now at 49f1966 add tests

$ git st
## HEAD (no branch)
```

You can see that HEAD now points to a specific commit.
```
$ cat .git/HEAD
49f1966cc55fd39c23d7d5d44625dd520818d030
```

branches and HEAD are sufficient to track the local state of your repo. There are other refs, for tracking what's going on remotely, but we don't need to worry about that to understand submodules.

HEAD points to branches, and branches point to commits. Naturally, let's explain what a commit is.

### objects
refs need to point to _something_, right? They can point to other refs, but at some point, there needs to be some substance.

#### commit
Commits are only slightly heavier than branches. The only data they really store is the commit message and a little metadata. For the most part, commits are actually also references.

Earlier, we saw that the main branch points to a specific commit.
```
$ cat .git/refs/heads/main
6c607c1cea9feffb63cad6ec0e6c38190c2d20a5
```

What _is_ a commit? Like, is it a file? Sort of! If we want to, we are totally allowed to go visit the file, but it's not human-readable.

```
$ cat .git/objects/6c/607c1cea9feffb63cad6ec0e6c38190c2d20a5
x10

Hi6Y8!-
a(      ~%BӸ]-"P@w,Fg"!g{
'zRBLh6Vw<i,(Ytd]Kaf8|5H        tv3wXWQPU6^먾IW%
```

The really cool thing is that git is shipped with utilities for reading object files.
```
$ git cat-file -p 6c607c1cea9feffb63cad6ec0e6c38190c2d20a5

tree cfa0eb4abae0b96ce3ef9caaea847ab58709f9d3
parent df04c42416fa9c5e8c3a324a3dec100ce155b0c9
author Dmitry Mazin <dm@cyberdemon.org> 1708461639 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708461639 +0000

add test README
```

Here we see that a commit has two references: one to its parent, and one to a "tree".

**end of stuff i wrote feb 20**
**what follows is stuff i wrote on feb 19**

# A brief primer on git internals
To understand how git tracks which commit of a submodule a repo points to, it's helpful to understand how git tracks things in the first place.

One revelation I had about git is that a branch is just a pointer to a commit. You may have heard of "HEAD" -- that is also a pointer. HEAD points to the current branch (or, sometimes, to a specific commit, if you have manually checked out a commit -- you will recognize this if git has ever told you that you were in a "detached" state).

Where does git store where HEAD and a branch (like main) point to? Let's climb into the `.git` directory and get aquainted. Most of the files in `.git` are plaintext, which means it's easy to explore. For binary files, git comes pre-packed with ways to read those files.

```
$ ls
FETCH_HEAD  ORIG_HEAD   description index       logs        packed-refs
HEAD        config      hooks       info        objects     refs
```

Today, we're going to dig into `HEAD`, `refs`, `objects`, `config`, and one more directory that git hasn't created yet.

When you look into cat, you can see that it's a plaintext file that simply points at the main branch.

```
$ cat HEAD
ref: refs/heads/main
```

If you check out a specific commit and get into a detached state, here's what HEAD looks like.

```
$ git co 1145d813c06535c27534a68644dbdb034b475239
Note: switching to '1145d813c06535c27534a68644dbdb034b475239'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

[...]

HEAD is now at 1145d81 something?
dmitry@dmitry-mba-m1 [21:33:21] [~/dev/outer-repo] git:(1145d81)

As you can see, HEAD now simply points at a specific commit:
```bash
$ cat .git/HEAD
1145d813c06535c27534a68644dbdb034b475239
```

(TODO: put the above in a footnote)

Let's keep exploring. What does `.git/refs/heads/main` hold?
```
$ cat .git/refs/heads/main
14e8fd62274fad8ab9b425a9cc4a76c9c86b0738
```

That looks like a commit. Indeed, it's the latest commit in my repo:
```
$ git log

commit 14e8fd62274fad8ab9b425a9cc4a76c9c86b0738 (HEAD -> main)
Author: Dmitry Mazin <dm@cyberdemon.org>
Date:   Mon Feb 19 21:22:13 2024 +0000

    add directory

commit 1145d813c06535c27534a68644dbdb034b475239 (origin/main)
Author: Dmitry Mazin <dm@cyberdemon.org>
Date:   Mon Feb 19 20:19:11 2024 +0000

    something?

[...]
```

You can see in `git log` also that `14e8f` is pointed to by HEAD, which itself points to main.

Now, here's where we get to the interesting stuff. The following starts to show why git is efficient.

# What's a commit?
What _is_ a commit? A commit is just a file. In git parlance, it's an _object_, and we can find objects in `.git/objects`.

A commit is, itself, a pointer[2], along with some metadata. How can we see it? 


What does it point to? A _tree_.

# What's a tree?

[2] need to make clear that it's not the same as a branch or HEAD, which are *refs*


**everything below was written feb 21**
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

However, this is not the end of the story of how git tracks submodules. To see why, we need to know one more core conceptual fact.

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



### dump
**unused attempt to explain how submodules must use a specific commit**
In Python, you can use a package manager called pip. pip allows you to use a "requirements" file, where you specify whatever packages you want, along with their versions. This file is always tracked via git.

So, let's say your `requirements.txt` looks like:
```
requests>=2,<3
```

When you commit this file, what you're saying is, "when you run a Python app after checking out commit abcd, you can use any version of requests so long as the version is, like, 2.x.x". 

Submodules aren't like that. Instead, they are more like a requirements file that pins to a specific version, like so:
```
requests==2.31.0
```

That is, you're saying, "if you're running after checking out commit a23d, you must use requests 2.31.0".

Submodules work a lot like that. git does _not_ say, "your submodule should be on the main branch." It says, "your submodule should use exactly a specific commit."
**end**
