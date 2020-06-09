# Graduate Introduction to Operating Systems - Summer 2020

### Table of Contents

* [Beyond Multiprocessing - Eykholt](#beyond-multiprocessing)

#### Beyond Multiprocessing (1992) - Eykholt et al

The SunOS/SVR4 kernel was re-structured around threads instead of adding locks and keeping the user process model unchanged. 

Goals:

1. The threads must be capable of executing system calls and handling page faults independently. 
2. The kernel should have a bounded dispatch latency for real-time threads. Real-time response requires control over scheduling, requiring preemption at almost any point in the kernel.

In the SunOS 5.0 kernel, the **kernel thread is the fundamental entity** that is scheduled and dispatched onto one of the CPUs of the system.

 A kernel thread is **very lightweight**, having only a small data structure and a stack. Switching between kernel threads **does not require a change of virtual memory address space** information. They **use synchronization primitives** like locks as well as service priorities to prevent priority inversion.

