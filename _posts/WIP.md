In [How does Linux really handle writes?](TODO), I covered the way that Linux (and operating systems, in general) shield applications from the slowness of disk access by making writes asynchronous. Today, we're going to keep exploring what happens when you perform a write in Linux. At the end of that article, I teased a (hopefully) intriguing detail: if you write 1 byte to a file on disk, 1 byte will not be written to disk. No. We will write far more: 65 THOUSAND bytes, maybe. Sometimes more. That's a huge disparity, right? It blew my mind. Let's dive right into what's going on.

First, I want to introduce how I discovered this. Six months ago, I began a [sabbatical](TODO) so I could play with Linux. The first thing I wanted to do was run an I/O benchmark against a hard disk (as in, measuring how many megabytes per second I could write to the disk, for example). I thought this would be a simple exercise, but yet the results stunned me and sent me down a disk and file rabbit hole that I am still exploring, 6 months later. So, anyway, what sent me down this rabbit hole?

Well, I ran a sequential write[2] benchmark, for which I used sysbench[1]. sysbench reported that my SSD's throughput was only 6 MB/s (megabytes a second). I knew that an SSD's throughput is much faster than that – 100+ MB/s. What gives?! So, I pulled up a tool, iostat, which can be used to measure the throughput of a disk, live. Indeed, iostat reported that the disk was writing at like 50 MB/s (which isn't terribly surprising, because it was a 10+ year old disk). Why the disparity between sysbench and iostat?

I spent a while flapping around, trying to figure it out (was there a bug in sysbench?) before I realized that I was missing some fundamental knowledge. I needed to read up on this. So, I picked up an excellent book that covers a bunch of important operating system and computing resources at just the perfect balance of superficial and deep, Gregg's System Design (2nd edition). While the book is about performance, in order to make something performant you must understand it, so the book does a great job explaining concepts. Anyway, indeed, I learned what I was missing. And here it is.

In my previous post, I mentioned the difference between logical files and the physical data that live on disk. In fact, the difference between the logical and the physical is profound. Let's cover those concepts now.

When you say, "hello operating system, please write 4 bytes to example.txt", that is a logical write. More specifically, invoking the `write` or `sendfile` syscalls are logical writes. We would say that the size of that logical write is 4 bytes. Generally, we call this "logical IO".

On the other hand, when a disk literally scratches information onto a platter or flashes it into its flash memory, that is a physical write. We call this "physical IO".

Why make this distinction? Well, as we are about to find out, when you say "hey disk write 4 bytes", the disk will write WAY more than 4 bytes. So, it's useful to be able to distinguish between what the application asked, and what actually happened.

Those definitions start to explain the disparity between the 6 MB/s sysbench reported and the 50 MB/s iostat reported: sysbench was measuring logical throughput, while iostat measured physical. To put it more thoroughly, I thought that sysbench answered this question: "how many MB/s can my disk write?" In fact, it answered the question: "how many MB/s could an application write to disk?" This is useful information. While we *do* want to know the theoretical limit of how much a disk can write, we also want to know how much an application could actually write.

At this point, we have two big questions to answer.
1. How do 6 MB/s of logical writes turn into 50 MB/s of disk writes? Similarly, how does asking 4 bytes to be written to disk turn into 65 kilobytes being written?
2. WHY is that disk only capable of 6 MB/s? Can we use our knowledge to increase that?

Let's dive into the first thing. It's a deep dive. If we learned anything, we will be able to answer why the sysbench stats were slow, and make them faster.

So, let's take a breath. We are going to cover a lot. But, unlike my previous post, I am going to do *a lot* of showing. You're going to see some cool traces and we're going to snoop exactly what is happening on disk.

Let's re-iterate what we want to know: how does asking 4 bytes to be written to disk turn into 65 kilobytes being written?

We can answer this! Attached to my computer is a hard disk that is not being used for *anything*. As in, no applications are able to write to it.[4] That means that we can see exactly what happens as a result of a logical write.

*How* will we see exactly what happens? We got a little preview of it in my last post, but Linux is deeply traceable. One, you can attach a trace to literally any kernel function. But we don't even need to do that. In the interests of tracing, kernel developers also add something called "tracepoints" all over the code. A tracepoint is kind of like adding a `console.log` or `print` statement to your code. As in, it's something you manually and explicitly do in your code. By default, a tracepoint has no performance overhead unless you subscribe to it. That's exactly what we'll do: we'll subscribe to a specific tracepoint (the performance overhead will still be insignifiant). Which tracepoint? Specifically, [`block:block_rq_issue`](https://elixir.bootlin.com/linux/v5.6/source/include/trace/events/block.h#L104)[5]. This is called exactly when the kernel asks the disk to write something.[7] This will let us see everything the kernel writes to disk as a result of our logical write. Importantly, the traces have useful metadata, so we will know *why* those physical writes are happening.

So, let's jump right into it.

First, let's craft our logical write. Here it is.

```bash
dd if=/dev/random of=/data/outfile-${RANDOM} bs=4 count=1
```
[8]

In another terminal, I have fired up bpftrace to trace exactly what happens. This article is already going to be long, so I will not cover the trace script here. However, bpftrace is super fast to learn – inspired by awk, it balances expressivity and simplicity really well – here is a well-annotated bpftrace script[9].

First, here's what happened in the terminal where we ran `dd`.
```bash
$ dd if=/dev/random of=/data/outfile-${RANDOM} bs=4 count=1
1+0 records in
1+0 records out
4 bytes copied, 0.000613199 s, 6.5 kB/s
```

As you can see, `dd` says we copied 4 bytes.

What traces did we capture?

```
$ sudo bpftrace trace_logical_physical_io.bt
Attaching 5 probes...
09:55:42 kfunc:vmlinux:vfs_write
inode=15, dev=8388657, buffer=, count=4, comm=dd
kernel stack:
        bpf_prog_b3aa61c265f42c9f_vfs_write+782
        bpf_prog_b3aa61c265f42c9f_vfs_write+782
        bpf_trampoline_6442492302_15+91
        vfs_write+5
        __x64_sys_write+25
        do_syscall_64+89
        entry_SYSCALL_64_after_hwframe+99


09:55:42 kretfunc:vmlinux:vfs_write
inode=15, dev=8388657, buffer=, count=4, comm=dd
kernel stack:
        bpf_prog_b896159dc3ee842d_vfs_write+782
        bpf_prog_b896159dc3ee842d_vfs_write+782
        bpf_trampoline_6442492302_15+249
        vfs_write+5
        __x64_sys_write+25
        do_syscall_64+89
        entry_SYSCALL_64_after_hwframe+99


09:55:47 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526864, bytes: 24576, comm: jbd2/sdd1-8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        jbd2_journal_commit_transaction+3481
        kjournald2+171
        kthread+235
        ret_from_fork+31


09:55:47 tracepoint:block:block_rq_issue
rwbs: W, dev: 8388656, sector: 284672, bytes: 4096, comm: kworker/u24:0
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:47 tracepoint:block:block_rq_issue
rwbs: FF, dev: 8388656, sector: 0, bytes: 0, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:48 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526912, bytes: 4096, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:48 tracepoint:block:block_rq_issue
rwbs: FF, dev: 8388656, sector: 0, bytes: 0, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 19016, bytes: 4096, comm: kworker/u24:8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2600, bytes: 4096, comm: kworker/u24:8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2576, bytes: 4096, comm: kworker/u24:8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2056, bytes: 4096, comm: kworker/u24:8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526920, bytes: 8192, comm: jbd2/sdd1-8
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        jbd2_journal_commit_transaction+3481
        kjournald2+171
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: FF, dev: 8388656, sector: 0, bytes: 0, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: WS, dev: 8388656, sector: 526936, bytes: 4096, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:53 tracepoint:block:block_rq_issue
rwbs: FF, dev: 8388656, sector: 0, bytes: 0, comm: kworker/8:1H
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_sched_dispatch_requests+189
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_run_hw_queues+112
        blk_mq_requeue_work+356
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


09:55:58 tracepoint:block:block_rq_issue
rwbs: WM, dev: 8388656, sector: 2632, bytes: 4096, comm: kworker/u24:0
kernel stack:
        blk_mq_start_request+212
        blk_mq_start_request+212
        scsi_queue_rq+707
        blk_mq_dispatch_rq_list+368
        __blk_mq_do_dispatch_sched+186
        blk_mq_do_dispatch_sched+64
        __blk_mq_sched_dispatch_requests+274
        blk_mq_sched_dispatch_requests+53
        __blk_mq_run_hw_queue+63
        __blk_mq_delay_run_hw_queue+383
        blk_mq_run_hw_queue+165
        blk_mq_sched_insert_requests+114
        blk_mq_flush_plug_list+241
        __blk_flush_plug+233
        blk_finish_plug+49
        wb_writeback+660
        wb_do_writeback+552
        wb_workfn+98
        process_one_work+540
        worker_thread+80
        kthread+235
        ret_from_fork+31


^C

@[FF]: 0
@[W]: 4096
@[WM]: 20480
@[WS]: 40960

@bytes_written: 65536
```

[1] I used sysbench because I was familiar with it because of its usefulness for benchmarking MySQL, but note that smart people recommend fio instead for file benchmarks.
[2] Explain sequential write.
[3] Somewhere near the beginning, explain that I am using ext4.
[4] Link to "what's writing to my brand new disk" because that could be interesting and relevant.
[5] Update the link to the latest kernel source, and maybe reference github.
[7] Explain that we aren't using block:block_rq_complete because it makes it hard to figure out what corresponds to what. It doesn't have the metadata we need.
[8] Explain the command. also explain why we are writing 4 bytes, not 1.
[9] TODO

[10] I need to clean up the fact that originally I'm talking about SSD for sysbench, but my investigation is about HDD. Already, I am thinking of splitting this up into two articles