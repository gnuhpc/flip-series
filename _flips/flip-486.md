---
layout: page
title: "FLIP-486: DeltaJoin"
nav_order: 486
parent: FLIPs
permalink: /flips/flip-486/
description: "A New Solution for Join State Optimization in Stream Processing"
---

## Introduction

Imagine this scenario: an e-commerce platform needs to process order data in real-time, linking order creation information with payment success information to promptly update order status, shipping, and subsequent processing. This process generates two data streams: the order creation stream and the payment success stream. As time progresses, unmatched orders and payment data accumulate, causing the Join operation's state space to grow increasingly large, like an ever-expanding balloon that might eventually burst the entire system. This FLIP proposes a new solution called DeltaJoin to address this issue.

## Why DeltaJoin?

In Flink's stream processing tasks, Join node state has always been a persistent challenge:

1. State grows continuously over time, like a snowball rolling larger and larger
2. Large states cause tasks to slow down, like running with an increasingly heavy backpack
3. Checkpoint saving and recovery times increase, affecting task stability

While setting TTL (Time To Live) can help clean up expired data, this method isn't suitable for all scenarios and doesn't address the root cause.

## What is DeltaJoin?

DeltaJoin's core concept is to use bidirectional lookup to reuse source table data instead of storing Join states. This might sound abstract, so let's illustrate with diagrams:

```mermaid
graph TB
    subgraph "Traditional Join Solution"
        A1[Order Stream<br>e.g.: createOrder] --> C1[Join Operator]
        B1[Payment Stream<br>e.g.: paySuccess] --> C1
        C1 --> D1[Complete Order]
        
        E1[State Storage] -.->|"Must store all<br>unmatched orders"| C1
        F1[State Storage] -.->|"Must store all<br>unmatched payments"| C1
        G1[Disk] -.->|"Large checkpoints<br>slow recovery"| E1
        G1 -.->|"Large checkpoints<br>slow recovery"| F1
        
        style E1 fill:#f77,stroke:#333
        style F1 fill:#f77,stroke:#333
        style G1 fill:#faa,stroke:#333
    end
```

```mermaid
graph TB
    subgraph "DeltaJoin Solution"
        A2[Order Stream<br>e.g.: createOrder] --> C2[DeltaJoin Operator]
        B2[Payment Stream<br>e.g.: paySuccess] --> C2
        C2 --> D2[Complete Order]
        
        E2[Source Table Query] -->|"Direct record<br>lookup"| C2
        F2[Local Cache] -.->|"Hot data cache<br>size controllable"| C2
        
        style E2 fill:#7f7,stroke:#333
        style F2 fill:#7fa,stroke:#333
    end
```

## How DeltaJoin Works

Let's illustrate DeltaJoin's operation through a sequence diagram:

```mermaid
sequenceDiagram
    participant O as Order Creation Stream
    participant D as DeltaJoin
    participant P as Payment Records Table
    participant R as Result Stream

    O->>D: New Order Created
    D->>P: Look up Payment Record
    P-->>D: Return Matching Payment Info
    D->>R: Output Complete Order
    Note over D: Uses Cache to Optimize Lookup Performance
```

DeltaJoin's workflow includes these key steps:

1. **Data Arrival**: When a new order is created, DeltaJoin receives the order creation event.
2. **Match Finding**: DeltaJoin looks up corresponding payment information in the payment records table based on join conditions.
3. **Cache Optimization**: DeltaJoin caches frequently used payment records for improved efficiency.
4. **Result Output**: Once a matching payment record is found, DeltaJoin associates the order creation info with payment info and outputs a complete order record.

## Configuration Options

DeltaJoin provides several configuration options:

| Option | Values | Default | Description |
|--------|---------|---------|-------------|
| table.optimizer.delta-join.strategy | AUTO/FORCE/NONE | AUTO | Whether to enable DeltaJoin |
| table.exec.delta-join.cache-enable | true/false | true | Whether to enable caching |
| table.exec.delta-join.left-cache-size | numeric | 10000 | Left table cache size |
| table.exec.delta-join.right-cache-size | numeric | 10000 | Right table cache size |

The strategy options mean:
- AUTO: Optimizer tries DeltaJoin first, falls back to regular Join if conditions aren't met
- FORCE: Forces DeltaJoin use, throws error if conditions aren't met
- NONE: Doesn't use DeltaJoin

## Current Progress

This FLIP is currently in the discussion phase, with work focused on:
1. Finalizing implementation details
2. Developing performance benchmarks
3. Adding more test cases

## Summary

```mermaid
mindmap
  root((DeltaJoin))
    Core Concept
      Bidirectional Lookup
      Source Table Data Reuse
      State Storage Replacement
    Benefits
      Reduced State Size
      Improved Checkpoint Efficiency
      Faster Recovery
    Limitations
      Wide Table Build Only
      Stateless Operator Restriction
    Configuration
      Strategy Selection
      Cache Control
      Size Limits
```

DeltaJoin's design philosophy is practical - rather than storing large amounts of state data during joins, it makes efficient use of source table data itself. This approach not only reduces resource consumption but also improves system reliability. While there are currently some limitations and the architecture requires a storage engine capable of both stream reading and point queries, this direction shows great promise. We look forward to seeing more improvements and breakthroughs in future versions.
