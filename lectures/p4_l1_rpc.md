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

<img src="rpc_stack.png">

Let's start with an example. The server is a calculator. The client needs to send the operation it wants to perform as well as the data needed to perform that operation over to the server. The server contains the implementation of that operation, and will perform it on behalf of the client.

We can use **RPC to abstract away low-level communication details**. 

### Client

With RPC, the client is still allowed to call the function `add` with `i` and `j` even though the client doesn't hold the implementation for the `add` function.

In a regular program, when a procedure is called, the execution jumps to some other point in the address space where the implementation of that procedure is stored. In this example, the when the client calls `add`, the **execution will also jump to another location in the address space**, but it won't be where the implementations of `add` lives. Instead, it will be in a `stub` implementation. To the rest of the process, this stub will look like the real `add`.

The responsibility of the client stub is to create a buffer and populate the buffer with all of the appropriate information - the function name `add` and the arguments `i` and `j`. After the buffer is created, the RCP runtime sends the buffer to the server process. This may be via TCP/IP sockets or some other transport-level protocol.

The stub code itself is automatically generated via some tools that are part of the RPC package, that is, the programmer doesn't have to write the stub code. 

### Server

When the packets are received on the server, they are handed off to the **server stub**. This stub knows how to parse the received bytes, and it will be able to determine that this is an RPC request for `add` with arguments `i` and `j`.

Once this information is extracted, the stub is ready to make a call in the server process to the local `add` function with `i` and `j`. 

The server will then create a buffer for the result and send it back to the client via the appropriate connection. The packets will be received by the client stub, and the information will be stored in the client address space. 

Finally, the client function will return. The result of the call will be available and the client execution will proceed.

## Steps in RPC

Let's summarize the steps that have to take place in an RPC interaction between a client and server.


