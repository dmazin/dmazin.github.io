---
layout: post
title: "A disk is a bunch of bits"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-07-19
tags: featured
description: We've heard that a disk is a "bunch of bits", so let's get dirty and personal with those bits.
---
# Introduction
Have you ever heard someone say that a disk or memory is a "bunch of bits"?

I'm not sure about this idea's origin, but it's a pretty good idea. It reduces the mystery of computers. For example, it rules out the theory that inside of my computer is a very flat elf.

No, inside are bits, encoded on electrical components.

# Getting close and personal with bits
Yet, computers are still pretty mysterious. What *are* these bits? What do they mean? Can we play with them, parse them, make sense of them?

In this post, I will show you that, yes, absolutely we can! For your entertainment, I am going to stick my hand into my computer, pull up a bunch of bits, and we will examine and make sense of them.

What bits, exactly, should we explore? For this exercise, let's pick apart how a disk-backed file is represented on disk.

Say we have a file called `/data/example.txt`:
```
$ cat /data/example.txt

Hello, world!
```

Here's one big question: where does "Hello, world!" live?

Also, you may know that files have permissions (e.g. the file is executable), an owner, creation timestamp, etc. Where is this metadata stored?

I mean, literally, where are the actual bits that store this information? Let's find them and try to parse them.

First, a bit of theory.

# How do files work?
The following applies to the ext4 filesystem commonly used in Linux (and, in fact, the whole article is ext4-specific). These concepts are common to most filesystems, though.

What even is `/data/example.txt`? It's what we call a *directory entry*. A directory entry is just a human-readable name – `example.txt`.

Directory entries are stored on disk, but they are not terribly interesting, because they are just names.

A name *names* something, right? What does `example.txt` name? The thing it names is called an *inode*.

inodes *are* interesting. When you say, "a file lives on disk", what you really mean is, "an inode lives on disk". They are the on-disk collections of bits that describe a file.

An inode stores pretty much everything about a file, like the metadata we mentioned earlier.

We're almost done with the theory. You should also know that inodes, files, and directory entries are all elements of what's called a *filesystem*. A filesystem is the software that takes the bits on your disk and turns them into familiar files and directories.

With that, we're ready to get our hands dirty.

# Exploring inodes
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

Don't try too hard to understand all of the output. Just notice that you see metadata like the file size, owner, and timestamps. Everything you see comes from the inode (other than the name, which came from the directory entry<a name="inode-number-return"></a><sup>[[Also, the inode number itself]](#inode-number)</sup>).

# Exploring the innards of an inode
But we want to see the raw bits for this inode, right? How can we see the raw bits?

