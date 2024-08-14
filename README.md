# SysDesign Meetup Academy

Your challenge, should you accept it, is to implement a distributed system that satisfies a certain set of requirements.

## Setup

The system should fulfill all the functional requirements for the problem. Some requirements are qualitative hard yes/no constraints, other requirements are quantitative and can be measured. Not satisfying a qualitative requirement marks the implementation as incorrect. Correct implementations are ranked based on how well they perform on quantifiable requirements.

This distributed system consists of several nodes. Each node receives requests, and each node should respond to each of them. Nodes can also talk to each other.

The nodes are simulated, so that the tests are deterministic and fast to run. The code is implemented in a simplified programming language designed to make the codes focus on the aspects of distributed systems. The simulated machine that runs the code of these nodes is based on cooperative multithreading (async-await-friendly), has with limited amount of fast hot storage ("RAM") and an infinite, albeit slow, cold storage ("disk").

For the first version of this challenge, the nodes do not fail, the network connections do not deteriorate, and the number of nodes to use is known up front.
Submissions have to be implemented as correct code snippets. The domain-specific language for them is described _below_. We provide the ready-to-execute test runner, as well as several reference solutions. They can be tested and evaluated locally, or via Github Action runners.

## Problem

The challenge is to implement a strongly consistent distributed key-value store.

### Functional Requirements

* All keys are independent, all read/write operations access a single key.
  * Each key is of a dedicated type. Assume there will be at least one million different keys.
  * Each value is of a dedicated type. The user code does not need to know what it is, since it is only assigned and returned.
  * The types of both keys and values fit one "unit" of memory that of the virtual "CPU" that runs the code.
* For each key, the value of its version is also stored, as an integer.
  * The version is equal to the number of times the value for this key was mutated.
  * For a never-accessed key, the value is `None` and the version is zero.
  * During test runs, the version for each key will never exceed one billion.
* There are three operations to handle:
  * Mutate:
    * Request: the key and the expected next version.
    * Response, either:
      * `Updated`, if the precondition is satisfied and the mutation request has succeeded or
      * `Rejected`, if the value for this key was updated since.
    * Note:
      * This technique mimics the `If-Unmodified-Since` strategy. It can also be thought of as `compare-and-set`.
  * Simple read:
    * Request: the key.
    * Response, either:
      * The value and the version for this key, as a tuple, or
      * `TryLater` if the system chooses to discard this request.
    * Notes:
      * The nodes are allowed to return stale data, but only within the `kKVParams.MAX_REPLICATION_DELAY` time interval.
      * Responding with `TryLater` means that the system is overloaded, loosely equivalent to "HTTP 503".
  * Read at version:
    * This is the extension of the simple read request above.
    * Request: the key, the requested version, maximum wait timeout.
    * Response, either:
      * The value, if the requested version is the current one,
      * `Outdated`, if there exists a new  value for this key.
      * `NotAvailableYet`, if the requested version is not yet available on this node.
    * Notes:
      * `Outdated` is legal because the requirement is to only keep the latest value per key.
      * The implementation can always respond with `NotAvailableYet` if the node does not have the data locally.
      * The requirement for the node to wait for up to the maximum wait timeout and respond back once the data for the requested version becomes available is a non-functional one.
      * It is (or, rather, _it will be_) graded separately for more bonus points, but consider it an advanced requrement.
      
## Test Setup

The test traffic consist of a large number of "virtual users", each running a batch of requests. One batch operates on a subset of keys. The keys that each batch operates on are chosen from a power law distribution.

Different values of the parameter of this distribution result in different test cases. On one extreme value, each key is accessed with equal probability. On the other extreme, it is a small subset of keys that are accessed all the time.

The implementation should correctly handle both cases. When it comes to performance and quantatative metrics, we expect different submissions to perform better for different load patterns, and _analyzing this behavior in depth is part of the exercise_.

Each virtual user starts in a stateless manner, and it can send its requests to any node. The node that has received the request should send a response to it. This node can communicate with other nodes before sending the response to the user.

Each response from the system can include the _recommendation_ (stickiness hint) for the caller on which node should this caller go to for the requested key. Within one virtual user sending one batch, these recommendations are respected. The next batch starts from scratch, and its requests are at first sent to randomly selected nodes.

Clearly, the larger the batch size is, the larger the fraction of requests that will respect stickiness will be. Thus, batch size is another, orthogonal parameter to running the test.

These two orthogonal parameters define the test matix. For each cell in this matrix, the simulated traffic is gradually increased in requests-per-second until the system fails. For each cell in the matrix the maximum sustained load, in requests per second, is recorded as the score.

(Note: We will probably do some binary-searching here, with a threshold, since due to pseudo-deterministic nature of the simulated environment, the passing-to-failing "transition" may not be a fully monotonic function of the simulated QPS.)

## Implementations

The trivial implementation can use hashing to determine the source-of-truth for each key, route each request to the node that is the source-of-truth for each key, and store this key on disk of this node.

The obvious extension is to have nodes broadcast the mutations to the keys owned by them, so that read requests can be served by any node.

The next big step is to add caching per node.

I imagine the next step would be to optimize the in-memory layout of the cache (using bitsets), so that the maximum possible amount of RAM can be used specifically for the cache. I.e. utilize not 25% or 50% but 90+% of the available RAM for caching keys and values.

Various caching techniques can be utilized too.

On the advanced level the game will be about not failing under high load. Some ideas include:

