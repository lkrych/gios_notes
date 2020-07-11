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

