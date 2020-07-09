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

<img src="cache_coherence.png"/>

On some architectures, this problem needs to be dealt with completely in software. Otherwise, the caches will be incoherent. For instance, if one CPU writes a new version of X to its cache, the hardware will not update the value across the other CPU caches. These architectures are called **non-cache-coherent (NCC) architectures**.

On **cache-coherent (CC) architectures**, the hardware will take care of all the necessary steps to ensure that the caches are coherent. If one CPU writes a new version of X to its cache, the hardware will step in and ensure that the value is updated across CPU caches.

There are two basic strategies by which hardware can achieve cache coherence.

Hardware using the **write-invalidate (WI)** strategy will **invalidate all cache entries of X once one CPU updates its copy of X**. Future references to invalidate cache entries will have to pass through main memory before being re-cached.

Hardware using the **write-update (WU)** strategy will **ensure that all cache entries of X are updated once one CPU updates its copy of X**. Subsequent accesses of X by any of the CPUs will continue to be served by the cache.

With **write-invalidate, we pose lower bandwidth requirements on the shared interconnect** in the system. Why? We don't have to send the value of X, but rather just its address so it can be invalidated. 

If X isn't needed on any other CPUs anytime soon, it is possible to amortize the cost of coherence traffic over multiple changes. X can change multiple times on one CPU before its value is needed on another CPU.

For **write-update architectures, the benefit is that X will be available immediately on the other CPUs that need to access it**. We will not have to pay the cost of a main memory access. Programs that will need to access the new value of X immediately on another CPU will benefit from the design.

The user of write-update or write-invalidate is determined by the hardware supporting your platform. The software engineer has no say :p.

### Cache Coherence and Atomics

Let's consider a scenario where we have two CPUs. Each CPU needs to perform an atomic operation concerning X, and body CPUs have a reference to X present in their cache.

When an atomic instruction is performed against the cached value of X on one CPU, it is really challenging to know whether or not an atomic instruction will be attempted on the value of X on another CPU. We obviously do not want to have incoherent cache references because that would affect the correctness of our applications.

To address this, **atomic operations always bypass the caches and are issued directly to the memory location** where the particular target is stored. 

By forcing atomic operations to go directly to the memory controller, we enforce a central entry point where all the references can be ordered and synchronized in a unique manner. None of the race conditions that could have occurred had we let atomic instructions update the cache can occur now. 

That being said, **atomic instructions take much longer than other types of instructions since they can never leverage the cache**. 

In addition, in order to guarantee atomic behavior, we have to generate the coherence traffic to either update or invalidate all of the cached references to X, regardless of whether it has changed. 

In summary, atomic instructions on SMP systems are more expensive than on single CPU system because of bus or I/C contention. In addition, atomics in general are more expensive because they bypass the cache and always trigger coherence traffic.

## Spinlock Performance Metrics

We want spinlocks to have low latency, we can **define latency as the time it takes a thread to acquire a free lock**. Ideally, we want the thread to be able to acquire a free lock immediately with a single atomic instruction.

In addition, we want the spinlock to have low delay/waiting time. We can define delay as **the amount of time required for a thread to stop spinning and acquire a lock that has been freed**. 

Lastly, we would like a design that reduces contention on the shared bus or interconnect. Specifically, contention due to the actual atomic memory references as well as the subsequent coherence traffic. 

## Test and Set Spinlock

<img src="spinlock_api.png">

The `test_and_set` instruction is a very common atomic that most hardware platforms support. 

From a latency perspective, this spinlock is as good as it gets. We only execute on atomic operation and there is no way we can do better than this.

Regarding delay, this implementation performs well. We are spinning on the atomic. As soon as the lock becomes free, th every next call to test_and_set which is the next instruction, will immediately detect the free lock and acquire it.

From a contention perspective this lock does not perform well. As long as the threads are spinning on `test_and_set`, the processor has to continuously go to main memory on each instruction. This will delay all other CPUs that need to access memory.

The real problem with this implementation is that we are continuously spinning on the atomic instruction. Regardless of cache coherence, we are forced to constantly waste time going to memory every time we execute the atomic instruction.

## Test and Test and Set Spinlock

The problem with the previous implementation is that all of the CPUs are spinning on the atomic operation. **Let's try to separate the test part - checking the value of the lock - from the atomic**. 

The intuition is that CPUs can potentially test their cached copy of the lock and only execute the atomic if it detects that its cached copy has changed. 

Here is the resulting lock operation:

<img src="test_and_test_and_set.png">

First we check if the lock is busy. Importantly, this check is performed against the cached value. As long as the lock is busy, we will stay in the while loop and we won't need to evaluate the second part of the predicate. Only when the lock becomes free - when `lock == busy` evaluates to false, do we actually execute the atomic.

This spinlock is referred to as the **test-and-test-and-set spinlock**. It is also called a spin-on-read or spin-on-cached-value spinlock.

From a latency and delay standpoint it is okay. It is slightly worse than the test-and-set spinlock because we have to perform an extra step.

