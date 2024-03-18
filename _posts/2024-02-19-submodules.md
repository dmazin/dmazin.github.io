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
- [x] Question from Julia: "does `git config submodule.recurse true` make it automatically update submodules?"
- [ ] Julia mentioned an interesting post: https://diziet.dreamwidth.org/14666.html. I should see if I undertand the post. She mentioned that she still didn't understand some things after reading my psot
- [x] make shas easier to read
- [ ] each prompt should show the pwd
- [x] rearrange article as recommended by perplexity: https://www.perplexity.ai/search/The-flow-of-mISsQcI3SE.K2WOwkKVjbg#1
- [x] one recurser mentioned that sometimes people override the work of others by comitting a submodule change by accident; this should be noted
    - for example it seems easy to do this if you don't `git st` before beginning your work. you may do some stuff and then commit the submodule change without noticing
- [x] mention git clone --recursive
---

Git submodules cause a fair amount of grumbling. They certainly have made me grumble, because I didn't understand them, and kept getting myself into confusing situations that I couldn't fix.

So, I finally sat down and learned how git tracks submodules. Turns out, it's not that bad. But also, the way git tracks submodules is definitely different from how it tracks other stuff, so it makes sense that it's confusing.

Note that I do _not_ discuss if submodules are "good"/"bad", nor do I discuss if you *should* use them. I also do not discuss alternatives. The goal of my article is only to explain how submodules work.

Anyway, it's pretty great to not be confused anymore, so read on.

## The lay of the land
This will make more sense if we use examples, so let me describe a toy example that we'll be playing with.

Suppose you are working on a webapp. Call this repo **webapp**. Here's what the repo looks like.
```
README.md
tests/
```

Say you want to use some library. For example, maybe the library defines some Python modules you want to import.

That stuff lives in a different git repo called **library**. Here's what it looks like.
```
README.md
my_cool_functions.py
```

## A day in the life of someone who uses submodules
Before we understand submodules, let's see what it looks like *not* to understand them. I think I can best illustrate this via a dramatic re-enactment of something that has happened to me multiple times before I understood submodules.

Ah, 2018. What a time to be a "full-stack engineer"! I wonder what contributions await me on the main branch!

```bash
$ git pull

<some output>
```

Let's just confirm that my working tree is clean.

```
$ git st

## main...origin/main
 M library
```

What's this? I've made modifications? To `library`?

Hmmm... looks like `library` is a directory. What's in it?
```
$ ls library

README.md            my_cool_functions.py
```

That's weird. I'm not used to git saying a whole _directory_ has been modified. Usually it just says a specific _file_ has been modified.

Well, what does `git diff` have to say?

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

Apparently, I have deleted `Subproject commit library_new_commit_sha` and added `Subproject commit library_old_commit_sha`.

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

It did not! I am really confused now!

Usually, when `git diff` shows an `M`, it's because _I_ modified something.

I guess one way to address that is to `git add` the change.

*My future self teleports into the room*

Oh no! If you `git add` that change, you'll actually be _rolling back_ a change someone else made.

Sometimes, people don't even notice the diff above, and accidentally do this.

The issue is that `library` is a **submodule** and git treats it differently than other files.

## What's a submodule?
Let's start with the basics: a git submodule is a full repo that's been nested inside another repo. Any repo can be a submodule of another.

So, `library` is a full repo that has been nested inside `webapp` as a submodule.

That doesn't seem so confusing, does it? However, there are two important, and tricky, facts about submodules.

### 1. A submodule is always pinned to a specific commit
You know how package managers let you be relatively fuzzy when specifying a package version ("get me any version of requests so long as it's 2.x.x"), or to pin an exact version ("use requests 2.31.0 exactly")?

Submodules can _only_ be pinned to a specific commit. This is because a submodule isn't a package; it's code that you have embedded in another repo, and git wants you to be precise.

### 2. git does not automatically download or update submodules
If you clone `webapp` afresh, git _will not_ automatically download `library` for you. (Unless you clone using `git clone --recursive`)

Similarly, if a collaborator pins webapp to a new commit of library, and you `git pull` webapp, git _will not_ automatically update `library` for you.

This is where the majority of confusion for submodules stem from. That's, in fact, what's going on in the dramatic reenactment above. Let's dive into how git actually tracks submodules to understand what's going on.

## How git tracks submodules
### How does git know where to download a submodule from?
git uses a file called `.gitmodules` to track the basic facts of a submodule, like the repo URL.

```
$ cat .gitmodules

[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```

The nice thing about `.gitmodules` is that it's a regular file, tracked the regular way in git. That makes it not confusing.

But wait... doesn't git specify an _exact_ commit of a submodule? And yet that is nowhere to be found in `.gitmodules`. Then how does git track it?

### How does git pin a submodule to a specific commit?
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

Focus on `tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4`. The _tree_ object represents the directory listing of your repo. When you think trees, think directories.

Here, see it for yourself.

First, recall the contents of `webapp`.
```
.gitmodules
README.md
library
tests
```

Now, let's inspect the tree object.
```
$ git cat-file -p 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4

100644 blob     6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob     a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
040000 tree     a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
160000 commit   library_new_commit_sha                      library
```

See how it matches the directory listing?

Note how `tests` is a `tree` (just like directories can hold directories, trees can point to trees).

But `library` is a... commit?!

```
160000 commit   library_new_commit_sha                      library
```

This weirdness is how git tracks the fact that library is not a regular directory, but a submodule. And it points at an exact commit of library: `library_new_commit_sha`.

To recap: `webapp_new_commit_sha` points to `tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4`, which points to `library_new_commit_sha`.

