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

<img src="rpc_steps.png">

1. In the first step a binding occurs, the client finds and discovers the server that supports the functionality it needs. For connection-based protocols (like TCP), the actual connection will be established in this step.
2. The client makes the actual RPC call. Control is passed to the stub, blocking the rest of the client.
3. The client stub then **creates a data buffer and populates it** with the arguments passed in. This process is called **marshaling**. The arguments may be located in noncontiguous memory locations within the address space. The transport level, however requires a contiguous buffer for transmission. This is why marshaling is needed. 
4. Once the buffer is available, the RPC runtime will send the message to the server via whatever transmission protocol the client and server have agreed upon during the binding process.
5. The data is then received by the RPC runtime on the server which performs all the necessary checks to determine which server stub the request needs to be delivered to.
6. The server stub will then unmarshal the data, extracting the necessary byte sequences from the buffer and populating the appropriate data structures. 
7. Once the arguments are allocated and set to appropriate values, the actual procedure call can be made. This calls the implementation of the procedure that is actually part of the server process. 
8. The server will compute the result of the operation, which will be passed to the server side stub and return to the client.

Note that before any of this happens, the server must do something to let the world know what procedures it supports, what arguments it requires and where it is.

## Interface Design Language

When using RPC, the client and the server don't need to be developed together. They can be written by different developers in different programming languages. 

For this to work, however, there must be some type of agreement so that the server can explicitly say what procedures it supports and what arguments it requires. 

This information is needed so that the client can determine which server it needs to bind with. 

Standardizing how this information is represented is also important because it allows the RPC runtime to incorporate some tools that allow it to automate the process of generating stub functionality.

To address these needs, RPC systems rely on the use of **interface definition languages (IDLs)**. The IDL servers as **a protocol for how the client-server agreement will be expressed**. 

## Specifying an IDL

An interface definition language is **used to describe the interface that a particular server exports**. At a minimum, this will include the name of the procedure, the types of different arguments, and the result type.

Another important piece of information to include is a version number. If there are multiples servers that provide the same procedure, the version number helps clients find which server is the most up to date. In addition, version numbers allow clients to identify the server which has the version of the procedure that fits with the rest of the client program.

An RPC system can use an IDL that is language-agnostic or language-specific.

For programmers that know the language, a language-specific IDL is great, for those that don;t, learning a language-agnostic IDL is simpler.

Whatever the choice of IDL, the IDL is used solely to define the interface. 

## Marshaling

Let's look at marshalling from the example before with the `add` server. Initially the arguments `i` and `j` live somewhere in the client address space. When the client calls the add function, it passes in `i` and `j` as discrete entities. 

At the lowest level, a socket will need to send a contiguous buffer of information over to the server. This buffer will need to hold a descriptor for the procedure (`add`), as well as the necessary arguments.

**This buffer gets generated in marshaling code**. 

<img src="marshaling.png">

Generally, the marshaling process encodes the data into an agreed upon format so that it can be correctly interpreted by the server. This encoding specifies the layout of the data upon serialization into a byte stream. 

## Unmarshaling

In the unmarshaling code, we take the buffer provided by the network protocol and based on the procedure descriptor and the type of arguments required by the procedure, we extract the correct chunks of bytes from the buffer and use those bytes to initialize data structure that correspond to the argument types.

As a result of the unmarshaling process, the `i` and `j` variables will be allocated in the server address space and will be initialized to values that correspond to whatever was placed into the buffer sent by the client. 

The marshaling and unmarshaling routines are typically not written by the programmer. RPC systems usually include a special compiler that takes an IDL specification and generates the marshaling/unmarshaling routines from that. The programmer just needs to ensure that they link these autogenerated files with their program files. 

<img src="unmarshaling.png">

## Binding and Registry

**Binding** is the mechanism used by the client to determine which server to connect to based on the service name and the version number. In addition, binding is used to determine how to connect to a particular server. This means discovering the IP address and or network protocol required for the client/server connection to be established.

To support binding, there needs to be some systems software that maintains a database of all available services. This is often called the **registry**.

A registry is the yellow pages for services, you pass the registry the name of the service and the version you are looking for, and you receive the contact information for the matching server. This information will include the IP address, the port and the transport protocol.

At one extreme, this registry can be a distributed online platform where any RPC can register.

On the other hand, the registry can be a dedicated process that runs on every server machine and only knows about the processes running on that machine. In this case, clients must know the machine address of the host running the service, and the registry need only provide the port number the service is running on.

A registry needs a naming protocol. The simplest approach is that it requires an exact name and version number. 

## Pointers in RPC

In regular local procedures, it makes sense to allow pointer arguments. The pointer references some address within the address space of both the calling procedure and the called procedure, so the pointer can be dereferenced without issue.

This won't work in RPC. To solve this problem, an RPC system can disallow pointers to be used as arguments to any RPC procedure. Another solution is allow pointers to be used but ensure that the referenced data is marshaled into the transmitted buffer. 

## Handling Partial Failures

When a client hangs while waiting on a remote procedure call, it is often difficult to pinpoint the problem. Is the server down? Is the service down? Is the network down? Did the message get lost?

