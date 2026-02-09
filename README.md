# Assignment 1: Serverless Computing - Critical Analysis

## Alice Yang, 041200019, CST8917 - Serverless Applications | Winter 2026, 02/09/2026

## Assignment Tasks

### Part 1: Paper Summary

While serverless computing may be "one step forward" through fully managed autoscaling, execution, and requiring no server provisioning on the user end, it also may be "two steps back" because it is a poor fit for workloads that require efficiency with data, hardware acceleration, or distributed systems. To support this stance, the paper emphasizes that the cloud's strength in high compute potential is used for ease of multi-tenancy and scaling legacy data services, not cutting-edge programming models. For context, paper focuses on early FaaS options, particularly AWS Lambda, and the objective realities of how "serverless" solutions like those are implemented at that point in time in 2019.

The execution time constraints identified include its very short lifetime (timeout) and increased complexity required to fully execute long-running tasks. The paper identifies that Lambda functions are forcibly timed out after a max of 15 minutes. In addition, subsequent invocations cannot be guaranteed to run on the same VM as the prior execution, so they must be written "assuming that state will not be recoverable across invocations." Furthermore, long-running tasks like  model training must be broken into multiple invocations, which ultimately increases complexity and latency due to orchestration and state management. For example, the case study demonstrates that the ML training algo on Lambda was 21x slower and 7.3x more expensive than on EC2.

Some of the communication/network limitations include I/O bottlenecks, lacking direct network addressability, thus increasing latency. "A single Lambda function can achieve on average 538Mbps network bandwidth"  which is significantly slower than a modern SSD, up to 2.5x slower. In addition, since Lambda functions are not network addressable, they cannot receive direct messages. All communication must go through intermediary services like S3. As a result, according to Table 1, while direct EC2-EC2 messaging is ~290 µs, Lambda-S3-EC2 messaging is about ~106-108 ms, which is 365x-372x slower.

The "data shipping" anti-pattern refers to FaaS "shipping data to code" instead of "shipping code to data." This happens because FaaS runs code on isolated VMs separate from data with very limited persistence of state across invocations. As a result, data-intensive workloads like the aforementioned ML training results in high latency and bandwidth costs. Time is wasted on fetching data from S3 over and over again instead of actual compute.

Another problem identified is hardware limitations. FaaS only exposes CPU slices and RAM, not GPU or any other hardware. Furthermore, the largest Lambda instances still only has 3 GB RAM, which is inadequate for larger in-memory models. This also means that FaaS is inadequate for any hardware-accelerated use cases, like for deep ML training.

Like previously mentioned, FaaS solutions struggle with distributed computing and stateful workloads. There is no direct way to maintain long-lived, in-memory state with typical Lambda functions besides writing and reading from storage like S3 each invocation. This also negatively impacts distributed communication, since event-driven designs also must face the inconsistency and latency of repeated read/writes to "slow storage."

Authors in the paper propose that future cloud programming must address the following issues:

1. Support stateful/long-lived computation

Functions must be enabled to run beyond the timeout limited when needed and be able to be addressed to directly over network. In addition, there must be a way to maintain state across multiple invocations without the need of repeated R/W to slow storage.

2. Negate the anti-pattern by "shipping code to data" with low-latency access to storage

The goal proposed here is to allow data-intensive workloads like ML training be competitive with similar workloads on traditional server-hosted services. FaaS solutions must be able to provide higher bandwidth as functions scale out, balancing compute with data retrieval needs.

3. Enable hardware specialization under serverless models

In order to make serverless solutions viable for ML training and other heavy workloads, CSPs must enable access to hardware features like GPU and appealing pricing models for autoscaling.

---

### Part 2: Azure Durable Functions Deep Dive

1. Orchestration model: How do orchestrator, activity, and client functions work together? How does this differ from basic FaaS?

