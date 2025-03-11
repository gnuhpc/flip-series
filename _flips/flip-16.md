---
layout: page
title: "FLIP-16: Reliable Iterative Stream Processing"
nav_order: 16
parent: FLIPs
permalink: /flips/flip-16/
description: "State Recovery Solution for Iterative Jobs in Flink"
---

## Introduction

In some data processing scenarios, data needs to be processed multiple times to achieve the final result. For example, in machine learning training, the same batch of data needs repeated iterations to obtain the optimal model; in graph computation, multiple rounds of propagation are needed to get the final result. This cyclic processing pattern is also common in Flink's stream processing. However, if system failure occurs during iteration, how do we ensure data isn't lost or duplicated? This is the problem FLIP-16 attempts to solve.

## What's the Challenge?

### The Special Nature of Iterative Processing

Let's understand this problem with a simple analogy. Imagine a factory production line. Ordinary assembly lines are linear: raw materials enter from one end and emerge as finished products from the other end. But some special products might need repeated processing at certain stages, forming a circular production line.

```mermaid
graph LR
    A[Raw Material] --> B[Station 1]
    B --> C[Station 2]
    C --> D[Inspection]
    D -->|Failed| B
    D -->|Passed| E[Finished Product]
    
    style D fill:#ffb6c1
    style B fill:#b0e0e6
```

In this circular production line, if the system needs to create a backup (called checkpoints in Flink), it faces a tricky problem: how to handle products currently in the loop?

### Limitations of Existing Solutions

In regular stream processing, Flink uses a method called "asynchronous barrier snapshots" to create checkpoints. It's like placing a marker on the assembly line and only backing up after all products before the marker are processed. But this method faces challenges in iterative processes:

```mermaid
graph TD
    subgraph "Regular Stream Processing"
    A1[Data] --> B1[Operator 1]
    B1 --> C1[Operator 2]
    C1 --> D1[Operator 3]
    end
    
    subgraph "Iterative Stream Processing"
    A2[Data] --> B2[Loop Head]
    B2 --> C2[Process Operator]
    C2 --> D2[Loop Tail]
    D2 -->|Continue Loop| B2
    end
    
    style B2 fill:#ffb6c1
    style D2 fill:#b0e0e6
```

Waiting for all data in the loop to finish processing before creating a checkpoint could take a very long time, possibly forever. It's like waiting for a never-ending game to finish.

## FLIP-16's Solution Approach

FLIP-16 proposed a clever solution. The core idea is: rather than waiting for all iterative data to complete processing, temporarily store it instead. Specifically:

1. When checkpoint begins, loop head enters "logging mode"
2. In logging mode, data returning from loop tail is recorded instead of being processed directly
3. After checkpoint completes, recorded data is reprocessed

This process can be shown in the following sequence diagram:

```mermaid
sequenceDiagram
    participant R as Runtime
    participant H as Loop Head
    participant T as Loop Tail
    
    R->>H: Initiate Checkpoint
    Note over H: Enter Logging Mode
    H->>T: Pass Checkpoint Barrier
    T->>H: Return Data R1
    Note over H: Record R1
    T->>H: Return Data R2
    Note over H: Record R2
    T->>H: Return Checkpoint Barrier
    Note over H: Complete Checkpoint<br/>Exit Logging Mode
    H->>H: Process R1, R2
```

## Why Was This Proposal Abandoned?

Although this solution looked promising, it was ultimately abandoned for several reasons:

1. **Memory Consumption**: In logging mode, all returning loop data needs to be stored in memory. Large volumes of loop data could lead to memory exhaustion.

2. **Implementation Complexity**: Adding this special handling to the existing checkpoint mechanism would require modifying many core components, potentially affecting system stability.

3. **Runtime Overhead**: Frequent entering and exiting of logging mode, plus data recording and replay, would introduce significant performance overhead.

## Lessons from FLIP-16

Although this FLIP wasn't adopted, its exploration process provides valuable insights:

It highlights that data consistency in iterative stream processing remains a challenging technical problem.

The exploration process demonstrates the complexity of distributed system design - an apparently elegant solution may face unexpected challenges in practical application.

The case also shows that system design must consider not only functional correctness but also implementation feasibility and operational efficiency.

For scenarios requiring iterations in stream processing, current recommendations are:
- Try to convert iterative logic into batch-processable form
- Handle iteration interruption and recovery logic at the application level
- Or use other frameworks more suitable for iterative computation

## Summary

FLIP-16 attempted to solve a challenging problem in distributed stream processing: ensuring reliability in iterative data processing. Although its proposed solution was ultimately abandoned due to implementation difficulties and performance issues, the exploration process provided valuable reference for future research.
