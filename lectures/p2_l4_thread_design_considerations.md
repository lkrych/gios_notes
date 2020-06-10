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