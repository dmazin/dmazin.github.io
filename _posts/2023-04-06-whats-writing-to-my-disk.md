---
layout: post
title: "What's writing to my brand-new disk?"
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

Here I'm using `sar`, a disk monitoring utility that I use when I want to print average stats over a given period. The `-d` option prints disk statistics, `--dev=sdd` analyzes `/dev/sdd`, and `1` outputs every second.

The `wkB/s` (average kilobytes per second written) should be 0, but it's not! Something is writing to the disk! Can I find out what it is?

To trace all requests writing to the disk, we can subscribe to the `block:block_rq_issue` tracepoint using `bpftrace`, which is a front-end for eBPF. This method even allows us to see requests made by the kernel itself. To me, this is surprisingly powerful. Until fairly recently, I did not realize how deeply one can inspect the kernel.

The trace tells you which command initiated the request, so we can use that to figure out what's writing to the disk!

Feel free to skip over my script, but here it is, for posterity.
```bash
# The device ID reported by the tracepoint is a combination of the major and minor numbers of the device, packed into a single integer value. This is the first time in my life I've actually had to do any bit manipulation.
# By the way, what is the major and minor number? This is how the kernel *actually* identifies disks. This is similar to how your numeric user ID is how the kernel *actually* identifies you – your actual username is a mere convenience.
function get_dev_id() {
    local dev=$1
    local major_number=$(lsblk -n -o MAJ:MIN $dev | cut -d: -f1)
    local minor_number=$(lsblk -n -o MAJ:MIN $dev | cut -d: -f2)
    echo $((major_number << 20 | minor_number))
}

export DEVICE_PATH=/dev/sdd

# The stuff inside the backslashes, e.g. args->rwbs == "W", is how you filter events in bpftrace.
# We are filtering down to rwbs == "W" because this corresponds to requests to actually write to the disk.
# Then, just like with awk, you can give bpftrace a function to call for each trace. In our case, we're just printing the bits of the trace payload that we care about: the command, the size of the request, the device, and the request type.
sudo bpftrace -e "t:block:block_rq_issue \
  /args->dev == $(get_dev_id $DEVICE_PATH) && args->rwbs == \"W\"/ \
  { \
    printf(\"comm: %s, dev: %d, rwbs: %s, bytes: %d\n\", \
      comm, args->dev, args->rwbs, args->bytes); \
  }"
```

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
