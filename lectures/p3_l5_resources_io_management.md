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