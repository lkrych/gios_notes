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

<img src="hardware_support.png">

Hardware mechanisms help make memory management decisions easier, faster and more reliable.

Every CPU package contains a **memory management unit (MMU)**. The CPU issues **virtual addresses** to the MMU, and the **MMU is responsible for converting these into physical addresses**. 

If there is an issue, the MMU can generate a **fault**. A fault can signal that the memory access is illegal (the memory hasn't been allocated). A fault could also signal that there are insufficient permissions to perform a particular access. A third type of fault can indicate that the requested page is not present in memory and must be fetched from disk.

Another way the hardware supports memory management is by **assigning designated registers to help with the memory translation process**. For instance, registers can point to the currently active page table in a page-based memory system. In a segmentation system, the registers may point to the base address, the size of the segment and the total number of segments.

Since the memory address translation happens on every memory reference, **most MMU incorporate a small cache of virtual/physical address translations -- the translation lookaside buffer (TLB)**. This buffer makes the translation process faster.

Finally, the actual translation of the physical address from the virtual address is done by the hardware. While the OS maintains page tables which are necessary for the translation process, the hardware actually performs memory access. This implies that hardware dictates what type of memory management modes are available. 

## Page Tables

Pages are the most popular method for memory management. They are used to **convert virtual memory addresses into physical memory addresses**. 

<img src="page_table.png">

For each virtual address, an entry in the page table is used to determine the actual physical address associated with that virtual address.

The sizes of the **pages in virtual memory are identical to the sizes of the page frames in physical memory**. By keeping the size the same, the OS does not need to manage the translation of every single virtual address within a page. Instead, we translate the first virtual address in a page to the first physical address in a page frame. The rest of the memory addresses in the page map to the corresponding offsets in the page frame. As a result, we can reduce the number of entries in the page table. 

<img src="vpn_pfn.png">

What this means is that **only the first portion of the virtual address is used to index into the page table**. We call this part of the address the **virtual page number (VPN)**, and the rest of the virtual address is the **offset**. The VPN is used as an index into the page table, which will produce the **physical frame number (PFN)**, which is the first physical address of the frame in DRAM. To complete the full translation, the **PFN needs to be summed with the offset specified in the latter portion of the virtual address to produce the actual physical address**. The PFN with the offset can be used to reference a specific location in DRAM.

### How does it actually work though?

Let's say the we want to initialize an array for the first time. We have already allocated the memory for that array into the virtual address space for the process, we have just never accessed it before. 

The first time we access this memory, the OS will realize that there isn't physical memory that corresponds to the range of virtual memory addresses, so it will take a free page of physical memory and create a page table entry linking the two. 

The **physical memory is only allocated when the process is trying to access it**. This is called **allocation on first touch**. The reason for this is to make sure that physical memory is only allocated when it it really needed. 

If a process hasn't used some of its memory pages for a long time, it is possible that those pages will be reclaimed. The contents will no longer be present in physical memory. They will be swapped out to disk, and some other content will end up in the corresponding physical memory slot. 

In order to detect this, p**age table entries also have a number of bits that give the memory management system some more information about the validity of the access**. If the page is in memory and the mapping is valid, its valid bit will be 1. Otherwise it will be 0. 

If the **MMU sees that this bit is 0 when an access is occurring, it will raise a fault and trap to the OS**. The OS has to decide if the access should be permitted, and if so, where the page is located and where it should be brought into DRAM. Ultimately, **if the access is granted, there will be a new page mapping that is reestablished after access is granted**.

In summary, the **OS creates a page table for every single process in the system**. That means that whenever a context switch is performed, the OS must swap in the page table associated with a new process. **Hardware assists with page table access by maintaining a register that points to the active page table**. On x86 platforms, this register is the CR3 register. 

<img src="cr3.png">