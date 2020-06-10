# Thread Design Considerations

Threads can be implemented at the kernel-level, at the user-level or both. In this lecture, we will take a look at the data structures and mechanisms that are necessary to support threads.

## Table of Contents

* [Thread Data Structures]()
* [Thread Management Interactions]()
* [Interrupts and Signals]()

## Kernel vs User Level Threads

<img src="p2_resources/kernelvuser.png">

Supporting threads at the kernel level means that the OS itself is multithreaded. The OS must maintain a thread abstraction, and be able to perform scheduling and synchronization.

At the user-level, it means there is some library that supports all of these mechanisms 

### Thread Data Structures - Single CPU

In a single-threaded process, the process is described in the PCB. 

<img src="p2_resources/thread_related_ds.png">

In a user-level multi-threaded environment where there is one kernel thread to many user threads.  The user-level library will need to represent threads so that it can track their resource use and make decisions around scheduling and synchronization.

If we want there to be multiple kernel level threads, many-to-many, we will need to split up the PCB into a data structure that is useful for tracking the threads in the kernel. 

Kernel-level threads look like virtual CPUs. 

### Thread Data Structures - At Scale

<img src="p2_resources/mapping_threads.png">

Let's say we have multiple multi-threaded processes in user-land. We will need to start maintaining relationships between all the data structures. Some threads belong to one application, others belong to another. We also need to maintain a mapping between these user-level data structures and the PCBs maintained by the kernel and the Kernel-level thread data structures. 

Lastly, if using threading to achieve parallel execution, there needs to be a mapping between the threads at the kernel level and the CPU on the machine. 

### Hard and Light Process State

When two kernel level threads belong to the same address space, there is information in the PCB that is relevant for the entire process, but there is also information that is specific to a single thread. 

When we are context-switching between these two threads, there is a portion of the PCB we want to preserve (virtual address mappings), and there is a portion that is really specific to the specific thread and it depends on which user-thread is currently executing. 

To manage this separation of concerns, we will split the process state into:
* **hard process state** - Relevant to all user-level threads. ex: virtual address mappings
* **light process state** -  Process state that is only relevant for a subset of user-level threads that are currently associated with a specific kernel-level thread. Ex: signal mask and sys call args. 

### Rationale for Data Structures

When we had a **single PCB**, it was a large contiguous data structure. Each PCB was private. Whenever we needed to context switch, we had to restore the entire data structure back into memory.

<img src="p2_resources/why_multiple_ds.png">

Thus there are fundamental limitations in what this data structure can accomplish: scalability, overhead, performance and flexibility. 

In contrast, **multiple smaller data structures** are easier to share, and context switch only saves and restores what needs to change. 

As a result, many OSes today use the multiple data structure approach to organize their execution context.

<img src="p2_resources/linux_thread.png">

### Sun OS Threading Model

<img src="p2_resources/sun_model.png">

This is a diagram of the Stein and Shah paper. It illustrates the threading model supported by the OS. The OS is intended for multiprocessor systems with multiple CPUs. The kernel itself is multi-threaded.

At the user level, the processes can be single or multi-threaded. And both many-to-many or single-to-single mapping between the boundaries are supported. 

The **lightweight process data structure** is an abstraction that virtualizes the CPU that the thread/process is executed on. 

Let's look closer at the user-level thread data structures. These are not POSIX threads, but they are similar. 

Thread creation returns a thread ID. This is not a direct pointer to the thread data structure, instead it is an index in a table of pointers, which in turn point to the data structure.

<img src="p2_resources/user_thread_design.png">

The thread data structure contains a local storage area that includes the variables that are defined in the thread at compile time. This means the compiler can allocate store on a per-thread basis. This improves locality, and make it easier for the thread to find the next thread. 

One problem is that stack growth can be dangerous, because if too much is written to it, it is possible that one thread can overwrite another thread. The solution is to separate the information between threads with a red zone that is not allocated and used as a canary to detect unrestricted stack growth. Any writes to the red zone will cause a fault.

