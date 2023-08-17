---
layout: post
title: "TODO: IO Inflation"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-08-16
tags: featured
description: TODO
---
# Personal note
Hey, everyone. I'm back from vacation, with some good news: in a couple weeks I'll be starting a new job as a "Senior Linux Engineer". I mean, knock on wood -- I don't want to jinx it. Needless to say, I am absurdly excited that, I am *actually* going to get to work on-premises with Linux.

# Introduction
Today, let's continue talking about file and disk IO.

One major idea behind my last two articles was the difference between an abstract interface and the low-level implementation. Specifically, I'm talking about the difference between a file and "the thing that lives on disk that I used to think was called a file".

In [file writes](TODO) I introduced this distinction, and in [bunch of bits]() I dove into the stuff that *actually* lives on disk (namely, the inode, which holds metadata about the file, and the file contents themselves).

In this post, we're going to explore a very dramatic example of the distinction between file IO (specifically, the act of us writing to a file using a syscall) and disk IO (specifically, the act of the kernel sending write requests to the disk driver).

This is what we are going to explore: if you write just 14 bytes to a file (`echo 'Hello, world!' > /data/example.txt`), the ext4 file system will write 73 *thousand* bytes to disk as a result.

That is *very* dramatic, right? I certainly found it shocking when I learned about it.

Why do we need to write so much to disk for such a tiny file? What, even, are we writing?

That's what we'll answer. Read on.

## This post is ext4-specific
Note that this post is ext4-specific, but the general ideas apply to all file systems. I'll try to make it clear when I'm discussing something ext4-specific. That said, like with my other articles, on top of learning about the main subject of the article, you'll walk away having also learned about file systems and ext4.

What's ext4, by the way? It's simply an extremely common file system for Linux. It's the default for many distributions.

And if you haven't read my previous posts, a file system is the software that turns the raw bytes on your disk into your familiar disks and directories.

# Abstraction and implementation
All of computing deals with two sides of a coin: abstraction and implementation.

When you write some piece of software, it needs an interface so others can use it. For example, if you wrote a web app, no one could use it unless you gave it a REST API. We call this the abstraction because, I guess, an interface is a very abstract thing, because it's separated from how something actually works.

Of course, your software needs to actually do something. You need to write some code so that *something* happens when people call your REST API. This is the implementation.

While this seems like theoretical academic stuff, I'm learning that this boundary is the core topic of software design. For example, some hold strongly that a programmer using someone else's software or hardware should focus entirely on the interface. Others contend that we should always expect programmers will learn about the vagaries of any implementation and even come to depend on it. Riding this boundary is what software is all about.

# File IO and its implementation
How does the above relate to file IO?

If you've been following along, you understand that file IO is abstract, and that files are interfaces. Indeed, we call file IO *logical* IO.

But this article is not about file IO. It is all about one specific implementation that powers writes to a disk-backed file, specifically, ext4. There are, of course, many implementations, like other file systems such as ZFS. (This is the power of abstractions: the same interface abstracts away many implementations).

The thing about implementations is that they go all the way down. By this I mean, when I say "implementation", it's not actually clear what I'm talking about. Am I talking about how the C standard library handles writes? How the Linux kernel handles block IO? How it handles IO requests? How the disk driver communicates with the disk hardware? How the disk's on-board controller handles operations? How the disk hardware implements operations? How the disk electronics actually flip the information-carrying bits? And so on.

So, I need to say exactly what level of implementation I'm going to focus on: I am going to focus on what requests ext4, via the Linux kernel, sends to the disk driver. Let me put it more plainly. We are interested in exactly what ext4 and the kernel write to disk when we write to a file.

# IO inflation
As we saw above, ext4 writes *a lot* more to disk than merely the file contents.

Forgive me for introducing one more term, but this is called **IO inflation**. It refers to the fact that, in order to write n bytes to the file, we need to actually write much more to disk.

