---
layout: post
title: "What's Writing to My Brand-new Disk?"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-04-06
tags: featured
description: Tracking down what's writing to a brand-new disk.
---
I am running some disk-related experiments and want to control *exactly* what's using the disk. So, I attached a new disk, created, and mounted a new ext4 filesystem on it.

Sweet! Let's confirm that nothing is touching this pristine disk.

```
-> % sar -d --dev=sdd 1
[...]
Average:          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
Average:          sdd      0.86      0.00    294.29      0.00    343.33      0.03     26.00      1.54
```
(See [notes](#explanation-of-sar) for an explanation of `sar` and its output.)

Uh oh. Something *is* writing to the disk! What is it?

To trace all requests writing to the disk, we can subscribe to the `block:block_rq_issue` tracepoint using `bpftrace`, which is a front-end for eBPF. This method even allows us to see requests made by the kernel itself. To me, this is surprisingly powerful. Until fairly recently, I did not realize how deeply one can inspect the kernel.

The trace tells you which command initiated the request, so we can use that to figure out what's writing to the disk!

If you're interested in the details of how this works, see my [shell script to trace block requests](https://gist.github.com/dmazin/fc8921400eb7ded1770acdc6734fb9da).

And… we have a culprit!

```
Attaching 1 probe...
comm: ext4lazyinit, dev: 8388656, rwbs: W, bytes: 524288
comm: ext4lazyinit, dev: 8388656, rwbs: W, bytes: 524288
comm: ext4lazyinit, dev: 8388656, rwbs: W, bytes: 524288
[...]
```

What is this ext4lazyinit?
From the mkfs.ext4 man page:
> *[…] the inode table will not be fully initialized by mke2fs. This speeds up file system initialization noticeably, but it requires the kernel to finish initializing the file system in the background when the file system is first mounted.*  

[This helpful wiki](https://www.thomas-krenn.com/en/wiki/Ext4_Filesystem#Lazy_Initialization) points out that you should be careful to wait for this to finish before running any benchmarks. This is good advice, worth remembering: in a professional setting, benchmarking is common after setting up a new disk/filesystem.

I'll have to wait for this to finish. Meanwhile, it's empowering to take a mystery (“I expected there to be nothing touching my new disk. What is touching my disk?”) to a solid answer. 

### Notes
#### explanation of sar
Here I'm using `sar`, a disk monitoring utility that I use when I want to print average stats over a given period. The `-d` option prints disk statistics, `--dev=sdd` analyzes `/dev/sdd`, and `1` outputs every second.

The `wkB/s` (average kilobytes per second written) should be 0, but it's not!
