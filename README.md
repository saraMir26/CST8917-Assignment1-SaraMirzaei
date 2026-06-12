# Assignment 1: Serverless Computing - Critical Analysis

**Name:** Sara Mirzaei   
**Student Number:** 040-467-655   
**Course:** CST8917 – Serverless Applications      
**Assignment:** Assignment 1: Serverless Computing - Critical Analysis     
**Date:** June 11, 2026    

---

## Part 1: Paper Summary

In *Serverless Computing: One Step Forward, Two Steps Back*, Hellerstein et al. (2019) argue that first-generation serverless computing, especially Function-as-a-Service (FaaS), is an important improvement in cloud programming but also a major regression for data-intensive and distributed systems. The “one step forward” is that serverless platforms allow developers to upload code without managing servers, automatically scale execution based on demand, and pay only for actual usage. This is valuable because it removes much of the operational work involved in provisioning, monitoring, and scaling virtual machines. However, the “two steps back” are that current FaaS platforms make it difficult to process data efficiently and difficult to build true distributed systems.

The authors identify execution time limits as one of the main restrictions. FaaS functions are designed to be short-lived. In the paper’s AWS Lambda example, functions were limited to 15 minutes, and developers could not rely on a future invocation returning to the same machine. This means functions must be written as if local state will disappear after each run. For simple event processing this is acceptable, but for long-running workflows or stateful applications it creates complexity.

Another major limitation is communication. FaaS functions can usually make outbound network calls, but they are not directly addressable as running processes. Because one function instance cannot easily communicate directly with another function instance, developers often use storage services, queues, or object stores as intermediaries. The paper argues that this creates an I/O bottleneck because communication through storage is slower and more expensive than direct network communication. It also prevents “sticky” client sessions where a client can continue communicating with the same function instance.

The paper also criticizes FaaS as a “data shipping” architecture. Instead of moving computation close to the data, FaaS often moves data from storage to short-lived code. This is inefficient because modern systems depend heavily on memory hierarchy, caching, locality, and high-bandwidth data access. Pulling large datasets into temporary function instances can increase latency, bandwidth use, and cost.

The authors also point out limited hardware access. First-generation FaaS generally provides CPU and memory but does not give developers direct access to GPUs, accelerators, or specialized hardware. This is a serious limitation for machine learning, scientific computing, and other workloads that depend on hardware acceleration.

Finally, the paper argues that these limitations make FaaS weak for distributed computing and stateful workloads. Distributed systems often require fine-grained communication for coordination, membership, leader election, consistency, and transactions. FaaS forces much of this coordination through external storage, which makes it difficult to use thousands of cloud cores efficiently for anything beyond embarrassingly parallel workloads.

For the future, the authors propose cloud programming models that go beyond basic FaaS. They recommend fluid code and data placement, where cloud infrastructure can move code closer to data or reorganize data placement for performance. They also recommend support for heterogeneous hardware so programs can use GPUs or specialized resources when needed. Another proposal is long-running, addressable virtual agents: cloud-managed software entities that can persist over time, have stable identities, and communicate efficiently. They also discuss more declarative or “disorderly” programming models, common intermediate representations, explicit service-level objectives, and stronger security models for cloud-scale programming.

---

## Part 2: Azure Durable Functions Deep Dive

### 1. Orchestration Model

Azure Durable Functions extends basic Azure Functions by adding a workflow model built around client functions, orchestrator functions, and activity functions. A client function starts an orchestration, often through an HTTP trigger, queue trigger, or another event source. The orchestrator function defines the workflow steps in code and coordinates the order of execution. Activity functions perform the actual work, such as calling an API, processing data, or writing to a database. This differs from basic FaaS because a regular function invocation is isolated and short-lived, while a Durable Functions orchestration represents a larger workflow with an identity and lifecycle. Microsoft describes Durable Functions as an Azure Functions extension for building stateful workflows using orchestrator, activity, and entity functions, with the runtime managing state, checkpoints, retries, and recovery. This directly addresses part of Hellerstein et al.’s criticism that raw FaaS is too isolated for composed applications, although it does so by adding a workflow framework rather than changing the basic nature of function execution.

### 2. State Management

Durable Functions manages workflow state through event sourcing, checkpointing, and replay. Instead of keeping the orchestrator’s state only in memory, the Durable Task Framework records the orchestration’s history in an append-only store. When the orchestrator reaches an `await` or `yield`, the framework checkpoints progress and can unload the orchestrator from memory. Later, when more work is available, the orchestrator replays from the beginning and uses the saved history to rebuild local variable state and avoid re-running completed activities. This addresses the paper’s criticism that FaaS functions are stateless by giving developers a reliable stateful workflow abstraction. However, the state is not preserved because the same function process remains alive. It is preserved through durable storage and replay. This makes workflows reliable, but it also confirms that Durable Functions depends on the kind of storage-backed coordination that the paper viewed as a performance concern.

### 3. Execution Timeouts

