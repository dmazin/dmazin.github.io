---
layout: post
title: "Demystifying git submodules"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2024-03-20
tags:
    - git
    - featured
description: Demystifying git submodules by showing exactly how they work.
---
Throughout my career, I have found git submodules to be a pain. Because I did not understand them, I kept getting myself into frustrating situations.

So, I finally sat down and learned how git tracks submodules. Turns out, it's not complex at all. It's just different from how git tracks regular files. It's just one more thing you have to learn.

In this article, I'll explain exactly what I needed to know in order to work with submodules without inflicting self-damage.

(This article doesn't discuss whether submodules are good/bad, or if you should use them or not -- a valid discussion, but out of scope.)

## The lay of the land
This article will make more sense if we use concrete examples.

Allow me to describe a toy webapp we're building. Call this repo `webapp`. Here are the contents of the repo.
```
$ [/webapp] ls

.git/
README.md
tests/
```

Say you want to import some library. It lives in its own repo, `library`.
```
$ [/library] ls

.git/
README.md
my_cool_functions.py
```

Shortly, I'll explain how submodules work. But, first, let me dramatically re-enact something that has happened to me multiple times. This is what it looks like to use submodules without understanding them.

## A day in the life of someone who doesn't understand submodules
Ah, 2012. What a time to be a "full-stack engineer"! I wonder what contributions await me on the main branch!

(For the sake of readability, in this article, instead of using real commit SHAs, I'm going to use fake descriptive ones.)

Let's pull to make sure I'm up-to-date with the remote.

```bash
$ [/webapp] git pull

remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (1/1), done.
remote: Total 2 (delta 1), reused 2 (delta 1), pack-reused 0
Unpacking objects: 100% (2/2), 237 bytes | 118.00 KiB/s, done.
From https://github.com/dmazin/webapp
   webapp_old_commit_sha..webapp_new_commit_sha  main -> origin/main
Updating webapp_old_commit_sha..webapp_new_commit_sha
Fast-forward
 library | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

After I pull, I like to confirm that my working tree is clean.

```
$ [/webapp] git st

## main...origin/main
 M library
```

What's this? I've made modifications to `library`? I never touch that directory.

It's weird that I've modified a _directory_. Usually git just says I've modified a specific _file_.

Well, what does `git diff` have to say?

```
$ [/webapp] git diff

diff --git a/library b/library
index library_old_commit_sha..library_new_commit_sha 160000
--- a/library
+++ b/library
@@ -1 +1 @@
-Subproject commit library_new_commit_sha
+Subproject commit library_old_commit_sha
```

Apparently, I deleted `Subproject commit library_new_commit_sha` and added `Subproject commit library_old_commit_sha`.

Surely I didn't do that. That's weird, let me do a hard reset.

```
$ [/webapp] git reset --hard origin/main

HEAD is now at webapp_new_commit_sha point submodule to newest commit
```

Did it make the git diff go away?

```
$ [/webapp] git st

## main...origin/main
 M library
```

It did not! I am really confused now!

Well, the usual way I make local modifications go away is `git reset --hard`, and that didn't work. The other way is to commit the changes.

(Sometimes, people don't even notice the diff above, and accidentally do this.)

**My future self**: *Don't do it! If you `git add` that change, you'll be rolling back a change someone else made!*

What's going on, of course, is that `library` is a submodule, and you have to do special stuff to deal with them.

Let's dive into submodules.

## What's a submodule?
A git submodule is a full repo that's been nested inside another repo. Any repo can be a submodule of another.

So, `library` is a full repo that has been nested inside `webapp` as a submodule.

That doesn't seem so confusing, does it? However, there are two important, and tricky, facts about submodules. These facts are why so many people trip up on submodules.

### 1. A submodule is always pinned to a specific commit
You know how package managers let you be  fuzzy when specifying a package version ("get me any version of `requests` so long as it's 2.x.x"), or to pin an exact version ("use `requests` 2.31.0 exactly")?

Submodules can _only_ be pinned to a specific commit. This is because a submodule isn't a package; it's code that you have embedded in another repo, and git wants you to be precise.

We'll see exactly how this pinning works shortly.

### 2. git does not automatically download or update submodules
If you clone `webapp` afresh, git _will not_ automatically download `library` for you (unless you clone using `git clone --recursive`)

Similarly, if a collaborator pins `webapp` to a new commit of `library`, and you `git pull` `webapp`, git _will not_ automatically update `library` for you.

This is actually what's happening in the dramatic re-enactment above. Let me rewind a little bit to show what happened.

## What happens when someone updates a submodule?
In the beginning, `webapp` pointed to `webapp_old_commit_sha`, which pinned `library` to `library_old_commit_sha`.

<img src="/assets/submodules1.png" alt="Hand-drawn diagram of two git repositories, webapp and library. It shows that the old_sha commit of the webapp repo points to the old_sha commit of the library repo. The old_sha commit of the webapp repo has a purple border around it, saying 'HEAD'. The old_sha commit of the library repo also has a purple border around it, saying 'HEAD'.">

(Think of `HEAD` as "current commit".)

Then, my collaborator made changes to `library`. Remember, `library` is a full repo, so after they did their work, they did what you always do after you make changes: they committed and pushed the new commit, `library_new_commit_sha`.

They weren't done, though. `webapp` must point to a specific commit of `library`, so in order to use `library_new_commit_sha`, my collaborator then pushed a new commit to `webpapp`, `webapp_new_commit_sha`, which points to `library_new_commit_sha`.

Here's the thing, though! _git does not automatically update submodules_, so `library` still points to `library_old_commit_sha`.

<img src="/assets/submodules2.png" alt="Hand-drawn diagram of two git repositories, webapp and library. It shows that the old_sha commit of the webapp repo points to the old_sha commit of the library repo. The new_sha commit of the webapp repo points to the new_sha of the library repo. The new_sha commit of the webapp repo has a purple border around it, saying 'HEAD'. The old_sha commit of the library repo has a purple border around it, saying 'HEAD'. A red arrow points to the purple border around old_sha in the library repo. The red arrow is linked to a speech bubble which says, 'library still points at old_sha!'">

I think this will be a lot less confusing if we look at exactly how git tracks submodules.

## How git tracks submodules
### How does git pin a submodule to a specific commit?
The latest commit of `webapp` is `webapp_new_commit_sha`. Let's inspect that commit.

A commit is just a file on disk. However, it's optimized/compressed, so we use a built-in utility to view it. Here's what the commit stores.
```
$ [/webapp] git cat-file -p `webapp_new_commit_sha`

tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4
parent webapp_old_commit_sha
author Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000
committer Dmitry Mazin <dm@cyberdemon.org> 1708717288 +0000

point submodule to newest commit
```

What we care about is `tree 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4`. The _tree_ object represents the directory listing of your repo. When you think trees, think directories.

Let's inspect the tree object.
```
$ [/webapp] git cat-file -p 92018fc6ac6e71ea3dfb57e2fab9d3fe23b6fdf4

100644 blob     6feaf03c7a9c805ff734a90a245a417e6a6c099b    .gitmodules
100644 blob     a72832b303c4d4f1833da79fc8a566e8a0eb37af    README.md
040000 tree     a425c23ded8892f901dee7fbc8d4c5714bdcc40d    tests
160000 commit   library_new_commit_sha                      library
```

Note how `tests` is a `tree` (just like directories can hold directories, trees can point to trees).

But `library` is a... commit?!

```
160000 commit   library_new_commit_sha                      library
```

That weirdness, right there, is precisely how git knows `library` points to `library_new_commit_sha`.

In other words, the way git implements submodules is by doing a weird trick where a tree points to a _commit_.

<img src="/assets/submodules3.png" alt="Hand-drawn diagram showing the text 'webapp_new_commit_sha' connected, via arrow, to 'tree a425' which is itself connected, via arrow, to 'library_new_commit_sha'">

Let's use this knowledge to understand the `git diff` from earlier.

## Understanding git diff
Here's the diff again.

```
$ [/webapp] git diff

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

git has no idea which change happened first. It only knows that your working tree is different from the commit. And, so it tells you: `library_new_commit_sha` is saying that library should point to `library_new_commit_sha`, but it doesn't.

Understanding the above took the pain out of submodules for me. However, I still haven't told you how to update a submodule.

## How to update a submodule
We now understand that we need to point `library` to `library_new_commit_sha`. How?

Because `library` is a full repo, I could just `cd` into it and literally check out that commit:
```
$ [/webapp] cd library

$ [/library] git checkout library_new_commit_sha

Previous HEAD position was library_old_commit_sha README
HEAD is now at library_new_commit_sha add some cool functions
```

If we go back into `webapp`, we'll see that `git st`/`git diff` finally look clean.

```
$ [/webapp] git st

## main...origin/main
# (no output)

$ [/webapp] git diff

# (no output)
```

However, you don't actually need to do the above.

## How to really update a submodule
From `webapp`, we can invoke `git submodule update`. This updates _all_ of a repo's submodules.

People often use certain flags with `git submodule update`, so let's understand them.

### Initialize a submodule: `git submodule update --init`
Remember how I said that if you `git clone webapp`, git won't actually download the contents of `library`?

What you're supposed to do is, after cloning webapp:
1. Run `git submodule init` to initialize the submodules. This doesn't actually download them, though 🙃️.
2. Run `git submodule update` to actually pull the submodules.

This is kind of a silly dance, so git lets you just do `git submodule update --init`. This initializes any submodules and updates them in one step. I _always_ pass `--init` because there is no harm in doing so.

You can skip `--init` by cloning with `--recursive`: that is, you could have done `git clone webapp --recursive`. I never remember to do this, though. Plus, you end up having to do `git update submodule` anyway.

### Update submodules of submodules: `git submodule update --recursive`
Submodules can nest other submodules. Yeah.

So, to take care of updating submodules _all the way down_, pretty much just always pass `--recursive` to `git submodule update`.

**So, the command I always end up using is `git submodule update --init --recursive`.**

### Make git automatically update submodules: `git config submodule.recurse true`
`submodule.recurse true` makes submodules automatically update when you `git pull`, `git checkout`, etc. In other words, it makes submodules automatically point to whatever they are supposed to point to. It's only available in git 2.14 and newer.

That makes running `git submodule update` unnecessary.

I don't use this setting, because I'm not sure if there are drawbacks or not. Plus, I work on submodules enough that I think it could cause conflicts. Let me know if you're aware of shortcomings, or if you've been using this setting forever without issue!

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

But note that `git submodule update --remote` will do this to _all_ your submodules. You likely do not want that.

For that reason, you have to do `git submodule update --remote -- library` to limit this to library only. (If you're thrown off by the fact that you have to do `-- library` -- yeah, it's kind of weird.)

Because `--remote` might accidentally update all the submodules, honestly I usually do the "without a command" method.

### The .gitmodules file
How does git know where to download `library` from?
git uses a file called `.gitmodules` to track the basic facts of a submodule, like the repo URL.

```
$ [/webapp] cat .gitmodules

[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
```

The nice thing about `.gitmodules` is that it's a regular file, tracked the regular way in git. That makes it not confusing.

(What I don't understand is, why git didn't just put the submodule commit right in .gitmodules? The commits of `webapp` would _still_ be able to specify exact commits of `library` to use. What am I missing?)

### Making submodules use branches other than main
If you want to, you can make `library` track whatever branch you want. Otherwise, it defaults to whatever the "main" branch is.

```
[submodule "library"]
        path = library
        url = https://github.com/dmazin/library.git
        branch = staging
```

Thanks for reading!
