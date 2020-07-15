# Distributed File Systems

## Table of Contents

## Introduction

Modern OS's **export a high-level filesystem interface** to abstract all of the **different types of storage devices** present on a machine and **unify them under a common API**.

The OS can hide the fact that there isn't even local physical storage on the machine, rather the files are maintained on a remote filesystem that is being accessed over the network.

Environments that involve **multiple machines for delivery of the filesystem** service are called **distributed filesystems**(DFS).

## DFS Models

A simple DFS model involves clients being served files from a server running on a different machine.

Often the server is not running on a single machine, but rather is distributed across multiple machines. Files may be **replicated** across every machine or **partitioned** amongst machines.

In **replicated systems**, all of the files are replicated and **available on every server machine**. If one machine fails, other machines can continue to serve requests. Replicated systems are **fault tolerant**. In addition, requests entering a replicated system can be serviced by any of the replicas. Replicated systems are **highly available**.

In **partitioned systems**, each server **holds only some subset of the total files**. If you need to support more files, you can just partition across more machines. In the replicated model, you will need to upgrade your hardware. This make partitioned systems more **scalable**. 

It's common to see a DFS that utilizes both partitioning and replication; for example, partitioning all the files and then replicating each partition.

Finally, files can be stored and served from all machines. This blurs the line between server and clients, because all the nodes in the system are **peers**. Every node is responsible for maintaining the files and providing the filesystem service. Each peer will take some portion of the load by servicing some of the requests, often those for files local to that peer. 

## Remote File Service: Extremes

At one extreme, we have the u**pload/download model**. When a client wants to access a file, it downloads the entire file, performs the modifications and then uploads the entire file back to the server. 

<img src="upload_download.png">

The **benefit of this model** is that all of the **modifications can be done locally**, which means they can be done **quickly**, without incurring any network cost.

One downside of this model is that the client has to download the entire file, even for small modifications. A second downside of this model is that it takes away file access control from the server. Once the server gives the file to the client, it has no idea what the client is doing with the file or when it will give it back.

At the other extreme, we have the true **remote file access**. In this model, the file remains on the server and every single operation has to pass through the server. The client makes no attempt to leverage any kind of local caching or buffering.

<img src="true_remote_access.png">

The benefit of this extreme is that the **server now has full control and knowledge of how the clients are accessing and modifying a file.** This makes it **easier to ensure that the state of the filesystem is consistent**.

The downside of this model is that **every file operation pays a network cost**. In addition, this model is **limited in its scalability**. Since every file operation goes to the server, the server will get overloaded more quickly, and will not be able to service as many clients.

## Remote File Service: A compromise

We should allow clients to benefit from using their local memory/disk and to store at least some part of the file they are accessing. This may include downloading blocks of the file they are accessing, as well as prefetching blocks that they may soon be accessing.

**When files can be served from a local cache, lower latencies on file operations can be achieved since no additional network cost is incurred**. In addition, server scalability is increased as local service decreases server load.

Once we allow clients to cache portions of the file, however, it becomes necessary for the clients to start interacting with the server more frequently. On the one hand, **clients need to notify the server of any modifications they make**. In addition, they need to query the server at some reasonable frequency to **detect if the files they are accessing from their cache have been changed by someone else**.

This approach is beneficial because the server continues to have insight into what the clients are doing and retains some control over which accesses can be permitted. This makes it easier to maintain consistency. At the same time, the server can be somewhat out of the way, allowing some proportion of operations to occur on the client. This helps to reduce the load ont he server.

The downside with this compromise is that the server becomes more complex. The server needs to perform additional tasks and maintain additional state to make sure it can provide consistency guarantees. This also means that the client has to understand file sharing semantics that are different from what they are used to in a normal filesystem.

## Stateless vs Stateful File Server

A **stateless** server keeps no state. It has no notion of which files/blocks are being accessed, which operations are being performed, and how many clients are accessing how many files.

