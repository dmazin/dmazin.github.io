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

But if you clone `inner-repo` afresh, you'll see that `inner-repo` is utterly empty:
```
$ ls -halt inner-repo
total 0
drwxr-xr-x  5 dmitry  staff   160B 19 Feb 20:33 ..
drwxr-xr-x  2 dmitry  staff    64B 19 Feb 20:28 .
```

So, how does git know anything about `inner-repo`? For example, if we ran `git submodule update --init --recursive` to actually initialize `inner-repo`, how would git know where to get the data?

This is stored in a file `.gitmodules`:
```
$ cat .gitmodules
[submodule "inner-repo"]
        path = inner-repo
        url = https://github.com/dmazin/inner-repo.git
```

`.gitmodules` lives in your normal repo, _outside_ `.git`. That is, it's totally normal file. This is nice, because changes to it are tracked just like any other file.

# How does git know which commit of the submodule to point to?
However, that file doesn't store everything. The outer repo _always_ points at a specific commit of the submodule: it never simply points at a branch or tag (the way that a Docker image can always point at "latest").

It has always confused me that `.gitmodules` does not store this information. If not there, then where is it stored?

One would be tempted to say that this is stored in the git repo of the submodule, that is, somewhere in `inner-repo/.git`. But, no, recall that `inner-repo` is empty.

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
```
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
