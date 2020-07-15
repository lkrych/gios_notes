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