Note that IO inflation is *not* [write amplification](https://en.wikipedia.org/wiki/Write_amplification). Whereas IO inflation is necessary and fine, write amplification is undesirable.

# Causes of IO inflation
We are ready to start answering our question: why do we write 73 KiB just to store "Hello, world!"?

To answer that, we'll trace all the requests the kernel sends to the disk driver. We're going to use bpftrace (an eBPF-based tracing program with an excellent scripting language) to trace the [tracepoint](TODO) that represents these requests. Specifically, the tracepoint we'll trace is [block:block_rq_issue](TODO).

In one terminal, I'll use `dd` to write to a file. In another, I'll run the tracing program.

First, here's the output when I run `dd`.
```bash
$ echo 'Hello, world!' | dd of=example.txt
0+1 records in
0+1 records out
14 bytes copied, 2.7956e-05 s, 501 kB/s
```

As you can see, `dd` says we wrote 14 bytes – but remember, this is the *logical* size.

Now, here are all the requests the kernel sends to the driver.

(Don't strain to understand these traces yet.)

```
$ sudo bpftrace trace.bt
[sudo] password for dmitry:
Attaching 1 probe...
15:18:53 tracepoint:block:block_rq_issue
rwbs: W, dev: 8388656, sector: 268288, bytes: 4096, comm: kworker/u24:0

15:18:53 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526416, bytes: 28672, comm: jbd2/sdd1-8

15:18:53 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526472, bytes: 4096, comm: kworker/8:1H

15:18:58 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 19016, bytes: 4096, comm: kworker/u24:3

15:18:58 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2600, bytes: 4096, comm: kworker/u24:3

15:18:58 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2576, bytes: 4096, comm: kworker/u24:3

15:18:58 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2048, bytes: 8192, comm: kworker/u24:3

15:18:59 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526480, bytes: 8192, comm: jbd2/sdd1-8

15:18:59 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526496, bytes: 4096, comm: kworker/8:1H

15:19:04 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2632, bytes: 4096, comm: kworker/u24:0

^C

@[W]: 4096
@[WM]: 24576
@[WS]: 45056
@bytes_written: 73728
```

Let's discuss what we see.
* Our file write turned into 10 disk writes!
* You can see that we wrote 73 KiB.
* From the `sector` field, you can see that we wrote in a *bunch* of different locations.

Let's dive deeper. Let me sort these traces by disk location and annotate them.

```
// actual file contents
rwbs: W, dev: 8388656, sector: 268288, bytes: 4096, comm: kworker/u24:0

// metadata: inode table, superblock, etc (TODO)
rwbs: WM, dev: 8388656, sector: 2048, bytes: 8192, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2576, bytes: 4096, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2600, bytes: 4096, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2632, bytes: 4096, comm: kworker/u24:0

// metadata: probably dentry (TODO)
rwbs: WM, dev: 8388656, sector: 19016, bytes: 4096, comm: kworker/u24:3

// journaling
rwbs: WS, dev: 8388656, sector: 526416, bytes: 28672, comm: jbd2/sdd1-8
rwbs: WS, dev: 8388656, sector: 526472, bytes: 4096, comm: kworker/8:1H
rwbs: WS, dev: 8388656, sector: 526480, bytes: 8192, comm: jbd2/sdd1-8
rwbs: WS, dev: 8388656, sector: 526496, bytes: 4096, comm: kworker/8:1H
```

Let's pull up our sleeves and dive into each trace.

# Cause of IO inflation 1: sector size, block size, and memory page size
Let's start with the trace where we actually wrote the file contents:
```
rwbs: W, dev: 8388656, sector: 268288, bytes: 4096, comm: kworker/u24:0
```

How can we prove that this is the trace that wrote the file contents? Well, the trace holds the disk location, so let's print it out.

```bash
$ sudo dd if=/dev/sdd skip=268288 count=8 bs=512 2>/dev/null | hexdump -C

00000000  48 65 6c 6c 6f 2c 20 77  6f 72 6c 64 21 0a 00 00  |Hello, world!...|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000
```

There's our hello world!

But, wait. Did it say `bytes: 4096`? Why did we write 4 KiB if "Hello, world!" only takes 14 bytes?

Ah, so there's our first bit of IO inflation. It turns out that you *cannot* write just 14 bytes to either a spinning disk or SSD.

## Disks are broken up into sectors


# Files are not inodes
In [How does Linux really handle writes?](TODO), we discussed the difference between files and the stuff that actually lives on disk (inodes, etc):

> files are an interface, much like a OOP instance with methods and attributes. The file is not the stuff on disk. It's just the abstract interface for it.

One way to look at this distinction is that a file is a *logical* entity while a disk is a *physical* entity.

The most obvious manifestation of this is the size of a file: 
```
$ dd if=/dev/zero of=/data/example.txt bs=1 count=4
4+0 records in
4+0 records out
4 bytes copied, 0.000562553 s, 7.1 kB/s

$ stat /data/example.txt
  File: /data/example.txt
  Size: 4               Blocks: 8
```

In the stat output, "Size" tells us the *logical* size of the file (4 bytes), while "Blocks" tells us the *physical* size: 8 512-byte blocks, i.e. 4 KiB.

# File IO is not disk IO
It follows from this distinction that file I/O is *logical I/O* and disk I/O is *physical I/O*. When a program writes to a file, this is logical I/O. Physical I/O refers to the process of actual information-carrying bits being flipped on an actual physical device.

As you can see above, when we wrote 4 bytes to the file, the disk actually wrote 4 KiB. This is **IO inflation**: when logical IO size is smaller than physical IO.

It's important to distinguish IO inflation from [write amplification](https://en.wikipedia.org/wiki/Write_amplification). IO inflation is not bad; it's merely a reflection of the fact that we need to store more than just the file contents. Write amplification sounds similar ("the actual amount of information physically written to the storage media is a multiple of the logical amount intended to be written") but write amplification *is* bad.

There is also **IO deflation**. This refers to the fact that we may actually read less from disk than the size of the logical read. A common reason for this is because the file contents were cached in the page cache: in that case, there is *no* physical disk read! (You can see an example of this at the very end of [How does Linux really handle writes?](TODO)).

So, the short answer for why a 1 byte file write turns into a 65 KiB disk write, is "IO inflation". However, we can do much better than that. Let's understand exactly what gets written to disk as a result of our file write.

# Tracing IO inflation
We are going to use bpftrace, which is a very good frontend for eBPF, to trace the communication between the kernel and the disk device driver. Specifically, the kernel doesn't write to disk: it asks the disk driver to do so. It does so by sending it a "block request", which we can trace via a [tracepoint](TODO) called `block:block_rq_issue`.

I want to say that this will let us trace *exactly* what gets written to disk, but the reality is that the disk might do some weird/magical stuff under-the-hood after the driver sends it the write operation. We're going to ignore that, and say that we are tracing exactly what *the kernel* writes to disk.

So, without further ado, here is everything the kernel writes to disk when we do `echo "hello, world" > /data/example.txt` (note that first I deleted `/data/example.txt`).

This output won't make sense yet. But understanding it will solidly explain how 13 (TODO) bytes turns into 65 KiB.

```
```

First, note the sum: 65 KiB!

Second, let me rearrange this output. Whereas previously it was ordered by timestamp, now allow me to order it by disk location, and also annotate what is actually happening.

```
```

So, here's what we have: writing the actual file contents, writing metadata, and journaling. Let's dive into each one.

# Causes of IO inflation
## Disk sectors, file system blocks, and memory pages
The trace that corresponds to writing the file contents is this one:
```
```

How do I know? Well, let's print out the disk contents at the given location ():
```
```

See? There's hello world!

Having proven that this trace corresponds to the file contents, we can note that, whereas the `stat` output above showed that the file contents take up 4 KiB on disk, the trace confirms that, when we wrote the file contents, we wrote 4 KiB.

So... why do the file contents take up 4 KiB?

There are (at least) 3 reasons.

### Disk sectors
The most fundamental is the disk sector size. Even though in a previous post I said that a disk is a "bunch of bits", the reality is that (so far as I know) no computer program can actually write or read a single byte from a disk. Instead, disks are broken up into *sectors*, which these days are 4 KiB in size. Aha! 4 KiB, see?[1]

*Why* are disks broken up into bigger sectors? The biggest reason, probably, is that in order to write or read from a location on disk, you need an address, and it's simply not feasible to address each individual byte. It's not desirable, either: each write has overhead, so writing one byte at a time would be slow. Bigger sectors increase throughput (which is another way of saying that we pay an overall lower cost of overhead).
 
In the 1980s, disk sectors were standardized to 512 bytes each. However, between the early 2000s and 2010s, sectors increased to 4 KiB. I think the biggest reason, rather than throughput, was the fact that each disk sector actually has an error-correcting code at the end. This code is more efficient the bigger the sector. This upgrade was called [advanced format](https://en.wikipedia.org/wiki/Advanced_Format).

The above applies only to hard disks. SSDs are an even more complex story. The short story is that SSDs are broken up into pages, not sectors (but SSDs emulate 512/4 KiB sectors for compatibility). SSD pages are usually 4 to 8 KiB, but they are grouped into blocks. These blocks are 256-512 KiB in size. Therefore, when you read/write an SSD, you actually read/write 256-512 KiB at once. The SSD story is even more complex, because of the whole write amplification thing. The ugly SSD story is something I need to learn about, because SSDs are more relevant than hard disks, which I know more about.

### File system blocks
File systems are similarly broken up. A very common block size today is 4 KiB. I'm not sure about the history of file system block size - though common sense says that it makes no sense for a file system block to be smaller than a disk sector - but I can talk about two of the pressures that determine file system block size.

The first is addressing. Like disks, file system blocks need to be addressed. The bigger a file system block, the more overall storage we can address using the same number of addresseds, which was even more important when computers had a lot less memory.

The second is again throughput. Reading/writing any file system block incurs overhead; the bigger the blocks, the less overall overhead.

### Memory pages
Finally, memory is broken up into pages for the same reason that disks are: addressing. Julia Evans explains it perfectly in this comic, so I won't even paraphrase. (TODO)

The reason memory pages are relevant is that, as we learned in (TODO previous blog post), the kernel writes by dumping an entire memory page to disk. Given that, if you are writing to disk via the page cache (which, unless you specifically ask not to, you almost always are), you can't write (or read) than an entire memory page at once.

## Metadata
Even though I spent a lot of words on the 4 KiB thing, in fact it only accounts for a small amount of what gets written when we write to a file.

## Journaling

# Notes
[1] Some hard disks and SSDs make it seem like their sectors are 512 bytes long, though this is just something they do for the sake of older hardware.

TODO file systems -> filesystems