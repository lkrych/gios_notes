# I/O Management

## Table of Contents

## I/O Devices

The execution of applications doesn't rely on only the CPU and memory, but other hardware components as well. Some of these components are specifically tied to receiving inputs or directing outputs and these devices are referred to as **I/O devices**. 

Examples of I/O devices include: keyboards, microphones, displays, speakers, mice, network interface cards.

## I/O Device Features

Devices come in all shapes and sizes with a lot of variability in their hardware architecture, functionality, and interfaces. Our discussion will focus on the key features of a device that enables its integration into a system.

In general, a device will have **a set of control registers which can be accessed by the CPU** and permit CPU/device interactions. These registers are typically divided into:
1. **Command Registers** - The CPU uses these to control the device.
2. **Data Registers** - Used by the CPU to transfer data in and out of the device.
3. **Status Registers** - Used by the CPU to understand what is happening in the device.

<img src="io_device.png">

Internally, the device will incorporate all other device-specific logic. This include the **microcontroller**, which is the devices CPU, on-device memory, as well as any other logic needed by the device (Some devices need chips for converting analog to digital signals, network devices need chips to interact with the physical network medium).

## CPU/Device Interconnect

**Devices interface with the rest of the system via a controller** that is typically integrated as part o the device packaging. It is used to connect the device with the rest of the CPU complex via a CPU/Device interconnect.

<img src="pci_bus.png">

In this figure, all of the controllers are connected to the rest of the system via a **Peripheral Component Interconnect (PCI)** bus. 

Modern platforms typically support **PCI express**, which is more technologically advanced than PCI-X and PCI. PCI Express has more bandwidth, is faster, has lower latency, and supports more devices than PCI-X. For compatibility reasons, most platforms still include PCI-X which follows the original PCI standard.

The PCI bus is not the only possible interconnect that can be present in the system. In the example above we can see a SCSI bus that connects SCSI disks and an expansion (peripheral) bus that connects things like keyboards.

The **device controllers determine what type of interconnect a device can attach to**.  **Bridging controllers** can handle any difference between different types of interconnects.

## Device Drivers

Operating systems support devices via **device drivers**. 

<img src="device_drivers.png">

Device drivers are **device-specific software components**. The operating system needs to include a device driver for every type of device that is included in the system.

**Device drivers are responsible for all aspects of device access, management and control**. This includes logic that determines how requests are passed from high level components to the device and how those components respond to errors or notifications from the device. Generally, device drivers govern any device-specific configuration/operation details.

The manufacturer of a device is responsible for making drivers available for all the operating systems where the device will be used. 

**Operating systems standardize their interfaces to device drivers.** Typically this is done by providing some device driver framework so that device manufacturers can develop their drivers within that framework. **This helps to decouple the operating system from a fixed set of supporting devices**, which makes the OS independent of the devices it supports. 

## Types of Devices

Devices can be roughly grouped into categories.

**Block Devices** operate at the granularity of blocks of data that are delivered to and from the device via the CPU interconnect. A key property of block devices is that individual blocks can be accessed. Disks are block devices, so if you have ten blocks of data on disk, you can directly request the ninth one. 

**Character Devices** work with a serial sequence of characters and support a get/put character inteface. A keyboard is an instance of a character device.

**Network Devices** are somewhere in between block and character devices. They deliver more than a character at a time, but their granularity is not a fixed block size. They are more like a stream of data chunks of potentially different sizes.

The interface a device exposes to the OS is standardized based on the type of device. All block devices should support operations for reading and writing a block of data. All character devices should support operations to get and put a character.

Internally, the **OS maintains some representation** for each of the devices available on the platform. This is **typically done using a file abstraction** where each file represents a device. This allows the OS to interact with devices via the same interface it already maintains for interacting with files.

On Unix-like systems, all devices appear as files under the `/dev` directory. They are treated by the filesystems `tmpfs` and `devfs`.

## CPU/Device Interactions

The **device registers appear to the CPU as memory locations at a specific physical address**. When the **CPU writes to these locations, the integrated PCI controller realizes that these accesses should be routed** to the appropriate device.

