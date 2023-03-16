---
layout: post
title: "Lab Notes: Something weird is happening with sysbench"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-10
tags: labs
---
## Hello
Hello. It is 2 PM. This morning, I took Eamon to see my cousin Sasha. We got some coffee, and Eamon was very cute, asking the bartenders their names and thanking them.

## What am I doing?
Yesterday, I ran initial sysbench MySQL benchmarks and found that my Intel Mac can handle about 10,000 queries per second without ruining the query duration (keeping it around ~60ms). I reached 10k qps by twiddling with how many threads sysbench used: 16 sysbench threads took me to 10k qps.

I didn’t write this down, but yesterday I noticed that the disk was 100% utilized even at 1,600 qps. It was also 100% utilized at 10,000 qps. What I am confused about is how I was able to squeeze more performance out of MySQL despite the disk already being fully utilized at 1,600 qps.

### An experiment involving dead uncles and elevators
I do have a theory! Despite the disk being 100% utilized, it doesn’t mean it was at 100% capacity. How can that be? Let me illustrate using an elevator.

Say that your rich uncle died, and you have inherited his elevator. I don’t know! And, I guess, now you are going to run experiments on this elevator. Say that the elevator has a capacity of 10 people. Your friends are extremely enthusiastic people, and you convince them to try an experiment. You are all on the bottom floor, and you send 1 friend up. It takes 30 seconds. The friend gets off on the top floor. The elevator comes back down, which takes 30 seconds, and immediately you send another 1 friend up. You repeat. The elevator is at 100% utilization: it is always busy. The throughput is 1 friend per minute. Now, instead of sending 1 friend at a time, you send 10. Now, your utilization is still 100%, but your throughput has increased to 10 friends per minute.

I don’t know if this is handled in the OS or in the disk itself, but I am sure there is something similar happening for writes.

### How can I verify this theory?
I can use `iostat` to monitor the disk. (What is iostat? It’s a monitoring utility for IO, like disks). What I will look for is the utilization %, the throughput (MB per second) [this will be indicated by the `wkB/s` field], and the request size (this will correspond to how many friends we are putting in the elevator at the same time) [this will be the `wareq-sz` field].

I expect the utilization to remain at 100%, but the throughput will increase, which will happen thanks to us stuffing more friends in the elevator (`wareq-sz` will go up).

The specific command I will use is `iostat -xh 1 sda`. `-x` is a flag which prints a ton more info than default, and `-h` “humanizes” it, e.g. converting 1,000 KB into 1 MB. `1` tells it to refresh the output every second, and `sda` limits it to the 1 disk I’m interested in.

Alright, so, I have that running in one window. In another, I’ll run the 1-thread benchmark: `docker compose run sysbench --threads=1 oltp_read_write run`.

Alright, check it out. The `%util` is 100%, `wkB/s` is 45 MB, and the `wareq-sz` is 22.5 KB. We’ll compare these to the next run.

(Here’s the full output)
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.9%    0.0%    3.5%   10.5%    0.0%   83.1%

     r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz Device
 2062.00     45.4M     0.00   0.0%    0.48    22.5k sda

     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     f/s f_await  aqu-sz  %util Device
  576.00    1.30    1.73 100.0% sda
```

Now, let’s run 16 threads: `docker compose run sysbench --threads=1 oltp_read_write run`.

Aha! The `%util` is  still 100%, but we are now writing 59 MB per second, and the `wareq-sz` is 25.4 KB.
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          18.5%    0.0%   10.7%    8.9%    0.0%   62.0%

     r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz Device
 2380.00     59.0M     0.00   0.0%    0.45    25.4k sda

     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     f/s f_await  aqu-sz  %util Device
  631.00    1.23    1.85 100.0% sda
``` 

This means that, as expected, we are able to increase the disk throughput even though the utilization was already 100%.

### What is the maximum write speed of my disk?
Even if I increase the number of sysbench threads (so that I can send more queries per second to MySQL), the disk write speed doesn’t go above 60 MB per second. This is probably the disk’s maximum write speed. However, I’d like to confirm this. The reason is that `oltp_read_write` is not a pure test of disk write speed. For example, as I increase the number of sysbench threads, MySQL has to spend more resources juggling the connections, which could prevent it from fully utilizing the disk.

