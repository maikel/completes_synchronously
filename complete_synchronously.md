---
title: "Proposal for Clarifying Semantics of `is_blocking()` and `completes_synchronously()` for Asynchronous Operations"
date: 2023-11-17
audience: "SG-1"
author:
  - name: "Maikel Nadolski"
    email: "<maikel.nadolski@gmail.com>"
  - name: "Lauri Vasama"
---

# Abstract

This paper aims to introduce the semantics of `is_blocking()` and `completes_synchronously()` queries in the context of asynchronous operations involving senders and receivers from the current `std::execution` proposal P2300. Building upon previous discussions and proposals, the paper delineates clear distinctions between these two queries and their practical implications for implementation and use cases.

# Proposal:

Let `sender` and `receiver` be connected, let `env` be the receivers environment and `op` an operation that is initiated by `execution::start(op)`. The following definitions apply:

`completes_synchronously()`:

True: When `completes_synchronously(sender, env)` is `true`, the connected receiver's completion-signal is guaranteed to occur before the return from `execution::start()`. This setting implies synchronous completion of the operation, enabling stack allocation of operation state or avoidance of certain synchronizations (e.g., in `sync_wait()`).

False: Returning `false` from `completes_synchronously(sender, env)` indicates that the completion-signal operation may occur after the return from `execution::start()`. This setting allows asynchronous completion, potentially requiring heap allocation of operation state or necessitating synchronization for `sync_wait()`.

Invalid Expression: Returning an invalid expression from `completes_synchronously(sender, env)` indicates that the completion-signal operation may occur after the return from `execution::start()`.

`is_blocking()`:

True: When `is_blocking(sender, env)` returns `true`, it denotes that the forward progress of `execution::start(op)` may depend on external conditions beyond the thread's execution steps. This setting could encompass scenarios where `start()` might encounter blocking conditions, even if the completion does not happen synchronously.

False: `is_blocking(sender, env) = false` signifies that `execution::start(op)` should not block, irrespective of external conditions beyond the thread's execution steps. This setting allows `start()` to proceed without waiting for external conditions, but it doesn't guarantee synchronous completion.

It's crucial to note that `completes_synchronously()` focuses solely on the timing of completion-signal operations, whereas `is_blocking()` pertains to the potential blocking nature impacting the forward progress of `start()` based on external conditions.

# Exploring the Role of `completes_synchronously(sender, env)` in Asynchronous Operations

The `completes_synchronously(sender, env)` query serves as a crucial determinant in understanding the behavior of asynchronous operations within different contexts. Its role extends across various functionalities and implementations.

1. Within the `sync_wait` algorithm: One possible implementation of `sync_wait(sender)` is to synchronize the completion of the async operation with an internal condition variable. On the other hand, if `completes_synchronously(sender, env)` is `true`, the operation can be completed synchronously without the need for an extra thread synchronization. This optimization is possible because the completion-signal is guaranteed to occur before the return from `execution::start()`.

2. Imagine a `repeat(sender)` algorithm that repeats the asynchronous operation indefinitely until being stopped. 
If the resulting operation completes synchronously, the usual implementation results in a recursive call to `execuction::start()` within the completion signal `set_value()` of an intermediate receiver. Usually, some kind of prevention needs to be implemented to avoid recursive call stacks and stack overflow, e.g. using a trampoline scheduler.
One possible solution is to use a `completes_synchronously(sender, env)` query to determine whether the operation is completed synchronously. If `completes_synchronously(sender, env)` is `true`, the operation is completed synchronously, and the algorithm can be implemented such that the recursion is avoided.

3. Assume a coroutine task `execution::task<T>` that ensures that each awaited expression completes on the current scheduler. If the awaited expression completes synchronously, the task can be implemented such that the coroutine is resumed immediately. If the awaited expression completes asynchronously, the task can be implemented such that the coroutine is resumed on the current scheduler when the awaited expression completes.

# Exploring the Role of `is_blocking(sender, env)` in Asynchronous Operations

1. Managing Thread resources: A thread pool might have extra resource for blocking operations. In this case, `is_blocking(sender, env)` can be used to determine whether the operation is blocking or not. If `is_blocking(sender, env)` is `true`, the operation can be executed on a thread that is allowed to be blocked. If `is_blocking(sender, env)` is `false`, the operation can be executed on a thread that is not allowed to block.

# Conclusion:

Clear differentiation between `completes_synchronously()` and `is_blocking()` helps programmers understand the distinct scenarios and implications associated with each query. This clarity enables precise control and optimization in managing the lifetime of asynchronous operations within C++.
