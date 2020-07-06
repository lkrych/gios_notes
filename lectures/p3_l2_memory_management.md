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

## Page Table Entry

Every page table entry will have at least a PFN and a valid bit. This bit is also called a present bit, as it represents whether of not the contents of virtual memory are present in physical memory or not.

There are a number of other bits that are present in the page table entry which the OS uses to make memory management decisions and also that the hardware understands and knows how to interpret.

<img src="page_table_entry.png">

For example, most hardware supports a **dirty bit** which gets set whenever a page is written to. This is useful in file systems, where files are cached in memory. The OS uses the dirty bit to see which files need to be updated on disk.

It is also useful to keep track of an **access bit**, which tells the OS whether the page has been accessed. This is useful for deciding which entries to swap out of the page table.

There are also **protection bits** which specify whether a page is allowed to be read, written or executed.

The **MMU uses the page table entry not just to perform the address translation, but also to rely on these bits to determine the validity of the access**. If the hardware determines that a physical memory access cannot be performed, it causes a page fault.

If this happens, then the CPU will place an error code on the stack of the kernel, and it will generate a trap into the OS kernel, which will in turn invoke the page fault handler. This handler determines the action to take based on the error code and the faulting address.

On x86 platforms, the error code is generated from some of the flags in the page table entry and the faulting address is stored in the CR2 register. 

## Page Table Size

A page table has a number of entries that is equal to the number of virtual page numbers that exist in a virtual address space. If we want to calculate the **size of a page table we need to know both the size of a page table entry and the number of entries contained in a page table**. 

On a 32-bit architecture, where each address is represented by 32 bits, each of the page table entries is 4 bytes (32 bits) in size, and this includes both the PFN and the flags. The total number of page table entries depends on the total number of VPNs.

In a 32-bit architecture there are 2^32(4GB) of addressable memory. A common page size is 4KB. In order to represent 4GB of memory in 4KB pages, we need 2^32/2^12 = 2^20 entries in our page table. Since each entry is 4 bytes long, we need 2^22 bytes (4MB) of memory to represent our page table. 

If we had a 64-bit architecture, we could represent 2^64 bytes of physical memory in chunks of 2^12 bytes. Since each entry is now 8 bytes long, we need 2^3 * 2^64/2^12 = 32PB.

Remember that page tables a per-process allocation.

It is important to know that a process will not use all fo the theoretically available virtual memory. Even on 32-bit architectures, not all 4GB is used by every type of process. The problem is that page tables assume that there is an entry for every VPN, regardless of whether the VPN is needed by the process or not.

## Multi-level Page Tables

OS's don't use flat page tables anymore, they use hierarchical page table structures.

<img src="hierarchical_pt.png">

The outer level is referred to as the **page table directory**. Its elements are not pointers to individual page frames, but rather **pointers to page table themselves**.

The inner level has proper page tables that actually point to page frames in physical memory. Their entries have the page frame number and all the protection bits.

The internal page table exists only for those virtual memory regions that are actually valid. Any holes in the virtual memory space will result in lack of internal page tables.

If a process requests more memory to be allocated via malloc, the OS will check and potentially create another page table for the process, adding a new entry to the page table directory. The new internal page table entry will correspond to some new virtual memory region that the process has requested.

To find the right element in the page table structure, the virtual address is split into more components. 

<img src="multilevel_page_translation.png">

The last part of the logical address is still the offset, which is used to actually index into the physical page frame. The first two components of the address are used to index into the different levels of the page tables, and they ultimately produce the PFN that is the starting address of the physical memory region being accessed. **The first portion indexes into the page table directory to get the page table, and the second portion indexes into the page table to get the PFN**. 

In this particular scenario, the address format is such that the outer index occupies 12 bits, the inner index occupies 10 bits, and the offset occupies 10 bits.

This means that a given page table can contain 2^10 entries, and each entry can address 2^10 bytes of physical memory. This means that each page table can address about 1MB of memory. Whenever there is a gap in virtual memory that is 1MB or greater, we don't need to fill in the gap with unused page tables. For example, if we have a page table containing pages 0-999, and we allocate another page table containing pages 30000-30999, we don't need to allocate the 29 page tables in between. This will reduce the overall size of the page table(s) that are required for a particular process. This is in contrast to the "flat" page table, in which every entry needs to be able to translate every single virtual address and it has entries for every virtual page number. There can't be any gaps in a flat page table.

The notion of a hierarchical page table can be extended to use even more layers. This technique is important to 64-bit architectures. The page table requirements are larger in these architectures and as a result are often more sparse. The third level can contain pointers to page table directories, and the fourth level is a map of page table directory pointers.

Because of this we have larger gaps between page table that are actually allocated. With four level addressing structures, we may be able to save entire page table directories from being allocated as a result of these gaps. 

