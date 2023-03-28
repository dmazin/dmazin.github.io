---
layout: post
title: "Lab Notes: Confusing myself with data"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-17
tags: labs
---
Yesterday, I started investigating what happens when you ask your system to write to disk. Today, I won’t have much time to work as we are unpacking over 400 sq ft of crap we brought from the States. But I wanted to run an experiment I thought of yesterday: write far more than 512 bytes, and see how much total data we write instantly, and 30s later. I want to see the ratio so that maybe we can identify how it scales.

* 512 bytes 512 instantly 20480 30s
* 2048 bytes (current fs block size) 2048 instantly 20480 30s
* 4096 bytes (the virtual disk sector size) 4096 instantly 20480 30s
* 8192 bytes 8192 instantly 20480 30s
* 16384 bytes 16384 instantly 20480 30s
* 65536 bytes 65536 instantly 28672 30s
* 131072 bytes 131072 instantly 12288 30s
* 1024*1024 bytes (1 mebibyte) 1048576 instantly 20480 30s

The 30s numbers get interesting at 65536 bytes. Let me rerun those to make sure the numbers are consistent between runs.

Interesting. This time around, the after-30s bytes written changed, but it was always either 20480 or 12288.

For sanity, let me try deleting `/data/foo` before each run. I’ll rerun the entire experiment.

```bash
declare -a sizes=(
    512
    2048
    4096
    8192
    16384
    65536
    131072
    1048576
)

for size in ${sizes[@]}; do
    echo "### running experiment for $size ###"
    rm /data/foo
    python3 disk_stat_diff.py sda5 &> /dev/null
    dd if=/dev/zero of=/data/foo bs=$size count=1
    python3 disk_stat_diff.py sda5
    sleep 30
    python3 disk_stat_diff.py sda5
done
```

Wow. Removing `/data/foo` first had a huge effect: the “instantly” bytes copied stat is now always 0. However, the after-30s stat is now really random, and for bytes above 65536, smaller than the number of bytes I’m asking it to write! I think this is because I’m not waiting long enough for the cache to be flushed (or something). I will now wait 60s.

All of these are after 60s. Instantly, it’s always 0.
* 512 38912
* 2048 45056
* 4096 47104
* 8192 45056
* 16384 59392
* 65536 108544
* 131072 174080
* 1048576 1091584

These are very different than before! Also note that from 4096 to 8192 it got smaller.

I want to try something else now. Now that I know that removing `/data/foo` first makes the instant number 0, I want to try the same with O_DIRECT, bypassing the filesystem cache, to see what the results will be.

* 512 2048 32768
* 2048 2048 32768
* 4096 4096 32768
* 8192 8192 32768
* 16384 16384 32768
* 65536 65536 40960
* 131072 131072 36864
* 1048576 1048576 36864

I am rerunning the 65536 test to see if it comes back with 40960 again. Ah, no, this time it’s 36864. So, unfortunately this test is not quite repeatable, but I think 36864 is the number that gives the clue about what’s going on.

I think 32768 might be the true block size of the SSD.

Oh, yesterday I was looking at what happens when we write 512 bytes with various settings. I want to try 2 more experiments: 1) don’t remove the file first, and wait 60s 2) remove the file first, and wait 30s.

* Yesterday’s result with O_DIRECT that I want to compare to: when we write 512 bytes with oflag=direct, right after dd exits, we see we see 2048 bytes getting written, and 30 seconds later, a further 20480 bytes.
* When we write 512 bytes with oflag=direct and deleting `/data/foo` first, right after dd exits, we see we see 2048 bytes getting written, and 30 seconds later, a further 18432 bytes.
* When we write 512 bytes with oflag=direct but not deleting `/data/foo` first, right after dd exits, we see we see 2048 bytes getting written, and 60 seconds later, a further 20480 bytes. (This was the same as waiting 30 seconds).

Note that 20480 - 18432 = 2048.

It’s clear that removing `/data/foo` first has some sort of effect.

Anyway, I didn’t have much time today, but I wanted to collect some numbers, mostly just to confuse myself, because I am definitely more more confused. That's OK, though! We just filled our house with 80 boxes of shit we brought from the States. First, everything will be a mess. But hopefully we'll come out better off. Same thing with my file writing adventure.