Even if the RPC runtime incorporates some timeout/retry mechanisms, there are still no guarantees that the problem will be resolved or that the runtime will be able to provide some better insight into the problem.

RPC systems incorporate a special error notification that tries to capture what went wrong with a request without claiming to provide the exact detail. This serves as a catchall for all types of failures that can potentially happen during a call.

## What is SunRPC?

SunRPC is an RPC package originally developed by Sun in the 1980s for UNIX systems. It is now widely available on other platforms.

In SunRPC, it is assumed that the server machine is known up front and therefore the registry design choice is such that there is a registry per machine. When a client wants to talk to a particular service, it must first talk to the registry on that particular machine.

SunRPC makes no assumption regarding the programming language used by the client or the server. SunRPC relies on a language-agnostic IDL, XDR, which is used both for the specification of the interface and for the specification of the encoding of data types.

SunRPC allows the use of pointers and serializes the pointed-to data.

SunRPC supports mechanisms for dealing with errors. It includes a retry mechanism for re-contacting the server when a request times out. The RPC runtime tries to return meaningful error messages.

## SunRPC Overview

SunRPC allows the client to interact via a procedure call interface.

The server specifies the interface that it supports in a .x file written in XDR. SunRPC includes a compiler called **rpcgen** that will compile the interface specified in the .x file into language-specific stub for the client and the server.

On start, the server process registers itself with the registry daemon available on the local machine. The per-machine registry keeps track of the name of the service, the version, the protocol, and the port number. A client must explicitly contact the registry on the target machine in order to obtain the full contact information about the desired service.

When the binding happens, the client creates an RPC handle, which is used whenever the client makes any RPC calls. This allows the RPC runtime to track all of the RPC-related state on a per-client basis.

Let's look at an example. In this program, the client sends an integer x to the server and the server squares it.

<img src="sunrpc.png">

In the .x file the server specifies all of the datatypes that are needed for the arguments the results of the procedures that it supports.

In this case, the server supports one procedure `SQUARE_PROC` that has one argument of `square_in` and returns a result of the type `square_out`.

The datatypes of these arguments are defined in the .x file. They are both structs that have one member that is an int.

In addition to the data types, the .x file describes the actual RPC service and all of the procedures it supports.

First there is the RPC service, `SQUARE_PROG`, that will be used by the clients trying to find an appropriate service to bind with. 

A single RPC server can support one or more procedures. In our case, the `SQUARE_PROG` service supports one procedure `SQUARE_PROC`. There is an ID number associated with a procedure that is used internally by the RPC runtime to identify which particular procedure is being called.

In addition to the procedure ID, and the input and output data types, each procedure is also identified by a version. This version applies to all of the procedures for a service.

Finally, the .x file specifies a service ID. This ID is the number used by the RPC runtime to differentiate between services.

The client will use service name, procedure name and service numbers, whereas the RPC runtime will refer to the service id, procedure id and the version id.

## Compiling XDR

To generate the client/server C stubs defined in the .x file, run

```bash
rpcgen -C <interface>.x
```

The outcome of this operation generates several files:
* <interface>.h -  contains all of the language specific definitions.
* <interface>_svc.c - contains the server side stub
* <interface>_clnt.c - contains the client side stub
* <interface>_xdr.c - contains code for marshaling and unmarshaling routines

The main function in `svc.c` includes code for the registration step and also some additional housekeeping operations. The second part contains all of the code that is related to the particular RPC service. This code handles request parsing and argument marshaling.

In addition, the autogenerated code will include the prototype for the actual procedure that is invoked in the server process. This has to be implemented by the developer.

The client stub will include a procedure that is automatically generated and this will represent a wrapper for the actual RPC call that the client makes to the server-side process.

Once we have this, the developer can just call the function.

<img src="compiling_xdr.png">
<img src="compiling_xdr2.png">

## Encoding

Since the server can support multiple programs, versions, and procedures, it is not enough just to pass procedure arguments from client to server.

RPCs must also contain information about the service procedure id, version number and request id in the header of the request. This header will be included in the response from the server as well.

In addition to these metadata fields, we clearly need to put the actual data (arguments or results) onto the wire. These datatypes are encoded into a byte stream which depends on the data type.

There may be a 1-1 mapping between how the data is represented in memory and how it is represented on the wire, but this may not always be the case. What is important is that the data is encoded in a standard format that can be deserialized by both the client and server.

Finally, the packet of data needs to be preceded by the transport header (TCP/UDP) in order to actually be sent along the wire in accordance with these transmission protocols.

## Java RMI

Another popular RPC system is Java Remote Method Invocations (Javas RMI).

This system was also pioneered by Sun as a form of client/server communication methods among address spaces in the JVM.

Since Java is an object-oriented language, the entities interact via method invocations, not procedure calls. 

Since all the generate code is Java and the RMI system is meant for address spaces in the JVM, the IDL is language-specific.

The RMI runtime is separated into two components.

The **Remote Reference Layer** contains all the common code needed to provide reference semantics. For instance, it supports unicast or broadcast. In addition, it can specify return semantics.

The **Transport Layer** implements all of the transport protocol-related functionality.