As we add more levels, the internal page tables/directories end up covering smaller regions of the virtual address space. As a result, it is more likely that the virtual address space will have gaps that will match that granularity, and we will be able to reduce the size of the page table as a result.

The downside of adding more levels to the page table structure is that there are more memory accesses required for translation, since we will have access to more page table components before actually accessing the physical memory. Therefore, the translation latency will be increased. 

## Speeding up Translation with a TLB

Adding levels to our page table structure adds overhead to the address translation process.

In the simple, single-level page table design, we need to memory references: one to access the page table entry, and one to access the actual physical memory.

In the four-level page table, we will need to perform four memory accesses to navigate through the page table entries before we can perform our physical memory reference. 

The standard technique to **avoid these accesses to memory** is to use a **page table cache**.  On most architectures, the MMU integrates **a hardware cache used for address translations**. This cache is called the **translation lookaside buffer (TLB)**.

On each address translation, the TLB cache is first referenced, and if the results can be generated from the lookup we can bypass all the page table navigation. If we have a miss, we still need to perform the lookup.

In addition to proper address translation, the TLB entries will contain all of the necessary protection/validity bits that ensure that access is correct and the MMU will generate a fault if needed. 

## Inverted Page Tables

Standard page tables serve to map virtual memory to physical memory on a per process basis. Each process has its own page table, so the total amount of virtual memory "available" in the system is proportional to the number of processes currently in the system.

A different way to think about structuring virtual memory is to make it closer to physical memory layout. **Inverted page tables** are **managed on a system-wide basis, not on a per-process basis**, and **each entry in the inverted page table points to a frame in main memory**.

The representation of a logical memory address when using inverted page tables is slightly different. The memory address contains the process ID (PID) of the process attempting to access the memory address as well as the virtual address and the offset.

A linear scan of the inverted page table is performed when a process attempts to perform a memory access. When the correct entry is found - validated by the combination of PID and virtual address - it is the index of that entry that is the frame number in physical memory. That index combined with the offset serves to reference the exact physical address. 

<img src="inverted_page_table.png">

Linear scans are slow, but thankfully the TLB comes to rescue to speed up lookups. That being said, there are ways to improve lookups. One common way is to use **hashing page tables**. The address is hashed and looked up in a hash table, where the hash points to a small linked list of possible matches. This allows us to speed up the linear search to consider just a few possibilities.

<img src="hashing_page_table.png">

## Segmentation

Virtual to physical memory mappings can also be maintained with **segments**. With segments, **the address space is divided into components of arbitrary size**, and the components will correspond to some logically meaningful section of the address sapce, like the code, heap, data, or stack.

A virtual address in segmented memory mode includes a **segment selector** and an offset. The selector is used in combination with a **descriptor table** to produce a physical address which is combined with the offset to describe a precise memory location.

In its pure form, a segment could be represented with a contiguous section of physical memory. In that case, the segment would be defined by its base address and its limit registers. 

In practice, segmentation and paging are used together. The linear address that is produced from the logical address by the segmentation process is then passed to the paging unit to ultimately produce the physical address. 

<img src="segmentation.png">

## Page Size

The size of the memory page, or frame, is determined by the number of bits in the offset. For example, if we have a 10-bit offset, our page size is 2 ^ 10 bytes, or 1KB. A 12-bit offset means we have a page size of 4KB.

In practice, systems support different page sizes. For Linux, on x86 platforms there are several common page sizes. 4KB pages are pretty popular, and are the default in the Linux environment. Page sizes can be much larger.

Large pages in LInux are 2MB in size, requiring an offset of 21 bits. Huge pages are 1GB in size, requiring 30 bits.

One benefit of larger pages is that more bits are used for the offset, so fewer bits are used for the virtual page number. This means there are fewer pages, which means we have a smaller page table. Large pages reduce the page table size by a factor of 512, and huge page tables reduce the size by 1024 compared to page tables with 4KB page size.

## Memory Allocation

**Memory allocation** incorporates **mechanisms to decide what are the physical pages that will be allocated to a particular virtual memory region**. Memory allocators can exist at the kernel level as well as the user-level.

**Kernel level allocators** are responsible for **allocating pages for the kernel** and also for **certain static state of processes** when they are created (code, stack, etc.). In addition, kernel level allocators are responsible for keeping track of the **free memory that is available in the system**. 

**User level allocators** are used for dynamic process state - **the heap**. This is memory that is dynamically-allocated during the process's execution. The basic interface for these allocators includes `malloc` and `free`. These calls request some amount of memory from the kernel's free pages and then ultimately release it when they are done.

Once the kernel allocates some memory through malloc, the kernel is no longer involved in the management of the memory. That memory is now under the purview of the user level allocator. 

## Memory Allocation Challenges