From a contention standpoint our performance varies.

If we don't have a cache coherent architecture, there is no difference in contention. Every single reference will go to memory.

If we have a cache coherence with the write-update strategy the contention improves. The only problem with write-update is that all the processors will see the values of the lock as free and thus all of them will try to unlock the lock at once.

If we have cache coherence with the write-invalidate strategy, our contention is terrible. Every single attempt to acquire the lock will generate contention for the memory module and will also create invalidation traffic.

One outcome of executing an atomic instruction is that e will trigger the cache coherence strategy regardless of whether or not the value protected by the atomic changes.

If we have a write-update situation, that coherence traffic will update the value of the other caches with the new value of the lock. If the lock was busy before the write-update event, and the lock remains busy after the write-update event, there is no problem. The CPU can keep spinning on the cached copy.

However, write-invalidate will invalidate the cached copy. Even if the value hasn't changed, the invalidation will force the CPU to go to main memory to execute the atomic. What this means is that any time another CPU executes an atomic, all of the other CPUs will be invalidated and will have to go to memory.

## Spinlock "Delay" Alternatives

<img src="spinlock_delay.png">

This implementation introduces a delay every time the thread notices that the lock is free.

The rationale behind this is to **prevent every thread from executing the atomic instruction at the same time**. 

As a result, the contention in the system will be improved. When the delay expires, the delayed thread will re-check the value of the lock, and it's possible that another thread executed the atomic and the delayed thread will see the lock is busy. If the lock is free, the delayed thread will execute the atomic.

There will be fewer cases in which threads see a busy lock as free and try to execute an atomic that will not be successful.

From a latency perspective, this solution is okay. We still have to perform a memory reference to bring the lock into the cache, and then another to perform the atomic. However, this isn't too different from the test-and-test-and-set.

From a delay perspective, the performance has decreased. Once a thread sees that a lock is free, we have to delay for some amount of time. If there is no contention for the lock, that delay is wasted time.

An alternative delay-based lock introduces a delay after each memory reference.

The main benefit of this is that it works on NCC architectures. Since a thread has to go to main memory on every reference on NCC architectures, introducing an artificial delay greatly increases the number of references the thread has to perform while spinning.

Unfortunately, this alternative will hurt the delay much more, because the thread will delay every time the lock is referenced, not just when the thread detects the lock has become free.

## Picking a Delay

One strategy is to pick a **static delay**, which is based on some fixed information. One benefit of this approach is the simplicity and the fact that under high load, static delays **will likely spread out all the atomic references such that there is little contention**.

The problem with it is that **it will create unnecessary load under low contention**. 

To avoid the issues of excessive delays without contention, **dynamic delays** can be used. With dynamic delays, each thread will take a random delay value form a range of possible values that increase with the perceived contention in the system.

Under high load, both dynamic and static delays will be sufficiently enough to reduce contention within the system.

How is contention evaluated? A good metric is to track the number of failed `test_and_set` operations. The more these operations fail, the more likely it is that there is a higher degree of contention.

If we delay after each lock reference, however, our delay grows not only as a function of contention but also as a function of the length of the critical section. If a thread is executing a large critical section, all spinning threads will be increasing their delays even though the contention in the system hasn't actually increased.

## Queueing Lock

The reason for introducing a delay is to guard against the case where every thread tries to acquire a lock once it is freed.

Alternatively, if we can prevent every thread from seeing that the lock has been freed at the same time, we can indirectly prevent the case of all threads rushing to acquire the lock simultaneously.

The **lock that controls which threads see that the lock is free at which time is the queuing lock**.

<img src="queueing_lock.png">

THe queueing lock uses an array of flags with up to `n` elements, where `n` is the number of threads in the system. Each element in the array will have one of two values: either `has_lock` or `must_wait`. In addition, one pointer will indicate the current lock holder (will have `has_lock`), and another pointer will reference the last element on the queue.

When a new thread arrives at the lock, it will receive a ticket, which corresponds to the current position of the thread in the lock. This will be done gby adding it after the existing last element in the queue.

Since multiple threads may enter the lock at the same time, it's important to increment the queuelast pointer atomically. This requires some support for the `read_and_increment` atomic.

For each thread arriving at the lock, the assigned element of the flags array at the ticket index acts like a private lock. As long as this value is `must_wait`, the thread will have to spin.  When the value of the element becomes `has_lcok`, this will signify to the threads that the lock is free and they can attempt to enter their critical section.

When a thread completes a critical section and needs to release the lock, it needs to signal the next thread. Thus `queue[ticket + 1] = has_lock`.

This strategy has two drawbacks. First, it requires support for the `read_and_increment` atomic, which is less common than `test_and_set`.

In addition, this lock requires much more space than the other locks. All other locks require a single memory location to track the value of the lock. This lock requires `n` such locations, one for each thread.

## Queueing Lock Implementation

<img src="queueing_lock_implementation.png">

