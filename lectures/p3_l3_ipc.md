# Inter process Communication (IPC)

## Introduction

**Inter process communication (IPC)** refers to **a set of mechanisms that the OS must support in order to permit multiple processes to interact with each other**. This includes mechanisms related to **synchronization, coordination and communication**.

IPC mechanisms are broadly categorized as either **message-based** or **memory-based**. 

Message-based IPC mechanisms include **sockets, pipes, and message queues**.

Memory-based IPC mechanisms utilize **shared memory**. This may be in the form of unstructured shared physical memory pages or memory mapped files which can be accessed by multiple processes.

Another mechanism that provides higher level semantics with regards to IPC is **remote procedure calls (RPC)**. RPC is more than just a channel for passing information between processes. This method provides some additional information as to the protocols that will be used, which includes information about data format and the data exchange procedure.

Finally, the need for communication and coordination illustrates the necessity of synchronization primitives.

## Message-based IPC

<img src="ipc_summary.png">

In message-based IPC, **processes create messages and then send and receive them**. The OS is responsible for creating and maintaining the channel that is used to send these messages.

The OS provides an interface to the processes so that they can send messages via this channel. The processes send/write messages to a port, and then recv/read messages from a port. The channel is responsible for passing the message from one port to the other.

Since the OS is required to establish communication and perform each IPC operation, **every send and receive call requires a system call and a data copy**. When we send, the data must be copied from the process address space into the communication chanel. When we receive, the data must be copied from the communication channel into the process address space.

This means that a request-response interaction between two processes **requires a total of four user/kernel crossings and four data copying operations**. These **overheads** are one of the negatives of message-based IPC.

One of the positives of this approach is the **relative simplicity**. The OS kernel will take care of all the operations regarding channel management and synchronization.

### Forms of Message Passing - Pipes

<img src="pipes.png">

Pipes are **characterized by two endpoints**, so **only two processes can communicate via a pipe**. There is no notion of a message with pipes, instead **there is a stream of bytes pushed into the pipe** from one process and read from the pipe by the other process.

One popular use of pipes is to connect the output from one process to the input of another.

```bash
cat /some/large/file | grep "search for something specific"
```

### Forms of Message Passing - Message Queues

<img src="message_queues.png">

Message queues understand the notion of messages that they can deliver. **A sending process must submit a properly formatted message to the channel, and then the channel can deliver this message to the receiving process**.

The OS level functionality regarding message queues includes mechanisms for message priority, custom message scheduling and more.

The use of message queues is supported via different APIs in Unix-based systems. Two common APIs are SysV and POSIX.

### Forms of Message Passing - Sockets

<img src="sockets.png">

With sockets, processes send and receive messages through the socket interface. The socket API supports `send` and `recv` operations that allow processes to send message buffers in and out of the kernel-level communication barrier.

The `socket` call itself **creates a kernel-level socket buffer**. In addition, it will associate any kernel level processing that needs to be associated with the socket along with the actual message movement.

For instance, the socket may be a TCP/IP socket, which means that the entire TCP/IP protocol stack is associated with the socket buffer.

Socket-based communication can happen between processes on two different machines. If the processes are on two different machines, then the communication buffer is really between the process and the network device that will actually send the data.

## Shared Memory IPC

<img src="shared_memory_ipc.png">

In shared memory IPC, processes **read and write into a shared memory region**. The **operating system is involved in establishing the shared memory channel** between the processes.

**The OS will map certain physical pages in memory into the virtual address space of both processes**. The virtual addresses in each process pointing to a shared physical location do not have to be the same (they are virtual after all). In addition, the shared physical memory section does not need to be contiguous.

The **big benefit of this approach is that once the physical memory is mapped into both address spaces, the operating system is out of the way. System calls are only used for the setup phase**.

Data copies are reduced, but not necessarily avoided. For data to be available to both processes, it needs to explicitly be allocated from the virtual addresses that belong to the shared memory region. If that is not the case, the data within the same address space needs to be copied in and out of the shared memory region.

Since the shared memory area can be concurrently accessed by both processes, this means the **processes must explicitly synchronize their shared memory operations**. In addition, it is **now the developer's responsibility to handle any protocol-related implementations, which adds to the complexity** of the application.

Unix-based systems support two popular shared memory APIs: SysV and POSIX. In addition, shared memory IPC can be established between processes by using a **memory-mapped file**.

### Copy vs Map

The **goal of both message-based and memory-based IPC is to transfer data from the address space of one process to the address space of another process**.

In message-based IPC, this requires that the CPU is involved in copying the data, and CPU cycles are spent every time data is copied to/from ports.

In memory-based IPC, CPU cycles are spent to map physical memory into the address spaces of the processes. The CPU may also be involved in copying the data into the shared address space, but not that here is no user/kernel switching in this case.

The memory-mapping operation is costly, but it is a one-time cost, and can pay off even if IPC is performed once. In particular, when we need to move large amounts of data from one address space to another address space, the time it takes to copy via message-based IPC greatly exceeds the setup cost of memory-mapping.

## SysV Shared Memory