* Do not broadcast the value for each key to every node, but only to its "secondary" and "ternary" nodes.
* Delay these broadcasts, so that while the replication delay constraint is still respected, the total amount of network traffic and CPU time needed to process these mutations is lower.
* Analyze the load of the system to intelligently throttle certain updates; for instance, by keeping the values for the most-frequently-updated keys in RAM only, persisting them to disk only as they become less frequently updated.

At this point of sophistication the challenge is more of the challenge for test writers than to those implementing the system. Which is exactly what we are planning on, because:

1. The language itself needs to be polished before we can go live with this SysDesign Challenge,
2. The test framework (Github Action runners and local executors) needs to be battle-tested,
3. Having people impement test workloads to "challenge" other submissions is a great exercise by itself, and
4. We have no shortage of truly nontrivial problems, since nodes can and will fail, in various ways, and their network connections will too.

## Language

The language is duck-typed.

In the future, if it evolves into something serious, there will static typing and linting, of course.

There are multiple "primitive types", including integers (fixed range; and no floating point), keys and values for the keys and values from the KV-store problem (keys can be hashed and compared to one another), absolute dates and time intervals, booleans, and, of course, a dedicated type to store references to other nodes (convertible, albeit only explicitly, to and from integers). 

For now, we expect the language section to be written as we have the simulator and some test submissions =)

One thing I have a strong opinion on: let's reserve any `CamelCase` identifier as a "compile-time" enum value. 

Tuples can be introduced organically `(python, style)`, variant types emerge naturally as Haskell algebraic types (`None | Maybe x`), and data classes of a small number of fields are effectively dictionaries where the key is a compile-time "enum value" term (`{ TheAnswer: "42", TheQuestion: "why not?" }`).

When it comes to async-await, I am thinking that for the first version of this pseudo-DSL we can omit the syntactic sugar and live with callbacks.

In terms of functionaliy, we also need the means to obtain:

* [pseudo-]random numbers,
* the "current time", to only be compared to other "current time" moments on the local node, and
* some ways to get into the insights of the VM, i.e. "get the number of currently pending I/O operations and/or sleep()-s".

Similar to JavaScript, we can treat `x[y]` and `x.y` as identical operations.

Global variables could be accessed as `global.`, and per-coroutine variables -- as `local.`. I think we need to have variables declared explicitly, so that if and when the code fails with "insufficient RAM", it fails on the line where the variable is being introduced.

Network and disk operations would be functions returning promises. (Yes, callbacks and/or `.then()`-style would work for now. We'll get to `await` soon.)

For algebraic types, the `match` construct is my favorite. So instead of strongly typing network commands -- be they user requests or cross-node communication -- we can have some single `onRequest(message, responder)`, which would start from some `match message with`. Since `message` can be anything, we can dedicate some compile-time `KeyValueGet` and `KeyValueMutate` algebraic types for user requests, so that they are just the `match` constructs themselves. To keep things simple, let's just say each message has to be responded to, exactly once.

## Constraints

## Memory Constraints

For the sake of simplicity:

* Each singular value takes one "unit" of simulated RAM.
* A tuple of M elements takes M units of RAM. 
* An array of N elements takes `max(integer_index_used) - min(integer_index_used) + 1` units of RAM.
* A dictionary of K values takes `3 * K + sum(sizeof(each_value))` units of RAM.
* An algebraic type is `1 + sizeof(inner)`, so for `None | Some x` it's either 1 or `1 + sizeof(x)`.

## Reference Constraints

For the virtual machine:
 
| Name | Value | Description |
| ---- | :---: | ----------- |
| kCycleTime | 1us | Between yielding control, each CPU operation is considered an O(1) operation, and it runs for 1us. In other words, the CPU performs a million of "trivial operations" per second, where everything that does not await on async I/O is considered a trivial operation. |
| kMaxCoroutines | 1000 | A node can have at most this many async operations -- coroutines -- scheduled. This includes network, disk, and `sleep()`-s combined. Inbound network request are counted in too, and the node is considered dead if it receives an inbound network message when `kMaxCoroutines` is reached. |
| kMaxGlobals | 1000 | A node can use at most this many "units" of memory. See in the text for details on exact calculation. |
| kMaxLocals | 100 | A coroutine can use at most this many "units" of memory. |  
| kDiskReadLatency | 10us-12us | How long until a cold storage ("disk") read request is fulfilled. |
| kDiskReadThroughput | 1us | Add this number of microseconds to read each unit of data from disk. | 
| kDiskWriteLatency | 50us-60us | How long until a cold storage ("disk") write request is fulfilled. | 
| kDiskWriteThroughput | 5us | Add this number of microseconds to write each unit of data to disk. | 
| kNetworkLatency | 500us | Add this number of microseconds to send a packet to another node, so that the "ping" between them is double this number. |
| kNetworkThroughput | 20us-24us | Add this number of microseconds to send one unit of data over the network to another node. |  

For the key-value store problem:

| Name | Value | Description |
| ---- | :---: | ----------- |
| kKVParams.N | 7 | The numbers of nodes to use. |
| kKVParams.MAX_LATENCY | 1s | The upper bound on the acceptable user-perceived latency of responses. |
| kKVParams.MAX_REPLICATION_DELAY | 0.1s | The maximum replication delay. At most after this much time after receiving the "OK" for a mutation from one node, other nodes should be up to date. |
| kKVParams.MAX_DISCARDS_RATIO | 0.01 | The system should not respond with `TryLater` to more than 1% of the requests. Measured per node, with a sliding window of length of one second. |
