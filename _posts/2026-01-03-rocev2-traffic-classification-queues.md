---
layout: post
title: "RoCEv2 on Ethernet (Part 2): Traffic Classification and Queue Design"
date: 2026-01-03
tags: [rocev2, rdma, qos, queues, datacenter, arista]
---

## Recap: where we are in the stack

In Part 1, we established a simple but critical idea:

> RoCEv2 only works when Ethernet is carefully controlled, layer by layer.

We identified four control layers:
1. Traffic identification
2. Loss prevention (PFC)
3. Early congestion signaling (ECN)
4. End-host rate control (DCQCN)

This article focuses entirely on **Layer 1**:
**Traffic classification and queue design**.

If you get this wrong, *everything above it breaks*.

---

## Why classification matters more than configuration

Before PFC, ECN, or DCQCN can function correctly, the network must answer one question with absolute certainty:

> “Is this packet RDMA traffic or not?”

If the answer is ambiguous:
- RDMA mixes with best-effort traffic
- Congestion behavior becomes unpredictable
- Lossless queues fill with the wrong packets

RoCEv2 is not tolerant of ambiguity.

---

## The goal of classification

The classification layer has one job:

> **Ensure RDMA traffic lands in a dedicated, lossless queue — and nothing else does.**

That means:
- RDMA traffic must be **positively identified**
- Non-RDMA traffic must be **excluded**
- The mapping must be consistent end-to-end

Classification is not about priority.  
It is about **isolation**.

---

## Common classification signals

In practice, RDMA traffic is identified using one or more of the following:

### Layer 3: DSCP
Most deployments mark RoCEv2 packets with a specific DSCP value.

Common choices:
- DSCP 26 (AF31)
- DSCP 48 (CS6)

The exact value matters less than **consistency**.

### Layer 2: 802.1p Priority
Switches ultimately make queuing decisions using internal traffic classes or priorities.

Typical mapping:
- RDMA traffic → priority 3 or 5
- Best-effort traffic → priority 0

DSCP is translated into internal traffic class, which is then mapped to a priority.

---

## A key design rule (do not violate this)

> **One lossless queue per switch.**

Not:
- One per application
- One per tenant
- One per VLAN

Exactly one.

Why?
- Multiple lossless queues dramatically increase deadlock risk
- Buffer accounting becomes unpredictable
- PFC interactions explode in complexity

If you think you need more than one lossless queue, the design problem is elsewhere.

---

## Queue design philosophy

Think of your switch queues like lanes on a highway.

- Best-effort traffic: many lanes, occasional congestion, acceptable loss
- RDMA traffic: a **single, protected lane** with strict rules

The lossless queue:
- Carries only RDMA traffic
- Has PFC enabled
- Has ECN enabled
- Has carefully sized buffers

All other traffic:
- Uses lossy queues
- Relies on TCP for recovery
- Must never trigger PFC

---

## What *not* to put in the lossless queue

This is where many designs fail.

Do **not** put the following into the RDMA queue:
- Storage traffic “just to be safe”
- Control plane traffic
- Management traffic
- “High priority” application traffic

Lossless does **not** mean “important”.

Lossless means:
> “This traffic cannot tolerate packet loss and has its own congestion control.”

Most traffic does not meet that bar.

---

## End-to-end consistency

Classification must be consistent across:
- NICs
- Top-of-Rack switches
- Spine switches
- Any intermediate hops

If one device:
- Fails to recognize the DSCP
- Maps it to the wrong traffic class
- Sends it to a lossy queue

Then PFC and ECN upstream become meaningless.

This is why RoCEv2 fabrics are typically **closed and homogeneous**.

---

## How Arista EOS fits this layer

Arista EOS provides:
- Explicit DSCP → traffic-class mapping
- Explicit traffic-class → priority mapping
- Clear visibility into queue counters
- Deterministic QoS pipelines

EOS does not “auto-detect” RDMA.
You must **tell it exactly what RDMA looks like**.

This is a feature, not a weakness.

---

## Validation mindset (even before configuration)

Before writing a single CLI line, you should be able to answer:

- Which DSCP value represents RDMA?
- Which traffic class does it map to?
- Which priority does that traffic class map to?
- Which queue is lossless?
- Can *any* other traffic reach that queue?

If any answer is unclear, stop and fix the design first.

---

## What comes next

Now that traffic is:
- Cleanly identified
- Isolated into a dedicated queue

We can safely talk about **PFC**.

In the next article:

**Part 3: PFC done safely**
- Why PFC is dangerous
- How deadlocks happen
- How to scope PFC correctly
- How Arista implements PFC per priority
- How to validate it under load

Only after that will we move into ECN and DCQCN.

---

## Final takeaway

RoCEv2 performance problems rarely start with ECN or DCQCN.

They almost always start here:
- Poor classification
- Mixed queues
- Lossless misuse

Get the queues right, and the rest of the system becomes manageable.

