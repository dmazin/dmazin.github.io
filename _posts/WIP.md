---
layout: post
title: "TODO"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: TODO
---
## Intro
My friends, I want to tell you a shocking thing I learned some time ago about files in Linux. I was living my little life, carrying around a mental model. This mental model got me far, but it was wrong. The truth is more interesting, and you may agree. Read on.

## The (wrong) mental model
Files live on disk, right? So, if you write to a file – like, if you `echo "foo" > bar.txt` – that means you are writing to disk.

This mental model is kind of useful, but false.

"Of course it's false," you may say. *You* don't write to disk. The operating system does. You just ask the operating system to.

No, the above mental model is way more false than that.

The model subtly implies that the write is durable – that is, once the `echo >` command returns, that means the data has been written to disk, and you can unplug your computer.

I certainly used to think this, because I equated files with disks.

In fact, after the command returns, *10s of seconds* might pass before the data is written to disk! Is is *not* safe to unplug your computer.

## Is this shocking?
If this is shocking to you, then this article will teach you some exciting things about how Linux handles file writes (and reads).

If it's not shocking to you, then you are an old hand, but you may still want to read this article so you can laugh at whatever I got wrong. (And, please, do email me, so I can learn).

## Why is this shocking?
You know how, sometimes, during a major technical disagreement at work, it turns out that you and your interlocutor were using conflicting definitions of some concept?

I think the reason my intro is shocking is caused by the fact that the words "write" and "read" can actually mean two different things. Let's set both definitions.

## Logical vs physical IO
When you do `echo "foo" > bar.txt`, this is called a *logical* write. You are saying: operating system, please write this data to a file.

This is opposed to a *physical* write: that is when the disk driver actually causes data to get physically encoded to a disk.

Similarly, files are *logical* concepts. They are abstractions. They are not the same thing as the actual *physical* data that lives on disk. When you do a *logical* write, you are operating on files, which are *not* the same thing as data on disk.

## Now, we can say why the above was shocking.
Now we can unpack my original sentence and show where the subtle errors lie.

> Files live on disk, right?

No, not really. Files are logical entities that represent disk data (of course, files can represent things that have nothing to do with disks, like sockets, but that's not what we're talking about here).

> So, if you write to a file – like, if you `echo "foo" > bar.txt` – that means you are writing to disk.

So, no. If you write to a file, you are telling the operating system that it should put "foo" in a file called bar.txt. Again, files are logical entities, so the operating system has *a lot* of room to interpret how, and when, the disk should be accessed.

## So, what really happens?
Let me tell you what really happens (by default) when you do `echo "foo" > bar.txt`: the kernel copies the bytes representing "foo" into a special place in RAM. As soon as the kernel has copied those bytes, it returns successfully.

So, in fact, `echo "foo" > bar.txt` writes things *to memory*, not to disk.

How do those bytes end up on disk? The simplest case is that a periodic kernel task (which, by default, runs every 30s) notices those bytes, and writes them to disk.

## Why does it work like that?
Simply put, because we want computers to be fast, and disks are the slowest thing about computers.

Indeed, before SSDs, spinning hard drives were the last remaining mechanical thing about computers (aside from fans).

So, we have this world of computers, where we are pushing electrons around super fast. And yet, in the midst of all that, we have to push *atoms* around? Ugh! That is, of course, orders of magnitude slower.

So, operating systems do what they can to protect fast applications from slow disk access.

## Analogy: the like button
Let's say that you made a photo sharing app with Like buttons. Photo Likes *are* stored in a database. However, if you personally were designing this app, would you...
1) Make the Like heart appear as soon as the user clicks the Like button?
2) Make the Like heart appear only once the Like has been persisted to disk?

You would go with option 1, of course, because UI responsiveness is really important.

This is an example of shielding a user from a slow operation via *asynchronicity*: you tell the user you did it, but actually, you do it later when they're not looking.

## Just how slow are disk writes?
My above analogy is a little inaccurate, because it makes it sound like the operating system is *lying*. It's not. `echo >` returns only once "foo" has been successfully copied to a special place in kernel memory. It's not like it returns as soon as you literally hit the enter key.

In other words, `echo >` doesn't return instantaneously. It's still only as fast as memory access, which takes, like, ~100 nanoseconds.

What if, instead, we really did want `echo >` to return only after "foo" was written to disk?

For an SSD, it would be *1,000x* slower.
For a spinning hard drive, it would be *1,000,000x* slower.

Orders of magnitude! It's a LOT!

## This is a trade-off
Of course, this is a trade-off. We have seen the benefit: a 1,000-1,000,000x speedup. What is the cost?

The cost has probably been nagging at you this whole article. Isn't there a chance that we will lose power before "foo" is written to disk, and the data will be lost? Yes! That is the cost.

## What if we do need durable writes?
What about databases? What about writes that really do need to be durable?

What I have described so far is *default* behavior.

Linux expects programmers who need durable writes to manually request durable writes.

### `sync` syscall
There's a system call called `sync`, which you can call at any time. It says: operating system, write everything in your special memory space to disk NOW! And the operating system will do it. You can even put that command into your shell right now just to try it.

In programs, you will often see a series of logical writes (which, remember, don't actually cause anything to get written to disk). Every `n` writes, there will be a `sync` call. This allows applications to finely tune the trade off between durability and write throughput.

### `O_SYNC` file mode
Programmers can also open a file in `O_SYNC` mode. This works much closer to the mental model at the beginning of this article: the logical write only completes once the data has been written to disk.

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

## There are names for this
When I originally wrote this article, I explained things in a lot more detail, but I decided to instead remove that information so I could focus on the behavior. That said, I want to mention some names, as names are useful.

### Look-aside caching
You are likely familiar in look-aside caching. This is when an application first looks in a cache, and if it doesn't find what it's looking for, looks in a database.

I define this merely so we can compare it to what Linux does.

### Write-back caching
This is how Linux handles writes by default: your application writes to the cache, and the cache asynchronously writes to disk.

### Read-through caching
This is how Linux handles reads by default: your application reads from the cache. If the cache has the data, it returns it. Otherwise, first the cache loads the data from disk into itself, then returns it.

## Write-through caching
This is how Linux handles writes in `O_SYNC` mode: your application *still* writes to the cache. However, the cache returns only once it has *synchronously* written to disk.

## Page cache
Earlier, I mentioned that the kernel copies the data to a "special place in memory". This special place is fascinating, and has tons of features and interesting details. This place is called the page cache. A common way to define the page cache is "where the kernel caches files" but I think it's more interesting to think of the way I presented it in this article: the page cache is how the kernel shields you from the slowness of the disk.