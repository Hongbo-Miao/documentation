---
id: tasks
title: Tasks
sidebar_label: Tasks
description: Temporal Task Queues and Worker Processes are tightly coupled components.
toc_max_heading_level: 4
---

<!-- THIS FILE IS GENERATED. DO NOT EDIT THIS FILE DIRECTLY -->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Temporal Task Queues and Worker Processes are tightly coupled components.

:::info WORK IN PROGRESS

This guide is a work in progress.
Some sections may be incomplete.
Information may change at any time.

:::

A Task is the context that a Worker needs to progress with a specific [Workflow Execution](/docs/workflows/#workflow-executions) or [Activity Execution](/docs/activities/#activity-execution).

There are two types of Tasks:

- [Activity Task](#activity-task)
- [Workflow Task](#workflow-task)

### Workflow Task

A Workflow Task is a Task that contains the context needed to make progress with a Workflow Execution.

- Every time a new external event that might affect a Workflow state is recorded, a Workflow Task that contains the event is added to a Task Queue and then picked up by a Workflow Worker.
- After the new event is handled, the Workflow Task is completed with a list of [Commands](/docs/workflows/#commands).
- Handling of a Workflow Task is usually very fast and is not related to the duration of operations that the Workflow invokes.

### Workflow Task Execution

A Workflow Task Execution is when a Worker picks up a Worker Task and uses it to make progress on the execution of a Workflow function.

### Activity Task

An Activity Task contains the context needed to proceed with an [Activity Task Execution](#activity-task-execution).
Activity Tasks largely represent the Activity Task Scheduled Event , which contains the data needed to execute an Activity Function.

If Heartbeat data is being passed, an Activity Task will also contain the latest Heartbeat details.

### Activity Task Execution

An Activity Task Execution is when the Worker uses the Context provided from the [Activity Task](#activity-task) and executes the [Activity Definition](/docs/activities/#activity-definition) (also known as the Activity Function).

The [ActivityTaskScheduled Event](/docs/workflows/#events#activitytaskscheduled) corresponds to when the Temporal Cluster puts the Activity Task into the Task Queue.

The [ActivityTaskStarted Event](/docs/workflows/#events#activitytaskstarted) corresponds to when the Worker picks up the Activity Task from the Task Queue.

Either [ActivityTaskCompleted](/docs/workflows/#events#activitytaskcompleted) or one of the other Closed Activity Task Events corresponds to when the Worker has yielded back to the Temporal Cluster.

The API to schedule an Activity Execution provides an "effectively once" experience, even though there may be several Activity Task Executions that take place to successfully complete an Activity.

Once an Activity Task finishes execution, the Worker responds to the Cluster with a specific Event:

- ActivityTaskCompleted
- ActivityTaskFailed
- ActivityTaskTimedOut
- ActivityTaskCanceled
- ActivityTaskTerminated

## Task Queues

A Task Queue is a lightweight, dynamically allocated queue that one or more [Worker Entities](/docs/workers/#worker-entity) poll for [Tasks](#).

Task Queues do not have any ordering guarantees.
It is possible to have a Task that stays in a Task Queue for a period of time, if there is a backlog that wasn't drained for that time.

There are two types of Task Queues, Activity Task Queues and Workflow Task Queues.
But one of each can exist with the same Task Queue name.

![Task Queue component](/diagrams/task-queue.svg)

Task Queues are very lightweight components.

- Task Queues do not require explicit registration but instead are created on demand when a Workflow Execution or Activity spawns or when a Worker Process subscribes to it.
- There is no limit to the number of Task Queues a Temporal Application can use or a Temporal Cluster can maintain.

Workers poll for Tasks in Task Queues via synchronous RPC.
This implementation offers several benefits:

- Worker Processes do not need to have any open ports, which is more secure.
- Worker Processes do not need to advertise themselves through DNS or any other network discovery mechanism.
- When all Worker Processes are down, messages simply persist in a Task Queue, waiting for the Worker Processes to recover.
- A Worker Process polls for a message only when it has spare capacity, avoiding overloading itself.
- In effect, Task Queues enable load balancing across a large number of Worker Processes.
- Task Queues support server-side throttling, which enables you to limit the Task dispatching rate to the pool of Worker Processes while still supporting Task dispatching at higher rates when spikes happen.
- Task Queues enable what we call [Task Routing](#task-routing), which is the routing of specific Tasks to specific Worker Processes or even a specific process.

All Workers listening to a given Task Queue must have identical registrations of Activities and/or Workflows.
The one exception is during a Server upgrade, where it is okay to have registration temporarily misaligned while the binary rolls out.

#### Where to set Task Queues

There are four places where the name of the Task Queue can be set by the developer.

1. A Task Queue must be set when spawning a Workflow Execution:

- [How to set `StartWorkflowOptions` in Go](/docs/go/startworkflowoptions-reference/#taskqueue)
- [How to spawn a Workflow Execution using tctl](/docs/tctl/workflow/start#--taskqueue)

2. A Task Queue name must be set when starting a Worker Entity:

- [How to develop a Worker Program in Go](/docs/go/how-to-develop-a-worker-program-in-go)
- [How to develop a Worker Program in Java](/docs/java/how-to-develop-a-worker-program-in-java)
- [How to develop a Worker Program in PHP](/docs/php/how-to-develop-a-worker-program-in-php)
- [How to develop a Worker Program in TypeScript](/docs/application-development-guide/#run-worker-processes)

Note that all Worker Entities listening to the same Task Queue name must be registered to handle the exact same Workflows Types and Activity Types.

If a Worker Entity polls a Task for a Workflow Type or Activity Type it does not know about, it will fail that Task.
However, the failure of the Task will not cause the associated Workflow Execution to fail.

3. A Task Queue name can be provided when spawning an Activity Execution:

This is optional.
An Activity Execution inherits the Task Queue name from its Workflow Execution if one is not provided.

- [How to set `ActivityOptions` in Go](/docs/go/activityoptions-reference/#taskqueue)

4. A Task Queue name can be provided when spawning a Child Workflow Execution:

This is optional.
A Child Workflow Execution inherits the Task Queue name from its Parent Workflow Execution if one is not provided.

- [How to set `ChildWorkflowOptions` in Go](#)

## Sticky Execution

A Sticky Execution is when a Worker Entity caches the Workflow Execution Event History and creates a dedicated Task Queue to listen on.

A Sticky Execution occurs after a Worker Entity completes the first Workflow Task in the chain of Workflow Tasks for the Workflow Execution.

The Worker Entity caches the Workflow Execution Event History and begins polling the dedicated Task Queue for Workflow Tasks that contain updates, rather than the entire Event History.

If the Worker Entity does not pick up a Workflow Task from the dedicated Task Queue in an appropriate amount of time, the Cluster will resume Scheduling Workflow Tasks on the original Task Queue.
Another Worker Entity can then resume the Workflow Execution, and can set up its own Sticky Execution for future Workflow Tasks.

- [How to set a `StickyScheduleToStartTimeout` on a Worker Entity in Go](/docs/go/how-to-set-workeroptions-in-go/#stickyscheduletostarttimeout)

Sticky Executions are the default behavior of the Temporal Platform.

## Task Routing

Task Routing is when a Task Queue is paired with one or more Workers, primarily for Activity Task Executions.

In some use cases, such as file processing or machine learning model training, an Activity Task must be routed to a specific Worker Process or Worker Entity.
For example, suppose that you have a Workflow with the following three separate Activities:

- Download a file.
- Process the file in some way.
- Upload a file to another location.

The first Activity, to download the file, could occur on any Worker on any host.
However, the second and third Activities must be executed by a Worker on the same host where the first Activity downloaded the file.

In a real-life scenario, you might have many Worker Processes scaled over many hosts.
You would need to develop your Temporal Application to route Tasks to specific Worker Processes when needed.

Code samples:

- [Java file processing example](https://github.com/temporalio/samples-java/tree/master/src/main/java/io/temporal/samples/fileprocessing)
- [PHP file processing example](https://github.com/temporalio/samples-php/tree/master/app/src/FileProcessing)
- [Go file processing example](https://github.com/temporalio/samples-go/tree/master/fileprocessing)

### Sessions

Some SDKs provide a Session API that provides a straightforward way to ensure that Activity Tasks are executed with the same Worker without requiring you to manually specify Task Queue names.
It also includes features like **concurrent session limitations** and **worker failure detection**.

- [How to create Worker Sessions in Go](/docs/go/how-to-create-a-worker-session-in-go)
