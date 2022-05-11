---
id: list_tasks
title: tctl admin taskqueue list_tasks
description: Listing Tasks in a Task Queue
  - reference
  - tctl
---

The `tctl admin taskqueue list_tasks` command  lists the tasks of a Task Queue.

#### Usage
`tctl admin taskqueue list_tasks [command modifiers] [arguments...]`

### Modifiers

#### `--more`
Alias: `-m`
Lists more pages of Tasks.

Default: one page of 10 Tasks

#### `--pagesize value`
Alias: `--ps value`
Size of the result page.

Default: 10

#### `--taskqueuetype`
The type of Task Queue.

Default: activity
Values: workflow, activity

#### `--min_task_id value`
The minimum value that can be set as a TaskId.

Default: -12346

#### `--max_task_id value`
The maximum value that can be set as a TaskId.

Default: 0

#### `--print_json`
Prints the list in raw JSON format.