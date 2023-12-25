---
layout: post
title: "Why does writing 14 bytes to a file write 73 kilobytes to disk?"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-08-17
tags: featured
description: I investigate why writing 14 bytes to a file writes 73 kilobytes to disk. In other words, I explain the difference between file and disk IO, and explain IO inflation.
---
# Personal note
Hey, everyone. I'm back from vacation, with some good news: in a couple weeks I'll be starting a new job as a "Senior Linux Engineer". I mean, knock on wood -- I don't want to jinx it. Given that [6 months ago I set out to switch from screwing around with the cloud to working with Linux](/2023/03/03/declare-sabbatical.html), I am absurdly excited that I am *actually* going to get to work on-premises with Linux.

# Warning
Throughout this article, I found myself referring back to two recent articles:
* [A disk is a bunch of bits](/2023/07/19/bunch-of-bits.html)
* [How does Linux really handle writes?](/2023/06/27/file-writes.html)

While they aren't hard prerequisites, I recommend reading them if you get confused.

# Introduction
Today, let's continue talking about file and disk IO.

One major idea behind my last two articles is the difference between an abstract interface and the low-level implementation. Specifically, I'm talking about the difference between a file and "the thing that lives on disk that I used to think was called a file".

In [How does Linux really handle writes?](/2023/06/27/file-writes.html) I introduced this distinction, and in [A disk is a bunch of bits](/2023/07/19/bunch-of-bits.html) I dove into the stuff that *actually* lives on disk (namely, the inode, which holds metadata about the file, and separately, of course, the file contents themselves).

In this post, we're going to explore a very dramatic example of the distinction between file IO (specifically, the act of us writing to a file using a syscall) and disk IO (specifically, the act of the kernel sending write requests to the disk driver).

This is what we are going to explore: if you write just 14 bytes to a file (`echo 'Hello, world!' > /data/example.txt`), the ext4 filesystem will write 73 *thousand* bytes to disk as a result.

That is *very* dramatic, right? I certainly found it shocking when I learned about it.

Why do we need to write so much to disk for such a tiny file? What, even, are we writing?

That's what we'll answer. Read on.

## This post is ext4-specific
Note that this post is ext4-specific, but the general ideas apply to all filesystems. I'll try to make it clear when I'm discussing something ext4-specific. That said, like with my other articles, on top of learning about the main subject of the article, you'll walk away having also learned about filesystems and ext4.

What's ext4, by the way? It's simply an extremely common filesystem for Linux. It's the default for many distributions.

And if you haven't read my previous posts, a filesystem is the software that turns the raw bytes on your disk into your familiar files and directories.

First, allow me to lay some (probably overly theoretical and verbose) groundwork.

# Abstraction and implementation
All of computing deals with two sides of a coin: abstraction and implementation.

When you write some piece of software, it needs an interface so others can use it. For example, if you wrote a web app, no one could use it unless you (for example) gave it a REST API. We call this the abstraction because, I guess, it abstracts away how something actually works: the implementation.

Of course, your software needs to actually do something. You need to write some code so that *something* happens when people call your REST API. This is the implementation.

# File IO and its implementation
How does the above relate to file IO?

If you've been following along, you understand that file IO is abstract, and that files are interfaces. Indeed, we call file IO *logical* IO.

But this article is not about file IO. It is all about one specific implementation that powers writes to a disk-backed file, specifically, ext4. There are, of course, many implementations, like other filesystems such as ZFS. (This is the power of abstractions: the same interface abstracts away many implementations).

The thing about implementations is that they go all the way down. By this I mean, when I say "implementation", it's not actually clear what I'm talking about. Am I talking about how the C standard library handles writes? How the Linux kernel handles block IO? How it handles IO requests? How the disk driver communicates with the disk hardware? How the disk's on-board controller handles operations? How the disk hardware implements operations? How the disk electronics actually flip the information-carrying bits? And so on.

So, I need to say exactly what level of implementation I'm going to focus on: I am going to focus on what requests ext4, via the Linux kernel, sends to the disk driver. Let me put it more plainly. We are interested in exactly what ext4 and the kernel write to disk when we write to a file.

# IO inflation
As we saw above, ext4 writes *a lot* more to disk than merely the file contents.

Forgive me for introducing one more term, but this is called **IO inflation**. It refers to the fact that, in order to write n bytes to the file, we need to actually write much more to disk.

