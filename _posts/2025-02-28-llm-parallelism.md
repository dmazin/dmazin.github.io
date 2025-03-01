---
layout: post
title: "LLMs make embarrassingly parallelizable scripts even more embarrassingly parallelizable"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2025-02-28
tags:
    - technical
description: You can speed up your one-off scripts with an LLM's help.
---
One of my favorite LLM tricks is to quickly make a script take advantage of parallelism.

Before we get too far, let me clarify that I use "parallelism" loosely to describe concurrent execution.

Modern computers, of course, have lots of cores. While much of the serious software we use and write takes advantage of this, usually scripts (especially one-offs) do not warrant the boilerplate required to use those extra cores.

That means that many embarrassingly parallelizable scripts don't get parallelized. As the name suggests<a name="as-name-suggests-return"></a><sup>[[1]](#as-name-suggests)</sup>, that's embarrassing.

For me, LLMs have made such low-hanging parallelism even more embarrassing, because they make it super easy to add that boilerplate to a script.

I mostly use Python, so my example is Python-specific, but this idea applies broadly to quick scripts and one-off programs.

## An example in Python

Sometimes, I need to do something very quick for a bunch of files, hostnames, rows in a CSV, etc.

For example, recently I wanted to download a bunch of pages from a website. Very simple – a call to `requests.get` in a for loop.

```py
import requests

BASE_URL = "https://www.example.com/pages/"


def download_posts():
    for page in range(1, 33):
        filename = f"{page}.html"
        url = f"{BASE_URL}{page}"
		
        response = requests.get(url)

        with open(filename, "w", encoding="utf-8") as file:
            file.write(response.text)


if __name__ == "__main__":
    download_posts()
```

How long does it take?

```
$ time python download_posts.py
python download_posts.py  0.44s user 0.12s system 2% cpu 25.780 total
```

That's slow. Almost 26 seconds of my precious wall time, and I'm only downloading 32 pages because it's a toy example. Usually I'm looping over at least thousands of items to process.

## This is an embarrassingly parallelizable script

Downloading 

