---
layout: post
title: "A crash course in black-box monitoring"
byline: By <a href="http://archvile.net/">Dmitry Mazin</a>.
date: 2023-02-04
---
### Note
This is an alpha-release article: a good strategy for anything you make is to release it so early that it embarrasses you, and then iterate. I'm applying the same idea to what I write. If nothing else, currently this article is too long.

### Congratulations
Congratulations! You are the proud owner of a service. Has someone already used the “operational excellence” words at you? Sure, that sounds good, you said. But what does it mean? It means that our service is generally alive, and it works well (which mostly means fast, but also, like, we aren’t losing people’s data). When the service is down or not performing well, it’s your problem. Here’s another problem, though: *how* do you know it’s down or not performing well?

### Black-box vs white-box monitoring
There are broadly two ways to monitor a service: measure the clients’ experiences, from the outside (“black-box”) and measure stuff going on inside (“white-box”).

What *is* black-box monitoring? It means directly measuring the “operational excellence” of your service. Since that is our ultimate goal, I think it’s worth thoroughly discussing it. I’m going to focus on it in this post. But, let me discuss the difference between client and internals monitoring.

Your measurements of the client’s experience are the ultimate source of truth of your quality of service. Even if your internal monitoring says that nothing is wrong, yet the client’s experience is degraded, then something *is* wrong.

If black-box monitoring is the source of truth, why do we monitor the internals? Because if we only measure the current client experience, we will only catch issues as they happen. We prefer to catch them before, preventing any client degradation. But, for the sake of scope, I will focus on black-box monitoring here.

### We're not talking about client-side/frontend monitoring
Just to be clear, this article is about monitoring the requests clients make to your service. There is another concept, client/frontend monitoring, which tracks the lifetime of the user action: it measures things like time to first byte, page draw time, etc. This is important stuff, but it's not what this article is about.

### Making things concrete
I understand things a lot better when I have concrete examples to look at. Maybe you do too. So, in our case, let’s say that we are running a service called Ping. It’s very simple. Clients talk to it using HTTP. They can `GET /api/v1/ping` and it returns `HTTP 200 pong`.

### Monitoring is not alerting
Note that monitoring is different than alerting. Alerts are generated from monitors, but you don’t need to be alerted about everything that you monitor. When something is worth alerting on, I’ll make that clear. Why, you may ask, would we monitor things that we don’t alert on? Mostly so that we can check them when investigating an issue.

### What do individual clients care about?
In the intro, I briefly defined operational excellence. It’s worth dwelling on this. 

Let’s say you’re a client of Ping. What do you care about?

* Did Ping respond to your request? (Is it available)
* Did Ping respond correctly to your request (remember that it’s supposed to return `HTTP 200 pong`. If it didn’t (e.g. `HTTP 502`) that’s not good)
* Did Ping respond quickly to your request?

### Why is speed important?
Here’s what matters about almost every computer-related service: “is it there when we need it?”, “does it work?”, and “is it fast?” I think the “fast” point deserves a bit of explanation. It’s not obvious. In real life, I think we can agree that everything should be there when we need it, and it should work. But fast? We don’t want everything to be fast. Maybe your bus ride to work is really pleasant and you’re sad when it’s over. With computing, though, speed is a fundamental virtue. Can you think of anything that you want to be slower? Page loads, database queries, key press response times. For all of them, more speed is better.

### Zooming out
We outlined what an individual client cares about, but we are the operators of Ping, so we want to monitor the aggregate experience of all clients at once.

### The first metric: load
First, I want to introduce one more thing to monitor that only becomes relevant when talking about multiple clients: load. *How many* clients are using Ping? For an HTTP service like Ping, this almost always means requests per second averaged over some time period. 

I mentioned earlier that you won’t alert on *everything*. Load is such a thing. Instead, it’s one of the things you will check on when investigating an issue. In fact, it is almost always the very first thing you check. “There’s an issue!” -> “Is the service under abnormally high load?” Here’s an example of how you might discuss/think about load: usually, you’ll be investigating some time period, and you may say, “the average load over the past 5 min is so and so, which is above the historical average,”, Or you may say, “the load is actually really low”.

