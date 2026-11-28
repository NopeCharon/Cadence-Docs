---
layout: default
title: Distributed Cron
description: How to run a recurring workflow using the cron_schedule option in the Cadence Python SDK, and how to handle a run that overlaps the next fire time with cron_overlap_policy.
keywords:
  - cadence python cron
  - cadence python distributed cron
  - cadence python cron overlap policy
  - cadence python recurring workflow
  - cadence python distributed cron tutorial
permalink: /docs/python-client/distributed-cron
---

# Distributed Cron

The `cron_schedule` field on `start_workflow` runs a workflow on a recurring cadence. This is the legacy approach; for new use cases consider [Schedules](/docs/python-client/schedules), which add pause/unpause, backfill, more overlap policies, and visibility.

## Samples

Related sample:

| Sample | Description | Code |
|--------|-------------|------|
| **Schedule operations** | Schedules, the recommended alternative to `cron_schedule` | [schedule_samples](https://github.com/cadence-workflow/cadence-samples/tree/master/python_sdk_samples/schedule_samples) |

## Starting a cron workflow

```python
from datetime import timedelta
from cadence.client import Client

CADENCE_TARGET = "localhost:7833"  # replace with your Cadence frontend address

async with Client(domain="my-domain", target=CADENCE_TARGET) as client:
    execution = await client.start_workflow(
        "DailyReportWorkflow",
        workflow_id="daily-report",
        task_list="report-workers",
        cron_schedule="0 9 * * *",   # every day at 9 AM UTC
        execution_start_to_close_timeout=timedelta(hours=2),
    )
```

Standard five-field cron syntax is used (`minute hour day-of-month month day-of-week`). All times are UTC.

## How it works

After each run completes (or fails), the Cadence server automatically starts the next execution at the next scheduled time. Each execution is independent: it gets a fresh workflow history and runs from the beginning of the workflow function.

By default, if a run is still executing when the next fire time arrives, the new fire is skipped. To start one late run instead, set a [cron overlap policy](#handling-overlapping-runs).

## Handling overlapping runs

The `cron_overlap_policy` option decides what the server does when a run closes after the next fire time has already passed, because the run took longer than the interval between two fire times.

| Policy | What happens when a run closes after the next fire time |
|---|---|
| `CRON_OVERLAP_POLICY_SKIPPED` | The fire times that passed while the run was open are skipped. The next run starts at the first fire time after the run closes. This is the default. |
| `CRON_OVERLAP_POLICY_BUFFER_ONE` | The next run starts as soon as the run closes. The server starts only one late run, however many fire times passed. |

For example, with the cron schedule `*/5 * * * *`, a run that starts at 10:00 and closes at 10:08 is followed by a run at 10:10 with `CRON_OVERLAP_POLICY_SKIPPED`, or by a run at 10:08 with `CRON_OVERLAP_POLICY_BUFFER_ONE`.

```python
from datetime import timedelta
from cadence.api.v1 import workflow_pb2

execution = await client.start_workflow(
    "InventorySyncWorkflow",
    workflow_id="inventory-sync",
    task_list="inventory-workers",
    cron_schedule="*/5 * * * *",
    cron_overlap_policy=workflow_pb2.CRON_OVERLAP_POLICY_BUFFER_ONE,
    execution_start_to_close_timeout=timedelta(hours=1),
)
```

The policy is set when the cron workflow starts, and the server passes it on to every later run, including retries. `signal_with_start_workflow` accepts the same option. It needs Python client v0.2.0 or later and server v1.3.3 or later.

## Passing state between runs

Each execution of the workflow function is independent. To carry state from one run to the next, use the workflow result: the next execution receives the previous run's result as its first argument.

```python
from datetime import timedelta
from cadence import workflow
from cadence.worker import Registry
from cadence.workflow import execute_activity

registry = Registry()

@registry.workflow()
class DailyReportWorkflow:
    @workflow.run
    async def run(self, last_cursor: str | None = None) -> str:
        # Process from where last run left off
        new_cursor = await execute_activity(
            "generate_report",
            str,
            last_cursor,
            start_to_close_timeout=timedelta(hours=1),
        )
        return new_cursor  # passed to next execution
```

## Stopping a cron workflow

Cancel or terminate the workflow execution. Cancelling stops the next scheduled run; no built-in mechanism pauses and resumes a cron workflow. For pause/unpause support, use [Schedules](/docs/python-client/schedules).

```python
await client.cancel_workflow("daily-report", "")
```

## Cron vs Schedules

| | `cron_schedule` field | Schedules |
|---|---|---|
| Overlap control | Skip or buffer one late run | Skip, buffer, run concurrently, or cancel or terminate the previous run |
| Pause/unpause | No | Yes |
| Backfill | No | Yes |
| Visibility | No | Yes |
| Update without restart | No | Yes |

For new recurring workflows, prefer Schedules.
