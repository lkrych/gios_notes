# Datacenter Technologies

## Table of Contents

## Internet Services

An internet service is any type of service that is accessible via a web interface. Users communicate with these services via web requests that are issued by their web browsers.

Most commonly, these services are divided into three tiers: 
1. The **presentation tier** which is typically responsible for static content related to webpage layout.
2. The **business logic tier** which integrates all of the business specific processing, including dynamic user-specific content.
3. The **database tier** deals with all the data storage and management.

These different tiers do not need to run as separate processes on separate machines. For example, the Apache HTTP web server can fulfill the presentation and business logic tier.

Many middleware components exist to connect and coordinate these tiers.

## Internet Service Architectures

For services that need to deal with high or variable request rates, choosing a configuration that requires multiple processes - potentially on multiple nodes - becomes necessary.

We can **scale out** a service deployment by running the service on more nodes.

A front-tend load-balancing component would route the incoming request to the appropriate machine that implements the service.

Behind the load balancer, we can have two different setups. In a **homogenous setup**, **all nodes are able to execute any possible step in the request processing pipeline**, for any request that can come in. In a **heterogenous setup**, n**odes execute some specific steps** in the request processing.

## Homogenous Architectures

Any node in a functionally homogenous setup can process any type of request and can perform any of the processing in the end-to-end request processing.

The benefit of this is that the front-end (load balancer) can be kept very simple. It doesn't have to keep track of which node can service which type of request. Instead it can assign requests in a round-robin manner.

This design doesn't mean that every node has all of the data. Instead, data may somehow be replicated or distributed across the nodes. Importantly, every node is able to get to any type of information that is necessary for the execution of the service.

One downside of this approach is that there is little opportunity to benefit from caching. A simple front-end will not keep enough to understand the locality of each task on a node by node basis.

## Heterogenous Architectures

In a heterogenous architecture, different nodes are designated to perform certain functions or handle certain types of requests.

For instance, requests may be divided among the servers based on request content. In the case of eBay, servers may be specialized for browsing requests vs bidding/buying requests.

**Since different nodes perform different tasks, it may not be necessary to ensure that data is uniformly accessible everywhere**. Certain types of data may be much more easily accessible from the servers that manipulate those data types.

One benefit of this approach is that the system can benefit from caching and locality, as each node is specialized to repeatedly perform one or a few actions.

One downside of this approach is that the front-end needs to be more complex. The front-end now needs to perform some request parsing to determine to which node it should route the request.

In addition, the overall management of this system becomes more complex. When a machine fails, or requests increases, it is important to understand which types of servers need to be scaled out.

Also, if something changes fundamentally in the workload pattern, the server layout needs to be reconfigured. It's not as easy as scaling up or down.

## Cloud Computing

Traditionally, business would need to buy and configure the resources that were needed for their services. The number of resources to buy/configure would be based on the peak expected demand for those services.

If the demand exceeded the expected capacity, the business would into a situation where requests have to be dropped, which would result in lost opportunity.

Ideally, we would like the capacity of the available resources to scale elastically with the demand.

The scaling should be instantaneous: as soon as the demand increases, so does the capacity, and vice versa. This means that the cost to support this service should be directly proportional to the demand.

Cloud computing provides shared resources. These resources can be infrastructure resources; that is, compute, storage or networking resources. Cloud computing can also provide higher-level software resources, like email or database services.

These infrastructural/software resources are made available via some APIs for access and configuration. These resources need to be accessed and manipulated as necessary over the Internet. APIs can be web-based, library-based, or command line-based.

Providers offer different types of billing and accounting services. Some marketplaces incorporate spot pricing, reservation pricing, or other pricing models.

Billing is often not done by raw usage, as overheads with monitoring at that level of fine-grained control are pretty high. Instead, billing is done based on some discrete step function. For example, compute resources may be billed according to "size": tiny, medium, extra-large.

All of this is managed by the cloud provider. Common software stacks more managing cloud resources include the open source OpenStack and VMware's vSphere software stack.

## Cloud Deployment Models

In **public clouds**, the infrastructure belongs to the cloud provider, and third party customers/tenants can rent the hardware to perform their computational tasks.

In **private clouds**, the infrastructure and software is owned by the same entity. Cloud computing technology is leveraged to maintain some of the flexibility/elasticity on machines that are in-house.

**Hybrid clouds** combine public and private clouds. Private clouds may comprise the main compute resources for the applications, with failover/spikes being handled on public cloud resources. In addition, public clouds may be used for auxiliary tasks, such as load testing.

Finally, **community clouds** are public clouds that are used by certain types of users.

