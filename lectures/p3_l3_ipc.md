# Inter process Communication (IPC)

## Introduction

**Inter process communication (IPC)** refers to **a set of mechanisms that the OS must support in order to permit multiple processes to interact with each other**. This includes mechanisms related to **synchronization, coordination and communication**.

IPC mechanisms are broadly categorized as either **message-based** or **memory-based**. 

Message-based IPC mechanisms include **sockets, pipes, and message queues**.

Memory-based IPC mechanisms utilize **shared memory**. This may be in the form of unstructured shared physical memory pages or memory mapped files which can be accessed by multiple processes.

Another mechanism that provides higher level semantics with regards to IPC is r**emote procedure calls (RPC)**. RPC is more than just a channel for passing information between processes. This method provides some additional information as to the protocols that will be used, which includes information about data format and the data exchange procedure.

Finally, the need for communication and coordination illustrates the necessity of synchronization primitives.

## Message-based IPC

In message-based IPC, processes create messages