This means that **a portion of the physical memory on the system is dedicated for device interactions**. We call this **memory-mapped I/O**. The portion of the memory that is reserved for these interactions is controlled by the **Base Address Registers (BAR)**. These registers get configured during the boot process in accordance to PCI protocol.

In addition, the **CPU can access devices via special instructions**. x86 platforms specify certain in/out instructions that are used for accessing devices. Each instruction needs to specify the target device - the I/O port - as well as some value that will be passed to the device. This model is called the **I/O port model**.

The **path from the device to the CPU complex** can take two routes. **Devices can generate interrupts** to the CPU. **CPUs can poll devices** by reading their status registers to determine if they have some response/data for the CPU. 

With interrupts, the downside is overhead from interrupt handlers: the actual steps of the handler as well as the setting/resetting of interrupt masks as well as indirect effects due to cache pollution.

That being said, interrupts can be triggered by the device as soon as the device has information for the CPU.

For polling, the OS has the opportunity to choose when it will poll. The OS can choose to poll at times when cache pollution will be at its lowest. However, this strategy can introduce delay in how the event is observed or handled, since the handling happens at some point after the event has been generated by device. In addition, to much polling may introduce CPU overhead.

## Device Access PIO

With just basic support from an interconnect like PCI, a CPU can request an operation from an I/O device using **programmed I/O (PIO)**.

The CPU issues instructions by writing into the command registers for the device. The CPU controls data movement to/from the device by reading/writing into the data registers for the device.

Let's consider how a process running on the CPU transmits a network packet via a **network interface card (NIC)** device.

First, the CPU needs to write to a command register on the device. This command needs to instruct the device that it needs to perform a transmission of the data that the CPU will provide. The CPU then copies the packet into the data registers and will do this until the entire packet is sent.

For example, we may have a 1500B packet that we wish to transmit using 8 byte data registers. The whole operation will take 1 CPU access to the command register and then 188 - 1500 / 8 rounded up - accesses to the data register. In total, 189 CPU accesses are needed to transmit the packet.

## Device Access DMA

An alternative to using PIO is to use **Direct Memory Access (DMA)** supported devices. This method requires additional hardware support in the form of a DMA controller.

For devices that have DMA support, the CPU still writes commands directly to the command registers on the device. However, the data movement is controlled by configuring the DMA controller to know which data needs to be moved from memory to the device and vice versa. 

Let's again consider how a process running on the CPu transmits a network packet via a network interface card device using DMA.

First the CPU needs to write to a command register on the device. This commands needs to instruct the device that it needs to perform a transmission of the data that the CPU will provide.

This command needs to be accompanied with an operation that configures the DMA controller with the information about the memory address and size of the buffer that is holding the network packet.

If we have a 1500B packet that we wish to transmit using 8 byte data registers, the whole operation will take 1 CPU access to the command register and then 1 DMA configuration operation.

DMA configuration is not trivial. It takes more cycles than a memory access. For smaller transfers, PIO is more efficient.

In order for DMA to work, the data buffer must be in physical memory until the transfer completes. It cannot be swapped out to disk since the DMA controller only has access to physical memory. This means that **the memory regions involved in DMA are pinned**.

## Typical Device Access

When **a user process needs to perform an operation that requires a device, the process will make a system call** specifying the appropriate operation.

The OS will then run the in-kernel stack associated with the specific device, and may perform some preprocessing to prepare the data received by the user process for the device. For example, the kernel may form a TCP/IP packet from an unformatted buffer.

Then the OS will invoke the appropriate device driver for the device. The device driver will then perform the configuration of the request to the device. For example, a device driver to a network interface card will write a record that configures the device to perform a transmission of the packet sent from the OS.

The device driver issues commands and sends data using the appropriate PIO or DMA operations. The drivers are responsible for ensuring that any commands and data needed by the device are not overwritten or undelivered.

Finally, once the device is configured, the device will perform the actual request. For example, a NIC will actually transmit the packet onto the network.

Any results/events originating on the device will traverse this chain in reverse: from the device to the driver to the kernel and finally back to the user process.

<img src="user_to_device.png">

## OS Bypass

