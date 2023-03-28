---
layout: post
title: "Using ChatGPT to generate git commit messages"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-27
tags: ai, featured
---
More than 5 years ago, in my [Pink Lexical Slime article](/2017/12/12/pink-lexical-slime.html), I warned that choosing an AI’s suggestion over coming up with your own was tantamount to voting yourself out of existence.

And yet here I am, using ChatGPT all day long. Here’s how I try to resolve in my head: there are some things I actually am totally happy to stop doing. (Plus, I'm using ChatGPT more and more to speed up my own learning, rather than to stop doing things.)

One of these things I'm completely happy to stop doing is writing git commit messages. I do not like writing commit messages. I am lazy.

And so, I’ve recruited ChatGPT to write commit messages for me. I called this tool `git-lazy-commit` (and, yes, ChatGPT came up with this name).

Let’s cut straight to the demonstration.

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

**gpt-3.5-turbo output**: Refactored uncommitted changes retrieval to use subprocess instead of git module

**gpt-4 output**: Replaced gitpython with subprocess for getting staged changes and updated Assistant class methods accordingly

```diff
diff --git a/assistant/core.py b/assistant/core.py
index 7524ed4..d5a7a28 100755
--- a/assistant/core.py
+++ b/assistant/core.py
@@ -14,7 +14,7 @@ class Assistant:

     def get_uncommitted_changes(self):
         staged_changes = subprocess.run(["git", "diff", "--staged"], capture_output=True, text=True)
-        uncommitted_changes = staged_changes.stdout.split('\n')
+        uncommitted_changes = staged_changes.stdout.strip().split('\n')
         return uncommitted_changes

     def generate_commit_message(self, changes_summary):
```

**what I'd have written if I wasn't lazy**: Strip trailing whitespace because it breaks a test otherwise

**gpt-3.5-turbo output**: Fix parsing of staged changes output to remove empty line.

**gpt-4 output**: Improved parsing of uncommitted changes output

## Installation
Full documentation is available in the [README](https://github.com/dmazin/git-lazy-commit/blob/main/README.md).

You can get it from PyPi: `pip install git-lazy-commit`. You’ll need an [OpenAPI API key](https://platform.openai.com/account/api-keys).

And here's the [source repo](https://github.com/dmazin/git-lazy-commit). Feel free to open issues and make pull requests! One of the funny things about ChatGPT is that additions and modifications are not daunting given that ChatGPT is so good at generating code.

## Caveats
It works surprisingly well. Sometimes, I have to ask it a couple times before it describes my changes well. Of course, I doubt this will work super well for giant diffs. Interestingly, I am finding myself committing more often so that it’s easier for `git-lazy-commit` to generate its changes. This is a positive change: I think breaking work into multiple commits makes tracking your work easier. We just don’t do it because writing commit messages is a bit of a pain!

One thing you may notice between my commit messages and ChatGPT’s is that I explain the why while ChatGPT doesn’t. This is important. I think the output is great for quick, lazy commits in personal projects, but for collaborative projects you should add the *why* as well.

## How it works
`git-lazy-commit` uses ChatGPT’s API, which I’m using via a module written by [Simon Willison](https://til.simonwillison.net/gpt3/chatgpt-api). Everything about my assistant is plumbing, except for the system message. This is the core prompt for the assistant. For example, it’s used to establish the personality and identity of the assistant. Right now, the system message is: “Please generate a commit message given the output of `git diff`. Please respond with a commit message without any additional text.”

Anyway, give it a spin, and feel free to report any issues and make suggestions in the repo. I am especially interested to see other people’s examples, and will add them to this blog post and the repo if you send me any really good ones. Thanks for reading!