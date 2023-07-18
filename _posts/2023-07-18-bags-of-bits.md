---
layout: post
title: "A disk is a bunch of bits"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-07-18
tags: featured
description: TODO
---
I'd like to show you something that greatly reduced the mystery of how a computer works. Perhaps it'll do the same for you.

I've heard before that a hard drive, or an SSD, or computer memory, is just a "bunch of bits".

That already demystifies computers somewhat. There are no elves inside: just bits, encoded on electrical components. All a computer does is push those bits around.

But what do these bits look like? What do they mean? Can we play with them, parse them, and make sense of them?

Yes, absolutely we can! In this post, we're going to roll up our sleeves, stick our hands into the bowels of the computer, and pull up a fistful of bits. At first, they will make no sense, but we will force sense onto them.

For our exercise, let's explore the bits behind a disk-backed file. That is, say we have a file called `/data/example.txt`. It has some contents.
```
$ cat /data/example.txt

Hello, world!
```

Let's explore where the bits for `/data/example.txt` live and what they look like.

The first question is, what *is* `/data/example.txt`?

We could say, well, `/data/example.txt` is a file. It lives on disk.

In my [previous post](TODO), we discussed how this is not true. A file doesn't live on disk. No, a file is an interface that lets us get a grip on the stuff that *really* lives on disk.

> Files are an interface, much like a OOP instance with methods and attributes. The file is not the stuff on disk. It’s just the abstract interface for it.

So, what is the thing that the file abstracts away? It's called an *inode*. One way to think of an inode is the same way you think of a jpeg: it’s a way to order bits in a certain way. Whereas a jpeg is how we arrange bits for a picture, an inode is how we arrange bits for a disk-backed file.

For example, you may know that files have permissions (e.g. the file is executable, or, everyone can read it). This is stored on the inode. As another example, the inode stores when the file was created and when it was modified. Essentially, the inode stores almost everything about the file.

I'm not awesome at explaining things, so let me take another stab at it. 

On computers, we have directories with files in them, right? For example, here's my `/data` folder.
```
$ ls /data
example2.txt  example.txt  hello.txt
```

The things you see in `/data`, like `example.txt`, are called *directory entries*. A directory entry is just a nice name (like `example.txt`) and a pointer to the inode, which holds the actual information. (footnote: Though, I should say, this is only the case for stuff that lives on disk. A directory entry could point at something other than an inode. For example, it could point at a socket.)

I don't want this to turn into an article about file systems, but, files, inodes, and directory entries are all aspects of a *file system*. The file system is the thing that organizes the bits on your disk into meaningful structures like inodes.

So, where do these inodes actually live? What do they look like?

To answer that, let's get our hands dirty. Let's play with the file system. To do that, we are going to use a utility called `debugfs`. I can't explain it better than its own manpage:

> The debugfs program is an interactive file system debugger. It can be used to  examine and change the state of an ext2, ext3, or ext4 file system.

We can use `debugfs` to see the contents of the inode and to find out where it lives on disk. From the manpage:

```
stat filespec
    Display the contents of the inode structure of the inode filespec.
```

So, let's see what the inode actually holds.
```
$ sudo debugfs /dev/sdd1

debugfs: stat example.txt

Inode: 11   Type: regular    Mode:  0664   Flags: 0x80000
Generation: 2347119513    Version: 0x00000000:00000001
User:  1000   Group:  1000   Project:     0   Size: 14
File ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x64b68b34:d9975ee4 -- Tue Jul 18 13:53:08 2023
 atime: 0x64b68b40:c0c52cbc -- Tue Jul 18 13:53:20 2023
 mtime: 0x64ac1348:2f9c34fc -- Mon Jul 10 15:18:48 2023
crtime: 0x64ac1348:2f9c34fc -- Mon Jul 10 15:18:48 2023
Size of extra inode fields: 32
Inode checksum: 0xfc206171
EXTENTS:
(0):33280
```

We won't dwell on this too much, but note how the inode stores the file permissions (`Mode:  0664`) and a bunch of timestamps about the file!

