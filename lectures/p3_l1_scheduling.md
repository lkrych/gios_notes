# Scheduling

In the subsequent lectures we will look at how the Operating System manages resources. First we will look at how the OS manages CPUs, and how it decides how processes and threads will execute on the CPU. We will review basic scheduling mechanisms, and data structures. 

We will also look at Linux schedulers and scheduling on multi-CPU platforms.

## A Visual Metaphor

<img src="p3_resources/visual_metaphor.png">

Running with our toy shop metaphor, an OS scheduler is like a toy shop manager responding to work loads. 

## Scheduling Overview

The **CPU scheduler decides how and when processes access shared CPUs**. In this lesson we will use **task** to interchangeably mean processes or threads.

The scheduler concerns itself with **both user-level tasks and kernel-level tasks**.

<img src="scheduling_overview">

The responsibility of the scheduler is to **pick a task from the ready queue and dispatch it to the CPU**. 

**Whenever the CPU becomes idle, we need to run the scheduler**. The goal is to pick another task to run as quickly as possible so that the CPU doesn't sit idle for very long. We also need to run the scheduler when a new task becomes ready. **A common way schedulers share the CPU is to set a timer (timeslice)**. When a timeslice expires, the scheduler must run.

Which task should be selected? The answer **depends on the scheduling policy/algorithm**.

How is the scheduling policy accomplished? The details depend on the **runqueue data structure**.  The design of the scheduling algorithm is coupled with the design of the runqueue. 

### Run-to-Completion Scheduling

Run-to-completion scheduling assumes that as soon as a task is assigned to a CPU, it runs until it completes. 

For the purpose of the discussion, it's necessary to make some assumptions. First we can assume that we have a group of tasks that need to be scheduled. Second, we can assume the we know the execution times of the tasks. Third, we can assume that there is no preemption in the system. Fourth, there is a single CPU.

Because we will be comparing scheduling algorithms, we need some metrics to compare them. 

* throughput
* average job completion time
* average job wait time
* cpu utilization

<img src="fcfs_metrics.png">

The first algorithm we will talk about is **First-Come First-Server (FCFS)**. In this algorithm tasks are scheduled in the **order that they arrive**. 

Clearly a good way to organize the runqueue, would be a **queue**, so that tasks can be picked up in a FIFO manner.  All the scheduler would need to know is the head of the queue and how to dequeue tasks.

To make a decision, all the algorithm will need to do is pop from the queue. 

This design is simple, but the wait time is really long if a long running job is scheduled in the middle of the queue.

<img src="sjf_design.png">

A variation on this algorithm is **Shortest Job First (SJF)**, which schedules tasks in order of their execution time. 

We could organize the data structure as a queue, but we will need to iterate through it to find the shortest run-time each time to find the next job O(n). **One thing we can do is maintain the queue as an ordered queue**. This makes the insertion more complex, but the selection of a task easier.  Or we can use a tree.

<img src="sjf_metrics.png">

### Preemptive Scheduling: SJF + Preempt

So far we've assumed that a task on our CPU cannot be preempted. Let's relax that requirement. 

Let's look at some tasks that don't all arrive at the same time. Using SJF, when shorter tasks arrive, the running process is preempted and the shorter jobs are scheduled. 

<img src="preemptive_scheduling.png">

One of the details we've glossed over in these simple examples is that **the scheduler will not actually know the execution time of a task**. 

The Scheduler uses heuristics to guess what the execution time will be. Usually the scheduler uses an average of the past execution (windowed average) times to estimate how long it will take. The variability of these execution times has to do with what's in cache, how the network is working, etc. 


### Preemptive Scheduling: Priority

<img src="priority_preemptive.png">

Another criteria that could drive preemption is **priority**. This is a pretty common scenario, OS kernel level tasks that manage critical system components typically have higher priority than user-level threads.

The scheduler needs to know how to run the highest priority task next. 

**This algorithm can be achieved using multiple runqueue structures, a different one for each priority level**. The scheduler will then select a ready task from the highest priority queue possible. 

One danger with priority scheduling is **starvation**, where a low priority task never gets run because higher priority tasks keep getting scheduled.

A mechanism to prevent starvation is **priority aging**, the priority is a function of the task and the time spent on the queue, so the longer it is on the queue, the higher priority it becomes. 

<img src="preemption_quiz.png">

### Priority Inversion

<img src="prioritiy_inversion.png">

**Priority inversion** is the phenomenon where high priority threads end up running last because they need to acquire locks held by preempted lower priority threads. 

One solution to this problem is to **temporarily boost the priority of the mutex owner** during execution. In the example above, this would mean boosting T3 to P1 priority and have it execute quickly so that it can unlock the mutex. 

### Round Robin Scheduling

<img src="rr_io.png">

A popular scheduling algorithm is round robin scheduling. In round robin, the first task from queue is picked up (like FCFS). The task may be preempted (unlike FCFS).

<img src="rr_noio.png">

Round Robin can also include priorities. When a higher task arrives, the lower priority task is preempted. Otherwise the task is served liked FCFS. 

Another modification is round robin with interleaving. In this strategy, the scheduler uses timers and timeslices to cycle between the different tasks.

<img src="rr_interleaving.png">

### Time Sharing and Timeslices

A **timeslice** is the maximum amount of uninterrupted time given to a task. It is also referred to as a quantum.

A timeslice is a maximum amount of time, so a task may run with a shorter duration for instance if it needs to do I/O or is blocked on synchronization. In this case, it will be placed on a queue. 

Using timeslices allow tasks to timeshare the CPU. CPU-bound tasks are preempted after timeslice. 

<img src="time_slicing_metrics.png">

You can see that time slicing allows us to achieve fairly good metrics without having to use a shortest job first algorithm. The benefit of this method is that shorter tasks finish sooner, it is more responsive, and lengthy I/O operations are initiated sooner. 

The downside of this strategy is all of the interruptions, this creates overhead from the context switching. 

The timeslice should always be bigger than the time to context switch to minimize these overheads. 

To answer **how long a timeslice should be depends on whether we are mainly servicing I/O-bound operations or CPU-bound operations**. 

### CPU Bound Timeslice length

<img src="cpu_bound.png">

For **CPU bound tasks, larger time slices are better**. We don't really care about their responsiveness, the user really cares about when they complete and overall when all the tasks finish. 

### IO Bound Timeslice length

<img src="io_bound.png">

For IO bound tasks, in the scenario above, the two tasks are equivalent because of IO blocking. 

<img src="io_bound2.png">

For a scenario with only one IO-bound task, we can see that a **smaller timeslice results in better performance**. It allows us to keep the CPU and the I/O devices busy.

<img src="timeslice_summary.png">
<img src="timeslice_utilization.png">

### Runqueue Data Structure