# Distributed Shared Memory

## Table of Contents

## Introduction

Distributed file systems are an example of a distributed service in which the state - the files - are stored on some set of server nodes and are then accessed by some set of client nodes. The servers own and manage the state and provide a service - the file access operations - which are requested by the clients and operate on the state.

The primary focus of the previous lesson (DFS) was the caching mechanisms, which was a useful discussion because it helped us understand how to improve performance for clients.

We didn't talk about how the multiple file servers share and coordinate the application state amongst themselves. We also didn't talk about the scenario in which all of the nodes in the system were both servers and clients (peers).

## Peer Distributed Applications

Often it is hard to distinguish between servers and clients in a system. The state of an application can be shared across all nodes, with different nodes both requesting and serving state at different points in time. 

Every node in the system owns some portion of the state. There is some state that is locally stored on a particular node. The overall application state is the union of all the pieces of state that are present on any one node.

All of the nodes are **peers**, they all require accesses to the state located elsewhere in the system and provide accesses to their local state.

This is slightly different from **peer-to-peer** applications: it's likely that there will still be some nodes that provide some overall configuration or management of the entire system. In a peer-to-peer system, these tasks would still be performed cooperatively by all peers.

## Distributed Shared Memory (DSM)

Distributed shared memory is a service that manages memory across multiple nodes so that applications will have the illusion that they are running on a single shared-memory machine.

Each node in the system owns some portion of the physical memory, and provides the operations - reads and writes - on that memory. The reads and write may originate from any of the nodes in the system.

For instance, when nodes read and write shared memory locations, these reads and writes must be ordered and observed by the remaining nodes in the system in consistent ways.

**Distributed shared memory mechanisms are important to study because they permit scaling beyond the limitations of how much memory we can include in a single machine.**

Single machines with large amounts of memory can cost hundreds of thousands of dollars per machine. In order to scale up memory affordably, it's imperative to understand DSM concepts and semantics so that many cheap machines can be connected to give the illusion of a high memory "single" machine.

Naturally the overall memory access will be slower in a DSM environment as a result of network costs, but this is often affordable and optimizable. It is possible to hide these network delays with certain design choices.

Distributed shared memory is becoming more relevant today because commodity interconnect technologies offer really low latencies between nodes in a system via **Remote Direct Memory Access (RDMA)** intefaces.

## Hardware vs Software DSM

DSM can be supported in hardware or software. 

Hardware-supported DSM relies on some physical interconnect. The OS running on each physical node is under the impression that is has access to much larger physical memory, and is allowed to establish virtual to physical mappings that point to physical addresses on other nodes. 

Memory accesses that reference remote emory locations are passed to the network interconnect card, which translates remote memory accesses into interconnect messages that are then passed to the correct node.

<img src="hardware_dsm.png">

The NICs are involved in all aspects of the memory management, access and consistency  and even support some atomics.

While it is very convenient to rely on hardware to do everything, this type of hardware is typically very expensive and as a result is reserved for very high-end machines.

Instead, DSM is often realized in software. The software will have to"
1. Detect local vs remote memory access
2. Create and send messages to the appropriate node
3. Accept messages from other nodes and perform the encoded memory operations
4. Be involved in memory sharing and consistency support

This can be done at the level of the OS or with a programming language runtime.

## DSM Design: Sharing Granularity

In SMP systems, the granularity of sharing is the cache line. The hardware tracks concurrent memory accesses at the granularity of a single cache line, and triggers the necessary coherence mechanisms if it detects that a shared cache line has been modified.

For DSM, generating coherence traffic at the granularity of the cache line will be too expensive, given that this traffic has to travel over the network and incur network cost.

Instead we look at larger granularities for sharing:
1. Variable granularity
2. Page granularity
3. Object granularity

Variables are a meaningful unit from a programmer's perspective, so maybe DSM solutions can operate at the level of variable granularity. Unfortunately, this is likely still too fine-grained. Variables are still too small and occur too frequently in an application, making it likely that we will still incur very high coherence traffic overheads.

Using something larger, like a page or object makes more sense. If the DSM system is to be integrated at the operating system level, page-based granularity makes sense since the OS doesn't understand objects. Pages are larger than variables, so it's possible to amortize the cost of remote access across these larger granularities.

With some help from a compiler, application level objects can be laid out on different pages, and then we can rely on the page-based operating system level mechanism.

Alternatively, we can have a DSM that is supported by the programming language and the runtime, where the runtime understands which objects are local and which are remote. For remote objects, the runtime will generate all of the necessary operations to communicate with the remote nodes.

In the case of object granularity, the OS doesn't need to know anything about DSM, and therefore doesn't have to be modified. That being said, the language level solution is not generalizable outside of programming languages that have this DSM support.

Once we start increasing the granularity of sharing, we have to beware of **false sharing**.

Consider a page that internally has two variables, x and y. A process on one node is exclusively accessing and modifying x. Similarly, a process on another node is exclusively accessing and modifying y. When x and y are on the same page, the DSM system will interpret the two write accesses as an indication of concurrent access to a shared page. This will trigger coherence mechanisms which, while logically viable, are functionally superfluous.

## DSM Design: Access Algorithm

Another important design point when looking at DSM solutions is to understand what types of memory accesses the higher-level applications expect to perform.

The simplest kind of application is the **single reader/single writer**. For these kinds of application, the main role of the DSM layer is to provide the application with the ability to access additional, remote memory.In this case, there are not any consistency, or sharing-related challenges.

