# Synchronization Concepts

## Table of Contents

* [Synchronization Constructs](#introduction)
    * [Spinlocks](#spinlocks)
    * [Semaphores](#semaphores)
        * [Posix Semaphore API](#posix-semaphores)
    * [Reader/Writer Locks](#reader-writer-locks)
    * [Monitors](#monitors)
    * [More Synchronization Constructs](#more-synchronization-constructs)
* [Discussion: The Performance of Spin Lock Alternatives for Shared Memory Multiprocessors](#the-performance-of-spin-lock-alternatives-for-shared-memory-multiprocessors)

## Introduction

Mutexes by themselves can only represent simple binary states. We cannot express arbitrary synchronization conditions using the constructs we have seen already.

Synchronization constructs require low-level support from the hardware in order to guarantee correctness of the construct. Hardware provides this type of low-level support via **atomic instructions**.

The discussion in these notes will cover alternative synchronization constructs that can help solve some of the issues we experienced with mutexes and condition variables as well as examine how different types of hardware achieve efficient implementations of the synchronization constructs.

### Spinlocks

One of the most basic synchronization constructs is the **spinlock**.

Spinlocks provide mutual exclusion, and it has some constructs that are similar to mutex lock/unlock operations. However they are different from mutexes in an important way.

When a spinlock is locked, and a thread is attempting to lock it, that thread is not blocked, it is spinning. It is **running on the CPU and repeatedly checking to see if the lock has become free**. With mutexes, the thread attempting to lock the mutex would have relinquished the CPU and allowed another thread to run. **With spinlocks, the thread will burn CPU cycles until the locks become free or until the thread becomes preempted**.

Because of their simplicity, spinlocks are a synchronization primitive that can be used to implement more complicated synchronization concepts.

### Semaphores

As a synchronization construct, **semaphores allow us to express count-related synchronization requirements**.

They are initialized with an integer value. **Threads arriving at the semaphore with a nonzero value will decrement the value and proceed with their execution. Threads arriving at a semaphore with a zero value will have to wait.** As a result, the **number of threads** that will be allowed to proceed at a given time will equal the **initialization value of the semaphore**.

When a thread leaves the critical section, it signals the semaphore, which will increment the semaphore counter.

If a semaphore is initialized with a 1 - a binary semaphore - it will behave like a mutex only allowing one thread at a time to pass.

### POSIX Semaphores

The POSIX semaphore API defines one type, `sem_t`, as well as three operations that manipulate that type. These operations:

1. Create the semaphore.
2. Wait on the semaphore.
3. Unlock the semaphore.

<img src="posix_semaphore.png">

### Reader/Writer Locks

When specifying synchronization requirements, it is sometimes useful to distinguish among different types of resources access.

For instance, we commonly want to distinguish **"read" access** - those that do not modify a shared resource - from **"write" access** - those that do modify a shared resource. A read access can be shared concurrently, a write requires exclusive access.

This is a common scenario, so many OSs and language runtime support a construct known as reader/writer locks.

A reader/writer lock behaves similarly to a mutex, but the developer only has to specify the type of access they wish to perform (read or write), and the lock takes care of access control for them.

### Using Reader/Writer Locks

<img src="rwlockslinux.png">

The datatype defined in Linux is `rwlock_t` and we can perform operations on the data type. We can lock/unlock for reads and writes.

Some implementations of this API differ across systems, these might be referred to as shared/exclusive locks in other APIs. It may also differ in how recursively-obtained read locks are unlocked. When unlocking a recursive read lock, some implementation unlock all of the recursive locks while other only unlock the most recently acquired lock. 

### Monitors

All of the constructs that we have seen so far require developers to explicitly invoke the lock/unlock and wait/signal operations. This introduces the opportunity to make errors.

**Monitors** are a higher level synchronization construct that **allow us to avoid manually invoking these synchronization operations**. A monitor will specify a shared resource, the entry procedures for accessing that resource, and potential condition variables used to wake up different types of waiting threads.

When invoking the entry procedure, all of the necessary locking and checking will take place by the monitor behind the scenes. When a thread is done with a shared resource and is exiting the procedure, all of the unlocking and potential signaling is again performed by the monitor behind the scenes.

### More Synchronization Constructs

**Serializers** make it easier to define priorities and also to hide the need for explicit signaling and explicit use of condition variables from the programmer.

**Path Expressions** require the programmer to specify a regular expression that captures the correct synchronization behavior. As opposed to using locks, the programmer would specify something like "Many Reads, Single Write", and the runtime will make sure that the operations satisfy the regular expressions.

**Barriers** are like reverse semaphores. While a semaphore allows n threads to process before it blocks, a barrier blocks until n threads arrive a barrier point. Similarly, rendezvous points also wait for multiple threads to arrive at a particular point of execution.

To further boost scalability and efficiency metrics, there are efforts to achieve concurrency without explicitly locking and waiting. These **wait-free synchronization** constructs are optimistic in the sense that they bet on the fact that there wont be any concurrent writes and that it is safe to allow reads to proceed concurrently. An example of this is **read-copy-update(RCU) lock** that is part of the Linux kernel.

All of these methods require support from the underlying hardware to atomically make updates to a memory location. This is the only way that they can guarantee that a lock is properly acquired and that protected state changes are performed in a safe, race-free way.

### The Performance of Spin Lock Alternatives for Shared Memory Multiprocessors

Checking and setting a lock value needs to happen **atomically** so that we can guarantee that only one thread at a time can successfully obtain the lock.

Let's look at the implementation of our spin lock:

```c
spinlock_lock(lock) {
    while(lock == busy) {
        //spin
    }
    lock = busy
}
```

The problem with this implementation is that it takes multiple cycles to perform the check and the set and during these cycles threads can be interleaved in arbitrary ways.

To make this work we need **hardware-supported atomic instructions**.

### Atomic Instructions

Each type of hardware will support a number of atomic instructions. Some examples include:
* test_and_set
* read_and_increment
* compare_and_swap

Each of these operations performs some multi-step, multi-cycle operation. Because they are atomic instructions however, t**he hardware makes some guarantees**. 

First, these operations will happen **atomically**, that is, **completely or not at all**. In addition, the **hardware guarantees mutual exclusion** -- threads on multiple cores cannot perform the atomic operations in parallel. **All concurrent attempts to execute an instruction will be queued and performed serially**.

Here is a new implementation for a spinlock locking operation which uses an atomic instruction.

```c
spinlock_lock(lock) {
    while(test_and_set(lock) == busy);
}
```

When the first thread arrives, `test_and_set` looks at the value that `lock` points to. This value will initially be set to zero. `test_and_set` atomically sets the value to one, and returns to zero. `busy` is equal to one. Thus, the first thread can proceed. When subsequent threads arrive, `test_and_set` will look at the value, these threads will spin.

Which specific atomic instructions are available on a given platform varies from hardware to hardware. There may be differences in the efficiencies with which different atomic operations execute on different architectures. For this reason, **software built with specific atomic instructions needs to be ported**; that is, we need to make sure the implementation uses one of the atomic instructions available on the target platform

### Shared Memory Multiprocessors

<img src="shared_memory_multip.png">

In the interconnect-based configuration (on the right), multiple memory references can be in-flight at a given moment, one to each connected memory module. In a bus-based configuration, the shared bas can only support one memory-reference at a time.

Shared memory multiprocessors are also referred to as symmetric multiprocessors(SMP).

Each of the CPUs in a SMP platform will have a cache.

<img src="smp_cache.png">

In general, **access to the cache data is faster than access to data in main memory**. Put another way, cache hides memory latency. **This latency is even more pronounced in shared memory systems because there may be contention among different CPUs for the shared memory components.** This contention will cause memory accesses to be delayed, making cached lookups appear much faster.

When CPUs perform a write, many things can happen.

For example, we may not even allow a CPU to write to its cache. A write will go directly to main memory, and any cache references will be invalidated. This is called **no-write**.

As one alternative, the CPU write may be applied both to the cache and in memory. This is called **write-through**.

A final alternative is to apply the write immediately to the cache, and perform the write in main memory at some later point in time, perhaps when the cache entry is evicted. This is called **write-back**.

### Cache Coherence

So what happens when multiple CPUs reference the same data?