Let's use this knowledge to understand the `git diff` from earlier.

## Understanding what happened
Let's go back to the confusing situation, where the `git diff` looked weird after I `git pull`ed.

What must have happened is that a collaborator pushed up a new commit to `library` -- `library_new_commit_sha`. Because my collaborator wanted `webapp` to take advantage of `library_new_commit_sha`, they pushed a new commit to `webapp` -- `webapp_new_commit_sha` -- that pointed to `library_new_commit_sha`. So, when I `git pull`ed, I checked out `webapp_new_commit_sha`. But, of course, git did not update `library.`

That hopefully makes sense, but the git diff is still confusing. Let's pick it apart.

## Understanding git diff
Here's the diff again.

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

It's confusing that it's saying that **I** modified `library`. I didn't modify it, someone else did!

Usually, I think of `git diff` as "here are the changes I have made". But this isn't exactly correct.

When you invoke `git diff`, you're asking git to tell you the difference between your working tree (that is, your unstaged, uncommitted local changes) and the most recent commit of your branch (`webapp_new_commit_sha`).

When you look at it that way, the above git diff starts to make sense. In `webapp_new_commit_sha`, `library` points to `library_new_commit_sha`, but in our working tree, `library` still points to `library_old_commit_sha`.

git has no idea which change happened first. It only knows that your working tree is different from the commit.

So... that's all fine and good, but how do we actually work with submodules? How do we make the git diff go away?

## How to update a submodule
We now understand that we need to point `library` to `library_new_commit_sha`. How?

Because `library` is a full repo, I could just `cd` into it and literally check out that commit:
```
$ cd library

$ git checkout library_new_commit_sha

Previous HEAD position was library_old_commit_sha README
HEAD is now at library_new_commit_sha add some cool functions
```

If we go back into `webapp`, we'll see that `git st`/`git diff` finally look clean.

```
$ git st

## main...origin/main
# (no output)

$ git diff

# (no output)
```

However, you don't actually need to do the above.

## How to really update a submodule
From `webapp`, we can invoke `git submodule update`. This updates _all_ of a repo's submodules.

People often use certain flags with `git submodule update`, so let's understand them.

### `git submodule update --init`
Remember how I said that if you `git clone webapp`, git won't actually download the contents of `library`?

What you're supposed to do is, after cloning webapp:
1. Run `git submodule init` to initialize the submodules. This doesn't actually download them, though 🙃️.
2. Run `git submodule update` to actually pull the submodules.

This is kind of a silly dance, so git lets you just do `git submodule update --init`. This initializes any submodules and updates them in one step. I _always_ pass `--init` because there is no harm in doing so.

Note that, when you clone webapp, you can do `git clone --recursive`. Of course, you may still need to update submdules later, and I recommend using the `--init` flag.

### `git submodule update --recursive`
Submodules can nest other submodules. Yeah.

So, to take care of updating submodules _all the way down_, pretty much just always pass `--recursive` to `git submodule update`.

So, the command I always end up using is `git submodule update --init --recursive`.

### `git config submodule.recurse true`
I have not used this setting before, but `submodule.recurse true` makes submodules automatically update when you `git pull`, `git checkout`, etc. In other words, it makes submodules automatically point to whatever they are supposed to point to.

That makes running `git submodule update` unnecessary. I'm going to give this setting a try, but I'm not sure what its limitations are. Let me know if you know of any.

This setting definitely does _not_ apply to `git clone`. So you still need to do `git clone --recursive` or init/update submodules using the commands above.

## Recap
I think I can summarize submodules pretty simply.

It's possible to embed a repo within another repo. This is called a submodule.

Each commit of the outer repo always specifies an _exact_ commit that submodule. This is done by the `outer commit -> tree -> submodule commit` link.

When you check out commits, git doesn't automatically update submodules for you. You have to do that using `git submodule update`.

And there we have it!

## Further topics in submodules
The above is enough to hopefully take the confusion out of submodules. However, there are more common commands and configs that I'd like to explain.

### How to add a submodule: `git submodule add`
Let's say that I start `webapp` fresh, and I have not added `library` to it yet.

To add `library`, I'd do `git submodule add https://github.com/dmazin/library.git library`.

This will add (or update) the `.gitmodules` file of `webapp`, download `library`, and point `webapp` at the latest commit of `library`.

Remember, this actually modifies `webapp`, so you need to commit after that. But you thankfully don't need to do `git submodule update` after doing `git submodule add` or anything.

### What do I do after I've modified a submodule?
Remember that `library` is a full repo, so if you want to make changes to it, you can. Just make changes and commit them to the main branch.

But how do you make `webapp` point at the new commit? There are a couple ways.

#### Without a command
You can go into `webapp`, then `cd library`, and just do `git pull` in there. When you `cd` back into `webapp`, if you `git diff` you'll see that `webapp` points to the newest branch of `library`. You can commit that.

#### Using `git submodule update --remote -- library`
This tells git "make the submodule point to the latest remote commit". Since you have pushed the latest commit of library to library's remote, this will make `webapp` point to that commit.

But note that `git submodule update --remote` will do this to _all_ your submdules. You likely do not want that.

For that reason, you have to do `git submodule update --remote -- library` to limit this to library only. (If you're thrown off by the fact that you have to do `-- library` -- yeah, it's kind of weird.)

Because `--remote` might accidentally update all the submodules, honestly I usually do the "without a command" method.

### Making submodules use branches other than main
Remember .gitmodules?

If you want to, you can make `library` track whatever branch you want. Otherwise, it defaults to whatever the "main" branch is.

```
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
        branch = staging
```

Thanks for reading!
