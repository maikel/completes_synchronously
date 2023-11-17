---
title: "Proposal for Clarifying Semantics of `is_blocking()` and `completes_synchronously()` for Asynchronous Operations"
date: 2023-11-17
audiance: SG1
author:
  - name: "Maikel Nadolski"
    email: "<maikel.nadolski@gmail.com>"
  - name: "Lauri Vasama"
---

# Abstract

This paper aims to refine the semantics of `is_blocking()` and `completes_synchronously()` queries in the context of asynchronous operations involving connected senders and receivers. Building upon previous discussions and proposals, the paper delineates clear distinctions between these queries and their practical implications for implementation and use cases.

# Introduction:
The paper addresses the need for precise definitions of `is_blocking()` and `completes_synchronously()` queries to enhance understanding and usage in asynchronous operations within C++.

# Proposal:

`completes_synchronously()`:

    True: When completes_synchronously() is set to true, the connected receiver's completion-signal operation is guaranteed to occur before the return from execution::start(). This setting implies synchronous completion of the operation, enabling stack allocation of operation state or avoidance of certain synchronizations (e.g., in sync_wait()).

    False: Setting completes_synchronously() to false indicates that the completion-signal operation may occur after the return from execution::start(). This setting allows asynchronous completion, potentially requiring heap allocation of operation state or necessitating synchronization for sync_wait().

`is_blocking()`:

    True: When is_blocking() is set to true, it denotes that the forward progress of execution::start() may depend on external conditions beyond the thread's execution steps. This setting could encompass scenarios where start() might encounter blocking conditions, even if the completion does not happen synchronously.

    False: Setting is_blocking() to false signifies that execution::start() should not block, irrespective of external conditions beyond the thread's execution steps. This setting allows start() to proceed without waiting for external conditions, but it doesn't guarantee synchronous completion.

It's crucial to note that `completes_synchronously()` focuses solely on the timing of completion-signal operations, whereas `is_blocking()` pertains to the potential blocking nature impacting the forward progress of start() based on external conditions.

Conclusion:
Clear differentiation between `completes_synchronously()` and `is_blocking()` helps programmers understand the distinct scenarios and implications associated with each query. This clarity enables precise control and optimization in managing asynchronous operations within C++.