### The second metric: availability
When discussing the experience of an individual client, we asked: did Ping respond to your request? This is availability. For most intents and purposes, you can think of availability as “what % of time was the service reachable?” In practice, you would measure this by periodically sending requests to your service. Note that we are talking about black-box monitoring, so the requests should originate outside your system.

While load is a metric that helps with investigations, availability is one of the metrics that grade the quality of your service. It’s the core metric for many of the services you might use; for example, cloud providers make promises about availability. Those are SLAs (worth covering in another post). For now, what you should know is that availability is so fundamental that you will likely make goals around improving or keeping it up, and customers and people in your organisation will scrutinise it. It is common to publicly expose it and let your customers know when it’s violated (note to self: incident communications, and public post-mortems, are also worth covering).

### The third metric: errors
We agreed earlier that one of the non-controversial qualities of a tool is, “does it work?”. Did your hair dryer turn on when you wanted it to? I think this is a fairly straightforward metric: we want to know when our service is broken.

In terms of Ping, we’d track our HTTP responses. Specifically, we’d measure 5XX. *Not* 4XX, though – those are client errors. 

This *is* something you should alert on. How you alert on it depends on the service. For some services, issuing any errors is a problem, so you may alert on the absolute number of errors (“more than 5 errors in the past minute”). More likely, you will alert on the error rate: (“more than 1% of requests errored out in the past minute”).

### Aside: where are we measuring this stuff?
I want to pause to address a subtlety that I did not address yet. Where are we actually measuring this? Since this is black-box monitoring, you may think we measure this from the client. This *is* a thing, and we *should* measure client experience directly on the client (and there are many other things we should measure and care about, like “time to first byte”). We do not measure from the client.

In terms of availability, we measure from something pretending to be a client, but in terms of the other metrics, we’re measuring with something that sits inside of our own system. We could measure using something external to our service (a proxy that tracks every request in and out of our service) or internal (for example, if we use a tracing library). I won’t go into implementation details here; what’s important is to disambiguate that while we are measuring client experience, we are actually doing it from our side.

### The fourth metric: duration
We agreed that speed is important. In fact, request duration is just as important as availability and error rate. Query durations directly measure the lived experience of our clients: if requests are slower, they are less happy.

When something is wrong with your service, in many cases it manifests not as errors or downtime, but slowness. This is because our computing systems are full of queues. This could end up being a major tangent, but…

### Aside: Utilization and saturation
Let’s say we operate an elevator. Briefly, utilisation is how *busy* the elevator is. If it’s in motion, it’s busy. Then, for some time period, its utilisation is what % of time it was in motion.

What do we do when a person arrives at the lobby, but the elevator is in motion? Do we turn them away? No – we let them queue. The same is true for CPUs, disks, buses (I mean literal road buses). Mostly, what it’s *not* true for is boats. “That ship has sailed.”

The length of the queue measures the *saturation* of the elevator. How backed-up is the service? Another way is latency. Latency just means “how long did I spend standing in line?” Latency and queue length are equivalent: the longer the queue, the longer we spend in the queue.

### Aside-aside: query duration is not query latency
These terms are almost always conflated. They don’t mean the same thing. Query duration is how long the query takes, end-to-end. It’s one of the big 4 metrics we discuss in this article. There is also query latency, which measures *how long the request waits before we begin to process it*.

For the elevator example, latency is how long we stand in line, while duration is how long we stand in line plus how long we spend inside the elevator.

### Aside-aside-aside: The relationship between query duration and latency
There’s an interesting relationship between the two. At the risk of confusing everyone all out of proportion, I want to explain it.

