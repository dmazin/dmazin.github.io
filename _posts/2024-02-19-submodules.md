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
- [x] Julia Evans found the fact that git doesn't automatically update submodules surprising. Like she says, it "breaks git's model". I should focus on that more at the beginning.
- [x] One thing she suggested I expand on is what happens after you git pull. "After you git pull, git add used to be a no op" but (paraphrasing now) now you will undo the person's work.
- [x] Julia suggested cutting discussion of what a git branch is.
- [x] In fact, she suggested cutting all history and just focusing on a single commit.
    - [-] If I don't cut that, Vaibhav suggested that a commit isn't necessarily a linked list; it's a graph.
- [ ] Question from Julia: "does `git config submodule.recurse true` make it automatically update submodules?"
- [ ] Julia mentioned an interesting post: https://diziet.dreamwidth.org/14666.html. I should see if I undertand the post. She mentioned that she still didn't understand some things after reading my psot
- [x] make shas easier to read
- [ ] each prompt should show the pwd
- [ ] rearrange article as recommended by perplexity: https://www.perplexity.ai/search/The-flow-of-mISsQcI3SE.K2WOwkKVjbg#1
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

Before we jump into understanding submodules, let's show what it looks like *not* to understand them. I think I can best illustrate this via a dramatic re-enactment of something that has happened to me multiple times before I understood submodules.

## A day in the life of someone who uses submodules
Ah, 2013. What a time to be a "full-stack engineer"!

I wonder what contributions are awaiting me on the main branch?

```bash
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

OK, but why is `library` showing up as modified? Why isn't it saying that some specific file within library is modified? That's weird. Well, what does `git diff` have to say?

```
$ git diff
diff --git a/library b/library
index library_old_commit_sha..library_new_commit_sha 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit library_new_commit_sha
+Subproject commit library_old_commit_sha
```

(For the sake of readabitlity, in this article, instead of using real commit SHAs, I'm going to use fake descriptive ones)

Apparently, I have deleted `Subproject commit library_new_commit_sha` and added `Subproject commit library_old_commit_sha`?

Surely I didn't do that. That's weird, let me do a hard reset.

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

I am really confused now!

Usually, when `git diff` shows an `M`, it's because _I_ modified something. I guess one way to address that is to `git add` the change.

But if you do that, you will, without knowing it, actually be _rolling back_ a change someone else made. To understand why, read on.

What I did not know is that while `library` looks like a directory in my local filesystem, `git` does not really see it as a directory. It sees it as a _submodule_.

## What's a submodule?
A git submodule is a full repo that's been nested inside another repo. Any repo can be a submodule of another.

So, `library` is a full repo that has been nested inside `webapp` as a submodule.

That doesn't seem so confusing, does it? However, there are two important, and tricky, facts about submodules.

### 1. A submodule is always pinned to a specific commit
You know how package managers let you be relatively fuzzy when specifying a package version ("get me any version of requests so long as it's 2.x.x"), or to pin an exact version ("use requests 2.31.0 exactly")?

Submodules can _only_ be pinned to a specific commit. This is because a submodule isn't a package; it's code that you have embedded in another repo. I think the reason for this pinning is to force us to specify exactly what version of code we're working with.

### 2. git does not automatically update submodules
If you clone `webapp`, git _will not_ automatically download `library` for you.

Similarly, if a collaborator points webapp at a new commit of library, and you `git pull`, git _will not_ automatically update `library` for you.

## What happened before I `git pull`ed
Let's say that, the day before the above monologue took place, my coworker wanted to add `my_really_cool_function()` to `my_cool_functions.py` in `library`. How did they do that?

Updating library itself was straightforward: because `library` is a normal repo, they added a new commit to the `main` branch of library: `library_old_commit_sha`.

Great! Now, how can `webapp` take advantage of `my_really_cool_function()`?

So, my coworker added a new commit to `webapp` -- `webapp_new_commit_sha` -- which points to `library_new_commit_sha`.

But when I `git pull`ed `webapp_new_commit_sha`, git _did not_ automatically point `library` to `library_new_commit_sha`.

## Proving that git pull does not update submodules
Let's illustrate this.

Because `library` is a full repo, we can see what its latest commit is.
```
$ cd library