Note that the inode *doesn't* store the file name: the directory entry stores that. Multiple directory entries can point at the same inode (you may have heard of this: hard links), so we decouple file names from inodes.

And, well, you may see that the inode doesn't actually hold the file contents. The reason is that, conceptually, you can think of a file system as a bunch of boxes to store file contents, and a database to manage those boxes. The inodes are the database entries. They are pure metadata. So, our inode merely points at the box that holds our file contents:
```
EXTENTS:
(0):33280
```

We'll return to the contents later.

Now, let's finally look at some binary. We can actually ask debugfs to print the raw bytes of the inode:
```
inode_dump filespec
    Print the contents of the inode data structure in hex and
    ASCII format.
```

<!-- footnote: You may notice a bit of sleight of hand here: I said we're going to look at binary, but then said debugfs would print out the bytes. The fact of the matter is, we *could* look at binary (00011010 etc) but there are way more human-readable ways to parse binary. Such a way is to turn the binary into hexadecimal, which is how binary is usually presented to humans. -->

Let's do it! Let's look at some raw bytes!

```
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

Well, there they are. The raw bytes for our inode. We can't make much sense of them, can we? The right column of the output is an attempt to interpret those bytes as text but, as you can see, that is meaningless too.

But, not to worry. This is where our experiment gets exciting. We're going to use technology to parse these bytes and make sense of them!

We are going to write a small C program that will read the raw bytes from disk, parse them into an inode, and give us input similar to what we saw when we ran `debugfs stat`.

But where on disk is that inode? `debugfs` can tell us. From the manpage:
```
imap filespec
    Print  the location of the inode data structure (in the inode table) of the
    inode filespec.
```

So, let's fire up `debugfs` again and find out where to find the inode for `example.txt`.
```
debugfs:  imap example.txt                                                                  
Inode 11 is part of block group 0
        located at block 73, offset 0x0a00
```

<!-- Note that I passed `/dev/sdd1` (the disk partition where `/data` lives) rather than `/data`. I think the reason for this is that `debugfs` can be used to debug a corrupted file system, which means we wouldn't have been able to mount it in the first place. (footnote) -->

Let me translate this into semi-human language: "The file system is broken up into 4,096 byte blocks. So, go to block 73, and you will find your inode 0x0a00 bytes (2560 bytes, in decimal) into the block."

<!-- (footnote: why is it broken up into 4 KiB blocks? subject for another article) -->

What that means is that our inode is `4096*73+2560=301568` bytes deep in the disk partition.

<!-- (footnote: note that we could actually use debugfs to print out the raw bytes of the inode, but that would not be as fun) -->

Let's verify this by dumping the raw bytes directly from the disk partition.

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

Here is where the ASCII interpretation at the right comes in handy: visually, we can see that this stuff looks identical to the raw bytes from `debugfs inode_dump`!

Let's take a moment to catch a breath. We found the inode on disk, and printed it *directly from disk*. That is already amazing: we know where the inodes are stored, and how to get them.

Now, let's actually prove that this *is* our inode by parsing it.

But... how do we parse it? For a long time, I sat on this, because I did not actually understand how to take raw bytes and tell the computer what those bytes *mean*.

After some weeks, though, it hit me: that's literally what a struct is for. Let me take a bit of time to explain what a struct is.

First, let's recall what a class looks like in Python. Something like this.

```python
class inode:
    mode: int       = 0
    uid: int        = 0
    size_bytes: int = 0