The more complex case is where the application needs to support **multiple readers/single writer** or **multiple readers/multiple writers**. In these cases, it's not just about how to read or write the correct physical location of memory.

With multiple writers and readers, it's important that the reads return the most recent value at a memory location. It is also important that all of the writes that are performed are correctly ordered. This is necessary so as to present a consistent view of the distributed state to all the nodes in the system.

## DSM Design: Migration vs Replication

For a DSM solution to be useful, it must provide good performance to applications. Since the core service provided by DSM solutions is access, the core performance metric to analyze is **access latency**.

Clearly, accessing local memory is faster than remote memory, so it is important to consider how to maximize the proportion of local memory accesses. 

One way to maximize the proportion of local memory accesses is through **migration**. Whenever a process on another node needs to access remote state, we copy the state over to that node.

Migration makes sense in the single reader/single writer case, since only one node at a time will be accessing the state. However moving data across nodes does incur overhead.

We should be careful about this solution, even in the single reader/writer case. Copying over a large chunk of state just for a single read or write is a cost that cannot be amortized.

With multiple readers and multiple writers, migrating the state all over the place doesn't make any sense since the state needs to be accessed by multiple nodes at the same time.

In this case, we use **replication** to copy the state on multiple nodes. While this solution is more general, it requires consistency management. We have to make sure that **writes are correctly ordered and that the most recent copies of the state are propagated across nodes** for up-to-date reads.

One way to control the consistency overhead is to limit the number of replicas that can exist in the system at any time, since the overhead is proportional to the number of replicas.

## DSM Design: Consistency Management

Once we permit multiple copies of the same data page to be stored in multiple locations, it becomes important to think about maintaining consistency. 

DSM systems are expected to behave similar to SMP systems. Remember that SMP manages consistency via two strategies:

1. **write-invalidate**
2. **write-update**

In write-invalidate, one processor's cached value of a variable is invalidated when another processor updates that variable. In write-update, the processor's cached variable is updated to reflect the new value.

These coherence operations are triggered by the shared memory support in the hardware on every single write operation. The overhead of supporting this strategy in DSM systems are too high, once network costs are considered.

One option is to push invalidation messages when data is written to. This is similar to the server-based approach that we talked about in the DFS lecture.

This approach is proactive, eager and pessimistic.

Another option is for one node to poll periodically for modification information regarding certain memory regions in the system. This can be done periodically or on-demand.

This approach is referred to as reactive, lazy and optimistic.

## DSM Architecture

Let's look at how a DSM system can be designed.

This system consists of a number of nodes, each with their own physical memory. Each node may contribute only a portion of their physical memory towards the DSM system. The rest of the memory can be used for caching, replication, or just local memory.

The pool of memory pages that each of these nodes contributes forms the global shared memory that is available for applications running in this system.

Every address in this system will be uniquely identified by a combination of the node identifier and the page frame number.

The node where a page is located is typically referred to as the **home node** for that page. 

Let's assume that the system needs to support the case of multiple readers and multiple writers.

In order for this system to be performant and achieve low latency, the DSM layer needs to incorporate caching. Pages will be cached on the nodes where they are accessed.

The home node for a page will be responsible for driving all of the coherence operations related to the page. As a result, all the nodes in the system are responsible for some part of the management of the overall distributed memory.

The home node will have to keep track of the state for its memory pages: the pages accessed, the modifications, whether caching is enabled/disabled, and the lock status.

All of this information is used for enforcing the shared semantics that a given DSM will implement.

When a particular page is accessed repeatedly, and sometimes exclusively, by a node that is not its home node, it becomes too expensive to continually contact the home node to perform state updates.

One mechanism that is useful in DSM systems is to separate the distinction of home node from an **owner node**. The owner is the node that **currently owns the page, and can control all of the state modifications and drive the coherence mechanisms**.

**The owner node may be different from the home node**, and the owner node may change, as a page is accessed by different nodes through the lifetime of an application's execution.

The role of the home node then becomes to keep track of who is the current owner of a given page that maps to its physical memory.

In addition to creating page copies via caching, page replicas can be explicitly created for load balancing, performance or reliability reasons. In datacenter environments, it makes sense to triplicate shared state: on the main machine, on a nearby machine in the same building, and on a remote machine in another datacenter. The consistency of these nodes is either managed by the home node or some manager node.

## Indexing Distributed State

When creating distributed memory systems, it's important to understand how we can determine where a particular page is physically located within the system.

In order to do this, the DSM component has to maintain some metadata. **Each page has an address, which is some combination of the node ID and the page frame number** on that node.

**The address directly identifies the manager/home node** that knows everything about that particular page. **The manager information is available on every node in the system**.

This can be captured via a **global map** - which takes in a page id and returns a manager id - and is replicated on each node.

Each manager will maintain the per-page metadata that is necessary to perform the specific access to that page or force its consistency.

This means that **the metadata for local pages is partitioned across the system**, while the **manager information for every page is replicated**.

Certain bits from the address are used to identify the manager. There will be a fixed manager identified from that mapping function. If we want some additional flexibility we can take the object id and use it as an index into a global mapping table that will give a manager node.

In this approach, we can change the manager node by updating the mapping table. We don't need to change the object identifier.

<img src="dsm_metadata.png">

## Implementing DSM