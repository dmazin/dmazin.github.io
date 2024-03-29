---
layout: post
title: "I Hereby Declare a Sabbatical"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-03
tags: featured
---
In November 2022 I quit my job (I ran the infrastructure team at Rollbar) and moved from New Jersey to London with my wife and 2-year-old. After running around the city a bunch and getting settled, I started preparing for job interviews.

Quickly, though, I found myself unable to make myself waste my soul on LeetCode. You see, I was carrying around all sorts of concepts I wanted to know more about, and LeetCode only got in the way of that.

The period between jobs is an opportunity to “capstone” what you learned in your previous job. For example, I started my Rollbar job with little theoretical education about reliability engineering. Therefore, everything I know in that domain is experiential. This meant that I came across tons of concepts that I knew how to deal with, but did not actually know much about them.

Let me give you an example. We ran a bunch of VMs, for example to host MySQL, and we configured the MySQL process using SystemD. I knew how to work with SystemD, but I knew absolutely nothing about what an init process it, where it stands the overall picture of an operating system, the history and criticism of SystemD, nor did I know anything about Linux features it uses like cgroups.

At a job, we come across many concepts, even work with them, without having much theoretical understanding of what they are. Personally, these little mysteries nag me, and I like to demystify them. At a fast-paced job, there may not be much time to actually demystify concepts, and so, after that job, when you come up for air is a great time to demystify.

And so I started reading and reading, eventually finding the [“Red Book”](https://www.oreilly.com/library/view/unix-and-linux/9780134278308/), a legendary manual of Unix system administration written primarily by similarly legendary sysadmin Evi Nemeth. I have deeply read most of it. (Seriously, if you’re an SRE/sysadmin, or want to understand Linux/Unix better, I highly recommend it.)

Now I comfortably understand SystemD, init, etc. I understand now why it has felt nicer to use SystemD on newer versions of Ubuntu than it felt to work with upstart on older versions.

This is only an example; I repeated this for a ton of concepts. I solidified – capstoned – my 3-year experiential education at Rollbar. I now feel like I actually deserve my staff title.

Anyway, all of this was in the context of preparing for job interviews. At least for SREs, a lot of your interviews are signal-gathering missions. You get asked lots of questions like “tell me what you think of SystemD” or “when would you use PubSub and when would you use AMPQ?” (I know, given that I have administered such interviews). The fact that I have worked with a bunch of these things is one thing, but being able to talk intelligently is another.

The thing is, though, the job market right now is not great. There are lots of jobs, mind you, but the jobs are not terribly attractive. A lot of them involve contracting with legacy businesses to help them make the extremely misguided decision to move everything to AWS Lambda or whatever.

And, so, probably like many tech workers, I now find myself taking a privileged (extremely privileged), yet forced, sabbatical. Instead of studying so that I can get a job, I am now studying, for the foreseeable future, for the sake of pure learning. I’m excited, because I love to learn, am learning tremendously, and have lots of energy.

Here are some of the things I’m interested in, and will be writing about.
* My opinions on cloud services, on which I have seriously soured. Not only are they not necessarily the right choice for a mature engineering operation, but I don’t think it’s a good idea for SREs to hyper-focus on cloud services in terms of professional development.
* Percona-style experiments with MySQL (e.g. somehow, one of your partition files is about to hit the 16 TB ext4 limit, what can you do?).
* Exploration and learning of Linux (I am super curious what components actually make a modern distro. What happens if you just boot into a pure Kernel – I know you get a TTY, and I think it’s called single-user mode, but then what? What are the utilities and daemons you want to install and run?).
* Exploration of queueing theory. Observability is one of my biggest interests (the other one is database administration), and I think the math behind it – namely, queuing theory – is fascinating.
* I also hope to write candidly about my experience as a parent in London and as simply a human.

One more thing, and a very important thing. Now that I’m studying for my own pleasure rather than job interviews, I can take it easier. And, so, for the first half of the day I’ve been watching my kid, and I’ll continue this. I mean, I’m present more than your average dad (it’s a super low bar), but the brightest thing about all of this is the chance to spend a lot more time with him.

Anyway, maybe no one will give a shit about this, I mean, look at this guy, congratulations, you have the privilege to not work for 6 months. Indeed, maybe this is boring. But note that I didn’t just talk about myself in this post. I voiced a strong opinion about all of us: I believe that people waste the opportunity to learn between jobs. Instead of building on what we learned in the job we just left, we hyper-optimize for job interviews, especially for LeetCode questions. I could have written a think piece about this opinion, but I find it easiest to write in the context of my own experience.
