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