Note that IO inflation is *not* [write amplification](https://en.wikipedia.org/wiki/Write_amplification). Whereas IO inflation is necessary and fine, write amplification is undesirable.

# Commercial interruption
If you're enjoying yourself, may I ask if you'd like to follow me via [RSS feed](/feed.xml), [Mastodon](https://file-explorers.club/@dmitry), or [Telegram channel](https://t.me/cyberdemon6)? Thanks!

# Causes of IO inflation
We are ready to start answering our question: why do we write 73 KiB just to store "Hello, world!"?

To answer that, we'll trace all the requests the kernel sends to the disk driver. We're going to use bpftrace (an eBPF-based tracing program with an excellent scripting language) to trace the tracepoint that represents these requests (a tracepoint is just a way to trace events in Linux). Specifically, the tracepoint we'll trace is [block:block_rq_issue](https://elixir.bootlin.com/linux/latest/source/include/trace/events/block.h#L220).

In one terminal, I'll use `dd` to write to a file. In another, I'll run the tracing program.

First, here's the output when I run `dd`.
```bash
$ echo 'Hello, world!' | dd of=/data/example.txt
0+1 records in
0+1 records out
14 bytes copied, 2.7956e-05 s, 501 kB/s
```

As you can see, `dd` says we wrote 14 bytes – but remember, this is the *logical* size.

Now, here are all the requests the kernel sends to the driver.

(Don't strain to understand these traces yet.)

```
$ sudo bpftrace trace.bt
Attaching 1 probe...

16:36:09 tracepoint:block:block_rq_issue
rwbs: W, sector: 268288, bytes: 4096, comm: kworker/u24:1

16:36:09 tracepoint:block:block_rq_issue
rwbs: WS, sector: 526408, bytes: 28672, comm: jbd2/sdd1-8

16:36:09 tracepoint:block:block_rq_issue
rwbs: WS, sector: 526464, bytes: 4096, comm: kworker/5:1H

16:36:14 tracepoint:block:block_rq_issue
rwbs: WM, sector: 2048, bytes: 8192, comm: kworker/u24:1

16:36:14 tracepoint:block:block_rq_issue
rwbs: WM, sector: 2576, bytes: 4096, comm: kworker/u24:1

16:36:14 tracepoint:block:block_rq_issue
rwbs: WM, sector: 2600, bytes: 4096, comm: kworker/u24:1

16:36:14 tracepoint:block:block_rq_issue
rwbs: WM, sector: 19016, bytes: 4096, comm: kworker/u24:1

16:36:14 tracepoint:block:block_rq_issue
rwbs: WS, sector: 526472, bytes: 8192, comm: jbd2/sdd1-8

16:36:14 tracepoint:block:block_rq_issue
rwbs: WS, sector: 526488, bytes: 4096, comm: kworker/5:1H

16:36:19 tracepoint:block:block_rq_issue
rwbs: WM, sector: 2632, bytes: 4096, comm: kworker/u24:0

^C

@[W]: 4096
@[WM]: 24576
@[WS]: 45056
@bytes_written: 73728
```

Let's discuss what we see.
* Our file write turned into 10 disk writes!
* You can see that we wrote 73 KiB (look at `bytes_written`).
* From the `sector` field, you can see that we wrote in a *bunch* of different locations.

Let's dive deeper. Let me sort these traces by disk location and annotate them.

```
// actual file contents
rwbs: W, sector: 268288, bytes: 4096, comm: kworker/u24:1

// metadata
rwbs: WM, sector: 2048, bytes: 8192, comm: kworker/u24:1
rwbs: WM, sector: 2576, bytes: 4096, comm: kworker/u24:1
rwbs: WM, sector: 2600, bytes: 4096, comm: kworker/u24:1
rwbs: WM, sector: 2632, bytes: 4096, comm: kworker/u24:0
rwbs: WM, sector: 19016, bytes: 4096, comm: kworker/u24:1

// journaling
rwbs: WS, sector: 526408, bytes: 28672, comm: jbd2/sdd1-8
rwbs: WS, sector: 526464, bytes: 4096, comm: kworker/5:1H
rwbs: WS, sector: 526472, bytes: 8192, comm: jbd2/sdd1-8
rwbs: WS, sector: 526488, bytes: 4096, comm: kworker/5:1H

@[W]: 4096    // file contents
@[WM]: 24576  // metadata total
@[WS]: 45056  // journaling total
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

Ah, so there's our first bit of IO inflation. It turns out that you *cannot* write just 14 bytes to a disk.

## Disks are broken up into sectors
Even though in a previous post I said that a disk is a "bunch of bits", the reality is that (so far as I know) no computer program can actually write or read a single byte (let alone a bit) from a disk. Instead, hard disks are broken up into *sectors*, which these days are 4 KiB in size. Aha! 4 KiB, see?

*Why* are hard disks broken up into sectors? The main reason, probably, is that in order to write or read from a location on disk, you need an address, and it's simply not feasible to address each individual byte. It's not desirable, either: each write has overhead, so writing even 1 megabyte, one byte at a time, would be super slow. Bigger sectors increase throughput (which is another way of saying that we pay an overall lower cost of overhead).

There are other pressures that drive disk sectors to be bigger or smaller, and the story with SSDs is far more complex. The short story is that software can only write to SSDs 256 KiB to 4 MiB at a time! This is a long and interesting topic, so I'll save it for another article.

## Filesystems are broken up into blocks
Filesystems are *also* broken up into *blocks*, and file contents cannot take up less than a block.

These days, a common block size for ext4 is 4 KiB. Indeed, that's the block size on my system:
```bash
$ sudo blockdev --getbsz /dev/sdd1

4096
```

Why are filesystems broken up into blocks? I do not know the history, but common sense says that, if disks are broken up into sectors, there is no sense having filesystem blocks smaller than a disk sector. Another important factor is that, especially back in the days of severely limited memory, bigger filesystem blocks mean that more overall disk space could be addressed with the same amount of memory.

## Memory is broken up into pages
Finally, both virtual and physical memory is *also* broken up, and this time we call the unit a *page*. The reason memory is broken up is the same as disks: if a memory address is 32 bits, then there is literally not enough space to refer to each individual byte. A common page size is 4 KiB, but it's configurable because page size has performance implications.

Why is this relevant here, though? Well, if you read [How does Linux really handle writes?](/2023/06/27/file-writes.html), you saw that the default write behavior for ext4 is for the kernel to copy the file contents to an intermediary in-memory location called the page cache, and then asynchronously dump the memory page to disk. So, even if the file only has 14 bytes of content in it, we will dump the entire page – all 4 KiB.

It just so happens that my disk sectors, filesystem blocks, and memory pages are all 4 KiB. This isn't a coincidence: it makes sense for them to match. That said, a good reader exercise would be to tune these parameters and see how each one affects how much you end up writing to disk. (In other words, I am too lazy to do it).

# Cause of IO inflation 2: metadata
We have still covered only 4 out of 73 KiB! So, let's keep moving.

24 KiB of what we wrote corresponds to metadata:
```
rwbs: WM, dev: 8388656, sector: 2048, bytes: 8192, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2576, bytes: 4096, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2600, bytes: 4096, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2632, bytes: 4096, comm: kworker/u24:0
```

How do I know that? Because the `rwbs` field helpfully specifies when the trace corresponds to metadata via the `M`. However, we are going to verify this for ourselves.

As you can see, we write a lot of metadata. What is all the metadata? Here comes a lot of information about ext4 (which mostly applies to other filesystems, because they have somewhat similar organization).

## metadata 1: ext4 inodes
In [A disk is a bunch of bits](/2023/07/19/bunch-of-bits.html), we thoroughly discussed a data structure called the inode. The inode, basically, is where we store metadata about a file.

Aha! In retrospect, it's obvious. Of course, when we create a 14 byte file, we must store more than 14 bytes: we have to store the metadata!

An ext4 inode struct [takes up 256 bytes](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Inode_Size), but, remember, we cannot write less than 1 disk sector at a time (I am also pretty sure ext4 cannot write less than one file block at a time, even for metadata). So, when we create or update an inode, we rewrite the entire inode table.

One question is: can we avoid writing the metadata? Sometimes, yes. For example, one piece of optional metadata is when the file was last accessed. It would cause a lot of work to update this every time a file is touched, so often people disable this behavior.

There are other kinds of metadata that *cannot* be avoided. The most important example is the logical file size. If the file contents take up 4 KiB on disk, how do we know how much of that is the actual file contents? We must store that information.

So... how do we know that we actually wrote the inode table? I found a really cool way to verify it. In [A disk is a bunch of bits](/2023/07/19/bunch-of-bits.html), I found that we could coerce the raw bytes of a disk into an ext4 data structure. This would prove that the stuff on disk really is an inode. So, I took each of the 4 traces and tried to parse inodes from the disk location indicated by the trace.

*This* is the trace where I found inodes:
```
rwbs: WM, dev: 8388656, sector: 2632, bytes: 4096, comm: kworker/u24:0
```

Here's a snippet from my inode seeker's output:
```
inode 1 found at offset 0 (160 bytes):
  0000  00 00 00 00 00 00 00 00 7d 81 42 64 7d 81 42 64  ........}.Bd}.Bd
  0016  7d 81 42 64 00 00 00 00 00 00 00 00 00 00 00 00  }.Bd............
  0032  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0048  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0064  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0080  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0096  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0112  00 00 00 00 00 00 00 00 00 00 00 00 6e b3 00 00  ............n...
  0128  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  0144  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................