I need to exercise the disk outside the context of MySQL. I can do this using sysbench’s `fileio` test. Note that I am not truly going to test the _disk’s_ write performance. Instead, I am testing the _OS_ write performance. They are different, but I can’t explain yet what the difference is. Maybe later I can explore that.

### Figuring out which benchmark to run
Anyway, the `fileio` benchmark has a number of modes: `seqwr, seqrewr, seqrd, rndrd, rndwr, rndrw`. I can tell that some of these correspond to sequential vs random reads/writes, but otherwise I can only guess what the modes mean. First, I need to find documentation or read the source. The `help` for `fileio` is not helpful.

```
$ docker compose run sysbench fileio help

sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)

fileio options:
  --file-num=N                  number of files to create [128]
  --file-block-size=N           block size to use in all IO operations [16384]
  --file-total-size=SIZE        total size of files to create [2G]
  --file-test-mode=STRING       test mode {seqwr, seqrewr, seqrd, rndrd, rndwr, rndrw}
  --file-io-mode=STRING         file operations mode {sync,async,mmap} [sync]
  --file-async-backlog=N        number of asynchronous operatons to queue per thread [128]
  --file-extra-flags=[LIST,...] list of additional flags to use to open files {sync,dsync,direct} []
  --file-fsync-freq=N           do fsync() after this number of requests (0 - don't use fsync()) [100]
  --file-fsync-all[=on|off]     do fsync() after each write operation [off]
  --file-fsync-end[=on|off]     do fsync() at the end of test [on]
  --file-fsync-mode=STRING      which method to use for synchronization {fsync, fdatasync} [fsync]
  --file-merged-requests=N      merge at most this number of IO requests if possible (0 - don't merge) [0]
  --file-rw-ratio=N             reads/writes ratio for combined test [1.5]
```

