---
title: "Introducing Sender Queries `completes_synchronously()` and `get_completion_concurrency()`"
date: 2024-01-11
audience: ""
author:
  - name: "Maikel Nadolski"
    email: "<maikel.nadolski@gmail.com>"
  - name: "Lauri Vasama"
---

# Abstract

This paper aims to introduce the semantics of two queries, namely `completes_synchronously()` and `get_completion_concurrency()`, within the framework of asynchronous operations that involve senders and receivers, building upon the current `std::execution` proposal P2300. These queries empower the sender to convey information about the completion behavior of the associated operation. This information proves instrumental in optimizing the behavior of algorithms, including but not limited to `sync_wait()`, `repeat()`-like algorithms and a scheduler-affine task type.

# Proposal

Let sender `s` and receiver `r` be connected, let `env` be the receivers environment and `op` the resulting operation that is initiated by `execution::start(op)`. The following definitions apply:

## `completes_synchronously(s, env)`:

This query primarily addresses the timing of the completion-signal operation concerning the return from `execution::start(op)`. It specifies whether the completion-signal happens synchronously before the return from `start()` or if it may occur asynchronously after the return.

### Possible Return Values

**Always:** When `completes_synchronously(s, env)` returns `always`, the connected receiver's completion-signal happens before the return from `execution::start()`. This setting implies synchronous completion of the operation, enabling stack allocation of operation state or avoidance of certain synchronizations (e.g., in `sync_wait()`).

**Maybe:** Returning `maybe` from `completes_synchronously(sender, env)` indicates that the completion-signal operation may occur after the return from `execution::start(op)`.

**Delayed:** Returning `delayed` from `completes_synchronously(sender, env)` indicates that the return from `execution::start()` happens before the completion-signal operation.

## `get_completion_concurrency(s, env)`:

This query is concerned with the concurrency of the completion operation itself.
It specifies whether the completion-signal operation is guaranteed to be executed sequentially with respect to other operations initiated by `execution::start()` (sequential) or if it can be executed concurrently, treating the completion and the return from `start()` as if initiated by different threads (parallel).

### Possible Return Values

**Sequential:** When `get_completion_concurrency(s, env)` is `sequential`, the completion operation is guaranteed to be executed sequentially with respect to other operations that are initiated by `execution::start()`.

**Parallel:** When `get_completion_concurrency(s, env)` is `parallel`, the completion-signal operation and the return from `execution::start()` are being executed as if they were initiated by different threads.

# Use Cases

The queries serve as a crucial determinant in understanding the behavior of asynchronous operations within different contexts. Its role extends across various functionalities and implementations.

## Within the `sync_wait` algorithm

One possible implementation of `sync_wait(sender)` is to synchronize the completion of the async operation with an internal condition variable. On the other hand, if `completes_synchronously(s, env)` is `true`, the operation will be completed synchronously automatically when it is being started and the need for synchronization is avoided.

## `repeat` algorithms
Consider a typical `repeat(s)` algorithm that repeats the asynchronous operation, which is associated with the sender `s`.
If the operation completes synchronously, the usual implementation results in a recursive call to `execuction::start()` within the completion signal `set_value()` of an intermediate receiver. Therefore, some kind of prevention needs to be implemented to avoid recursive call stacks and stack overflow, e.g. using a trampoline scheduler.
An improvment to this would be to use the `completes_synchronously(s, env)` query to determine whether the operation is completed synchronously. If `completes_synchronously(sender, env)` is `always`, the operation is completed synchronously, and the algorithm can be implemented such that the recursion is avoided.
The only case where the trampoline scheduler is still needed is when `completes_synchronously(sender, env)` is `maybe` (or not known) and `get_completion_concurrency(sender, env)` is `sequential`.

## scheduler-affine task type
Consider a coroutine task type `task<T>` that ensures that each awaited expression completes on the initial scheduler. If a co_await'ed sender completes synchronously, the task can be implemented such that the coroutine is resumed immediately. If the awaited expression completes asynchronously, the task can be implemented such that the coroutine is resumed on the current scheduler when the awaited expression completes.

# Implementation

- `just`:
  - `completes_synchronously()`: `always`
  - `get_completion_concurrency()`: `sequential`


- `then(s, fn)`:
  - `completes_synchronously(env)`: `completes_synchronously(s, env)`
  - `get_completion_concurrency(env)`: `get_completion_concurrency(s, env)`