<--snip-->

inode 11 found at offset 2560 (160 bytes):
  2560  b4 81 e8 03 0e 00 00 00 63 3e de 64 63 3e de 64  ........c>.dc>.d
  2576  63 3e de 64 00 00 00 00 e8 03 01 00 08 00 00 00  c>.d............
  2592  00 00 08 00 01 00 00 00 0a f3 01 00 04 00 00 00  ................
  2608  00 00 00 00 00 00 00 00 01 00 00 00 00 82 00 00  ................
  2624  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  2640  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  2656  00 00 00 00 9d 7b fc d4 00 00 00 00 00 00 00 00  .....{..........
  2672  00 00 00 00 00 00 00 00 00 00 00 00 ab 33 00 00  .............3..
  2688  20 00 37 6b 38 94 1d dd 38 94 1d dd 38 94 1d dd   .7k8...8...8...
  2704  63 3e de 64 38 94 1d dd 00 00 00 00 00 00 00 00  c>.d8...........
```

inode 11 is the one that corresponds to `/data/example.txt`. User inodes always start at 11, for [the first 10 are reserved for system use](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Special_inodes).

## metadata2: ext4 superblock
inodes are not the only metadata that ext4 updates when you write a file. In our case, we are also *creating* a new file, which is why we update the superblock (or maybe we *always* update the superblock – not sure. We definitely update the superblock when creating a new file, because [the superblock counts the number of unused inodes](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#The_Super_Block)).

I extended my inode seeker program to seek for a bunch of different ext4 data structures, and it found the superblock in the disk location corresponding to this trace:
```
rwbs: WM, dev: 8388656, sector: 2048, bytes: 8192, comm: kworker/u24:3
```

See?
```
superblock found at offset 1024 (1024 bytes):
  1024  00 80 00 00 00 00 02 00 99 19 00 00 32 e7 01 00  ............2...
  1040  f5 7f 00 00 00 00 00 00 02 00 00 00 02 00 00 00  ................
  1056  00 80 00 00 00 80 00 00 00 20 00 00 10 3d de 64  ......... ...=.d
  1072  10 3d de 64 11 00 ff ff 53 ef 01 00 01 00 00 00  .=.d....S.......
  1088  7d 81 42 64 00 00 00 00 00 00 00 00 01 00 00 00  }.Bd............
  1104  00 00 00 00 0b 00 00 00 00 01 00 00 3c 00 00 00  ............<...
  1120  c6 02 00 00 6b 04 00 00 73 56 0e 98 a2 ce 43 d4  ....k...sV....C.
  1136  ba 23 a9 86 e6 9e 06 cc 44 41 54 41 2d 48 44 44  .#......DATA-HDD
  1152  00 00 00 00 00 00 00 00 2f 64 61 74 61 00 00 00  ......../data...