The main `--help` output isn’t helpful either. Nice! Alright, what about the [source](https://github.com/akopytov/sysbench)? grepping the repo for `seqrewr` showed me [a list containing descriptive test names](https://github.com/akopytov/sysbench/blob/df89d34c410a2277e19f77e47e535d0890b2029b/src/tests/fileio/sb_fileio.c#LL1542-L1553C31):
```
    if (!strcmp(mode, "seqwr"))
      test_mode = MODE_WRITE;
    else if (!strcmp(mode, "seqrewr"))
      test_mode = MODE_REWRITE;
    else if (!strcmp(mode, "seqrd"))
      test_mode = MODE_READ;
    else if (!strcmp(mode, "rndrd"))
      test_mode = MODE_RND_READ;
    else if (!strcmp(mode, "rndwr"))
      test_mode = MODE_RND_WRITE;
    else if (!strcmp(mode, "rndrw"))
      test_mode = MODE_RND_RW;
```

What I want is sequential writes (`seqwr`), then. Why?
* I want writes only, because my `oltp_read_write` benchmarks only wrote to disk, didn’t read (I didn’t really expand on this previously, but I think this is because the entire database fits in memory, so the reads don’t hit the disk. I can explore this later!)
* I want to find the maximum write speed of the disk, and sequential writes are going to be faster than random writes because less time will be spent seeking.

### Running the sequential write fileio benchmark
In sysbench, for a benchmark you do 3 things: prepare it, actually run it, and then clean it up. I’ve been omitting this from my outputs. For some tests, you don’t actually need to clean up between runs, because the test cleans up after itself. For example, the `oltp_read_write` test deletes every row it creates. It would probably be best to actually clean up and prepare between every test, though... yeah, maybe I’ll start doing that. I can just use a small bash script to do so.

```bash
#!/usr/bin/env bash

docker compose run sysbench $* prepare
docker compose run sysbench $* run
docker compose run sysbench $* cleanup
```

`$*` expands to all positional arguments. I can use this script like so: `./run_test.sh --threads=16 oltp_read_write`.

Anyway, let’s actually run the `fileio` test: `./run_test.sh fileio --file-test-mode=seqwr`.

Oh, wow. The actual write throughput is WAY higher! 187 MB per second.

```
sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Extra file open flags: (none)
128 files, 16MiB each
2GiB total file size
Block size 16KiB
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing sequential write (creation) test
Initializing worker threads...

Threads started!


File operations:
    reads/s:                      0.00
    writes/s:                     11394.67
    fsyncs/s:                     14596.37

Throughput:
    read, MiB/s:                  0.00
    written, MiB/s:               178.04

General statistics:
    total time:                          10.0032s
    total number of events:              259904

Latency (ms):
         min:                                    0.00
         avg:                                    0.04
         max:                                   31.00
         95th percentile:                        0.01
         sum:                                 9928.30

Threads fairness:
    events (avg/stddev):           259904.0000/0.00
    execution time (avg/stddev):   9.9283/0.00
```

Alright, that makes sense, because these are sequential writes. What about random writes?

Ah, that’s strange. The `prepare` step clearly creates the files necessary for the test:
```
128 files, 16384Kb each, 2048Mb total
Creating files for the test...
Extra file open flags: (none)
Creating file test_file.0
Creating file test_file.1
Creating file test_file.2
...
```

And yet the `run` step complains that it can’t find them.
```
FATAL: Cannot open file 'test_file.0' errno = 2 (No such file or directory)
```

Strange. Indeed, I cannot see the test_files anywhere. Huh. Well, where are these files created during the prepare step?

Maybe I can use `strace` to figure out where the files are created. To help keep output readable, I’ll limit it to just 1 file: `docker compose run sysbench fileio prepare --file-test-mode=rndwr --file-num=1`.

Ah, wait, I don’t need to run anything. I completely forgot that I’m doing all of this in Docker. The files created during the prepare step are created in their own container, and are not persisted in any way that would expose them to the next invocation of the sysbench service.  

OK, so what I can do is rewrite my `run_test.sh`:

```bash
#!/usr/bin/env bash

sysbench $* prepare
sysbench $* run
sysbench $* cleanup
```

I can “upload” the file into the container, without modifying the Dockerfile, like so:
```
  sysbench:
    image: severalnines/sysbench:latest
    volumes:
      - /home/dmitry/dev/labs/sysbench-mysql-basic/scripts:/scripts
```

And then I can execute the prepare/run steps inside a single container: `docker compose run sysbench scripts/run_test.sh fileio --file-test-mode=seqwr`.

So… does it work? Yes!

```
File operations:
    reads/s:                      0.00
    writes/s:                     396.02
    fsyncs/s:                     514.83

Throughput:
    read, MiB/s:                  0.00
    written, MiB/s:               6.19

General statistics:
    total time:                          10.0986s
    total number of events:              9072

Latency (ms):
         min:                                    0.00
         avg:                                    1.10
         max:                                    8.89
         95th percentile:                        3.49
         sum:                                 9993.76

Threads fairness:
    events (avg/stddev):           9072.0000/0.00
    execution time (avg/stddev):   9.9938/0.00
```

Huh. It’s only doing ~6 MB/s. Very slow. Although, strangely the disk write throughput from iostat is extremely different: ~50 MB_s.

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.1%    0.0%    1.8%   11.0%    0.0%   86.1%

     r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz Device
 2256.00     53.2M     0.00   0.0%    0.46    24.2k sda

     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz Device
    0.00      0.0k     0.00   0.0%    0.00     0.0k sda

     f/s f_await  aqu-sz  %util Device
  643.00    1.21    1.83 100.0% sda
```

Let me see if there’s a similar discrepancy between iostat’s throughput and sysbench’s reported throughput for the `seqwr` benchmark.

Running `docker compose run sysbench scripts/run_test.sh fileio --file-test-mode=seqwr --threads=1`.

Sysbench’s throughput is 178 MB/s like before. Iostat’s: 200 MB’s. I mean, this is pretty close. This is not nearly as much of a discrepancy as iostat’s. Let me double-check iostat’s output against another monitoring utility (atop). No, atop agrees with iostat!

Something mysterious is going on. I’ll dive into this on Monday. Have a good weekend!

Meanwhile, we can safely say that 60 MB/s is not the disk's maximum write speed. It is more like 200 MB/s.

If you read this far, maybe you’ll want to hear from me again.
* Sign up to an [RSS feed of my posts on this site](/feed.xml).
* [Follow me on Mastodon](https://file-explorers.club/@dmitry).
* Sign up to [get my posts via email](https://docs.google.com/forms/d/e/1FAIpQLSePJIQBenOoP1GGe26exOhgPCKdqgY4j36D_WAvhTzudcioWA/viewform?usp=sf_link).
cyberdemon.org
If you'd like to discuss this post, please do reach out! My email is [dm@cyberdemon.org]().
cyberdemon.org