---
layout: post
title: "In the Age of AI, Don't Let Your Skills Atrophy"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-29
tags: ai, featured
description: Exploring the balance between relying on AI assistance like ChatGPT and maintaining personal skills in a world of increasing AI capabilities.
---
Feel free to check out the [Hacker News discussion](https://news.ycombinator.com/item?id=35361979).

I grew up in Los Angeles. I have driven hundreds of thousands of miles. Yet, I astonish my friends with my lack of road knowledge. The reason is simple: I rely entirely on Google Maps, turning off my brain and following directions. My skills as a road navigator have atrophied.

Google Maps isn't new, and the dilemma of letting a mental muscle atrophy due to technological innovations—such as handwriting skills declining due to typing—is nothing new either. What _is_ new is the increasing accessibility and skill of AI.

Technology, and specifically AI, has mostly resided in the domain of the most basic tasks. Autocorrect, which totally fits the modern (weak) definition of AI, [enables the very simple act of typing on a smartphone](/2017/12/12/pink-lexical-slime.html). Large language models and other forms of AI, though, are shockingly climbing the ranks of task complexity.

This development has led to a mix of worries and excitement about AI's potential impact. Personally, I'm experiencing a renaissance, as I'm now able to follow through on ideas that would have previously remained idle. Simon Willison [recently described it best](https://simonwillison.net/2023/Mar/27/ai-enhanced-development/):

> The thing I'm most excited about in our weird new AI-enhanced reality is the way it allows me to be more _ambitious_ with my projects.  
> In the past I've had plenty of ideas for projects which I've ruled out because they would take a day—or days—of work to get to a point where they're useful. I have enough other stuff to build already!  
> But if ChatGPT can drop that down to an hour or less, those projects can suddenly become viable.  
> Which means I'm building all sorts of weird and interesting little things that previously I wouldn't have invested the time in.

As a personal example, I am able to add visual explanations and other front-end tricks to my articles that I previously would not have bothered with. I am also writing utilities ([git-lazy-commit](/2023/03/28/git-lazy-commit.html) and [disk-stat-diff](https://github.com/dmazin/disk-stat-diff)) that I would previously have found too difficult or tedious to write.

There are many tasks, even complex ones, where I am fine with the fact that AI can do it while I cannot. I used ChatGPT to write a [tedious function](https://github.com/dmazin/disk-stat-diff/blob/main/disk_stat_diff#LL13C8-L13C8) for command-line output, and remain happily ignorant of the specifics. I'm also happy to let AI take over tasks I don't currently find interesting, like generating HTML.

However, each of us must walk a fine line: what are you willing to cede to AI?

Recently, I found that line. I used ChatGPT to write a script that generates data for an upcoming article. (Specifically, I used it to model the query durations against an imaginary service, and calculate some statistics about them.) The data the script generated looked right, so I felt a draw to move on. Here, I realized that I was being pulled toward a previously unknown line, a boundary I knew I must resist. By not delving deeper into the modeling of query durations, I was passing up the opportunity to learn about something I found genuinely interesting.

Finding that line did not, at all, mean ending my relationship with ChatGPT. ChatGPT generated the code I wanted to understand in the first place, and I continued to engage with that code. I used ChatGPT to explain the code (it's infinitely patient, at least until you hit the token limit). I asked ChatGPT follow-on questions and about alternate solutions. This is, by the way, how Simon engages with ChatGPT: to generate code, yes, but also to learn.

What I've realized is that there _are_ tasks I'm willing to let AI handle, as this enables me to take on more ambitious projects. But there is, very much, a line.

With ChatGPT, it's too easy to implement ideas without understanding the underlying concepts or even what the individual lines in a program do. Doing so for unfamiliar tasks means failing to learn something new, while doing so for known tasks leads to skill atrophy.

The age of AI has also very much caused an existential crisis. Need I argue why humans should remain skillful, why I choose to continue learning? I suppose I do. Instead of attempting to argue from a utilitarian standpoint (even though we are nowhere near the fully automated communist utopia), I will cede all utility and skip directly to joy. I am an existentialist: I am perfectly comfortable in a world with no inherent meaning. Sheer existence and action brings me joy. I spend hours of my day tinkering with computers not because of utility, but for the pleasure it brings. It's the same reason one plays piano: enjoyment rather than practicality. This is how I'm drawing the line. If that's not convincing to you, find some other way to draw it, but do draw it. Don't let the negotiation between AI and humans happen without you.
