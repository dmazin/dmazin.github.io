---
layout: post
title: "TODO: IO Inflation"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-08-16
tags: featured
description: TODO
---
# Introduction
Hey, everyone. I'm back from vacation, with some good news: in a couple weeks I'll be starting a new job as a "Senior Linux Engineer". I mean, knock on wood -- I don't want to jinx it. Needless to say, I am absurdly excited that, I am *actually* going to get to work on-premises with Linux.

Anyway, today we're going to continue talking about file and disk I/O (we have not even come close to exhausting the topic). At the end of my [How does Linux really handle writes?](TODO) article, I teased that writing to a file, and writing to a disk, are such different things that, in fact, if you write just one byte to a file, 65+ *thousand* bytes will get written to disk. So, let's explore that.

This post is ext4-specific, but lots of the general ideas apply to most file systems. I'll try to make it clear when I'm discussing something ext4-specific. That said, like with my other articles, on top of learning about the main subject of the article, you'll walk away having also learned about file systems and ext4.

Let's jump into it.

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

Now that we've defined IO inflation, we can answer our original question by discussing the causes of IO inflation.

# Causes of IO inflation
## Disk sectors, file system blocks, and memory pages
Let's return to the discrepancy between the logical and physical size of `/data/example.txt`. Why is the physical size 4 KiB?

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
Even though I spent a lot of words on the 4 KiB thing, in fact 

## Journaling

# Notes
[1] Some hard disks and SSDs make it seem like their sectors are 512 bytes long, though this is just something they do for the sake of older hardware.