# Thread Design Considerations

Threads can be implemented at the kernel-level, at the user-level or both. In this lecture, we will take a look at the data structures and mechanisms that are necessary to support threads.

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