<--snip-->
```

Just for completeness, when writing the superblock, we also write things called [block group descriptors](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Block_Group_Descriptors), which I don't know anything about, but I did find them:

```
<--snip-->
group description found at offset 4096 (64 bytes):
  4096  41 00 00 00 45 00 00 00 49 00 00 00 b5 77 f5 1f  A...E...I....w..
  4112  01 00 04 00 00 00 00 00 af 9b 51 84 e7 1f ec 52  ..........Q....R
  4128  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  4144  00 00 00 00 00 00 00 00 2c 6f 36 4a 00 00 00 00  ........,o6J....
group description found at offset 4160 (64 bytes):
  4160  42 00 00 00 46 00 00 00 49 02 00 00 be 7f 00 20  B...F...I......
  4176  00 00 05 00 00 00 00 00 c6 b1 00 00 00 20 d6 99  ............. ..
  4192  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  4208  00 00 00 00 00 00 00 00 6c 64 00 00 00 00 00 00  ........ld......
group description found at offset 4224 (64 bytes):
  4224  43 00 00 00 47 00 00 00 49 04 00 00 00 70 00 20  C...G...I....p.
  4240  00 00 05 00 00 00 00 00 4b 8c 00 00 00 20 2b aa  ........K.... +.
  4256  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  4272  00 00 00 00 00 00 00 00 76 23 00 00 00 00 00 00  ........v#......