This is I/O bound, so multithreading is appropriate.<a name="gil-footnote-return"></a><sup>[[2]](#gil-footnote)</sup> Download the pages in parallel – we've got CPUs and network bandwidth to spare.

I'll be first to admit, though, that I wouldn't be able to implement multithreading for this script without consulting the docs.  I'm just not going to do this for a one-off.

So, I say to the LLM, "Use Multithreading to parallelize this.".

The LLM (Claude 3.7 Thinking) uses `concurrent.futures`. It applies a boilerplate pattern I'm now used to seeing, so I know it looks right. <a name="error-handling-return"></a><sup>[[3]](#error-handling)</sup>

I set the `max_workers` to 32. This is a parameter you want to tune yourself. For example, in this case, if there were enough pages to matter, I'd tune this until I saturated my network interface.

```python
import concurrent.futures
import requests

BASE_URL = "https://www.example.com/pages/"


def download_single_page(page):
    filename = f"{page}.html"

    url = f"{BASE_URL}{page}"
    response = requests.get(url)
    
    with open(filename, "w", encoding="utf-8") as file:
        file.write(response.text)


def download_posts():
    with concurrent.futures.ThreadPoolExecutor(max_workers=32) as executor:
        # Submit all page downloads to the thread pool
        futures = [executor.submit(download_single_page, page) for page in range(1, 33)]
        
        # Wait for all downloads to complete
        for future in concurrent.futures.as_completed(futures):
            # This will raise any exceptions from the threads
            future.result()


if __name__ == "__main__":
    download_posts()
```

25 times faster with no effort.

```
$ time python download_posts.py
python download_posts.py  0.22s user 0.07s system 27% cpu 1.021 total
```

## If you know what you're doing, it's easy to parallelize non-embarrassing scripts too

Often, your scripts store some data in some sort of in-memory data structure (e.g. the list of paths you've already visited) or write to a file (e.g. update a CSV).

That is not embarrassingly parallelizable: it requires a little bit of work. You will need to make your script thread-safe.

Let's say that I wanted to log to a CSV for every page I visited. Maybe I want to keep track of the response codes or something.

```python
import requests
import concurrent.futures
import csv

BASE_URL = "https://www.example.com/pages/"

def download_single_page(page, csv_writer):
    filename = f"{page}.html"
    url = f"{BASE_URL}{page}"
    response = requests.get(url)
    
    with open(filename, "w", encoding="utf-8") as file:
        file.write(response.text)

    # Write to csv
    csv_writer.writerow([page, url, response.status_code])

def download_posts(csv_writer):
    with concurrent.futures.ThreadPoolExecutor(max_workers=32) as executor:
        # Submit all page downloads to the thread pool
        futures = [executor.submit(download_single_page, page, csv_writer) for page in range(1, 33)]
        
        # Wait for all downloads to complete
        for future in concurrent.futures.as_completed(futures):
            future.result()

if __name__ == "__main__":
    csv_file = open('download_results.csv', 'w', newline='')
    csv_writer = csv.writer(csv_file)
    
    download_posts(csv_writer)
    
    csv_file.close()
```

The above is not thread-safe. The download_single_page threads might try to write to that file concurrently, and `csv.writer` itself is not thread-safe.

One easy way to prevent this is to make each thread take a lock before writing.

You can tell the LLM to implement it for you, of course, but importantly you need to know that you need to do it in the first place, and you need to understand concurrency well enough to make sure it's implemented right.

So I say to the LLM, "Use a Lock when writing the CSV file". The LLM uses `threading.Lock`, which is a context manager we wrap around blocks to make sure they execute one at a time.

```python
import requests
import concurrent.futures
import csv
import threading

BASE_URL = "https://www.example.com/pages/"

def download_single_page(page, csv_writer, csv_lock):
    filename = f"{page}.html"
    url = f"{BASE_URL}{page}"
    response = requests.get(url)
    
    with open(filename, "w", encoding="utf-8") as file:
        file.write(response.text)
    
    # Use a mutually exclusive lock
    with csv_lock:
        csv_writer.writerow([page, url, response.status_code])

def download_posts(csv_writer, csv_lock):
    with concurrent.futures.ThreadPoolExecutor(max_workers=32) as executor:
        # Submit all page downloads to the thread pool
        futures = [executor.submit(download_single_page, page, csv_writer, csv_lock) for page in range(1, 33)]
        
        # Wait for all downloads to complete
        for future in concurrent.futures.as_completed(futures):
            future.result()

if __name__ == "__main__":
    csv_file = open('download_results.csv', 'w', newline='')
    csv_writer = csv.writer(csv_file)

    # Use a mutually exclusive lock
    csv_lock = threading.Lock()
    
    download_posts(csv_writer, csv_lock)
    
    csv_file.close()
```

This is far from a perfect implementation. For example, using a single writer thread with a queue would prevent contention for the lock and the file. But, this is safe and fast enough.

## Conclusion

Say what you will about LLMs, but they are killer at boilerplate. That often saves you the time you'd spend on monotonous implementation. They can also literally speed up your scripts.

Just remember that you should actually try to understand concurrency <a name="learn-concurrency-return"></a><sup>[[4]](#learn-concurrency)</sup>, because applying this without understanding what's going on can cause the loss of toes.

# Notes
<a name="as-name-suggests"></a>**1.** I know that's not what "embarrassingly parallelizable _really_ means." <a href="#as-name-suggests-return">(back)</a>

<a name="gil-footnote"></a>**2.** Can't blame the GIL for failing to parallelize in this case - that IO code runs outside Python. <a href="#gil-footnote-return">(back)</a>

<a name="error-handling"></a>**3.** In real life I'd add more error handling and logging, even to a one-off script (yes, LLMs make that easy too). <a href="#error-handling-return">(back)</a>

<a name="learn-concurrency"></a>**4.** What made concurrency click for me was Martin Kleppmann's DDIA (which, like, this is probably the 100th time you've seen this book recommended) (edition 2 is coming sometime soon!) and the <a href="https://www.cl.cam.ac.uk/teaching/2122/ConcDisSys/dist-sys-notes.pdf">notes from his Distributed Systems class</a>. <a href="#learn-concurrency-return">(back)</a>
