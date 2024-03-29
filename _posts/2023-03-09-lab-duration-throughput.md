---
layout: post
title: "Lab Notes: Throughput vs Duration"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-03-09
tags: labs
---
Hello. It is 2 PM. Today, I took Eamon to the Science Museum. He was sorely disappointed because, last time we visited, they were actually running one of the large steam engines. I believe the old men who are enthusiastic about this sort of thing call it “steaming”. Who am I kidding – I thought it was really cool. Eamon and I watched for a long time. Anyway, they were not doing it today. They probably do it very rarely.

### An update on the job search
I’m speaking to an external recruiter who came highly recommended. Maybe he’ll find me something good eventually, or maybe he won’t. I generally don’t place much hope in these kinds of recruiters.

I’m also talking to an internal recruiter soon about a database role that _might_ be good.

### A meta-note
[Yesterday](https://cyberdemon.org/2023/03/08/lab.html) I made a greater attempt than before to make these notes easier to follow. For example, previously I wasn’t often saying what commands I was issuing or what the output was. I am now trying to make it possible to actually follow what I’m doing.

### A recap of what I’m even doing
Over the past few days, my goal has been to learn how to use sysbench to profile MySQL. Unfortunately, nothing is easy.

* First, I tried running MySQL and sysbench using Docker Compose on my M1 Mac. sysbench was not compiled for this architecture, so it was running using Rosetta 2 and was segfaulting. I decided to use an x86 VM instead.
* Then, it turned out that the sysbench Docker image I was using was compiled using an old version of a MySQL client library, which meant that it couldn’t support MySQL 8’s new default authentication plugin. So, I ended up figuring out how to change the default authentication plugin to one supported in old versions. At this point I was able to run some benchmarks.
* Then, I decided that it was stupid to use the VM, what with its 2 CPUs and 2 GB RAM, when I had a much stronger Intel Mac. So I spent too much time setting it up, including some stupidity related to SSH auth.
* Finally, yesterday I spent the entire time simply installing Docker Engine. Though mostly it took so long because I read a lot about how apt verifies repo trustworthiness, and investigating Ansible’s apt_key module.

Today, I will actually run the sysbench benchmark. The goal with this benchmark is to establish baseline p95 query duration so that, during later experiments, I can see how this is affected.

### On to the benchmark
#### The setup
Recall that I am running everything using Docker Compose. I defined a service called `mysql` which is running MySQL 8, and I am giving it 3 CPUs and 8 GB memory. That is, not only am I reserving these resources for MySQL, but I am also limiting it to these. I think a benchmark makes most sense when you set clear constraints.

I have also defined a `sysbench` service, and I’m using it to run a test called `oltp_read_write`. I’ve set the resources to 1 CPU and 512 MB memory.

Here’s the [full Docker Compose file](https://github.com/dmazin/sysbench-mysql-basic/blob/fe57096e036453c664ed5d21e4af14bbdcba5d8b/docker-compose.yaml).

#### The benchmarks
I’ll put the full output the first time, but for future calls I’ll only write down the things I’m paying attention to.

With 1 sysbench thread, we reached 1840 queries per second (qps), with a p95 duration of 14ms. These are the only two numbers I care about: the throughput and a representative query duration. To understand why I’m focusing on them, see my [recent primer on black-box monitoring](https://cyberdemon.org/2023/02/04/crash-course-black-box-monitoring.html).

Note that I’m going to round things to 0 decimal places – sub-millisecond duration is not important for RDBS.

```
$ docker compose run sysbench --threads=1 oltp_read_write run

sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            12894
        write:                           3684
        other:                           1842
        total:                           18420
    transactions:                        921    (92.01 per sec.)
    queries:                             18420  (1840.11 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0086s
    total number of events:              921

Latency (ms):
         min:                                    8.23
         avg:                                   10.86
         max:                                   24.98
         95th percentile:                       14.46
         sum:                                10004.08

Threads fairness:
    events (avg/stddev):           921.0000/0.00
    execution time (avg/stddev):   10.0041/0.00
```

* With 2 threads, we reached 2,703 qps with a p95 duration of 24ms.
* 4 threads – 4,847 qps  – p95 duration 30ms.
* 8 threads – 8,201 qps  – p95 duration 36ms.
* 16 threads – 10,875 qps  – p95 duration 61ms.
* 32 threads – 11,612 qps  – p95 duration 132ms.
* 64 threads – 8,951 qps  – p95 duration 370ms.

After 8 threads the throughput started plateauing, and after 32 it actually dropped due to resource contention.

#### Interpreting the results
What do the numbers mean? What’s the point of all this?

First, about the numbers. With an OLTP database, extremely roughly you would be fine with 30-100 ms durations. I say extremely roughly, though, because it varies by application and query type. In general, once you go above 100ms it means you have a scaling problem, and, especially for reads, you would never try to reach sub-10ms durations because you would accomplish that with a caching layer.

What that means is that we can say that the database, as configured, can gracefully (i.e. with a good query duration) handle about 10k qps.

What’s the point, though? I hope to write about this in more detail outside the context of “lab notes”, but here are a two important things to understand, as someone operating a service:
1. Throughput, stated without duration, is meaningless, and vice-versa.
2. Throughput and latency are trade-offs.

#### Why are throughput and duration meaningless without each other?
Let me illustrate this with a non-technical example.

Let’s say you run a theme park, and there’s a queue to get in. Inside the theme park, there are a bunch of rides, each one with its own queue.

One day, you get drunk on the job, and you send out a marketing campaign proclaiming that there is absolutely no queue to enter the park, and entice 100,000 people to attend your park, all on the same day. “Very nice,” you tell yourself, “because my park can surely handle 100,000 visitors at the same time.”

Indeed, the next day you let all 100,000 visitors in, without waiting at the queue at the park gates. To your horror, soon hordes of angry visitors start leaving the park: the lines for all the rides are thousands-long. What you failed to think about is the fact that people care how long it takes them before they are able to get on a ride.

Let me explicitly draw parallels between this and database queries. In this case, throughput is how many park visitors you allow in during a time period (100,000 in a day). This is kind of abstract, but you can think of a query time as “after entering the park, how long passes until I am able to get onto a ride?” Because the queues for all the rides are so long, it takes a really long time before someone can actually get on a ride.

#### What are you supposed to do?
OK, you learned your lesson. You now know that visitors will be angry if it takes more than, say, 60 minutes before they can get on their first ride. But now what?

Well, unfortunately, what it means is that at your current capacity you simply cannot let more than, say, a few thousand visitors into your park at any time. The rest either have to wait in queue outside the park, or you need to turn them away.

If you have _demand_ for 100,000 visitors in a day, you should be able to find the capital resources to scale your park to meet that demand while trying to guarantee 60-minute waiting time. Until then, though, the only thing you can do is set expectations: you must ask people to wait outside your park, turn them away, or don’t invite so many in the first place.

#### A technical parallel
Hopefully the park example makes sense, but I might as well translate this back into the technical world.

Instead of talking about a park, let’s say you run some service, available via a REST API. I don’t know, let’s say this service summarizes news articles. So, the park is the service. Your service calls on a bunch of other sub-services, like MySQL and Memcached, and those correspond to the rides in the park.

You release your service, and everything is going great. You aren’t really thinking about throughput and query durations, because you are just trying to get a product off the ground, and, anyway, worrying about those would be a “good problem to have”. Your service grows, and grows, and you take some actions like horizontally scaling Memcached (which is very easy) and vertically scaling MySQL (because that is easier than sharding).

If you don’t plan ahead, eventually, you start running into problems: people are complaining that sometimes, it takes a really long time for an article to get summarized. A really huge customer, who uses you under-the-hood for their own service, leaves you because your service isn’t what it used to be. Oh, no!

What are you supposed to do? At the end of the day, technology is all about communication. Let’s say you charge your customers some flat fee per article summarized. You know you can only handle X articles per second, while guaranteeing Y query duration. So you implement rate limiting, and your API actually starts turning requests away with a HTTP 429. This is equivalent to saying: I don’t feel good charging you money if I can’t handle your request in a timely manner. Please try coming back another time. While this is not a good situation – you are turning away demand – this does allow you to guarantee a certain query duration. Meanwhile, you can work to improve your architecture.

### Oops
To be continued.

This basically ended up becoming a draft for the throughput vs duration article I mentioned earlier. I think in the future version of this draft, I’ll leave out the park analogy entirely. Analogies just tend to confuse me. What do you think?

### Next time
One thing I did not write about is that I actually used atop (which is like top on steroids, much more-so than htop, and it allows for historical analysis) to monitor resource usage during my benchmarks. What I found was that the CPUs were largely unstressed, but the disk write got maxed out at 30 MB/s even for the 1-thread benchmark, while the disk was not read at all (which makes sense given that the entire database fits in memory). Tomorrow, I might explore how I could also exercise the disk for reads or, see how much of the query duration is accounted for by internal queues for the disk. OR, I am also thinking about starting to play with Kafka. I want to continue exploring duration vs throughput, and Kafka is the perfect service for that.