Orchestrator functions define the workflow logic for managing various short-lived functions for a long-lived process via code. (support for Python, JS, C#, etc.) [1]

Activity functions are the basic units of work, being standard Azure Functions that perform tasks like database writes or API calls.[2] They return results to the orchestrator that then directs to the next step in the workflow.

Client functions as the entry point with typical function triggers of external events (ie. HTTP trigger) that activates downstream orchestration by the Durable Function. [3]

Compared to basic FaaS, Durable Function addresses the critique that FaaS lacks a way to coordinate compute due to its stateless nature. Durable Functions provides a stateful way to allow functions to execute together in a chain or parallel, rather than in isolation. Functions can now progress state  rather than R/W to storage for every result.

2. State management: How does Durable Functions manage state (event sourcing, checkpointing, replay)? How does this address the paper's criticism that functions are stateless?

The event sourcing pattern is when instead of saving the entire memory dump, the orchestration persists an append-only execution history of events to the storage. [4] When the orchestrator hits an `await` statement, it checkpoints progress by committing new events since the last checkpoint to storage and unloads from memory. [5] When resuming, the orchestrator replays from the start all the previously executed tasks and returns stored state to rebuild local state without re-executing everything. [1]

This solves the paper's critique about the stateless problems of serverless solutions, which relied on expensive and slow R/Ws from storage like S3. By "replaying" state restoration from checkpointed events following the event sourcing pattern, Durable Functions attribute statefulness to the orchestration and thus allows workflows to virtually persist data over a long-running process.

3. Execution timeouts: How do orchestrators bypass the timeout limits that apply to regular Azure Functions? What limits still apply to activity functions?

Orchestrators bypass typical FaaS timeout limits (15 min cap for Lambda & 10 min cap for Functions) using the aforementioned replay mechanism. Though the workflow may look like it is running for several days, the actual execution compute is made up of short bursts like a typical function. When the orchestrator is awaiting an async task, it checkpoints its state to storage and unloads from memory until it resumes using a Durable Timer. When the timer resets, the function replays from the start to rebuild state, thus virtualizing a seemingly long-running process. [1]

Activity functions do not have the replay function and are subject to typical timeout limits, like 10 min max for Functions. [6] As a result, any long-running tasks should be broken down into smaller activities or be hosted on higher cost plans that can ensure their completion. For example, anything above Consumption plan has an unbounded timeout limit at maximum [6]

4. Communication between functions: How do orchestrator and activity functions communicate? Does this address the paper's criticism about functions needing slow storage intermediaries?

Orchestrator & activity functions communicate via a Task Hub that uses queues for message passing and tables for storing execution history. [7] Since this is managed within Azure Storage, this does validate the paper's criticism that serverless solutions must use a slow storage intermediary for network communications, resulting in unappealing latency issues.

To address this, the fully-managed Durable Task Scheduler (DTS) (the successor to Netherite) aims to provide much higher throughput compared to the default storage polling option. It features direct gRPC connections between the workers and scheduler. [8] Example benchmarks show that DTS is about 5x faster than default Azure Storage. [8] As a result, Durable Functions now have modern solutions in the works to mitigate that slow storage critique.

5. Parallel execution: (fan-out/fan-in) How does the fan-out/fan-in pattern work? How does it address the paper's concern about distributed computing?

The fan-out/fan-in pattern allows orchestrators to execute multiple activity functions in parallel and only resume past the parallel operations once all have completed. This is done by scheduling the parallel batch of activity tasks (fan-out) without awaiting yet. Then, some sort of sync function/method is used to wait for results (fan-in) like `yield context.task_all(parallel_tasks)` in Python. [5]

This pattern directly addresses the concern the paper expresses that serverless solutions are incompatible with distributed computing. Rather than forcing functions to coordinate via shared storage and coordinating "fan-in" through complex series of queue triggers and external state management [9], Durable Functions lets developers treat distributed parallel operations as simple code. [5] Therefore, the gap between distributed requirements and stateless serverless computing is bridged.

---

### Part 3: Critical Evaluation

Building on your work in Parts 1 and 2, write an analysis (400-600 words) addressing the following:

1. **Limitations that remain unresolved**: Identify **two** criticisms from the paper that Azure Durable Functions has **not** resolved or only partially addresses. Explain why these limitations persist.

2. **Your Verdict**: Does Azure Durable Functions represent the kind of progress the authors envisioned for serverless computing, or does it merely work around the fundamental limitations without truly solving them? Take a clear position and support it with evidence from your research.

Based on the 2019 paper on first-gen FaaS and research evaluating how Azure Durable Functions (ADF) have managed to address some of the paper's primary critiques, I conclude that ADF represents a positive direction evolving from the FaaS' early stateless and orchestration issues. However, ADF still must resolve couple fundamental issues baked into the architecture of serverless platforms. ADF does not fully resolve the "data-shipping" anti-pattern or lack of hardware specialization, and some of its solutions presents as a layer over the "flawed" architecture.

To recap, the paper criticizes FaaS for lack of data efficiency; "shipping data to code" instead of "shipping code to data," essentially saying that decoupling compute from storage leads to unacceptably high I/O bottlenecks/latency for data-intensive workloads like ML training. While ADF can virtually persist workflow state through event sourcing, activity functions are still executed on compute separate from Azure Storage, where the data is stored. As a result, in ML training, the activity function would still have to read datasets, process them, write them back, and thus incur the same bandwidth/latency costs critiqued in the paper. DTS may optimize how execution history is stored (gRPC & logs), but that still does not address how the app data itself is processed.

Another key critique is that FaaS only provisions CPU hyperthread slices and RAM, with no way to access specialized hardware like GPU. ADF is usually used to orchestrate standard activity functions, which are heavily limited in their hardware. (Consumption has 1.5 GB memory and 1 CPU) ADF does not have a native way to enable hardware acceleration yet. Therefore, tasks that would benefit from hardware-acceleration like ML training must be in part completed by non-serverless solutions, or at least solutions outside of the core Azure Function suite. Up to this day, FaaS options are still unsuitable for the type of cutting-edge accelerated software innovation that the paper envisions.

Nevertheless, ADF still demonstrates a matured version of FaaS that improves on some of the critiques. For example, it primarily solves the distributed systems issue. The paper critiques FaaS as being incompatible with distributed computing since functions would be forced to use storage to communicate as an intermediary. ADF's Durable Task Framework allows for distributed patterns like fan-out/fan-in to be written in simple code. Furthermore, DTS replaces storage polling with gRPC or event streams, thus solving much of the slow storage critique. 

Still, it is important to keep in mind that ADF builds on top of existing FaaS as an orchestration layer rather than redefining its intrinsically flawed relationship between compute and storage. While it has made serverless much more usable for long-running processes, I cannot conclude that it fully unlocked it as the future for data-intensive computing in particular.

---

### References

[1]cgillum, “Durable Orchestrations - Azure Functions,” Microsoft.com, Dec. 11, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp-inproc (accessed Feb. 09, 2026).

[2]lily-ma, “Azure Durable Functions: FaaS for Stateful Logic and Complex Workflows,” TECHCOMMUNITY.MICROSOFT.COM, Sep. 06, 2024. https://techcommunity.microsoft.com/blog/appsonazureblog/azure-durable-functions-faas-for-stateful-logic-and-complex-workflows/4238858

[3]R. Dennyson, “The Ultimate Guide to Azure Durable Functions: A Deep Dive into Long-Running Processes, Best Practices, and Comparisons with Azure Batch,” Medium, Sep. 21, 2024. https://medium.com/@robertdennyson/the-ultimate-guide-to-azure-durable-functions-a-deep-dive-into-long-running-processes-best-bacc53fcc6ba

[4]Microsoft, “Event Sourcing pattern - Azure Architecture Center,” learn.microsoft.com. https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

[5]cgillum, “Durable Functions Overview - Azure,” Microsoft.com, Apr. 06, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=in-process%2Cnodejs-v3%2Cv1-model&pivots=python (accessed Feb. 09, 2026).

[6]ggailey777, “Azure Functions scale and hosting,” learn.microsoft.com. https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

[7]cgillum, “Durable Functions storage providers - Azure,” Microsoft.com, May 08, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-storage-providers (accessed Feb. 09, 2026).

[8]greenie-msft, “Announcing the public preview launch of Azure Functions durable task scheduler,” TECHCOMMUNITY.MICROSOFT.COM, Mar. 20, 2025. https://techcommunity.microsoft.com/blog/appsonazureblog/announcing-the-public-preview-launch-of-azure-functions-durable-task-scheduler/4389670 (accessed Feb. 09, 2026).

[9]J. Hellerstein et al., “Serverless Computing: One Step Forward, Two Steps Back,” Jan. 2019.

## AI Disclosure Statement

I used AI to find a list of 15-20 sources I could sift through that is relevant to Durable Function research. I extracted the 8 most relevant for manual IEEE citation.

