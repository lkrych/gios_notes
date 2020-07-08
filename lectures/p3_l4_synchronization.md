# Synchronization Concepts

## Table of Contents

## Introduction

Mutexes by themselves can only represent simple binary states. We cannot express arbitrary synchronization conditions using the constructs we have seen already.

Synchronization constructs require low-level support from the hardware in order to guarantee correctness of the construct. Hardware provides this type of low-level support via **atomic instructions**.

The discussion in these notes will cover alternative synchronization constructs that can help solve some of the issues we experienced with mutexes and condition variables as well as examine how different types of hardware achieve efficient implementations of the synchronization constructs.

### Spinlocks

One of the most basic synchronization constructs is the **spinlock**.

Spinlocks provide mutual exclusion, and it has some constructs that are similar to mutex lock/unlock operations. However they are different from mutexes in an important way.