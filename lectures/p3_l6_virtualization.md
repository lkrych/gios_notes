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

## Virtualization Models - Bare Metal

In **bare-metal** virtualization (also known as **hypervisor-based** or **type 1 virtualization**), the VMM manages all of the hardware resource and supports execution of the VMs.

<img src="bare_metal.png">

One issues with this model concerns devices. According to the model, the hypervisor must manage all possible devices. In other words, device manufacturers have to provide device drivers not just for the different OS's, but also for different hypervisors. 

To eliminate this problem, the hypervisor model typically integrates a special virtual machine, a **service VM**, that runs a **standardized OS with full hardware access** privileges, allowing it to manipulate hardware as if it was native. 

The privileged VM runs all of the device drivers and controls how the devices on the platform are used. This VM will run some other configuration and management tasks to further assist the hypervisor.

This model is adapted by Xen virtualization software and by the ESX hypervisor from VMware.

Regarding Xen, the VMs that run in the virtualized environment are referred to as domains. The privileged domain is referred to as **dom0** and the guest domains are referred to as **domUs**. Xen is the hypervisor and all of the device drivers run on dom0.

VMware used to have a control core based in Linux (similar to dom0 in Xen), but now all of the configuration is done via remote APIs.

## Virtualization Models: Hosted