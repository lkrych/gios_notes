# Memory Management

## Introduction

One of the goals of the OS is to manage the physical resources, in this lecture we will talk about DRAM, and how it is managed on behalf of one or more executing processes. 

The operating system provides an **address space** to an executing process as a way **to decouple the physical memory from the memory the process interacts with**.  Almost everything uses virtual addresses, and the OS translates these addresses into physical addresses where the actual data is stored.

The **range of virtual addresses** that are visible to a process **can be much larger than the actual amount of physical memory** behind the scenes. In order to mange the physical memory, **the operating system must be able to allocate physical memory and arbitrate how it is being accessed**. 

**Allocation** requires that the operating system incorporate mechanisms and data structures so that is can **track how memory is used and what memory is free**. It is possible that since some virtual memory are not present in physical memory, that they correspond to secondary storage like disk. The OS must be able to replace the contents in physical memory with these contents from the disk. 

**Arbitration** requires that the operating system be able to quickly **translate a virtual address into a physical address and validate it to verify that it is a legal access**. Modern OS's rely on hardware support and software implementations to accomplish this. 

The virtual address space is subdivided into **fixed-size segments called pages**. The **physical memory is divided into page frames** of the same size. In terms of allocation, **the OS maps pages into page frames**. In this type of page-based memory management, the **arbitration is done via page tables**. 

Paging is not the only way to decouple the virtual and physical memories. **Another approach is segmentation**.  With segmentation, the allocation process **doesn't use fixed-size pages**, but rather more **flexibly-sized segments that can be mapped to some regions in physical memory** as well as swapped in and out of physical memory. Arbitration of access uses segment registers that are typically supported in modern hardware. 

Paging is the dominant memory management mechanism.

## Memory Management: Hardware Support

Hardware mechanisms help make memory management decisions easier, faster and more reliable.