Let's look at the kernel level structures now.

<img src="p2_resources/kernel_level_data_structures.png">

For each process we maintain information about that **process**.
* what are the kernel level threads that execute in the address space.
* What are the user credentials
* What are the signal handlers that are valid for the process. 

Next we have the **lightweight process data structure**. This information is somewhat similar to what is maintained at the user-level, but this is what is visible to the kernel. 
* user-level registers
* system-call args
* resource usage info
* signal mask


The **kernel-level data structure** includes. This information is not swappable, it always needs to exist. This is different from the LWP. 
* kernel-level registers
* stack pointers
* scheduling class
* pointers to associated LWP, process, and CPU structures.

The **CPU data structure**
* The current thread executing
* list of kernel-level threads that ran there
* information about dispatching and interrupt handling

<img src="kernel_all_together.png">

## Basic Thread Management Interactions

<img src="thread_management_intro.png">

The basic problem here is that the user-level thread does nto know what is happening in the kernel, and the kernel does not know what is happening in the user-level library!

**System calls and special signals allow kernel and ULT library** to interact and coordinate. 

<img src="pthread_set_concurrency.png">

### Thread management visibility and design

We just introduced a problem statement, the kernel and the user level library don't have insight into each other's activities. 

At the kernel level, the kernel sees the kernel level threads, the CPUs and the kernel level scheduler. 

At the user-level the library sees the user-level threads, and the available kernel level threads. Depending on the thread association model (one-to-one, one-to-many, many-to-many), the ULT will see how many threads are available.

<img src="bound_thread.png">

A ULT can bound a particular KLT to a particular ULT. 

Let's illustrate the visibility problem. One user-level threads has a lock, and a KLT is supporting the execution of this critical section code. If this KLT is preempted, the execution of the critical section cannot continue. The ULT library will not be able to schedule any more threads because the halted ULT has the lock! Only after the KLT is rescheduled will the lock be released. 

<img src="thread_boundary_invisibility.png">

The essential problem here is that the kernel does not have any visibility into mutex variables or wait queues. 

Let's dig deeper here. The best way to understand these problems is to actually understand how the ULT is implemented and executed. 

<img src="ul_run.png">

The user-level library is part of the same address space as the process that is running all of the threads. Occasionally the execution jumps to the appropriate program counter into the user level scheduling library. There are multiple reasons for this to happen!

1. A User-level thread explicitly yields.
2. A timer set by the library expires.
3. ULT calls library functions like lock/unlock. 
4. Blocked threads become runnable.

### Issues on Multiple CPUs

Imagine a scenario where there are three user-level threads. There is a priority between the threads, the higher the thread number, the higher the priority. 

<img src="multiple_cpus.png">

Imagine T2 and T1 are both running on different CPUs, and a resource that T3 needs becomes unlocked by T2. It is time to evict T1, but the signaling for this scheduling decision comes from the KLT that belongs to T2. How do we signal the other KLT to preempt it?

### Synchronizaton-related issues

Another interesting case when we have multiple CPU systems is related to synchronization.

<img src="multi_cpu_sync.png">

Consider the following situation: One ULT running on top of a KLT on one CPU, this thread has a mutex and a number of ULTs are blocked. 

On another CPU, another ULT is scheduled. Let's say this ULT also needs that same mutex. The normal situation would be to put this ULT onto the mutex queue and have it wait for the mutex to be unlocked.

However, on a multi-CPU system if the critical section is small, it actually might be more efficient to keep that other thread spinning on the CPU until the mutex is unlocked. This saves the OS from having to do the overhead of context switching and putting the ULT on the mutex queue. 

This concept is known as **adaptive mutexes**. They only make sense on multi-cpu systems. 

<img src="destroying_threads.png">

Once a thread is no longer needed, it should be destroyed and its data should be freed. The allocation of threads takes some time and energy so sometimes, the data structures should be recycled. 


<img src="max_threads.png">

## Interrupts and Signals