As a result, every request has to be completely self-maintained. This type of server is suitable for the extreme models, but it cannot be used for any model that relies on caching. **Without state, we cannot achieve consistency management**. In addition, since every request has to be self-contained, more bits need to be transferred on the wire for each request.

One positive of this approach is that since there is no state on the server, there is no CPU/memory utilization required to manage that state. Another positive is that this design is very resilient. Since requests cannot rely on any internal state, the server can be restarted if it fails at no detriment to the client. 

A **stateful server** maintains information about the clients in the system, which files are being accessed, which types of accesses are being performed, which clients have a file cached, and which clients have read/written the file. 

Because of the state, **the server can allow data to be cached and can guarantee consistency**. In addition, state management allows for other functionality like locking. Since accesses are known, clients can request relative blocks, instead of having to specify absolute offsets.

On failure, however, all that state needs to be recovered so that the filesystem remains consistent. This requires that the state must be incrementally check-pointed to prevent too much loss. In addition, there are runtime overheads incurred by the server to maintain state and enforce consistency protocols and by the client to perform caching.

## Caching State in a DFS

Caching state in a DFS involves **allowing clients to locally maintain a portion of the state** - a file block, for example - and also **allows them to perform operations on this cached state**: `open/read/write`, etc. 

Keeping the **cached portions of the file consistent with the server's representation** of the file requires **cache coherence mechanisms**.

For example, two clients may cache the same portion of file. If one client modifies their local portion, and sends the update to the file server, how does the the other client become aware of those changes?

For client/server systems, these coherence mechanisms may be triggered in different ways. For example, the mechanism may be triggered on demand any time a client tries to access a file, or it may be triggered periodically, or it may be triggered when a client tries to open a file. In addition, the coherence mechanisms may be driven by the client or the server.

## File Sharing Semantics in DFS

### Single Node, UNIX

Whenever a file is modified by any process, that change is immediately visible to any other process in the system. This will be the case even if the change isn't pushed out to the disk because both processes have access to the same buffer cache. 

<img src="single_node_fs.png">


### DFS

Even if a write gets pushed to the file server immediately, it will take some time before that update is actually delivered to the server. It is possible that another client will not see that update for a while, and every time it performs a read operation it will continue seeing "stale" data. Given that message latencies may vary, we have no way of determining how long to delay any possible read operation in order to make sure that any write from anywhere in the system arrives at the file servers so that we can guarantee no staleness.

<img src="dfs_coherence.png">

In order to maintain acceptable performance, a **DFS will typically sacrifice some of the consistency and will accept more relaxed file sharing semantics**.

## Session Semantics

In **session semantics**, the client writes back whatever data was modified on close. Whenever a client needs to `open` a file, the cache is skipped, and the client checks to see if a more recent version is present on the file server. 

With session semantics, it is very possible for a client to be reading a stale version of a file. However, **we know that when we close a file or open a file that we are consistent with the rest of the filesystem at that moment**.

Session semantics are easy to reason about, but they are not great for situations where clients want to concurrently share a file. For example, for two clients to be able to write to a file and see each other's updates, they will constantly need to `close` and `open` the file. Also, files that stay open for longer periods of time may become severely inconsistent.

## Periodic Updates

In order **to avoid long periods of inconsistency, the client may write back changes periodically to the server**. In addition, the server can send invalidation notifications periodically, which can enforce time bounds on inconsistency. Once the server notification is received, a client has to sync with the most recent version of the file.

The filesystem can provide explicit operations to let the client `flush` its updates to the server, or `sync` its state with the remote server. 

## Other Strategies

With immutable files, you never modify an old file, but rather, create a new file instead.

With transactions, the filesystem exports some API to allow clients to group file updates into a single batch to be applied atomically.

## File vs Directory Service

Filesystems have **two different types of files: regular files and directories**. These two types of files often have **very different access patterns**. As a result, it is not uncommon to adopt one type of semantics for files, and another for directories. For example, we may have session semantics for files, and UNIX semantics for directories.