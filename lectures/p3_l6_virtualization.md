# Virtualization

## Table of Contents

## What is Virtualization?

In order to concurrently run diverse workloads on the same physical hardware without requiring that a single OS be used for all of the applications, it was necessary to come up with a model where **multiple OS's can concurrently be deployed on the same hardware**. This is **virtualization**.

<img src="virtualization.png">

With virtualization, each of the **operating systems** that are deployed on the same physical platform **have the illusion that they own the underlying hardware resources**. 

**Each OS, plus its applications and virtual resources** is called a **virtual machine (VM)**. 

Supporting the coexistence of multiple VMs on a single machine **requires underlying functionality in order to deal with the allocation and management of real hardware resources. It is also necessary to guarantee isolation across the VMs**.

This functionality is provided by the **virtualization layer**, also referred to as the **virtual machine monitor or hypervisor**. 

A virtual machine is an **efficient, isolated duplicate** of a real machine.

Virtualization is supported by the VMM, which has three responsibilities:

1. The VMM must provide an environment that is essentially identical to the original machine. The capacity may differ, but the overall setup should be the same. The VMM must provide **fidelity** that the representation of the hardware that is visible from the VM matches the hardware that is available on the physical platform.
2. The programs run on VMs must show at worst only minor decreases in speed. VMs are only given a portion of the resources available to the host machine. **The goal of a VMM is to ensure that the VM performs at the same speed as a native application**. 
3. The VMM is in complete control of system resources. It should control who accesses which resources and when, and it should be **relied upon to provide safety and isolation** among the VMs.

## Benefits of Virtualization

Virtualization enables **consolidation**, the ability to run multiple VMs on a single physical platform. This leads to fewer machines with less space, with fewer admins, with potentially a smaller electric bill!

It allows companies to decrease cost and improve manageability. 

Virtualization also makes **migration** easier. Since the OS and the applications are no longer coupled to a physical system, it is easy to setup, teardown and clone virtual machines.

Virtualization also helps us address **availability** and **reliability**. If we notice that a physical machine is about to go down, we can easily spin up a new VM on a different physical platform.

Because the OS and the applications are nicely encapsulated in a VM, it becomes easier to contain bugs or malicious code to those isolated containers without bringing down other VMs or the physical platform.

Lastly, virtualization is good for OS research and providing support for legacy OSes. 

## Virtualization Models: Bare Metal

In **bare-metal** virtualization (also known as **hypervisor-based** or **type 1 virtualization**), the VMM manages all of the hardware resource and supports execution of the VMs.

<img src="bare_metal.png">

One issues with this model concerns devices. According to the model, the hypervisor must manage all possible devices. In other words, device manufacturers have to provide device drivers not just for the different OS's, but also for different hypervisors. 

To eliminate this problem, the hypervisor model typically integrates a special virtual machine, a **service VM**, that runs a **standardized OS with full hardware access** privileges, allowing it to manipulate hardware as if it was native. 

The privileged VM runs all of the device drivers and controls how the devices on the platform are used. This VM will run some other configuration and management tasks to further assist the hypervisor.

This model is adapted by Xen virtualization software and by the ESX hypervisor from VMware.

Regarding Xen, the VMs that run in the virtualized environment are referred to as domains. The privileged domain is referred to as **dom0** and the guest domains are referred to as **domUs**. Xen is the hypervisor and all of the device drivers run on dom0.

VMware used to have a control core based in Linux (similar to dom0 in Xen), but now all of the configuration is done via remote APIs.

## Virtualization Models: Hosted

The other type of virtualization model is the **hosted** (or **type 2**) model. In this model, there is a **full-fledged host OS that manages all of the hardware resources**. The host OS integrates a VMM, which is responsible for providing the VMs with their virtual platform interface.

The VMM module will invoke device drivers and other host components as needed.

One benefit of this model is that is can leverage all of the services and mechanisms that are already developed for the host OS. Much less functionality needs to be developed for the VMM module itself.

In this setup, **you can run guest VMs through the VMM module as well as native applications** directly on the host OS.

<img src="hosted.png">

One example of the hosted model is **kernel-based VM (KVM)** which is built into Linux. The Linux host provides all aspects of the physical hardware management and can run regular linux applications directly.

The support for running guest VMs come from a combination of the KVM VMM module and a **hardware emulator called QEMU**.

