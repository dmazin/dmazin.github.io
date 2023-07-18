---
layout: post
title: "A disk is a bunch of bits"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-07-18
tags: featured
description: TODO
---
Have you ever heard someone say that a hard drive, or memory, is a "bunch of bits"?

I'm not sure about this idea's origin, but it's a pretty good idea. It reduces the mystery of computers. For example, it rules out the theory that inside of my computer is a very flat elf.

No, inside are bits, encoded on electrical components.

Yet, computers are still pretty mysterious. What *are* these bits? What do they mean? Can we play with them, parse them, make sense of them?

In this post, I will show you that, yes, absolutely we can! For your entertainment, I am going to stick my hand into my computer, pull up a bunch of bits, and we will examine and make sense of them.

What bits, exactly, should we explore? For this exercise, let's pick apart how a disk-backed file is represented on disk.

Say we have a file called `/data/example.txt`:
```
$ cat /data/example.txt

Hello, world!
```

Where does "Hello, world!" live?

Additionally, you may know that files have permissions (e.g. the file is executable), an owner, creation timestamp, etc. Where is this metadata stored?

I mean, literally, where are the actual bits that store this information? Let's find them and try to parse them.

First, a bit of theory.

What even is `/data/example.txt`? It's what we call a *directory entry*. A directory entry is just a human-readable name – `example.txt`.

Directory entries are stored on disk, but they are not terribly interesting, because they are just names.

A name *names* something, right? What does `example.txt` name? The thing it names is called an *inode*.

inodes *are* interesting. When you say, "a file lives on disk", what you really mean is, "an inode lives on disk". They are the on-disk collections of bits that describe a file.

An inode stores pretty much everything about a file, like the metadata we mentioned earlier.

We're almost done with the theory. You should also know that inodes, files, and directory entries are all elements of what's called a *filesystem*. A filesystem is the software that takes the bits on your disk and turns them into familiar files and directories.

With that, we're ready to get our hands dirty.

Let's start our exploration by listing some of the inode metadata. To do that, we can use `stat`.

```bash
$ stat /data/example.txt

  File: /data/example.txt
  Size: 14              Blocks: 8          IO Block: 4096   regular file
Device: 831h/2097d      Inode: 11          Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/  dmitry)   Gid: ( 1000/  dmitry)
Access: 2023-07-18 13:53:20.808536879 +0100
Modify: 2023-07-10 15:18:48.199691583 +0100
Change: 2023-07-18 14:52:26.349625767 +0100
 Birth: 2023-07-10 15:18:48.199691583 +0100
```

Don't try too hard to understand all of the output. Just notice that you see metadata like the file size, owner, and timestamps. Everything you see comes from the inode (other than the name, which came from the directory entry).

But we want to see the raw bits for this inode, right? How can we see the raw bits?

