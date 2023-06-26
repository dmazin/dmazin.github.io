---
layout: post
title: "TODO"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: TODO
---
## Intro
I am an experienced SRE. Yet, six months ago, Linux was mysterious to me. How is that possible? Long story short: cloud computing abstracts a lot, and it’s possible to work with the high-level abstractions without understanding what’s going on below. I always knew something major and interesting lived below the abstractions, and it bothered me that I did not know much about it. Generally, I find mysteries extremely annoying, and I squash them like bugs. I decided to take a sabbatic to squash a lot of these mystery-bugs. In this article series, I am going to share what I learned. You may find that this squashes many of your mystery-bugs too.

## What this article is about
In this article, I’m going to discuss a misconception I had about how operating systems handle writes. Here’s the misconception: files live on disk, so when you work with files, you are working with the disk. While this is true, it leaves out the fact that *memory completely intermediates your interactions with the disk*.

In this article, I will deeply elaborate on what that sentence means. In the second part of this article, we will pull up our sleeves, trace the actual events I describe below, and for extra griminess, step through the actual kernel code.

## A common misconception
So, let’s begin with the misconception that when you work with files, you are working with the disk. A child-misconception of this general misconception is that writes to a file are durable. That is, if you do `echo foo > bar.txt`, as soon as you hit enter, “foo” is going to get written to disk, and if you unplug your computer (quickly after hitting enter) and then start it back up, you will find “foo” on disk in a file called bar.txt.

None of that is true. Indeed, if you write to a file, then read from that same file, both of those operations are likely to complete well before the disk is accessed at all. Why?

Because, instead, that write/read goes through memory. That is, when you write, you are actually writing to memory. When you read, you read from memory. The data is written to/from the disk asynchronously. Why?

The reason is that disk access is supremely expensive, especially for hard drives. Operating systems do what they can to protect applications from that slowness. You may have heard the phrase “non-blocking I/O,” and this is one example of it.

The disparity is shocking. Allow me to illustrate just how slow disk access is compared to memory access: SSD access is roughly 1,000 times slower than memory access, and hard disk access is roughly 1,000,000 times slower!

<!-- TODO illustration -->

## What actually happens?
So, what actually happens when you write “foo” to a file?

1. Your application has stored “foo” in a variable. **Every user application runs in a process, and each process has its own space in memory. You may have heard of the “stack” – the stack is one of the sections of that process memory space. So, the string “foo” is in a variable that is in memory**. TODO bolded text should be a footnote
2. Your application asks the kernel to write “foo” to some file, using a system call like `write` (almost always via a standard library call, which is to say, it’s unusual for your application to use the system call directly; the library the application uses in turn uses that system call though). TODO this can be a footnote
3. The kernel copies “foo” from the application process’ memory space to the kernel’s own memory space.
4. The `write` system call returns in the application as soon as that happens, and your application executes the next line.
5. Some time later, a kernel thread syncs “foo” from the kernel’s memory space to disk.

## Kernel memory space? Huh?
It’s worth elaborating what I mean by the “kernel’s own memory space”. When I said earlier that disk writes/reads go “through memory”, that is actually referring to something called the page cache. When people say “page cache” they are referring to both an actual place in memory, where the kernel caches files, and also to the code that manages that memory.

By the way, why “page”? This is because operating systems don’t actually access memory one byte (let alone bit) at a time (the reason why is the subject of a future article, because it’s a surprisingly deep topic, especially its history TODO footnote). Instead, they do it in larger blocks. For memory, we call these blocks pages. 

So, when I say that the kernel copies “foo” from the process memory to kernel memory, what I am really saying is that the `write` syscall calls a function that causes “foo” to get stored in a place in memory called the page cache.

## How does the data actually end up on disk?
I mentioned that a kernel thread will asynchronously write “foo” to disk. What’s up with that?

One thing I often repeat to myself is that everything is simpler than I think. It is especially true to think that things are complex when they are mysterious. But, we are removing mystery.

Let’s say that we are writing an operating system. We know that applications are constantly writing data to memory, and we want to asynchronously follow up on that – to detect that data, and write it to disk. How would we do it? We’d run a process that looks for that data and writes it to disk!

This is pretty much what happens in Linux. There are programs called flusher threads that look for “dirty” pages and write them to disk.

This is not the only way that data can get written to disk. In a later section, I will discuss how applications can manually demand that the data get written to disk. There are other ways, but I think they are out of the scope of this article.

## There’s a name for this, by the way
This caching style, where the cache intermediates access to persistent storage. It is called write-back caching.

You are probably used to look-aside caching, where an application first consults the cache, and if the cache doesn’t have the info, the application asks persistent storage.

## That was writing. How about reading?
Let’s demystify what happens when you read that file!

If you read from that file again, “foo” will come from the page cache, not disk. **That is, unless the data has been evicted from the page cache. It’s a cache, after all, and caches are not infinite. Just like how Memcached evicts little-used entries to make room for new or highly-used ones, the page cache does too.** make bold text a footnote

What happens if you read that file, and the data is not in the page cache? Will your read once again go “through memory”, and will it go through the page cache directly? Yes! Here it is, briefly.

1. Your application asks to read the file.
2. The kernel looks in the page cache and does not find it.
3. The kernel reads “foo” from disk, and puts it in the page cache.
4. “Foo” is copied from the page cache to your application’s memory.

This is known as read-through caching.