group description found at offset 4288 (64 bytes):
  4288  44 00 00 00 48 00 00 00 49 06 00 00 bf 7f 00 20  D...H...I......
  4304  00 00 05 00 00 00 00 00 18 51 00 00 00 20 6f be  .........Q... o.
  4320  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  4336  00 00 00 00 00 00 00 00 71 f8 00 00 00 00 00 00  ........q.......
<--snip-->
```

## metadata3: ext4 directory entries
the inode does not store the filename (which we also call a "directory entry"). This is because many different directory entries can point to the same inode.

If you have heard of a hard link, that's what they are: a directory entry that points at an inode that is already pointed to by another directory entry. (A soft link is a directory entry that points at another directory entry!)

The directory entry is written in this trace:
```
rwbs: WM, dev: 8388656, sector: 19016, bytes: 4096, comm: kworker/u24:3
```

And here are the contents:
```
directory entry found at offset 0 (1024 bytes):
  0000  02 00 00 00 0c 00 01 02 2e 00 00 00 02 00 00 00  ................
  0016  0c 00 02 02 2e 2e 00 00 0b 00 00 00 dc 0f 0b 01  ................
  0032  65 78 61 6d 70 6c 65 2e 74 78 74 00 00 00 00 00  example.txt.....
<--snip-->
```

## metadata: mystery metadata
I have not been able to figure out what ext4 writes for these two traces.

```
rwbs: WM, dev: 8388656, sector: 2576, bytes: 4096, comm: kworker/u24:3
rwbs: WM, dev: 8388656, sector: 2600, bytes: 4096, comm: kworker/u24:3
```

At the end of the article is the hexdump for them. If anyone knows what we write to those locations, let me know!

# Cause of IO inflation 3: journaling
We have explained 28 out of 73 KiB. Thankfully, though, the rest belongs to just one category: journaling.

## Journals are like write-ahead logs
If you have learned about databases, you may know that they use something called a write-ahead log. Have you wondered why they do that? The biggest reason is probably crash recovery.

Let's say that your query updates multiple tables. Those tables are usually stored in different files. What if the database writes to one table file, but the computer dies before it's able to write the other one? Not only have we lost data, but the database is left in an inconsistent state!

For this reason, before writing to the table files, a database will first write the information to a single file – the write-ahead log. That way, if we crash while writing to the multiple table files, we can recover using the log.

Earlier, I said that the file cannot be read without metadata like the logical file size. So, metadata is critically important. For this reason, some filesystems – like ext4 (and earlier versions of it, starting I think with ext3) – use a write-ahead log, except they call this journaling. My understanding is that prior to journaling, it was absurdly easy to corrupt a filesystem, and very slow to recover.

Note, though, that by default ext4 does not journal the actual file contents. Journaling does incur a major performance penalty, and its purpose seems to be really to make filesystem recovery easier, *not* increasing data durability (if you want to make your file contents durable, use the sync syscall as soon as you are done writing to the file - this is the topic of [How does Linux really handle writes?](/2023/06/27/file-writes.html)).

## I can't actually explain journaling
I have to be honest with you, though. I haven't learned about ext4 journaling deeply enough to tell you what each of these traces corresponds to.

```
rwbs: WS, dev: 8388656, sector: 526416, bytes: 28672, comm: jbd2/sdd1-8
rwbs: WS, dev: 8388656, sector: 526472, bytes: 4096, comm: kworker/8:1H
rwbs: WS, dev: 8388656, sector: 526480, bytes: 8192, comm: jbd2/sdd1-8
rwbs: WS, dev: 8388656, sector: 526496, bytes: 4096, comm: kworker/8:1H
```

Here is what I do know. Note the `S` in the `rwbs` field. This means that the write is synchronous: it's not considered complete until the data has actually been sent to the disk. In other words, the metadata for the file needs to be written durably, or else we may leave the filesystem in an inconsistent state.

One more thing I know is that the contents of the ext4 journal are basically just whatever we will write later elsewhere (plus journal metadata). For that reason, everything we found on disk from the other traces can be found in the journal.

For example, let's look for meaning in the location represented by this trace:
```
rwbs: WS, sector: 526408, bytes: 28672, comm: jbd2/sdd1-8
```

12 KiB into the write, we wrote inode 1.
```
inode 1 found at offset 12288 (160 bytes):
  12288  00 00 00 00 00 00 00 00 7d 81 42 64 7d 81 42 64  ........}.Bd}.Bd
  12304  7d 81 42 64 00 00 00 00 00 00 00 00 00 00 00 00  }.Bd............
  12320  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12336  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12352  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12368  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12384  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12400  00 00 00 00 00 00 00 00 00 00 00 00 6e b3 00 00  ............n...
  12416  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
  12432  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