It is not necessary to go through the kernel to get to a device. It is possible to configure some devices to be accessible directly from the user level. This is called **operating system bypass**. In OS bypass, any memory/registers assigned for use by the device is directly available to the user process.

The OS is involved in making the device registers available to the user process on create, but then it is out of the way.

Since we don't want to interact with the kernel in order to control the device, we need a **user-level driver/library**, that the process links in order to interact with the device. These libraries are usually provided by the manufacturers.

**The OS has to retain some coarse-grain control**. For example, the OS can still enable/disable a device or add permissions to add more processes to use the device. The device must have enough registers so that the OS can map some of them into one or more user processes while still retaining access to a few registers itself so it can interact with the device at a higher level.

When the device needs to pass some data to one of the processes interacting with it, the device must figure out which process the data belongs to. The device must perform some protocol functionality in order to demultiplex multiple chunks of data that belong to different processes. Normally, the kernel performs the demultiplexing, but in OS bypass that responsibility is left to the device. 

## Sync vs Async Access

<img src="sync_v_async.png">

When an I/O request is made, the user process typically requires some response from the device, even if it is just an acknowledgement.

**What happens to a user thread once an I/O request is made depends on whether the request was synchronous or asynchronous**.

For **synchronous operations, the calling thread will block**. The OS kernel will place the thread on the corresponding wait queue associated with the device, and the thread will eventually become runnable again when the response to its request becomes available.

With **asynchronous operations, the thread is allowed to continue execution** as soon as it issues the request. At some later time, the user process can be allowed to check if the response is available. Alternatively, the kernel can notify the process that the operation is complete and that the results are available.


## Block Device Stack

**Block devices**, like disks are typically **used for storage**. The typical **storage related abstraction** used by applications is the **file**. A file is **a logical storage unit which maps to some underlying physical storage location**. At the level of the user process we don't think about interacting with blocks of storage, we think about interacting with files.

**Below the file-based interface used by applications is the filesystem**. The file system will receive read/write operations form a user process for a given file, and will have the information to find the file, determine if the user process can access it and which portion to access.

OS's allow for a filesystem to be modified or completely replaced with a different filesystem. To make this easy, **OS's standardize the filesystem interface** that is exposed to a user process. The standardized API is the POSIX API, which includes the system calls for read and write. The result is that **filesystems can be swapped out without breaking** user applications.