QEMU is used as a virtual interface between the VM and the physical hardware, and only intervenes during certain types of critical instructions, for example I/O management.

KVM has been able to leverage all of the advances of the open source Linux community. Because of this the KVM can quickly adopt new features and fixes. 

## Hardware Protection Levels

Commodity hardware has more than two protection levels. For example, x86 architecture has four **protection levels**, these levels are called **rings**.

**Ring 0** has the **highest privilege and can access all resources and execute all hardware supported instructions**. In a native model, the OS resides are ring 0.

In constrast, **Ring 3 has the least privilege**. This is where **applications reside**. When an application tries to perform some operation for which it does not have privilege, a trap will be caused and control will be switched to ring 0.

In virtualization setups, the hypervisor sits at ring 0, pushing the OS to ring 1. The applications remain at ring 3.

More recent x86 architectures introduce two different protection modes: root and non-root. Within each mode, the four rings exist.

When running in root mode, everthing is permitted. The hypervisor resides in ring 0 of the root mode. In contrast, in non-root mode, certain types of operations are not permitted. Guest VMs operate in non-root mode, with their applications in ring 3 of this mode, and their OS in ring 0.

Attempts by the guest OS to perform privileged operations cause traps called **VMExits**, which **trigger a switch to root mode, passing control to the hypervisor**. When the hypervisor completes its operation, it passes control back to the VM, by performing a **VMEntry**, which switches out of root mode.

## Processor Virtualization

**Guest instructions are executed directly by the hardware**. The VMM does not interfere with every instruction that is issued by the guest OS or its applications.

As long as the **guest OS is operating within the resources allocated to it by the hypervisor, the instructions will operate at hardware speeds**, which underlines the efficiency of virtualization!

Whenever a **privileged instruction is issued, the process causes a trap to the hypervisor**. At this point, t**he hypervisor can determine if the operation is to be allowed** or not. If the operation is illegal, the hypervisor can perform some punitive action, like shutting down the VM. If the operation should be allowed, the hypervisor must provide the necessary emulation to ensure that the guest OS receives the response it was expecting from the hardware. This is known as **trap-and-emulate**. The hypervisor intervention should be invisible to the guest OS.

## x86 Virtualization in the Past

Before 2005, x86 platforms had only the four privilege rings, without the root/non-root distinction. The basic strategy of virtualization software was to run the hypervisor in ring 0, and the guest OS in ring 1. 

However, there were exactly 17 hardware instructions that were privileged (required ring 0), but didn't cause a trap. Issuing them from another protection level wouldn't pass control to the hypervisor and they would fail silently.

For example, enabling/disabling interrupts requires manipulating a bit in a privileged register, which can be done with the POPF/PUSHF instructions. When these instructions were issued, they just failed silently.

Since control isn't passed to the hypervisor, the hypervisor has no idea that the OS wanted to change the interrupt status, so it cannot emulate that behavior.

At the same time, since the failure was silent, the OS doesn't know about it and assumes the change was successful. As a result, it continues with its execution.

## Binary Translation

One way to solve the issue of the 17 hardware instructions was to write the VM binary to **never issue those 17 instructions** :). This process is called **binary translation**.

The goal pursued by binary translation is to run **unmodified guest OS**. This is known as **full virtualization**.

To avoid the bad hardware instructions, interception and translation had to take place at the virtualization layer. Instruction sequences were caught by the VM binary to see if any of the 17 hardware instructions were present. If the code did not have any of these instructions, it was marked as safe and allowed to execute at hardware speeds. 

However, if one of the bad instructions was found, that instruction was translated into an alternative instruction sequence that emulated the desired behavior.

Binary translation adds overhead! Some mechanisms to reduce this overhead are caching translated code fragments and making sure to analyze kernel code executed in the guest OS.

## Paravirtualization

Another approach **gives up on unmodified guests, instead focusing on performance**. This approach is called **paravirtualization**.

In paravirtualization, the guest now knows that it is running in a virtualized environment on top of a hypervisor.

A paravirtualized guest OS may not directly try to perform operations it knows will fail, but will **instead make explicit calls to the hypervisor to achieve the desired behavior**. These calls are called **hypercalls** and they behave similar to system calls. The hypercall will trap to the hypervisor which upon performing the required operation with the data supplied by the guest, will pass control back to the guest.

This approach was taken by Xen.

## Memory Virtualization: Full Virtualization