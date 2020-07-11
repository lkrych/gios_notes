# Virtualization

## Table of Contents

# Introduction

Let's look at two applications. In the first application, the client gets a file from the server. In the second application the client sends an image to the server for some processing/modification.

<img src="why_rpc.png">

Notice that the setup for both applications is similar. The only difference is the request being made from the client to the server, and the data being transmitted.

Almost **all programs that implement some kind of basic client/server functionality will require these very similar setup steps**, and thus the need came to simplify these steps. This gave rise to RPC.

## Benefits of RPC

**RPC** is intended to **simplify the development of interactions across address spaces and machines**. 

One benefit of RPC is that it **offers a high-level interface for data movement and communication**. This means the developer can focus more on what their application does as opposed to the standard setup.

RPC also handles a lot of the errors that may arise form low-level communication/transmission interactions, freeing the developer from having to explicitly re-implement error handling in each program.

Finally, RPC hides complexities of cross-machine interactions.

## RPC Requirements

The model of IPC interactions that the RPC model is intended for needs to match client/server interactions. The server performs some complex tasks, and the client needs to know how to communicate with that server.

Since most of the languages at the time were procedural languages (hence the name remote procedure call), there was an expectation that this model followed certain semantics. As a result, **RPCs have synchronous call semantics**. When a process makes a remote procedure call, **the calling process will block until the procedure completes** and returns the result. This is the exact same thing that happens when we make a local procedure call.

RPCs also have type checking. If you pass an argument of the wrong type to an RPC, you will receive some kind of error. **Type checking affords us opportunities to optimize the implementation of the RPC runtime**. When packets are being sent between two machines, they are just a stream of bytes. Being able to transmit some information about they types can be useful when the RPC runtime is trying to interpret the bytes. 

Since the client and the server may run on different machines, there may be differences in how they represent certain data types. For instance, a machine may differ in their endianness, their representation of floating point numbers, negative numbers, etc. The RPC system should hide these differences from the programmer.

One way to deal with the conversion is for the **RPC runtime, and both endpoints to agree upon a single data representation**, for example network format of integer types. With this agreement, there is no need for endpoints to negotiate on how data should be encoded. 

Finally, **RPC is meant to be more than a transport-level protocol**. RPC should support different types of protocols, whether UDP, TCP or others. RPC should also incorporate some higher-level mechanisms like **access control, authentication and fault-tolerance**. 

## Structure of RPC