# Write-back caching is a trade off
Write-back caching is a trade-off. On the one hand, it means applications don’t need to wait for slow disk writes to finish. On the other, though, there’s a chance that something will go wrong (like power failure) after the application thinks the write finished (i.e. when the write syscall returns), but before the data has been written to disk. 

What about applications like databases, where durability is, like, the most important feature?

The key is that what I described is merely the default behavior. Linux allows applications to demand that the data should get written to disk ASAP.

### `sync` syscall
The application could call the `sync` syscall right after the write. This would cause the data to be immediately persisted to disk. This allows applications to finely tune the trade off between durability and write throughput. For example, an application could call `sync` every 100 writes.

[illustration] (something like `write` `write` `write` … `sync`)

### `O_SYNC` file mode
Another way applications can make write durable is by opening a file in `O_SYNC` mode. In this case, reads and writes would still be intermediated by the page cache. However, all writes would be blocking and would successfully return only once the data (and metadata) had successfully persisted to disk.

This is known as write-through caching: the cache still intermediates access to persistent storage, but the write is not considered successful until it has hit persistent storage.

# There are more reasons for this indirection
I am going to save this for another article, but I want to briefly mention that there are other benefits for intermediating writes and reads through the page cache, and for asynchronicity.
TODO this can be a footnote

* Show clock cycle, memory access, hard disk write, SSD write durations compared to each other. DONE
* Explain that they are so slow that if every write was durable, operating systems would be very slow. DONE
* TODO show that asynchronous allowed us to batch requests and cancel and coalescence 
* Instead, developers are expected to know this and that they need to tell the operating system needs to persist the data to disk *now*. DONE
* Explain that this is called write-back caching. DONE

# Rolling up our sleeves
I hope that was a solid conceptual explanation of how reads and writes work, but reading about this stuff was not enough to fully demystify things for me. The best way to put it is that I wanted to catch Linux, red-handed, actually doing the things I described above.

So, first, let’s use a tracing utility to demonstrate that a read and write will complete before the disk is accessed. When I first did this, I felt a huge surge of satisfaction: it’s one thing to read about something, but another to observe it happening.

Second, we are going to go even further. We are going to debug the kernel itself, step through the code, inspect memory, and see for ourselves that the way this actually works matches up with the above explanations. While tracing gave me satisfaction, doing this was utterly revelatory. What was previously mysterious had been made plain. I hope that at the end of these exercises, you will feel a similar way.

# Setup: a little bit of C code
* In order to be sure that the application works exactly the way we think it does, we’re going to write a simple C program that will write to a file and then read from it. To make sure that the things happening are exactly what we think is happening, we will not use the standard library. Instead, we will use system calls directly.
* To keep it simple, there is no error checking.

```
#include <fcntl.h>
#include <unistd.h>

int main() {
    char* str = "foo";
    int fd = open("example.txt", O_WRONLY | O_CREAT);
    write(fd, str, 3);
}
```

# Level 1: tracing using `bpftrace`
* In the next section, actually show this using bpftrace.
* Another misconception I used to hold is that you’re not meant to observe what’s going on under the hood in Linux. That is wildly wrong. The Linux community puts a lot of effort into letting you efficiently observe what’s going on[1].
* One way Linux exposes what’s going on is using tracepoints, which are traces that are actually hard-coded into the source. They are sort of like debug log statements, in the sense that developers hard-code those, and you need to subscribe to them to see them.
* One way to view tracepoints is using bpftrace, which is a simple command-line utility that lets you use eBPF for observability. What I like most about it is that you can easily express most of what you want with it.
* Another way Linux exposes what’s going on is using kfuncs. Unlike tracepoints, kfuncs don’t require the developer to explicitly add anything to their code. Instead, they allow us to trace any function call in the kernel. We can use them to track stuff that does not have tracepoints.
* We will specifically trace vfs_read and vfs_write to trace when the application wants to read and write. We should also trace when these return. We will trace `t:block:block_rq_insert` to show when the request is sent to disk, to show when it actually hits disk.

# Level 2: debugging the kernel and stepping through the code
* Focus on reading.
* Show that initially the application buffer is empty.
* Show it looking in the page cache and not finding it.
* Show it loading the data from disk and copying it to the page cache.
* Show the data in the page cache.
* Show it copying the data from the page cache to the application buffer.
* Show that now the application buffer holds the data.

```
ssize_t vfs_read(struct file *file, char __user *buf, ...)
{
  ... 

	if (file->f_op->read)
		ret = file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)
		ret = new_sync_read(file, buf, count, pos);

	...
}
```

We are going to hook into vfs_read.

Let’s cause it to get triggered: `dd if=example.txt` (we are not using `cat` because it does not actually hit the same code path in the kernel)

Aha! It’s been hit.

```
Thread 3 hit Breakpoint 4, vfs_read (file=file@entry=0xffff8881021ba500, buf=buf@entry=0x55a5b3a4d2a0 "", count=count@entry=512, pos=pos@entry=0xffffc900005d7ea0) at fs/read_write.c:451
451     {
```

First, as soon as we hit vfs_read, let’s confirm that the buffer is empty:
```
(gdb) p buf
$1 = 0x55a5b3a4d2a0 ""
```

Now, I’ll step through the function until we get to `new_sync_read`, and then we’ll step into it.

# Notes
[1] The efficiency allows you to do this on production.
[2] Gregg, 2.3.2 “Time Scales”
[3] Unless the data had been evicted from the page cache.
