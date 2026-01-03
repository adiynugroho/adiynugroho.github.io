---
layout: post
title: "RoCEv2 on Ethernet: A Layer-by-Layer Mental Model"
date: 2026-01-02
tags: [rocev2, rdma, datacenter, qos, arista]
---

## Why RoCEv2 exists

Modern AI, HPC, and large-scale distributed systems are no longer limited by compute.  
They are limited by **data movement**.

RDMA (Remote Direct Memory Access) exists to remove:
- Kernel overhead
- CPU context switching
- TCP retransmission latency

RoCEv2 brings RDMA to **Ethernet**, which gives us:
- Routing (L3 scalability)
- Cost efficiency
- Vendor interoperability

But Ethernet was never designed to be lossless by default.  
That gap is what the rest of this article explains.

---

## The core problem: Ethernet drops packets

Ethernet assumes:
- Packet loss is acceptable
- Upper layers will recover

RDMA assumes:
- Packet loss is catastrophic
- Recovery is extremely expensive

If you run RoCEv2 on a normal Ethernet fabric:
- Throughput collapses
- Latency spikes
- NICs enter retransmission storms

So the entire design goal becomes:

> **Make Ethernet behave predictably enough for RDMA**

---

## The four control layers of RoCEv2

Think of RoCEv2 not as a single feature, but as **four stacked control layers**.

### Layer 1: Traffic identification
Before anything else, the network must **recognize RDMA traffic**.

This is done using:
- DSCP values
- Traffic Classes
- 802.1p priorities

If you misclassify traffic:
- Everything above this layer fails

This is the foundation.

---

### Layer 2: Loss prevention (PFC)

Priority Flow Control (PFC) exists to answer one question:

> What happens when a queue is about to overflow?

For RDMA traffic, the answer must be:
- **Pause the sender**
- **Do not drop packets**

PFC is:
- Extremely powerful
- Extremely dangerous if overused

Correct principle:
- PFC only for RDMA traffic
- Never for general traffic

PFC is a **last-resort safety mechanism**, not a congestion control system.

---

### Layer 3: Early congestion signaling (ECN)

Pausing traffic is not enough.  
You must **avoid reaching the pause condition in the first place**.

Explicit Congestion Notification (ECN):
- Marks packets before queues overflow
- Signals congestion without dropping packets

ECN answers:
> “You are going too fast — slow down *now*, not later.”

Without ECN:
- PFC storms happen
- Entire fabrics can pause

ECN is what keeps PFC rare.

---

### Layer 4: End-host rate control (DCQCN)

DCQCN is the **brain** of the system.

It lives primarily on the NIC, not the switch.

DCQCN:
- Consumes ECN signals
- Adjusts RDMA send rates dynamically
- Stabilizes throughput across thousands of flows

If ECN is the signal,
DCQCN is the response.

No DCQCN = oscillation, unfairness, instability.

---

## Why all four layers are mandatory

A common mistake is enabling only one or two layers.

| Missing Layer | Result |
|---------------|--------|
| No classification | RDMA mixed with best-effort traffic |
| No PFC | Packet loss, RDMA collapse |
| No ECN | PFC storms |
| No DCQCN | Unstable throughput |

RoCEv2 works **only** when all layers cooperate.

---

## What switches do vs what hosts do

### Switch responsibilities
- Traffic classification
- Queue isolation
- PFC enforcement
- ECN marking
- Buffer management

### Host responsibilities
- DCQCN algorithm
- Rate adjustment
- Queue pair management

A switch alone cannot “fix” RoCE.
A host alone cannot “fix” congestion.

This is a **distributed control system**.

---

## Where Arista EOS fits

Arista EOS excels at:
- Clear QoS pipelines
- Predictable buffer behavior
- Fine-grained ECN control
- Operational visibility

EOS does **not** magically make RoCE work.
It gives you the **tools to engineer it correctly**.

Configuration comes later.
Understanding comes first.

---

## What’s next in this series

In the next articles, we will go one layer at a time:

**Part 2:** Traffic classification and queue design  
**Part 3:** PFC done safely (and how to avoid deadlocks)  
**Part 4:** ECN, DCQCN, and tuning for AI fabrics  

Each part will move from:
- Concepts → design rules → Arista EOS configuration → validation

---

## Final thought

RoCEv2 is not fragile.
**Misunderstood RoCEv2 is fragile.**

Once you understand the layers, the configs stop being scary —  
they become obvious.