One of the most OG kernel hackers, [Ted Ts'o](https://en.wikipedia.org/wiki/Theodore_Ts%27o), maintains a set of filesystem debugging tools knows as e2fsprogs. We can use one of these tools, debugfs, to play with the inode.

debugfs has a cool command that will spit out the raw binary for an inode. From the [manpage](https://www.man7.org/linux/man-pages/man8/debugfs.8.html):
```
inode_dump filespec
    Print the contents of the inode data structure in hex and
    ASCII format.
```

So, here comes the raw binary I promised. Except, it's not going to be binary like 0011000. It's much easier for humans to read binary when it's converted into something called [hexadecimal](https://wizardzines.com/comics/hexadecimal/).

```bash
$ sudo debugfs /dev/sdd1

debugfs:  inode_dump example.txt
0000  b481 e803 0e00 0000 408b b664 348b b664  ........@..d4..d
0020  4813 ac64 0000 0000 e803 0200 0800 0000  H..d............
0040  0000 0800 0100 0000 0af3 0100 0400 0000  ................
0060  0000 0000 0000 0000 0100 0000 0082 0000  ................
0100  0000 0000 0000 0000 0000 0000 0000 0000  ................
*
0140  0000 0000 9933 e68b 0000 0000 0000 0000  .....3..........
0160  0000 0000 0000 0000 0000 0000 7161 0000  ............qa..
0200  2000 20fc e45e 97d9 fc34 9c2f bc2c c5c0   . ..^...4./.,..
0220  4813 ac64 fc34 9c2f 0000 0000 0000 0000  H..d.4./........
0240  0000 0000 0000 0000 0000 0000 0000 0000  ................
*
```

Makes sense? Cool – thanks for reading!

Just kidding. Look, actually seeing the raw data of the inode is slightly cool, but we still don't know *where* on disk this lives and what the bits actually mean.

So, let's find the raw inode on disk. To do that, we can again use debugfs:
```
imap filespec
    Print  the location of the inode data structure (in the inode table) of the
    inode filespec.
```

So, let's find the location.
```
debugfs:  imap example.txt                                                                  
Inode 11 is part of block group 0
        located at block 73, offset 0x0a00
```

Let me decipher this: a filesystem is broken up into blocks. In my case, a block is 4,096 bytes (this is the default for many Linux distributions). So, this output is saying "start at the beginning of the filesystem, and walk forward 73 blocks, i.e. 73*4096 bytes". That sort of tells us what street the inode is on. The house number is the offset: `0x0a00` bytes. In decimal, that's 2560 bytes.

So, to find our inode, we need to start at the beginning of the disk partition (which is also the beginning of the filesystem), then skip forward `4096*73+2560=301568` bytes.

Let's do it! Let's dump the raw bits from my disk and see if they match the `debugfs inode_dump` output.

```bash
$ sudo dd if=/dev/sdd1 bs=1 skip=301568 count=256 2>/dev/null | hexdump -C

00000000  b4 81 e8 03 0e 00 00 00  40 8b b6 64 34 8b b6 64  |........@..d4..d|
00000010  48 13 ac 64 00 00 00 00  e8 03 02 00 08 00 00 00  |H..d............|
00000020  00 00 08 00 01 00 00 00  0a f3 01 00 04 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  01 00 00 00 00 82 00 00  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000060  00 00 00 00 99 33 e6 8b  00 00 00 00 00 00 00 00  |.....3..........|
00000070  00 00 00 00 00 00 00 00  00 00 00 00 71 61 00 00  |............qa..|
00000080  20 00 20 fc e4 5e 97 d9  fc 34 9c 2f bc 2c c5 c0  | . ..^...4./.,..|
00000090  48 13 ac 64 fc 34 9c 2f  00 00 00 00 00 00 00 00  |H..d.4./........|
000000a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000100
```

Here is where the ASCII interpretation at the right comes in handy: visually, we can see that this stuff looks identical to the raw bytes from `debugfs inode_dump`! We found where the inode lives!

This feels way cooler than the `inode_dump` output did. In that case, we had asked a program written by a core kernel developer to please tell us what the inode looked like. In this case, we found the information directly on the disk ourselves.

But we still don't know what these bytes mean. Can we parse them?

For a few weeks, I sat on this. How do we ask a computer to turn a bunch of random bits into an inode?

Then it hit me: that is exactly what a struct is for!

You may have come across structs. They're sort of like objects from dynamic languages, except confusing.

Well, I think here's how I think about structs now, and it makes a ton of sense: let's say you come across a random bunch of bits. A struct is simply a specification of what those bits mean.

With that realization in hand, the Linux kernel must define a struct for the inode somewhere, right? [It does](https://github.com/torvalds/linux/blob/fdf0eaf11452d72945af31804e2a1048ee1b574c/fs/ext4/ext4.h#L769)!

```c
/*
 * Structure of an inode on the disk
 */
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
    /* ... */
}
```

I will repeat, for effect: this struct is how anything that runs on Linux knows how to parse the bytes we saw previously!

It says, basically: the first 16 bits are the file permissions, the next 16 bits are the owner, and the next 32 are the file size, and so on.

Let's use this struct to parse the raw bytes from before!

We are going to write a small C program that will do the following.
1. Ask the computer to set aside 256 bytes in memory (because that's how big an ext4_inode struct is).
2. Ask it to copy 256 bytes from `/dev/sdd1/`, at location 301568, into that memory.
3. Tell it how to parse those bytes using our ext4_inode struct.