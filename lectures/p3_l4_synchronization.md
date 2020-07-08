# Synchronization Concepts

## Table of Contents

* [Introduction](#introduction)
* [Spinlocks](#spinlocks)
* [Semaphores](#semaphores)

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
