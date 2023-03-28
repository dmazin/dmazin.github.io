---
layout: post
title: "Lab Notes: Confirming That Writes Get Buffered"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-20
tags: labs
---
Hello. I am sat at a cafe in Kentish Town. Today I needed to return some packages at an Evri, up on Brecknock Road. I hate walking all the way up to it, so I rented a Lime bike. Thought I was going to die the whole time; this bike weighs 70+ lbs; 10/10.

Recently all my crap arrived from the States. One of the things that has arrived is my copy of Kerrisk’s Linux Programming Interface. I’ve had the PDF all along, but listing through it physically made me notice that it discusses file I/O extensively. I'm reading through it now.

First, Kerrisk gives a good way to think of file I/O syscalls like `write`: don’t think of them as writing to disk; think of them as transferring data from your program’s buffer to the kernel buffer.

How does the data leave the kernel buffer and get transferred to disk? A kernel thread named `pdflush` flushes kernel buffers to disk every 30s (this is configured by `/proc/sys/vm/dirty_expire_centisecs`):
```
$ cat /proc/sys/vm/dirty_expire_centisecs
3000
```

If we track what actually gets written to disk using the disk stats revealed through `/sys/dev`, we can see this happening.

```bash
# call disk_stat_diff at the beginning to checkpoint the current state of the disk
python3 disk_stat_diff.py sda5 &> /dev/null
# delete /data/foo first so we have a clean experiement
rm /data/foo
# use `dd` to write 512 bytes to this file
dd if=/dev/zero of=/data/foo bs=512 count=1

# now periodically check how much we wrote to disk in the past second
for time in $(seq 0 10); do
    sleep 1
    echo "written after ${time}s:"
    python3 disk_stat_diff.py sda5 | grep 'write_bytes'
done
```

You can see that the data does not actually get flushed to disk for a few seconds.
```
written after 0s:
| write_bytes     |    0 |
written after 1s:
| write_bytes     |    0 |
written after 2s:
| write_bytes     |    0 |
written after 3s:
| write_bytes     |    0 |
written after 4s:
| write_bytes     | 22528 |
written after 5s:
| write_bytes     |    0 |
written after 6s:
| write_bytes     |    0 |
written after 7s:
| write_bytes     |    0 |
written after 8s:
| write_bytes     |    0 |
written after 9s:
| write_bytes     | 16384 |
written after 10s:
| write_bytes     |    0 |
```

On the other hand, if you ask the fs to bypass its write buffer (in `dd`, this means setting `oflag=direct`) and write the data directly to disk, you can see that data is immediately flushed to disk.
```
written after 0s:
| write_bytes     | 2048 |
written after 1s:
| write_bytes     |    0 |
written after 2s:
| write_bytes     |    0 |
written after 3s:
| write_bytes     |    0 |
written after 4s:
| write_bytes     |    0 |
written after 5s:
| write_bytes     | 20480 |
written after 6s:
| write_bytes     |    0 |
written after 7s:
| write_bytes     |    0 |
written after 8s:
| write_bytes     |    0 |
written after 9s:
| write_bytes     | 16384 |
written after 10s:
| write_bytes     |    0 |
```

Why are we writing 20480 and 16384 bytes later? I think that’s metadata updates, and figuring that out is most definitely the top thing on my mind since I noticed this last week.

Also, if you’re wondering why we wrote 2048 bytes to disk when we asked `dd` to write only 512, that’s because the record size of my file system is 2048 bytes – see my previous posts to understand the meaning of this.
