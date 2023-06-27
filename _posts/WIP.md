---
layout: post
title: "Linux writes don't work the way you think they do"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: TODO
description: I examine a common, but wrong, mental model of file writes in Linux.
---
My friends – programmers and operators – I would like to talk to you about the way that file writes work in Linux.

I used to think they worked a certain way, and I think most people think the same. It turns out that they work very differently. The way they really work is interesting, and important to know.

Let us begin by stating the way I used to think file writes worked.

Files live on disk. So, if you write to a file – like, if you `echo "foo" > bar.txt` – that means you are writing to disk.

<!-- TODO explain echo and > -->

It turns out that most of the words in that statement are wrong! In this article, we will explore that statement in detail, and then discuss how writes *actually* work.

The goal of this article is to explore the difference between a thing, and an abstract representation of that thing. It's kind of out there, but hopefully we can make sense by repeatedly circling it until we get closer to the center.

So, let's start by picking that statement apart, piece-by-piece.

> files live on disk

This is not true. You may have heard that, in Unix, "everything is a file." Another way of saying that is "files are used as an abstraction to represent lots of things, like sockets and files on block devices."

That starts to explain why it's not true that files "live" on disk. Something else lives on disk.

What actually lives on disk, then? Well, bytes. These bytes have a structure, and we can refer to this structure as an *inode*. The reality is a bit more complicated, but let's start by saying that *inodes* live on disk.

Let's go back to files, then. What is a file? A file is an abstract interface that lets us do stuff to the inode. You can think of file in object oriented programming terms: a file is an instance of a class, with fields and methods.

Take a look at the next part of my original statement.

> if you write to a file

This helps explain the whole "abstract interface" thing: a file instance has a method that lets you write to it.

Let's finish the statement.

> that means you are writing to disk

This is again not true. *You* are not writing the disk. The operating system is. The *file* is how you ask the operating system to write to disk.

What is the point of this misdirection?

Before we get to that, let's reflect on the fact that our mental model of how file writes on Linux is already changing. We used to think that files lived on disk, but in fact they don't. It's more accurate to say that inodes live on disk, and files are the interface that lets us interact with inodes.

Now, let's try to answer the point of that misdirection.

Why do computer systems use abstractions?

One reason is that it allows us to define a common interface that can represent lots of different entities. For example, I mentioned above that files can represent both sockets and files on block devices. This means that a programmer can call the same methods for both sockets and block device files without having to learn a new set of methods each time.

That's not the point of this article, though. The point of this article is to deeply examine this sentence I said earlier:

> The *file* is how you ask the operating system to write to disk.

What happens after we ask the operating system to write to disk? What exactly does the operating system do?

That's what we'll focus on for the rest of the article.

Let's rewind a little bit again, and think about the implication of our original mental model. A file lives on disk, so when we write to a file, we write to disk. This implies that when you write to a file, then some data gets written to disk immediately. To put it concretely: we are saying that after `echo "foo" > bar.txt` completes, that means that "foo" has been written to disk and it's safe to unplug the computer.

That is not at all true. In fact, "foo" may be written to disk up to 30 seconds later, asynchronously.

Initially, I found this shocking. But it actually makes sense, and has good reasons. Let's explore them.

Here comes another reason why file abstraction is useful. It allows us to say, hey, operating system, write to this file. And the operating system can say "OK, sure. Done! As far as you're concerned, the write is complete." This misdirection allows the operating system to then implement the actual disk write however it sees fit.

So, again, the mental model: files "live" on disk.

So, if you write to a file – like, if you `echo "foo" > bar.txt` – that means you are writing to disk.

This is false.

"Of course it's false," you may say. *You* don't write to disk. The operating system does. You just ask the operating system to.

No, the above mental model is way more false than that. Let's examine why.

Files don't live on disk: bytes do. If you want to give these bytes a name, *inode* would be more accurate. An *inode* is a structured way to lay some bytes on disk so that we can parse that into an abstract representation called a *file*.

And *that's* what a file is: an abstract representation. A data structure with attributes and methods. It corresponds to 

It's clear where the misconception comes from: files represent bytes that live on disk. However, this is a case of the common mistake of mistaking the map for the territory.

## Why does this distinction matter?
Well, if you say that files *are* the things that live on disk, that implies that if you write to a file, you are writing to disk.

That is, once the `echo >` command returns, that means the data has been written to disk, and you can unplug your computer. In other words, by default, file writes in Linux are *durable*. They are not (by design!).

In fact, after the command returns, *10s of seconds* might pass before the data is written to disk! Is is *not* safe to unplug your computer.

## Is this shocking?
If this is shocking to you, then you are about to have your mental model updated in a useful way.

If it's not shocking to you, then you are an old hand, but you may still want to read this article so you can laugh at whatever I got wrong. (And, please, do email me, so I can learn).

## So, what really happens when we write?
Let me tell you what really happens (by default) when you do `echo "foo" > bar.txt`: the kernel copies the bytes representing "foo" into a special place in memory called the *page cache*. As soon as the kernel has copied those bytes, it returns successfully.

