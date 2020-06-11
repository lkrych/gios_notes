# Thread Performance Considerations

<img src="which_model_is_better.png">

If we consider execution time, the pipeline threading model is faster. However, if we care about average time to complete order, then the boss-worker model is better. 

<img src="are_threads_useful">

How did we draw these conclusions about threads? How do we determine what is a useful metric? For a matrix multiply application, execution time is probably the metric we want to focus on. For a web service application, it is probably the number of client requests per second. Or maybe response time might be a good metric.

To evaluate a solution, we need to determine the properties that matter, and figure out how to measure that property. 

## Performance Metrics

<img src="performance_metrics.png">

A metric is a measurable quantity that we can use to reason about the behavior of the system. Ideally we can run experiments with real software deployments and real workloads.

### But really, are threads useful?

The answer depends on the metrics and workload that we are interested in. Context matters.

## How to best provide concurrency?

<img src="multithread_vs_multiple_process.png>

Let's compare a **multi-threaded strategy vs a multiple processes** strategy in a web server. 

What are the steps in a web server?
1. client sends a request.
2. web server accepts a request. 
3. server processes a request.
4. server responds by sending a file.

<img src="web_server_steps.png

### Multi-process web server

Let's take a look at a multi-process web server. 

The benefits of this approach are that it is simple. The downsides are that there is high memory usage, context switching has a cost, and 

<img src="multi_process_web.png">

An alternative is the multi-threaded application. All of these threads are executed within the same address space, and every one is executing a request.

The benefits of this approach is that the threads share an address space. Context-switching between the threads is cheap. The memory requirements are lower. The downsides are that the implementation is not simple, it requires synchronization and underlying support for threads. 

<img src="multi_thread_web.png">

Another alternative is the **event-driven model**. 

The application is implemented in a single address space, there is a single process and a single thread of control. 

<img src="event_driven.png">

The main part of the process is the **event dispatcher** that continuously looks for incoming events and then based on those events, invokes a registered handler. Some events might be: receipt of request, completion of send, completion of a disk read.

The dispatcher is a state machine that responds to events by invoking the appropriate handler. 

If the handler needs to block, it hands execution back to the dispatcher which can then proceed responding to incoming events. 

### Concurrency in the event-driven model

How can we achieve concurrency if the event-driven model only has one thread?

In the multi-process and multi-threaded models, there is 1 request per execution context (thread or process).

In the event-driven model, many requests are interleaved within the same execution context. The single thread switches among processing of different requests. 

<img src="event_driven_concurrency.png">

What is the benefit of the event-driven model?

On a single CPU, threads can be useful because they can hide latency. The main takeaway from that discussion is that if a thread is going to idle for twice the time it takes to context switch, then it makes sense to context switch to another thread that will do some useful work. 

If there isn't any idle time. Then it doesn't make sense to context switch. 

There are alternative models wherein multiple CPUs are each running a specific event-driven process. The only gotcha here is that we need a mechanism that will steer the right set of events to the appropriate CPU. That can be complicated. 


An event is an input on a file descriptor (sockets or files). We can use the `select()` call to monitor file descriptors. Another alternative is to use `poll()`. The problem with these is that they have to actively scan file descriptors. 

An alternative is the `epoll()`, which eliminates some of the problems of `select()` and `poll()`. 

<img src="event_driven_benefits.png">

### Challenges with the Event-Driven Model

One big problem with the event-driven model is blocking requests. These can stop the entire model in its' tracks. 

A solution to this problem is **asynchronous I/O operations**. Asynchronous properties have the property that when the thread or process makes the system call, the OS obtains all the relevant info from the caller and where and how it should be returned once it becomes available. 

The event driven model takes advantage of **out-sourcing**. It makes other things go and do the blocking work while it processes its fast non-blocking events. [source](https://www.kislayverma.com/post/event-based-asynchronous-programming) 

<img src="async_io.png">

This is possible because the kernel itself is multi-threaded.  

What happens when async calls are not available?
 
To deal with this, Pai suggests helpers. Which are functions designated for blocking I/O operations only. The communication with the helper can be with sockets or other IPC mechanisms. Thus, `select()` and `poll()` can still be used. 

The helper blocks, but the main event loop will not. In the Pai paper, these helpers were independent processes. 

<img src="helpers.png">

One big downside is that the event-driven model is not generally applicable to all problems. 

## Flash Web Server

<img src="flash_web_server.png">

Flash is an event-driven web server that follows the AMPED model (asymmetric helper processes). 

Helpers are used for disk reads (blocking I/O). Pipes are used for communication with the dispatcher. The helper reads the file in memory via the `mmap()` call.

One optimization that is used by the flash web server is that the the dispatcher checks if pages of file are in memory (using `mincore()`). If they are in memory, it doesn't bother with the helper and just reads the file via a local handler, if they aren't it uses the helper to do the asynchronous work.  


### Flash Optimizations

<img src="flash_cache_optimization.png">

Flash implements application-level caching at multiple levels. It does this on both data and computation. It is common to cache files, this is data caching. However, it is also possible to cache computations. The specific computation that the Flash server caches is the pathname of the file in the web server. It also does response header caches. 

Another optimization is around the networking hardware. All of the data structures are aligned so that it is easy to do DMA operations without copying data. 


### Apache Web Server

<img src="apache.png">

The core component accepts requests and responses and modules are used for handling specific duties.

Like the event-driven model, each request passes through all of the modules. However, Apache is a combination of a multi-process and multi-threaded model.

A single instance (process) of Apache is a multi-threaded boss-worker process that has dynamic management of the number of threads. This is configurable. The total number of processes can also be dynamically adjusted.

### Comparing Flash and Apache

<img src="server_comparison.png">

The metrics used to compare the servers were bandwidth (bytes/time), and connection rate (request/time). 

Both of these were evaluated as a function of file size. These are dueling metrics, the larger the file size, the higher the bandwidth, more data can be sent across a single connection. However, there is also more work per connection, which means that the connection rate will diminish.