And 21 KiB into the write, we wrote the superblock.
```
superblock found at offset 21504 (1024 bytes):
  21504  00 80 00 00 00 00 02 00 99 19 00 00 32 e7 01 00  ............2...
  21520  f5 7f 00 00 00 00 00 00 02 00 00 00 02 00 00 00  ................
  21536  00 80 00 00 00 80 00 00 00 20 00 00 10 3d de 64  ......... ...=.d
  21552  10 3d de 64 11 00 ff ff 53 ef 01 00 01 00 00 00  .=.d....S.......
  21568  7d 81 42 64 00 00 00 00 00 00 00 00 01 00 00 00  }.Bd............
  21584  00 00 00 00 0b 00 00 00 00 01 00 00 3c 00 00 00  ............<...
  21600  c6 02 00 00 6b 04 00 00 73 56 0e 98 a2 ce 43 d4  ....k...sV....C.
  21616  ba 23 a9 86 e6 9e 06 cc 44 41 54 41 2d 48 44 44  .#......DATA-HDD
  21632  00 00 00 00 00 00 00 00 2f 64 61 74 61 00 00 00  ......../data...
```

Phew. I think that's mostly it.

# Wrapping up
Let me show you the traces again, but this time I'll annotate them with what we learned.

```
// actual file contents
rwbs: W, sector: 268288, bytes: 4096, comm: kworker/u24:1


// metadata
// superblock and group descriptors
rwbs: WM, sector: 2048, bytes: 8192, comm: kworker/u24:1

// mystery metadata!
rwbs: WM, sector: 2576, bytes: 4096, comm: kworker/u24:1
rwbs: WM, sector: 2600, bytes: 4096, comm: kworker/u24:1

// inode table
rwbs: WM, sector: 2632, bytes: 4096, comm: kworker/u24:0

// directory entries
rwbs: WM, sector: 19016, bytes: 4096, comm: kworker/u24:1


// journaling
rwbs: WS, sector: 526408, bytes: 28672, comm: jbd2/sdd1-8
rwbs: WS, sector: 526464, bytes: 4096, comm: kworker/5:1H
rwbs: WS, sector: 526472, bytes: 8192, comm: jbd2/sdd1-8
rwbs: WS, sector: 526488, bytes: 4096, comm: kworker/5:1H
```

OK. There we have it. We have quite thoroughly examined all the writing that happens as a result of writing just 14 bytes to a file!

I'm tired. Thanks for reading!

# Extra content
* [bpftrace script used to trace the disk writes](https://github.com/dmazin/parse_ext4_structures/blob/main/trace.bt)
* [Python code for dumping disk locations](https://github.com/dmazin/parse_ext4_structures/blob/main/read_locations_from_traces_into_dir.py)
* [C code for seeking for ext4 structures](https://github.com/dmazin/parse_ext4_structures/blob/main/parse.c)
* [dump of all the traces for my disk](https://github.com/dmazin/parse_ext4_structures/tree/main/bitmap-dump)
