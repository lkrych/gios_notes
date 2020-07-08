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
