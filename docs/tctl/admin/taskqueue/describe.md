---
id: describe
title: tctl admin taskqueue describe
description: Retrieving information about the Task Queue
tags:
  - reference
  - tctl
---

The `tctl admin taskqueue describe` command reveals information about pollers and the status of the Task Queue.

#### Usage
`tctl admind taskqueue describe [command modifiers] [arguments...]`

### Modifiers

#### `--taskqueue value`
Alias: `--tq value`
Describes the Task Queue.

#### `--taskqueuetype value`
Alias: `--tqt value`
Shows the optional TaskQueue type.

Default: workflow
Values: workflow, activity