If the files are stored on block devices, the filesystem will need to interact with these devices via their device drivers. Different types of block devices can be used for the physical storage and the actual interaction with them will require certain protocol-specific APIs. There are often differences among the APIs :(.

In order to **mask these device-specific differences**, the block device stack introduces another layer: the **generic block layer**. The intent of this layer is to provide a standard for a particular operating system to all types of block devices. The full device features are still available and accessible through the device driver, but are abstracted away from the filesystem itself.

Thus, in the same way that the filesystem provides a consistent file API to user processes, the OS provides a consistent block API to the filesystem. 

## Virtual File System

It would be nice if:

1. A user application could see files across multiple devices as a single filesystem.
2. Files could be presented that aren't even local to the machine.
3. Manufacturers didn't create devices that preferred specific filesystems.

To solve these underlying problems, OS's like Linux include a **virtual filesystem (VFS)** layer. This layer hides all details regarding the underlying filesystem(s) from the higher level customers.

<img src="vfs.png">

Applications continue to interact with the VFS using the same POSIX API as before, and the VFS specifies a more detailed set of filesystem-related abstractions that every single underlying filesystem must implement.

## VFS Abstractions

The **file** abstraction represents the elements on which a VFS operates.

The OS abstracts files via **file descriptors**. A file descriptor is an integer that is created when a file is first opened. There are many operations that can be supported on files using a file descriptor, such as `read`, `write`, and `close`.

For each file, the VFS maintains a **persistent data structure** called an **inode**. The inode maintains a list of all the data blocks corresponding to the file. In this way, the inode is the "index node" for a file. The **inode also contains** other information for that file, like **permissions associated with the file, the size of the file, and other metadata**. Inodes are important because files do not need to be stored contiguously on disk. The inode will keep track of where they are.

To help with certain operations on directories, Linux maintains a data structure called a **dentry**(directory entry). Each dentry object corresponds to a single path component that is being traversed as we are trying to reach a particular file. For example, if we are trying to access a file `/users/byron`, the filesystem will create a dentry for every path component: `/`, `/users`, and `/users/byron`.

This is useful because when we need to find another file in `/users/byron`, we don't need to go through the entire path and re-read the files that correspond to all of the directories in order to get to the right directory. The filesystem will maintain a **dentry cache** containing all of the directories that we have previously visited. Note that dentry objects live only in memory, they are not persisted.

The **superblock abstraction** provides information about **how a particular filesystem is laid out on some storage device**. The data structure maintains a map that the filesystem uses so it can figure out how it has organized the persistent data elements like inodes and the data blocks that belong to different files. 

## VFS on Disk

The **VFS data structures are software entities**. They are created and maintained by the OS filesystem component.

Other than dentries, the remaining components of the filesystem will correspond to blocks that are present on disk. The files are written to disk as block. The inodes - which track all the blocks of a file - are persisted as well in a block.

To make sense of all of this - which blocks hold data, which blocks hold inodes, and which blocks are free. The superblock maintains an overall map of the disks on a particular device. This map is used for both allocation and lookup.

## Linux - ext2 - Second Extended Filesystem

The ext2 filesystem was the default filesystem in Linux until it was replaced by ext3 and ext4 more recently.

A disk partition that is used in ext2 looks as follows: 

<img src="ext2.png">

The first block is not used by Linux and is often used to boot the system.

**The rest of the partition is divided into block groups.** The size of these block groups have no correlation to the physics of the actual disks these groups abstract.

Each block group contains several blocks.

The first block is the superblock, which contains information about the overall block group:
1. the number of inodes
2. the number of disk blocks
3. the start of the free blocks

The overall state of the block group is further described by the **group descriptor**, which contains information about:
1. bitmaps
2. number of free nodes
3. number of directories

**Bitmaps** are used to **quickly find free blocks and inodes**. Higher level allocators can read the bitmaps to easily determine which blocks and inodes are free and which are in use. 

The inodes are numbered from 1 to some maximum value. **Every inode in ext2 is a 128B data structure that describes exactly one file**. The inode will **contain information about file ownership as well as some pointers to the actual data blocks that hold data**. 

Finally, the block group contains the actual data blocks themselves that hold the data. 

## Inodes

Inodes play a key role in organizing how files are stored on disk because they are the index of all the disk blocks that correspond to a specific file.

<img src="inode.png">

A file is uniquely identified by its inode. The inode contains a list of all of the blocks that correspond to the actual file. In addition to the list of blocks, an inode contains metadata about the file.

The file shown above has 5 blocks allocated to it. If we need more storage for the file, the filesystem can allocate a free block and simply update the inode to contain a sixth entry, pointing to a newly allocated block.

**The benefit of this approach is that it is easy to perform both sequential and random accesses to the file.**

**The downside of this approach is that there is a limit on the file size** for files that can be indexed using this data structure. For example, if we have a 128B inode containing 4B block pointers, we can only address 32 blocks. If each block can store 1Kb of information, our file size limit is 32KB which is too restrictive. 

## Inodes with indirect pointers

One way to solve the issue of file size limits is to use **indirect pointers**. 

<img src="indirect_inode.png">

The first section of blocks contain blocks that point directly to data. The direct pointers will point to 1kb per entry.

To extend the number of disk blocks that can be addressed via a single inode element, while also keeping the size of the inode small, we use indirect pointers.

An indirect pointer will point to a block of pointers, where each pointers points to data. Given that a block contains 1 KB of space, and a pointer is 4B large, a single indirect pointer can point to 256KB of file content.

A double indirect pointer can point to 64MB of file content.

The benefits of indirect pointers is that it allows us to use relatively small inodes while being able to address larger and larger files.

The downside of indirect pointers is that file access is slowed down. Without any indirect pointers, we have at most two disk accesses to get a block of content, one access for the inode, one access for the block. With double indirect pointers, we double the number of accesses: inode, block and two pointers.

## Disk Access Optimizations