$ git log -1
commit library_old_commit_sha (HEAD)
Author: Dmitry Mazin <dm@cyberdemon.org>
Date:   Tue Feb 20 20:05:29 2024 +0000

    Initial commit
```

_git does not automatically update submodules_, so `library` still points at the old commit.

This helps explain why `git diff` looks weird.


## Understanding why git diff says I modified something
Let's think about the way `git` generates `git diff`. 

Each commit in git specifies exactly what the repo should look like. For example, what the contents of README.md should be. And, of course, what commit the submodule `library` should point at.

git compares that state against the _actual_ contents of your working tree -- that is, your local modifications. That is how it knows whether or not you have modified README.md (for example).

So, git knows that the latest commit of `webapp` specifies that `library` submodule should point at `library_new_commit_sha`. And, yet, locally the actual latest commit of `library` is `library_old_commit_sha`.

You can think of `git diff` this way: don't think of it as necessarily the changes you have made, but the difference between your local changes and a commit.

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

## Scrutinizing a commit
The latest commit of `webapp` is `webapp_new_commit_sha`. Let's inspect that commit.

A commit is just a file on disk. However, it's optimized/compressed, so we use a built-in utility to view it. Here's what the commit stores.
```
$ git cat-file -p `webapp_new_commit_sha`
tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
parent webapp_old_commit_sha
author Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000

point submodule to newest commit
```

You can see that the commit object stores the commit message, the author, and it points to another commit.

It also references a tree object. Briefly put, the _tree_ object represents the directory listing of your repo. When you think trees, think directories.

The easiest way is to see it for yourself.

First, recall the contents of the repo.
```
$ ls -a webapp
.git
.gitmodules
README.md
library
tests
```

Now, let's inspect the tree object.
```
$ git cat-file -p 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
100644 blob 6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
040000 tree a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
160000 commit library_new_commit_sha  library
```

See how it lists all the files tracked by git?

Let's unpack this further.

Briefly, **blobs** store the file contents, **trees** are other directories (just like directories can hold directories, trees can point to trees), and -- what's what? -- a **commit** object pointing to `library`!

```
160000 commit library_new_commit_sha  library
```

So... a commit references a tree, and the tree referefences yet another commit? Yes.

`commit1->tree->commit2`? YES.

It's pretty strange.

`tree->blob` and `tree->tree` make sense: that's just how directories work. Think `directory->file` and `directory->directory`.

But `tree->commit` doesn't map to any concept in the filesystem, nor is it based on any other git feature. We have nothing, conceptually, to hang our hat on. Plus, this way of tracking submodules is a one-off: so far as I know, **`tree->commit` is exclusively used to implement submodules**.

But how do we actually get out of the confusing situation I described earlier?

## How to update a submodule
The output of `git diff` is clear now: I need to point `library` to `library_new_commit_sha`. How?

Because `library` is a full repo, I could just `cd` into it and literally check out that commit:
```
$ cd library

$ git checkout library_new_commit_sha
Previous HEAD position was aa695bb README
HEAD is now at 2fb3d7b add some cool functions

# go back to webapp
$ cd ..

$ git st
## main...origin/main

$ git diff
# (no output)
```

See? Now git says there are no changes.

You don't actually need to do that: `git submodule update` does the same thing.

Usually, I use `git submodule update --init --recursive`. `--init` downloads the submodule if it hasn't already been downloaded; and is a no-op otherwise, so it's safe to always pass. `--recursive` is used in cases the submodules themselves reference other submodules (!).

## Recap
It's possible to embed a repo within another repo. This is called a submodule.

Each commit of the outer repo always specifies _exactly_ which commit of the submodule it wants by slightly obscure `commit->tree->commit` linkage.

When you check out commits, git doesn't automatically update submodules for you. You have to do that using `git submodule update`.

Hopefully, understanding how git tracks submodules makes them less confusing. It made them less confusing to me. Feel free to write me if anything still confuses you.
