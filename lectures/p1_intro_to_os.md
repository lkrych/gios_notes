# An Introduction to Operating Systems

## Table of Contents
* [Introduction](#introduction)
* [What does an OS do?](#what-does-an-operating-system-do?)
* [Elements of an OS](#elements-of-an-os)
* [OS Design Principles](#os-design-principles)
* [User/kernel boundary](#user-kernel-protection-boundary)
* [System Calls](#system-calls)
* [OS Services](#os-services)
* [OS Design: Monolith vs Modular](#monolithic-os-vs-modular-os)
* [Linux and Mac OS Architecture]

## Introduction

An **operating system** is a special piece of software that **abstracts and arbitrates** the underlying hardware system. In this context, 
1. abstracts means to simplify what the hardware looks like, and 
2. arbitrate means to manage. 

<img src=p1_resources/1.png>

**Computing systems are made up of a number of hardware components**, this includes processors (CPUs), memory, graphics cards, storage devices and network interconnects. 

<img src=p1_resources/2.png>

## What does an Operating System do?

An operating system is the layer of **systems software** that sits between the complex hardware and the applications that use the hardware. 

Operating systems
1. **Hide hardware complexity**
2. **Manages resources/hardware** (manages memory and schedules processes)
3. **Provides isolation and protection** (prevents applications from interfering with each other)

<img src=p1_resources/3.png>

### Operating Systems Examples

Certain OSes target desktops, ultra-highend machines, servers, and embedded devices.

<img src=p1_resources/4.png>

Why are there different operating systems? Because In each of these systems, there are unique design decisions made.

## Elements of an OS

To achieve its goal an OS supports high level **abstractions and mechanisms** that operate on top of the abstractions. Lastly, the OS uses policies to dictate the behavior of certain mechanisms. 

Let's look at a memory management example. The abstraction is a memory page.

<img src=p1_resources/5.png>

A common policy that operating systems use to identify what is kept in memory is the least-recently used policy that evicts the oldest pages.

## OS Design Principles

1. **Separation of mechanism and policy** -  we want to incorporate flexible mechanisms that can support a range of policies. Ex: If an OS has the ability to check how recently a memory page has been used, a variety of different algorithms (policies) can be created.
2. **Optimize for the common case** -  This means we need to know where the OS will be used, what the user will want and what the workload requirements are.

## User/Kernel Protection Boundary

Computer platforms distinguish between two modes, privileged or kernel mode and unprivileged or user mode. 

<img src=p1_resources/6.png>

Because on OS must have direct hardware access, it must operate in kernel mode. The applications operate in user mode. Crossing over to talk to hardware, has to be intermediated by the OS. So how does this work? Typically the CPU has a **privilege bit** that when set, causes any instruction to be run in privileged mode.

When in a user-mode instruction attempts to execute a privileged instruction on hardware, it causes a **trap instruction**. At this point, the OS can see if the trap instruction should proceed.

In addition to the trap method, the OS exports an interface called **system calls**, that allow applications to interact with privileged instructions.

Lastly, Operating Systems support **signals** that are a mechanism to pass notifications into processes.

## System Calls

<img src=p1_resources/7.png>

<img src=p1_resources/8.png>

In **synchronous** mode, the process waits until the syscall completes. **Asynchronous** mode does not block. 

A user/kernel transition involves a decent number of instructions. Another problem with these transitions is they **effect the hardware cache usage**. Why? Context switches will swap the data/addresses currently in the cache. Leaving the cache cold for the context switch back to the application.

## OS Services

The OS provides applications with access to the underlying hardware. It does so by exporting services. 

At the most basic level, these services are directly related to the hardware: **scheduler** (CPU), the **memory manager** (main memory).

Additionally it also exports higher level abstractions like the **file system**.  

Here is a comparison of System Calls between Windows and Unix. 

<img src=p1_resources/9.png>

On a 64-bit Linux-based OS, which system call is used to:
1. **kill** - Send a signal to a process
2. **setgid** - Set the group identity of a process
3. **mount** - Mount a file system
4. **sysctl** - Read/write system parameters

## Monolithic OS vs Modular OS

Operating systems can be organized in different ways. Historically, OSes had a **monolithic design** which means that any interface an application might need, or any type of hardware is already part of the OS. 

The benefit of this approach is that everything is included. This means that everything is compiled together and it's possible to get some compile-time optimizaitons. 

The downsides are that there is a huge memory footprint, and there is too much code to maintain.

A more common approach is the **modular approach**, like the Linux system. This system has some built-ins but anything you need has to be added as a module. This is possible because the OS specifies interfaces that allows for dynamic loading of the module. 

This is useful because OSes can be specialized for different things. An OS that needs to work with a database might want to support a memory manager that has been optimized for random I/O as opposed to sequential I/O.

<img src=p1_resources/10.png>

The **benefit of this approach is that it is easier to maintain, has a smaller memory footprint, and less resource needs**.

The downsides are that all the **modularity can impact performance**. This is because the interface between the OS and the module can have overhead. Maintenance can still be an issue. 

<img src=p1_resources/11.png>

Another example of OS design is the Microkernel. **Microkernels only require the most basic primitives at the OS level**.  

For instance, at the OS level they can support Address spaces, and threads. Everything else will run outside of the OS kernel at user-level.  This requires a lot of interprocess intercommunications. 

The **benefits of the microkernel are its size and verifiability**. 

The downsides are that they are **not portable, software development is difficult, and there is a performance hit from all of the user/kernel transitions**.

## Linux and Mac OS Architectures

<img src=p1_resources/12.png>

The kernel is itself modular. Each component can be replaced.

<img src=p1_resources/13.png>

<img src=p1_resources/14.png>

At the core of the Mac OS system is the Mach microkernel which implements key primitives. The BSD component implements linux interoperability. All application environments sit above this layer. 