Durable Functions helps developers build long-running workflows without keeping a single function execution active for the entire duration. Microsoft documentation states that orchestration instances can last seconds, days, months, or even be configured never to end. This works because the orchestrator checkpoints progress, unloads from memory while waiting, and later resumes through replay. As a result, a workflow can outlive the timeout of a single Azure Function execution. This addresses the paper’s concern about limited lifetimes for higher-level workflows such as approvals, booking processes, or multi-step data pipelines. However, it does not completely remove timeout limits from every part of the system. Microsoft states that activity, orchestrator, and entity functions are still subject to the same function timeout rules as other Azure Functions, and an activity timeout is treated like an unhandled exception. Azure Functions timeout limits also vary by hosting plan; for example, the traditional Consumption plan has a 5-minute default and 10-minute maximum.

### 4. Communication Between Functions

In Durable Functions, orchestrators communicate with activities by scheduling function calls and receiving their results through the Durable Task Framework. The framework records scheduled work, places messages in storage-backed queues or the configured backend, and later replays results into the orchestrator. This is much better for application development than manually wiring many independent functions together with queues and storage accounts because the developer writes the workflow in normal procedural code. It also gives a clear instance ID and execution history for monitoring and recovery. However, it only partially addresses the paper’s criticism about communication through slow intermediaries. Durable Functions improves the programming model and reliability, but it does not provide direct network addressability between running function instances. The orchestrator itself should not perform CPU-intensive work, blocking work, or I/O; Microsoft recommends moving that work into activity functions. Therefore, Durable Functions solves coordination at the workflow level, not low-latency peer-to-peer communication.

### 5. Parallel Execution: Fan-out/Fan-in

The fan-out/fan-in pattern allows an orchestrator to start many activity functions in parallel and then wait for all of them to finish before aggregating the results. For example, a workflow can split work across many files, call an activity for each file, and then combine the outputs after all activities complete. Microsoft lists parallel activity execution as a core Durable Functions performance scenario and connects it to the fan-out/fan-in pattern. This directly addresses the paper’s concern that basic FaaS is mainly useful for independent tasks and difficult to coordinate when tasks must be combined. Durable Functions makes this pattern easier because the orchestrator tracks all parallel tasks and checkpoints progress. However, this is still strongest for coarse-grained parallelism. It is not a full solution for tightly coupled distributed computing where tasks require frequent low-latency communication, shared memory, leader election, or custom networking protocols.

---

## Part 3: Critical Evaluation

Azure Durable Functions is a meaningful improvement over first-generation FaaS, but it does not fully solve the deeper architectural concerns raised by Hellerstein et al. My view is that Durable Functions represents practical progress for business workflows and serverless application development, but it mostly works around the fundamental limitations instead of eliminating them.

One unresolved limitation is the paper’s concern about direct communication and addressability. Durable Functions gives each orchestration an instance ID and allows clients to query status, raise events, or wait for results. This is useful, but it is not the same as having long-running, directly addressable compute agents that communicate at normal network speed. Activity functions are still stateless units of work. The orchestrator coordinates them through the Durable Task Framework, which stores history and schedules messages through a backend. This improves reliability and developer experience, but it does not create a model where thousands of functions can communicate with each other directly using fine-grained distributed protocols. For workloads like leader election, transaction commit protocols, high-performance scientific computing, or tightly coupled parallel processing, the original criticism still applies.

A second unresolved limitation is data locality and the “data shipping” anti-pattern. Durable Functions is excellent at coordinating steps, but it does not automatically move code to data or optimize physical placement of computation and storage. Activity functions can read from Blob Storage, Cosmos DB, SQL, or external APIs, but they still usually pull data into function executions. The orchestration history itself is stored externally, commonly in Azure Storage or another durable backend. This is reliable, but it reinforces the storage-centered model rather than replacing it with a data-centric execution engine. Durable Functions can organize a data pipeline, but it is not equivalent to a database optimizer, Spark engine, or cloud runtime that fluidly places code near data.

Durable Functions does, however, strongly address the problem of stateless workflow composition. In basic FaaS, developers often have to manually combine functions using queues, tables, blobs, and custom status tracking. Durable Functions turns that into a programming model where the workflow is written in code, state is checkpointed automatically, failed activities can be retried, and long-running processes can resume after restarts. For enterprise workflows such as order processing, approvals, appointment booking, file processing, and scheduled automation, this is a major step forward.

Overall, Azure Durable Functions is not the full future imagined by the paper. It does not provide fluid code/data placement, heterogeneous hardware scheduling, or low-latency addressable virtual agents. Instead, it provides a robust orchestration layer on top of existing serverless functions and storage. That is still valuable. In my opinion, Durable Functions should be seen as an important evolution from “stateless functions” to “stateful serverless workflows,” but not as a complete answer to the paper’s challenge. It makes serverless more useful for real applications, especially workflows, while leaving the hardest data-intensive and distributed-systems problems unresolved.

---

## References

- Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). *Serverless Computing: One Step Forward, Two Steps Back*. CIDR 2019. https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf
- Microsoft Learn. (2026). *Durable Functions overview*. https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview
- Microsoft Learn. (2026). *Durable orchestrations*. https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-orchestrations
- Microsoft Learn. (2026). *Durable orchestrator code constraints*. https://learn.microsoft.com/en-us/azure/durable-task/common/durable-task-code-constraints
- Microsoft Learn. (2026). *Performance and scale in Durable Functions*. https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-perf-and-scale
- Microsoft Learn. (2026). *Azure Functions scale and hosting*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

---

## AI Disclosure Statement

I used ChatGPT to help draft, organize, and edit this assignment.