So, in fact, `echo "foo" > bar.txt` writes things *to memory*, not to disk.

How do those bytes end up on disk? The simplest case is that a periodic kernel task (which, by default, runs every 30s) notices that the page cache has data that hasn't yet been written to disk, and writes it to disk.

## Why does it work like that?
First, consider this analogy.

Let's say that you made a photo sharing app with Like buttons. Photo Likes *are* stored in a database. However, if you personally were designing this app, would you...
1) Make the Like heart appear as soon as the user clicks the Like button?
2) Make the Like heart appear only once the Like has been persisted to disk?

You would go with option 1, of course, because UI responsiveness is really important.

This is an example of shielding a user from a slow operation via *asynchronicity*: you tell the user you did it, but actually, you do it later when they're not looking.

Now, let's tie this to disks.

Simply put, we want computers to be fast, and disks are the slowest thing about computers.

Indeed, before SSDs, spinning hard drives were the last remaining mechanical thing about computers (aside from fans).

So, we have this world of computers, where we are pushing electrons around super fast. And yet, in the midst of all that, we have to push *atoms* around? Ugh! That is orders of magnitude slower.

So, operating systems do what they can to protect fast applications from slow disk access.

Specifically, in this case the write happens later, asynchronously. The application is dependent only on (relatively fast) memory access. There's a name for this: write-back caching. That means that the application only talks to the cache, and not to the disk. The cache is responsible for (asynchronously) writing to the disk.

## Just how slow are disk writes?
`echo >` returns after "foo" has been successfully copied to a special place in kernel memory.

In other words, `echo >` doesn't return instantaneously. It's still only as fast as main memory access, which takes, like, ~100 nanoseconds.

What if, instead, we really did want `echo >` to return only after "foo" was written to disk?

For an SSD, it would be *1,000x* slower.
For a spinning hard drive, it would be *1,000,000x* slower.

Orders of magnitude! It's a LOT!

## This is a trade-off
Of course, this is a trade-off. We have seen the benefit: a 1,000-1,000,000x speedup. What is the cost?

The cost is that you could lose power before "foo" is written to disk, and the data will be lost.

Operating system designers have decided that this is a worthwhile tradeoff. Things would simply be too slow if we treated all writes as synchronous and durable.

## What if we do need durable writes?
This thought might be nagging at you: what about databases? What about applications where durability is, like, the main feature?

What I have described so far is *default* behavior.

Linux expects programmers who need durable writes to manually request durable writes.

### `sync` syscall
There's a system call called `sync`, which you can call at any time. It says: operating system, write everything in your special memory space to disk NOW! And the operating system will do it. You can even put that command into your shell right now just to try it.

In programs, you will often see a series of writes (which, remember, don't actually cause anything to get written to disk). Every `n` writes, there will be a `sync` call. This allows applications to finely tune the trade off between durability and write throughput.

### `O_SYNC` file mode
Programmers can also open a file in `O_SYNC` mode. This works much closer to the mental model at the beginning of this article: the write only completes once the data has been persisted to disk.

## A demonstration
Any good science lesson ends with a good demonstration. So, let me demonstrate to you the following.

I'm going to write to a file. I'm going to read from that file. I will show you that both operations will complete *long* before the disk is accessed.

This is what I'm going to paste into my shell, all at once.
```
echo "foo" > example.txt
dd if=example.txt
```

And here is the output of `bpftrace`, which allows us to trace kernel events.

Specifically, we are going to trace when `vfs_read` and `vfs_write` finish (all you need to know is that those are the functions that are called when we read and write a file). We will also trace `block_rq_issue`, which is called when the disk driver actually writes to disk.

```
# write starts
11:32:05 kfunc:vmlinux:vfs_write
comm=zsh

# write finishes
11:32:05 kretfunc:vmlinux:vfs_write
comm=zsh

# read starts
11:32:05 kfunc:vmlinux:vfs_read
comm=dd

# read finishes
11:32:05 kretfunc:vmlinux:vfs_read
comm=dd

# **5 seconds later** we actually write "foo" to disk!
11:32:10 tracepoint:block:block_rq_issue
rwbs: W, comm: kworker/u24:9
```

Just to re-iterate: we wrote, and read, the file LONG before we actually touched the disk! File IO is not the same thing as disk IO!

## Conclusion
I started this article months ago, when I noticed that a file IO benchmark vastly disagreed with the disk IO throughput reported by `iostat`. This sent me down a deep rabbit hole. This article originally discussed so many things, like write cancellation and coalescence. Anyway, all that's to say, this is a deep and interesting topic, and you can expect more on it. I urge you to sign up my RSS feed if you're interested.

## P.S. I want to work with Linux
I am a successful SRE and technical leader (feel free to ask for my CV; most recently I ran the infrastructure team at Rollbar). However, I'm tired of the cloud. I am tired of *asking* Google and AWS to run things for me. I want to work with Linux directly. I am based in London and am interviewing for Linux engineering and/or HPC roles. If you have something for me, reach out! I am not picky about titles or levels: I just want to work with Linux.