---
layout: post
title: "A disk is a bunch of bits"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-07-18
tags: featured
description: TODO
---
I'd like to show you something that greatly reduced the mystery of how a computer works. Perhaps it'll do the same for you.

Let's say we have a file, `/data/example.txt`. It has some contents.
```
$ cat /data/example.txt

Hello, world!
```

The mystery is, what is `/data/example.txt`? Where does it live? What does it look like? What does it want?

We could say, well, `/data/example.txt` is a file. It lives on disk.

In my [previous post](TODO), we discussed how this is not true. A file doesn't live on disk. No, a file is an interface that lets us get a grip on the stuff that *really* lives on disk.

> Files are an interface, much like a OOP instance with methods and attributes. The file is not the stuff on disk. It’s just the abstract interface for it.

Then what *is* the stuff on disk? Bits. A disk is a bunch of bits.

What are these bits? Can we see them? Can we make sense of them? *That's* what we're going to explore today. 

In this post, we're going to roll up our sleeves, stick our hands into the bowels of the computer, and pull up a bunch of bytes. At first, they will make no sense, but we will force sense onto them.

For me, this was a hugely revelatory exercise. It did more to reduce the mystery of computers than most other things I've done.

Let's start by giving the bunch of bits a name. The bits on disk are not random, of course. They have a structure. We call that structure the **file system**. The file system is where we get our familiar files and directories.

In essence, the file system is both the structured bits on disk as well as [the code](https://github.com/torvalds/linux/tree/master/fs/ext4) that knows what to do with the said bits.

A file system has different components, like something called the "super block", which contains information about the file system itself. We are most interested in `/data/example.txt`, and the on-disk structure for it is called an **inode**.

Here's one way to think about an inode. If the file is the interface for manipulating `/data/example.txt`, the inode is the actual bits on disk that the file manipulates.


So, let's go back to `/data/example.txt`. It has contents: `Hello, world!`

It also has metadata.
```
$ stat /data/example.txt

  File: /data/example.txt
  Size: 14              Blocks: 8          IO Block: 4096   regular file
Device: 831h/2097d      Inode: 11          Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/  dmitry)   Gid: ( 1000/  dmitry)
Access: 2023-07-10 15:28:22.159110437 +0100
Modify: 2023-07-10 15:18:48.199691583 +0100
Change: 2023-07-10 15:18:48.199691583 +0100
 Birth: 2023-07-10 15:18:48.199691583 +0100
```




You might say, well, `/data/example.txt` is a file. It lives on disk. That's a useful mental model, but, as we saw in my previous post, it's wrong.

In my [previous post](TODO), 


In my [previous post](TODO), we discussed how a file is actually an abstraction. One might think, "a file is a thing that lives on disk", but that's not right. 

In the case of `/data/example.txt`, it's an abstraction for a bunch of bits that live somewhere on a hard disk. 



But in my previous post, we discussed how a file is really an abstraction. What are we abstracting?

`/data/example.txt` represents some information that lives on a hard disk.

In in my [previous post](TODO), we discussed how a file is really an abstraction.

> It’s reasonable to think that files live on disk, because files are what we interact with in order to write stuff to disk.

> But, that’s exactly it: a file is an interface. The operating system uses this interface so that we can tell it what we want.

> This is kind of theoretical, so let me say it again: files are an interface, much like a OOP instance with methods and attributes. The file is not the stuff on disk. It’s just the abstract interface for it.

> What does live on disk, then? Bytes.

> A disk is just a bag of bytes. These bytes have a structure, of course. If they didn’t, they would be random, and we would have no way to ever make sense of them. The specific way we order bytes on disk is called an inode. An inode is what we end up representing using a file.






 that lets us manipulate a bunch of bits that live on disk.

What *are* these bits, though? Where, on disk, is the string "Hello, world!"? How does my computer know  

And it has some metadata.
```
stat /data/example.txt
```