```

That class is a template for what instances of `inode` should look like.

<!-- (footnote: It would be even more accurate if I said this was a @dataclass, but I didn't want to get too deep into Python stuff) -->

A struct is kind of like that, except that it's hyper-specific (and, of course, Python is dynamic while C is static). Here's what the same struct looks like.

```c
struct inode {
	uint16_t	mode;          /* unsigned 16-bit integer */ 
	uint16_t	uid;           /* unsigned 16-bit integer */ 
	uint32_t	size_bytes;    /* unsigned 32-bit integer */
}
```

This struct says, "if you come across a bunch of bits that you want to parse as an inode, here's how you do it: the first 16 bits are the file mode (the file permissions); the next 16 bits are the user ID of the owner of the file; the next 32 bits are the file size".

So, in plain English, here is what our program will do: it will ask the computer to set aside some space in memory (256 bytes, to be exact), and to copy 256 bytes from disk location 301568 and put them into that little space in memory. We will define a struct that will specify exactly what each byte means.

But, how will we know what the inode struct should look like? Here's the amazing thing: we are going to parse the bits of the inode *exactly* the same way that the operating system does it. It, itself, [defines the struct](https://github.com/torvalds/linux/blob/fdf0eaf11452d72945af31804e2a1048ee1b574c/fs/ext4/ext4.h#L769) we need:

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

Let me repeat: this is *the* struct used to parse the raw bytes we dumped previously. If we feed our bytes through this struct, we will get back the exact information we saw earlier! This is how we will make sense of the bytes!

<!-- (TODO some info about the program) -->

And... there we go!

```
$ sudo ./parse /dev/sdd1 /dev/sdd1 301568

Inode: 11   Type: regular   Mode:  0664   Flags: 0x00080000
User:  1000   Group:  1000   Project:  0   Size: 14
File ACL: 0
Links: 1   Blockcount: 8
ctime: 0x64b6991a -- (null)atime: 0x64b68b40 -- (null)mtime: 0x64ac1348 -- Mon Jul 10 15:18:48 2023
crtime: 0x64ac1348 -- (null)Size of extra inode fields: 32
Inode checksum: 0x0c5e4923
```

We have parsed the raw bytes into an inode.


<!-- Here's how. We are going to say: computer, copy the 256 bytes at disk location 301568 and put them in memory. Then, please interpret the bytes as so: the first 2 bytes are the file mode (the file permissions); the next 2 bytes are the user ID of the owner of the file; the next 4 bytes are the file size, and so on. -->



<!-- How do we do that? We can use a utility, `dd`, which is used to move data from one file to another. Since our disk partition is represented by a file, we can use `dd` to copy data from it to standard output (by that I mean we can use it to print the output to our shell).

As an example, here is roughly what our dd command will look like: `dd if=/dev/sdd1 bs=512 skip=?`. By default, `dd` spits the information out to the shell (standard output). `bs` stands for "block size" and it tells `dd` how many bytes to read at a time. `skip` is how we tell `dd` *where* we want to read from, except that instead of using bytes, we tell `dd` how many multiples of blocks to skip at a time. (note to myself: this makes a lot more sense if you understand disk sectords and file system blocks)

So, I'm not 100% sure what block 73 really means – it could be 73*512 (because disks are broken up into 512 byte sectors) or 73*4096 (because ext4 by default is broken up into 4 KiB blocks).

The `skip` is how we tell `dd` *where* we want to read from. The `bs` stands for "block size" and it tells it 

In many cases, you can say that each file you see when you list the contents of a directory corresponds to an inode, though this is not exactly true. For example, perhaps you have heard of hard links. A hard link is just a file that points to an inode.



What are these bits? Can we see them? Can we make sense of them? *That's* what we're going to explore today. 

In this post, we're going to roll up our sleeves, stick our hands into the bowels of the computer, and pull up a bunch of bytes. At first, they will make no sense, but we will force sense onto them.

For me, this was a hugely revelatory exercise. It did more to reduce the mystery of computers than most other things I've done.

Let's start by giving the bunch of bits a name.

If you read my previous article, you already know that 

We can do that by asking our OS to give us some metadata about our file.

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

From this output, we see a couple things.
1. The file has a bunch of metadata. That must be stored somewhere.
2. The metadata tells us where to find `Device: 831h/2097d      Inode: 11`


the file `/data/example.txt` is not the thing that lives on disk. The file is an interface that abstracts away the bits that *actually* live on disk.




Call it `/data/example.txt`. In my [previous post](TODO), we discussed how a file is really just an interface for manipulating the bits that actually live on disk.

Those bits have a name: **inode**. The inode is how we actually store information about `/data/example.txt`. For example, it's where we store when it was created, and how big it is.





`/data/example.txt`.

It has some contents.
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
``` -->