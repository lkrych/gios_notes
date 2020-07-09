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

## CPU Device Interconnect

**Devices interface with the rest of the system via a controller** that is typically integrated as part o the device packaging. It is used to connect the device with the rest of the CPU complex via a CPU/Device interconnect.

<img src="pci_bus.png">

In this figure, all of the controllers are connected to the rest of the system via a **Peripheral Component Interconnect (PCI)** bus. 

Modern platforms typically support **PCI express**, which is more technologically advanced than PCI-X and PCI. PCI Express has more bandwidth, is faster, has lower latency, and supports more devices than PCI-X. For compatibility reasons, most platforms still include PCI-X which follows the original PCI standard.

The PCI bus is not the only possible interconnect that can be present in the system. In the example above we can see a SCSI bus that connects SCSI disks and an expansion (peripheral) bus that connects things like keyboards.

The **device controllers determine what type of interconnect a device can attach to**.  **Bridging controllers** can handle any difference between different types of interconnects.