Let’s go back to the Ping example, except let’s say Ping is dependent on MySQL. That is, a request Ping works something like this:
1. Client makes an HTTP GET to the endpoint.
2. Ping establishes a connection to MySQL and does `SELECT 1;` to verify that we are able to connect to MySQL and perform a query. (This example is not as contrived as you may think. This is actually a real-life health check you might run, e.g. to power the readiness probe for a Kubernetes pod.)
3. If it’s successful, Ping returns an HTTP 200.

The request duration tracks the request end-to-end. Where does latency come in? Let’s say that you are using thread pooling in MySQL, and MySQL is getting a lot of requests, so Ping’s `SELECT 1;` gets queued. The time Ping waits before the `SELECT` is processed is latency. Notice something important, though: so far as clients are concerned, they don’t know or care that a request that Ping made got queued. That latency is externalised as query duration.

### Back to query duration
Let’s go back to what I was saying: “When something is wrong with your service, in many cases it manifests not as errors or downtime, but slowness. This is because our computing systems are full of queues.”

Any service you create will depend on other things. Perhaps it’ll depend on other services, like MySQL or Redis, some other internal service accessed via HTTP or gRPC, or an external service like a weather API. As you saw above, that means that your service may end up waiting before its request is processed. This wait might be longer as one of the services you depend on might have high load, or a configuration issue. This manifests as longer request durations experienced by your clients.

Even if your service doesn’t depend on any other services in an obvious way, it still depends on components which are likely shared. For example, maybe your service writes to disk, and that disk is under high load. Guess what happens when the disk is under high load? That’s right – your service’s request to write will be queued. In most instances, this is less likely than the above issue where your service is waiting on some other service.

Because modern systems are pretty complex (services that depend on shared services that depend on other shared services that depend on shared components), there are lots of ways for things to be slow. And, for this reason, query duration is where many issues will manifest.

### Measuring duration
We’re not done with duration yet. Next is the question of how to measure it. Remember, we are interested in duration in aggregate.

I know what you’re thinking: let’s take the average of query durations across some time period. Averages are easy to reach for, and are easy to understand, but unfortunately they don’t tell a very good story.

Your request durations are going to be all over the place. Some of them will be very fast, some will be slow. And some will be super duper fast or super duper slow. Those are outliers. You know what outliers do to averages? It’s very nasty. Just imagine that you get a bunch of requests at 10 ms, and everything is great. Then you get 1 request that takes a minute – a complete fluke – and suddenly all your alarms are going off. That’s not good.

Alright, so what should you measure instead? Preferably, you should look at a number of *percentiles*: p50, p75, p90, p99, p99.5. You’ve probably heard of medians. A median says “50% of values are smaller than this.” A percentile says (let’s take p99 as an example): “99% of values are smaller than this.”

In practical terms, it’s rare to be able to think about a bunch of percentiles at once. If you have to pick one, I think you should focus on p90, p99, or even p99.5 based on your business needs. I argue for looking at p95+ because the reality of the world is that your most valuable customers tend to have tons of records in the system and their requests tend to take longer. If you ignore the very tip of the tail of your requests, you may actually be ignoring the experience of your most valuable customers.

Just one more thing about percentiles: I won’t get into the details, but know that you cannot take an average of percentiles. Maybe store this fact in your pocket, because at some point you will be tempted to do it.

### Alerting on duration
Because duration captures a lot of issues, we should alert on it. In fact, like with availability, we should establish a goal and try to reach it or maintain it. For example, you may say, “over any 4 week period, we want the p99 duration to be under 1 second.” If we start to violate the desired duration, we should alert. Remember, our goal is to catch client issues, so if query duration is high, that means that a core client expectation (speed) is being violated.

### If you remember nothing else
I probably went into too many details, so let me summarise.

It’s important to monitor the client experience of a service, because it directly measures the things your clients care about. You should monitor whether or not your service is up (available), functional (errors), and fast (query duration), and you want to measure load as it is the most important factor that affects the other metrics.

Availability and duration are so important that both customers and people inside your company will pay attention to them. You should, too.

There are many intricacies to all of this – including how to actually implement it – but I hope this has been a worthy introduction.