Long-time kernel developer [Ted Ts'o](https://en.wikipedia.org/wiki/Theodore_Ts%27o) maintains a set of filesystem debugging tools called e2fsprogs. We can use one of these tools, debugfs, to play with the inode.

debugfs has a cool command that will spit out the raw binary for an inode. From the [manpage](https://www.man7.org/linux/man-pages/man8/debugfs.8.html):
```
inode_dump filespec
    Print the contents of the inode data structure in hex and
    ASCII format.
```

So, here comes the raw binary I promised. Except, it's not going to be binary like 0011000. It's much easier for humans to read binary when it's converted into a representation called [hexadecimal](https://wizardzines.com/comics/hexadecimal/), so that's what we'll use.

```bash
$ sudo debugfs /dev/sdd1

debugfs:  inode_dump example.txt
0000  b481 e803 0e00 0000 408b b664 1a99 b664  ........@..d...d
0020  4813 ac64 0000 0000 e803 0100 0800 0000  H..d............
0040  0000 0800 0100 0000 0af3 0100 0400 0000  ................
0060  0000 0000 0000 0000 0100 0000 0082 0000  ................
0100  0000 0000 0000 0000 0000 0000 0000 0000  ................
*
0140  0000 0000 9933 e68b 0000 0000 0000 0000  .....3..........
0160  0000 0000 0000 0000 0000 0000 2349 0000  ............#I..
0200  2000 5e0c 9c76 5b53 fc34 9c2f bc2c c5c0   .^..v[S.4./.,..
0220  4813 ac64 fc34 9c2f 0000 0000 0000 0000  H..d.4./........
0240  0000 0000 0000 0000 0000 0000 0000 0000  ................
*
```

Makes sense? Cool – thanks for reading!

Just kidding. Look, actually seeing the raw data of the inode is slightly cool, but we still don't know *where* on disk this lives and what the bits actually mean.

# Where on disk is my inode?
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

Let me decipher this: a filesystem is broken up into blocks. In my case, a block is 4,096 bytes (this is the default for many Linux distributions). So, this output is saying "start at the beginning of the filesystem, and walk forward 73 blocks, i.e. 73*4096 bytes". That sort of tells us what street the inode is on. The house number is the offset: `0x0a00` bytes. In decimal, that's 2560 bytes.<a name="2560-return"></a><sup>[[Why 2560?]](#2560)</sup>

So, to find our inode, we need to start at the beginning of the disk partition (which is also the beginning of the filesystem), then skip forward `4096 * 73 + 2560 = 301568` bytes.

Let's do it! Let's dump the raw bits from my disk and see if they match the `debugfs inode_dump` output.

```bash
$ sudo dd if=/dev/sdd1 bs=1 skip=301568 count=256 2>/dev/null | hexdump -C

00000000  b4 81 e8 03 0e 00 00 00  40 8b b6 64 1a 99 b6 64  |........@..d...d|
00000010  48 13 ac 64 00 00 00 00  e8 03 01 00 08 00 00 00  |H..d............|
00000020  00 00 08 00 01 00 00 00  0a f3 01 00 04 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  01 00 00 00 00 82 00 00  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000060  00 00 00 00 99 33 e6 8b  00 00 00 00 00 00 00 00  |.....3..........|
00000070  00 00 00 00 00 00 00 00  00 00 00 00 23 49 00 00  |............#I..|
00000080  20 00 5e 0c 9c 76 5b 53  fc 34 9c 2f bc 2c c5 c0  | .^..v[S.4./.,..|
00000090  48 13 ac 64 fc 34 9c 2f  00 00 00 00 00 00 00 00  |H..d.4./........|
000000a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000100
```

Here is where the ASCII interpretation at the right comes in handy: visually, we can see that this stuff looks identical to the raw bits from `debugfs inode_dump`! We found where the inode lives on disk!

This feels way cooler than the `inode_dump` output did. In that case, we had asked a program written by a core kernel developer to please tell us what the inode looked like. In this case, we found the information directly on the disk ourselves.

But we still don't know what these bits mean. Can we parse them?

# Commercial interruption
If you're enjoying yourself, may I ask if you'd like to follow me via [RSS feed](/feed.xml), [Mastodon](https://file-explorers.club/@dmitry), [email](https://cyberdemon.substack.com/?r=kk1l&utm_campaign=pub&utm_medium=web), or [Telegram channel](https://t.me/cyberdemon6)? Thanks!

# Making sense of the raw bits
For a few weeks, I sat on this. How do we ask a computer to turn a bunch of bits into an inode?

Then it hit me: that is exactly what a struct is for!

You may have come across structs. They're sort of like objects from dynamic languages, except confusing.

Well, here's how I think about structs now: let's say you come across a bunch of bits. A struct is simply a specification of what those bits mean.

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

I will repeat, for effect: this struct is exactly how the ext4 filesystem knows how to parse the bits we saw previously! This is the skeleton key.

It says, basically: the first 16 bits are the file permissions, the next 16 bits are the owner, and the next 32 are the file size, and so on<a name="padding-1-return"></a><sup>[[It's a tad more complicated because of padding.]](#padding-1)</sup>.

Let's use this struct to parse the raw bits from before!

We are going to write a small C program that will do the following.
1. Ask the computer to set aside 256 bytes in memory (because that's how big an ext4_inode struct is).
2. Ask it to copy 256 bytes from `/dev/sdd1/`, at location 301568, into that memory.
3. Tell it how to parse those bytes using our ext4_inode struct.

Here's the above program in C (boiled down to its essence).
```c
// open the partition file
int fd = open("/dev/sdd1/", O_RDONLY);

// seek to the inode location
lseek(fd, 301568, SEEK_SET);

// initialize the struct and copy 256 bytes from disk to memory
struct ext4_inode candidate_inode;
read(fd, &candidate_inode, sizeof(struct ext4_inode));

// now we can access the fields of the inode!
printf("User:  %u", inode->i_uid);
```

Here's [the full program, with error checking](https://gist.github.com/dmazin/ed608be38e4b0ec7ed2db65a74bc123a). If you want, you can build it and try it on your own computer.

Let's run it! Are you excited?! If this works, that means we have wrangled the bits. We have sussed out their structure.

```bash
$ sudo ./parse /dev/sdd1 301568

Inode: 11   Mode:  0664
User:  1000   Group:  1000   Size: 14
Links: 1   Blockcount: 8
Inode checksum: 0x0c5e4923
```

Yay! We've got ourselves a valid inode!

To verify that this output makes sense, here's the output of `debugfs stat example.txt`. Look, every common field – importantly, the checksum – match!

```
debugfs: stat example.txt

Inode: 11   Type: regular    Mode:  0664   Flags: 0x80000
Generation: 2347119513    Version: 0x00000000:00000001
User:  1000   Group:  1000   Project:     0   Size: 14
File ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x64b6991a:535b769c -- Tue Jul 18 14:52:26 2023
 atime: 0x64b68b40:c0c52cbc -- Tue Jul 18 13:53:20 2023
 mtime: 0x64ac1348:2f9c34fc -- Mon Jul 10 15:18:48 2023
crtime: 0x64ac1348:2f9c34fc -- Mon Jul 10 15:18:48 2023
Size of extra inode fields: 32
Inode checksum: 0x0c5e4923
EXTENTS:
(0):33280
```

To me, this is super cool. We set out to find the raw bits for an inode on disk, found them, and then made sense of the bits.

# Memory is a bunch of bits, too
Now, we're not done just yet.

At the beginning, I said that disks *and memory* are bunches of bits. Our program copies the raw inode bits into memory, right?

That means we should be able to find those bits, in memory, and confirm that they are the same bits that came from disk<a name="padding-2-return"></a><sup>[[Note about struct padding]](#padding-2)</sup>!

To do this, let's run our program in a debugger called `gdb` (kind of like Python's `pdb`). We'll use it to pause the program's process, and then spy on the process's memory.

```
$ sudo gdb parse
(gdb) break 167
(gdb) run /dev/sdd1 301568
(gdb) x/160xb &candidate_inode
0x7fffffffe410: 0xb4    0x81    0xe8    0x03    0x0e    0x00    0x00    0x00
0x7fffffffe418: 0x40    0x8b    0xb6    0x64    0x1a    0x99    0xb6    0x64
0x7fffffffe420: 0x48    0x13    0xac    0x64    0x00    0x00    0x00    0x00
0x7fffffffe428: 0xe8    0x03    0x01    0x00    0x08    0x00    0x00    0x00
0x7fffffffe430: 0x00    0x00    0x08    0x00    0x01    0x00    0x00    0x00
0x7fffffffe438: 0x0a    0xf3    0x01    0x00    0x04    0x00    0x00    0x00
0x7fffffffe440: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe448: 0x01    0x00    0x00    0x00    0x00    0x82    0x00    0x00
0x7fffffffe450: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe458: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe460: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe468: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe470: 0x00    0x00    0x00    0x00    0x99    0x33    0xe6    0x8b
0x7fffffffe478: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe480: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffe488: 0x00    0x00    0x00    0x00    0x23    0x49    0x00    0x00
0x7fffffffe490: 0x20    0x00    0x5e    0x0c    0x9c    0x76    0x5b    0x53
0x7fffffffe498: 0xfc    0x34    0x9c    0x2f    0xbc    0x2c    0xc5    0xc0
0x7fffffffe4a0: 0x48    0x13    0xac    0x64    0xfc    0x34    0x9c    0x2f
0x7fffffffe4a8: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```

It's not the easiest thing to read, so I used a [script](https://gist.github.com/dmazin/7348f61312c647e79e75eb6e8521ef68) to make the gdb output look more like `hexdump -C`:
```
b4 81 e8 03 0e 00 00 00 40 8b b6 64 1a 99 b6 64
48 13 ac 64 00 00 00 00 e8 03 01 00 08 00 00 00
00 00 08 00 01 00 00 00 0a f3 01 00 04 00 00 00
00 00 00 00 00 00 00 00 01 00 00 00 00 82 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 99 33 e6 8b 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 23 49 00 00
20 00 5e 0c 9c 76 5b 53 fc 34 9c 2f bc 2c c5 c0
48 13 ac 64 fc 34 9c 2f 00 00 00 00 00 00 00 00
```

Let's compare that to the raw bits from disk:
```
b4 81 e8 03 0e 00 00 00  40 8b b6 64 1a 99 b6 64
48 13 ac 64 00 00 00 00  e8 03 01 00 08 00 00 00
00 00 08 00 01 00 00 00  0a f3 01 00 04 00 00 00
00 00 00 00 00 00 00 00  01 00 00 00 00 82 00 00
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
*
00 00 00 00 99 33 e6 8b  00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00  00 00 00 00 23 49 00 00
20 00 5e 0c 9c 76 5b 53  fc 34 9c 2f bc 2c c5 c0
48 13 ac 64 fc 34 9c 2f  00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
```

They match!

(The asterisk just means that the row was all 0's. A keen observer would also notice that the last 16 bytes of 0s are missing from the `gdb` output – I think due to a compiler optimization).

What this shows is that the bits on disk and the bits in memory are the same. In retrospect, it may be obvious, but we have seen it with our very own eyes.

# Hey, where are my file contents?
I know what you're thinking: we haven't actually seen the file contents!

It's true. The inode does not itself store the file contents. They are elsewhere.

Allow me to briefly explain why. You can think of a filesystem as having two components: a bunch of boxes to put file contents into, and a database for managing those boxes. It's sort of like a distributed system, where you store records in a database (all the metadata), but you put actual file blobs in something like S3 or a disk.

So, the inode doesn't actually hold the contents; it points to them.

Let's use debugfs to parse the location from the inode. From the manpage:
```
blocks filespec
    Print the blocks used by the inode filespec to stdout.
```

```
debugfs:  blocks example.txt
33280
```

What this says is that the contents are 33,280 4 KiB blocks from the start of the filesystem.<a name="struct-extents-location-return"></a><sup>[[Could we have gotten the location straight from the inode struct?]](#struct-extents-location)</sup>

Let's dump the disk at that location!

```
$ sudo dd if=/dev/sdd1 skip=33280 bs=4096 count=1 2>/dev/null | hexdump -C
00000000  48 65 6c 6c 6f 2c 20 77  6f 72 6c 64 21 0a 00 00  |Hello, world!...|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000
```

There it is! Our hello world, hot and fresh from disk!

# What have we learned?
So, what have we learned? What have we done?

We started with the common refrain that a disk, and memory, are just bunches of bits.

So we set out on a quest to become familiar with those bits. Specifically, the bits that encode disk-backed files: inodes.

We got very familiar with said bits: we found them on disk, parsed them using a program that loaded them into memory and applied a struct to them, and then turned around and saw the very same bits in memory.

Along the way, we learned a little bit about the ext4 filesystem (and filesystems in general), too!

When I performed this exercise for myself, it was one of the most revelatory experiences I've had on a computer. It reduced the mystery. I hope the mystery has been reduced for you too.

# Notes
<a name="inode-number"></a>**Also, the inode number itself**

Well, also, the inode number itself (11 in this case) isn't stored in the inode either. Instead, it's the inode's position in the inode table. <a href="#inode-number-return">(back)</a>

<a name="2560"></a>**Why 2560?**

Recall that this inode's number is 11. That means that, on disk, there are 10 inodes before this one. Each inode is 256 bytes, so those inodes take up 2560 bytes. <a href="#2560-return">(back)</a>

<a name="padding-1"></a>**It's a tad more complicated because of padding**

Technically, the compiler will pad the struct, which means it will insert empty space throughout the struct. So, in that sense, the struct doesn't *exactly* specify the order of the bits. However, given that the inode was generated on the same computer where it will be read, this means that the struct truly *is* the skeleton key to the seemingly random bits. <a href="#padding-1-return">(back)</a>

<a name="padding-2"></a>**Note about struct padding**

Earlier, I mentioned that the compiler pads the struct, adding extra bytes between the fields. This would make the in-memory representation hard to compare to the on-disk representation, so I prevented padding as much as possible by appending `__attribute__((__packed__))` to the struct definition. That is why in the memory dump I pasted, we only printed 160 bytes – that is `sizeof(struct ext4_inode) when padding is disabled. <a href="#padding-2-return">(back)</a>

<a name="struct-extents-location"></a>**Could we have gotten the location straight from the inode struct?**

We could also parse the location from the inode we loaded into memory, via the `i_block` field, but the contents are a slightly cryptic array that it takes a bit of code to decode. It was easier to just call on the debugfs code to do it for us. For the curious, here's what that array looks like:
```
(gdb) p candidate_inode->i_block
$1 = {127754, 4, 0, 0, 1, 33280, 0, 0, 0, 0, 0, 0, 0, 0, 0}
```
You can see the 33280 from debugfs's output in that array. <a href="#struct-extents-location-return">(back)</a>