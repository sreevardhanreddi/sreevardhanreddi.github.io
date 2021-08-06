---
title: "Concurrency in Python"
date: 2020-08-15T15:06:51+05:30
draft: false
# cover: "https://images.unsplash.com/photo-1434871619871-1f315a50efba?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1053&q=80"
categories:
  - Development
  - Python
tags:
  - python
  - concurrency
  - multi-threading
keywords: "python concurrency multi-threading multithreading async-io"
description: "Concurrency is the ability of different parts or units of a program, algorithm, or problem to be executed out-of-order or in a partial order, without affecting the final outcome. This allows for parallel execution of the concurrent units, which can significantly improve the overall speed of the execution."
slug: "concurrency-in-python"
showFullContent: ""
---

<!-- # Concurrency in Python -->

Concurrency is the ability of different parts or units of a program, algorithm, or problem to be executed out-of-order or in a partial order, without affecting the final outcome. This allows for parallel execution of the concurrent units, which can significantly improve the overall speed of the execution.

Let's look at an example code, which is **not concurrent**.

```python

import requests

urls = [
    "https://docs.python.org/3/",
    "https://github.com/trending/",
    "https://www.youtube.com/feed/trending/",
    "https://stackoverflow.com/questions/",
    "https://docs.djangoproject.com/en/3.0/",
    "https://requests.readthedocs.io/en/master/",
    "https://docs.aiohttp.org/en/stable/",
    "https://pypi.org/",
    "https://itsfoss.com/",
    "https://www.freecodecamp.org/news/",
    "https://gist.github.com/discover",
]

def get_req(url):
    res = requests.get(url)
    print(res.status_code, url)

def synchronous_function():
    for url in urls:
        get_req(url)

synchronous_function()

```

The above program sends a http get request to all the urls specified in the list of the urls synchronously.
what does synchronously mean here?
It means that each request is sent only after the previous http request completes i.e after the previous request returns a response. So for the current request to complete it has to wait for the previous request to complete. Hence the code is executed in a sequential manner. But what if a single url takes a lot of time to respond? the whole code execution is paused until we get a response. Can we make it better? Yes, we can !!.

Let us rewrite the code so that each url request is independent of another url request.

```python
import threading

def get_req(url):
    res = requests.get(url)
    print(res.status_code, url)

def threaded_function():
    threads = []
    for url in urls:
        t = threading.Thread(target=get_req, args=(url,))
        t.start()
        threads.append(t)

    for thread in threads:
        thread.join()

threaded_function()

```

The above code uses python's threading module and uses threads. Threads are small set of individual programs that are executed independently, multiple threads can run at overlapping time intervals. Unlike other programming languages threads in python are different. They do not run parallelly, only one thread runs at a particular time due to GIL.

Let's look at the code which uses multithreading, the implementation goes like this :

- for every url we create a new thread and make a new http get request.
- the thread starts running i.e it is executed by python.

The implementation using a multithreading paradigm is better because all the urls run in different threads and don't have to wait for other urls to finish, each thread is now independent of other and all of them run concurrently in overlapping time.

Let's dig deeper on what overlapping time here means

- assume there are 2 urls and 2 threads
- we know that python can execute only one thread at a time
- **thread 1** sends http request and waits for a response for **url 1**
- while **thread 1** is waiting for the response **thread 2** sends another http request and waits for the response, so when **thread 2** is waiting we would have already got the response for **thread 1** url and later we get the response for **thread 2** url.

So by using multithreading, we can significantly reduce the waiting time for blocking operations such as **Network and IO** operations.

But there is another problem here, usually, multithreaded programs are hard to implement and design.

Python 3.5 introduced async/await. Asynchronous programming improves the performance for blocking operations by running concurrently. So when these features were introduced there weren't many libraries to leverage its power. Libraries like [aiohttp](https://docs.aiohttp.org/en/stable/), [httpx](https://www.python-httpx.org/) etc support async/await.

Let's re-write the above program to use async/await

```python

import asyncio

import httpx

async def async_get(url):
    client = httpx.AsyncClient()
    res = await client.get(url)
    print(res.status_code, url)
    await client.aclose()


async def async_func():
    async_funcs = [async_get(url) for url in urls]
    await asyncio.gather(*async_funcs)

asyncio.run(async_func())
```

The above version looks a lot cleaner than the threading version and performs as good as the threading version.

Here you can find the complete code with benchmarks

```python

import asyncio
import threading
import time

import httpx
import requests


urls = [
    "https://docs.python.org/3/",
    "https://github.com/trending/",
    "https://www.youtube.com/feed/trending/",
    "https://stackoverflow.com/questions/",
    "https://docs.djangoproject.com/en/3.0/",
    "https://requests.readthedocs.io/en/master/",
    "https://docs.aiohttp.org/en/stable/",
    "https://pypi.org/",
    "https://itsfoss.com/",
    "https://www.freecodecamp.org/news/",
    "https://gist.github.com/discover",
]


def get_req(url):
    res = requests.get(url)
    # print(res.status_code, url)


def synchronous_function():
    for url in urls:
        get_req(url)


def threaded_function():
    threads = []
    for url in urls:
        t = threading.Thread(target=get_req, args=(url,))
        t.start()
        threads.append(t)

    for thread in threads:
        thread.join()


def time_taken_for(func):
    start_time = time.perf_counter()
    func()  # or func.__call__()
    end_time = time.perf_counter()
    total_time = end_time - start_time
    print(
        "total time taken for function {} : {:.3f} seconds".format(
            func.__name__, total_time
        ),
    )


async def async_get(url):
    client = httpx.AsyncClient()
    res = await client.get(url)
    # print(res.status_code, url)
    await client.aclose()


async def async_func():
    async_funcs = [async_get(url) for url in urls]
    await asyncio.gather(*async_funcs)


def time_taken_for_async_req():
    start_time = time.perf_counter()
    asyncio.run(async_func())
    end_time = time.perf_counter()
    total_time = end_time - start_time
    print("total time taken for async function : {:.3f} seconds".format(total_time),)


if __name__ == "__main__":
    time_taken_for(synchronous_function)
    time_taken_for(threaded_function)
    time_taken_for_async_req()



# total time taken for function synchronous_function : 10.250 seconds

# total time taken for function threaded_function : 1.369 seconds

# total time taken for async function : 1.315 seconds
```
