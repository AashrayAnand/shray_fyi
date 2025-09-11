+++
title = "Paper Review: The NIC should be part of the OS."
date = 2025-09-11
+++

[Aleksey Charapko's reading list archives](https://charap.co/) has a ton of interesting systems papers, so I decided to join his Fall 2025 reading group, with the following set of [papers to be discussed](https://charap.co/fall-2025-reading-list-201-210/). The reading group can be joined on [discord](https://discord.gg/VS7J4PAU58). The paper for discussion in this week's session is ["The NIC should be part of the OS."](https://sigops.org/s/conferences/hotos/2025/papers/hotos25-207.pdf).

I usually don't read a lot of papers that are this deep in the internals of OS and networking, but this was an interesting paper for me to learn about what existing optimizations exist in a traditional networking stack, and in the process be able to evaluate the author's proposal for a new design to consider for improving the stack.

After reading this, I feel like I definitely should spend more time branching out in the areas that I read. I think the inertia of learning in an area you're unfamiliar with will always be way higher than the incremental knowledge gain in areas of expertise, so its definitely both eye-opening (how little I know about some things) and also very satisfying (learning a little where you know nothing feels like a lot, vs. a little where you already know a lot).

## Things I had to look up while reading this

**cache-coherent interconnects** -> The hardware and protocol that synchronize caches lines and enforce memory consistency across CPU cores. E.g. when a cache line is modified on a core, and another core later reads it, the interconnect is what ensures the other core will get the most up-to-date value, even if the original core did not flush it. This is reserved for operations where synchronization is necessary, as this is obviously performance-impacting.

**protocol offload** -> Normally, when an application sends data over the network, it will go to the kernel, which will chunk the data into segments, adding TCP/IP headers and checksumming these segments, and write these to the NIC's transmit buffer. To receive data, the NIC receives packets and raises an interrupt, forcing the kernel to validate headers/checksum, and copy the payload to a socket buffer for application processing. Protocol offload optimizes this by having the NIC maintains protocol state (e.g. TPC connection state) and take on protocol processing previously handled by the kernel. Applications write data to dedicated NIC memory for protocol handling. This makes the NIC more complex (needs to be protocol aware), but reduces memory copying and CPU cycles. The kernel will typically be uninvolved in the data path (per-packet processing), but still handles the control path e.g. socket creation, security, memory mapping of sockets etc.

**RPC deserialization acceleration** -> Similar to protocol offload and especially powerful in data-center environments with heavy RPC volume. Assigns RPC data transformation to dedicated, programmed hardware, and relieving the CPU of this work. Unlike protocol offload, this does not take on the work of dispatch, endpoint mapping etc, which is still handled by the kernel.

**FPGA-homed Cache Lines** -> FPGAs are programmable hardware chips that are typically used for purpose-specific processing e.g. RPC deserialization, with the purpose of hardware acceleration for specific tasks. In the proposed design, the NIC uses a set of FPGA-homed cache lines, which associates dedicated cache lines for each core, with corresponding "home" (authoritative copy of the data for these cache lines) NIC FPGAs that are doing RPC deserailization.

## Paper Summary

This paper proposes a new design for interaction between user applications, the OS, and the NIC, where the NIC is intimately aware with the application protocol (e.g. RPC) and OS scheduling state, enabling it to handle protocol deserialization (stripping down packets to the minimal RPC application information), and identifying the corresponding process/thread that is awaiting the RPC, writing the deserialized information directly into the corresponding core's cache lines, or in the case when the process isn't scheduled, to the kernel, which handles process scheduling and delivering the payload.

My layman understanding of the paper is basically that, if we combine a few independent networking improvements:

1. Using FPGA for hardware acceleration of RPC processing.
2. Sharing OS scheduling state with the NIC.

The NIC can go on to determine what application requests correlate to which corresponding processes/endpoints, and resultingly, which cores these processes correspond to, as well as be able to deserialize the underlyling request, and provide only the minimally stripped down data needed by the application to process the request. In the case of RPC, this amounts to just "a code
pointer and data pointer inside that process corresponding to
the request, and the call arguments".

"Each endpoint comprises a set of cache lines homed on the NIC: two control lines plus multiple auxiliary lines to handle payloads larger than a single cache line (128 B on Enzian). The transmit path uses a similar, disjoint set of cache lines...To receive a request, a process issues a load to one control
cache line to receive a request...The NIC will respond to this
load with the data listed above when an appropriate packet
arrives and has been decoded; until then the core is stalled. Of course, Lauberhorn cannot block a cache fill from a core indefinitely...We avoid this by returning TryAgain dummy messages after 15ms"

A mapping is created between RPC endpoints and a set of cache lines, with these cache lines used by processes to ask to receive RPC requests, and for the NIC FPGA to provide back the corresponding request payload, with a timeout of 15ms to avoid blocking indefinitely, but also avoiding having to spin in the meanwhile.

One point that Owen Hilyard, who mediated today's discussion, made was that this method of stalling cores on a cache line load, and blocking for the NIC to write back the instruction pointer to the cache line for subsequent execution, is inherently a synchronous model.

## How does this differ vs. existing technologies?

"Current approaches to server stacks navigate a trade-off between flexibility, efficiency, and performance: the fastest kernel-bypass approaches dedicate cores to applications, busy-wait on receive queues, etc. while more flexible approaches appropriate to more dynamic workload mixes incur much greater software overhead on the data path."

Above is how the author describes the shortcomings of existing optimizations in the network stack, as well as mentioning the following as a focus for the proposal:

"Our focus in this paper is on the network receive path, although it is also closely connected to the transmit path...We also focus on Remote Procedure Calls (RPCs), whether data center microservices or serverless function invocations. While some are large, the great majority of RPC requests and
responses are small"

As I understand this proposal, it is not really suggesting an idea that can be generalized to the pushdown of all application protocols into the NIC, but specifically, protocols like RPC, where payload size (for requests and responses) is fairly small.

This makes sense inherently given the proposal, since there are a dedicated set of cache lines for each core, that are associated with corresponding NIC FPGAs for request processing. If requests exceed the total size of the corresponding cache lines, then the request processing can no longer be offloaded to the NIC, and is forwarded to the kernel for processing.

## What are other Interesting Takeaways?

"Secondly, it’s time to trust the NIC. The NIC is a critical
part of the OS function of the machine...viewing it
as a potential part of the OS...is the only way to fully exploit its hardware resources and unique position in the data path."

Whether this proposal enables universaly better performance than other network optimizations, this is definitely a unique mentality compared generally to the view of the NIC as a dumb device, that should be insulated from the OS and just move data around.

"However, when the workload is dynamic with many more end-points than spare cores, the up-front cost of mapping the NIC’s demultiplexing to queues onto the scheduling of applications on cores quickly becomes cumbersome"

This is I think an important call out about where exactly this proposal would work effectively. Given that each RPC endpoint is allocated a fixed set of cache lines, there is a scaling bottleneck for this design in the face of systems with significant connections counts, whereas it would make more sense to devise this optimization in a system with a smaller or fixed number of endpoints, which are each receiving a high request/response volume.

Some quick math (where we assume 2 control lines and 4 auxiliary lines are allocated to each endpoint) would show that a 64KB L1 cache can (at most) support only (64 KB / (6 * 128b)) ~= 85 endpoints before being exhausted, and given we cannot allocate all cache lines to this network optimization, much less in practice.

As far as I can tell, this design also doesn't really acknowledge buffering in any specific fashion. It is optimized to handle fast-path RPCs, where requests and responses are small enough to fit in the allocated cache lines, and for larger requests that exceed these cache lines, or if the inbound request volume is saturing the NIC, and it already is processing a request for an endpoint, subsequent requests would have to be offloaded to the kernel (slow) path.

From a security side, I also assume this can really only be leveraged for servers which do not receive public traffic, since we are expecting the NIC to be able to simply demux and route parsed packets, which if malicious, could theoretically result in corrupting memory, misdirecting RPCs and other issues.

All these takeaways line up with the fact that this paper is funded by a grant from Google systems research, and I assume are driven by delivering a bespoke improvement provided a particular set of caveats:

1. Traffic into the system can be trusted e.g. internal microservices in a data center.
2. Small payloads, where requests and responses can be processed within a fixed set of cache lines.
3. The number of connections does not exceed the number of cores on the system, or more specifically, the total cache lines across the system allocated for NIC RPC processing and dispatch, over the number of cache lines allocated to each connection.
