---
layout: post
title: "Using ChatGPT to generate git commit messages"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-27
tags: ai, featured
---
More than 5 years ago, in [Pink Lexical Slime](/2017/12/12/pink-lexical-slime.html), I warned that choosing an AI’s suggestion over coming up with your own idea was tantamount to voting yourself out of existence.

> To put it bluntly, when you use [Gmail's] Smart Reply, you are effectively voting for your own agency to be automated away.

It's worth revisiting that article, but this blog post is not about that.

This post is about augmenting a tedious task that I, honestly, mislike: writing commit messages. I am lazy.

So, I wrote a tool, `git-lazy-commit`, which comes up with a surprisingly good commit message for your staged git changes.
You can install it from PyPi: `pip install git-lazy-commit`.

Here are some examples.

Here's a diff from my [dotfiles repo](https://github.com/dmazin/dotfiles) where I added the direnv plugin to oh-my-zsh.
```diff
diff --git a/.zshrc.base.inc b/.zshrc.base.inc
index 7926b0e..34b54e0 100644
--- a/.zshrc.base.inc
+++ b/.zshrc.base.inc
@@ -11,6 +11,7 @@ plugins=(
   asdf
   # git clone https://github.com/davidparsson/zsh-pyenv-lazy.git ~/.oh-my-zsh/custom/plugins/pyenv-lazy
   pyenv-lazy
+  direnv
 )
```

**what I'd have written if I wasn't lazy**: add direnv plugin to oh-my-zsh

**git-lazy-commit**: feat: add direnv plugin to zshrc base configuration file

Note that it prepended a silly little "feat:". It's not perfect! For this reason, git-lazy-commit lets you edit the proposed commit message using the $EDITOR of your choice.

Another example, Here's a diff from the git-lazy-commit repo, where I gave up trying to generate the diff with GitPython (I couldn't figure out how to make its output match `git diff --staged`) and just used subprocess to run `git diff --staged` directly.
```diff
diff --git a/assistant/core.py b/assistant/core.py
index bea132e..7524ed4 100755
--- a/assistant/core.py
+++ b/assistant/core.py
@@ -1,6 +1,7 @@
 import os
-import git
 import argparse
+import subprocess
+import git
 from .chatbot import ChatBot


@@ -11,8 +12,9 @@ class Assistant:
             model=model,
         )

-    def get_uncommitted_changes(self, repo):
-        uncommitted_changes = repo.git.diff().split("\n")
+    def get_uncommitted_changes(self):
+        staged_changes = subprocess.run(["git", "diff", "--staged"], capture_output=True, text=True)
+        uncommitted_changes = staged_changes.stdout.split('\n')
         return uncommitted_changes

     def generate_commit_message(self, changes_summary):
@@ -47,7 +49,7 @@ def main(args=None):
     assistant = Assistant(args.model)

     repo = git.Repo(os.getcwd())
-    uncommitted_changes = assistant.get_uncommitted_changes(repo)
+    uncommitted_changes = assistant.get_uncommitted_changes()
     changes_summary = "\n".join(uncommitted_changes)
     generated_commit_message = assistant.generate_commit_message(changes_summary)
```

**what I'd have written if I wasn't lazy**: Use subprocess to call `git diff --staged` instead of trying to get gitpython to print what I want

**git-lazy-commit**: Refactored uncommitted changes retrieval to use subprocess instead of git module

Here, I wanted update the interactivity of git-lazy-commit to allow a user to enter "y" instead of "yes" (etc). I also bumped the version.
```diff
diff --git a/assistant/core.py b/assistant/core.py
index 2c59fd1..d1c2fee 100755
--- a/assistant/core.py
+++ b/assistant/core.py
@@ -39,11 +39,11 @@ class Assistant:
             input("Do you approve this commit message? ((y)es/(n)o/(e)ditor): ").strip().lower()
         )
 
-        if user_input in ["yes", "y"]:
+        if user_input == "yes":
             return True, commit_msg
-        elif user_input in ["no", "n"]:
+        elif user_input == "no":
             return False, commit_msg
-        elif user_input in ["editor", "e"]:
+        elif user_input == "e":
             return self.edit_commit_message(commit_msg)
         else:
             print("Invalid input. Please enter 'yes (or y)', 'no (or n)', or 'editor (or e)'.")
diff --git a/setup.py b/setup.py
index fccccad..50e03a7 100644
--- a/setup.py
+++ b/setup.py
@@ -2,7 +2,7 @@ from setuptools import setup, find_packages
 
 setup(
     name="git-lazy-commit",
-    version="0.10",
+    version="0.9",
     packages=find_packages(),
     entry_points={"console_scripts": ["git-lazy-commit = assistant:main"]},
     install_requires=["openai", "GitPython"],
```

**what I'd have written if I wasn't lazy**: allow user to enter "y" instead of "yes" (etc); bump version

**git-lazy-commit**: Update version number in setup.py and add support for 'y' and 'n' input in Assistant class.

## Documentation 
Full documentation is available in the [README](https://github.com/dmazin/git-lazy-commit/blob/main/README.md).

You can get it from PyPi: `pip install git-lazy-commit`. You’ll need an [OpenAPI API key](https://platform.openai.com/account/api-keys).

And here's the [source repo](https://github.com/dmazin/git-lazy-commit). Feel free to open issues and make pull requests! One of the funny things about ChatGPT is that additions and modifications are not daunting given that ChatGPT is so good at generating code.

## Caveats
git-lazy-commit works much better with small diffs. I am finding myself committing more often for this reason. This is a positive change: breaking work into multiple commits makes tracking your work easier. We just don’t do it because writing commit messages is a bit of a pain!

One thing you may notice between my commit messages and ChatGPT’s is that I explain the why while ChatGPT doesn’t. This is important. I think the output is great for quick, lazy commits in personal projects, but for collaborative projects you should add the *why* as well. This is why `git-lazy-commit` augments, rather than replaces you: use it come up with an initial message, and then add the why. (This is honestly something we aren't great at in general. Our comments, commit messages, and PR descriptions need a lot more "why".)

## How it works
`git-lazy-commit` uses ChatGPT’s API, which I’m using via a module written by [Simon Willison](https://til.simonwillison.net/gpt3/chatgpt-api). Everything about my assistant is plumbing, except for the [system message](https://platform.openai.com/docs/guides/chat/instructing-chat-models). This is the core prompt for the assistant. For example, it’s used to establish the personality and identity of the assistant. Right now, the system message is: “Please generate a commit message given the output of `git diff`. Please respond with a commit message without any additional text.” The first message we send to ChatGPT is the diff. If a user is unhappy with the output, we say to ChatGPT, "Please come up with another commit message for the diff I sent earlier".
