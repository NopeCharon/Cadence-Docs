---
layout: default
title: Distributed CRON
description: This page explains how to schedule Cadence workflows to run on a cron schedule using CronSchedule in StartWorkflowOptions, including how to handle a run that overlaps the next scheduled time with CronOverlapPolicy and how to retrieve results from the last successful run.
keywords:
  - cadence cron
  - distributed cron
  - CronSchedule
  - CronOverlapPolicy
  - scheduled workflow
  - periodic workflow
  - cron expression
  - HasLastCompletionResult
  - GetLastCompletionResult
  - go client cron
  - cadence go distributed cron tutorial
permalink: /docs/go-client/distributed-cron
---

It is relatively straightforward to turn any Cadence :workflow: into a Cron :workflow:. All you need
is to supply a cron schedule when starting the :workflow: using the CronSchedule
parameter of
[StartWorkflowOptions](https://godoc.org/go.uber.org/cadence/internal#StartWorkflowOptions).

You can also start a :workflow: using the Cadence :CLI: with an optional cron schedule using the `--cron` argument.

## Samples

Runnable cron samples:

| Sample | Description | Code |
|--------|-------------|------|
| **Distributed cron** | Recurring workflow driven by the CronSchedule start option | [cron](https://github.com/cadence-workflow/cadence-samples/tree/master/cmd/samples/cron) |
| **Schedules** | Schedule operations, the recommended successor to cron for new use cases | [schedule](https://github.com/cadence-workflow/cadence-samples/tree/master/new_samples/schedule) |

---

For :workflow:workflows: with CronSchedule:

* Cron schedule is based on UTC time. For example cron schedule "15 8 \* \* \*"
  will run daily at 8:15am UTC. Another example "*/2 * * * 5-6" will schedule a workflow every two minutes on fridays
  and saturdays.
* If a :workflow: failed and a RetryPolicy is supplied to the StartWorkflowOptions
  as well, the :workflow: will retry based on the RetryPolicy. While the :workflow: is
  retrying, the server will not schedule the next cron run.
* Cadence server only schedules the next cron run after the current run is
  completed. By default, if the next schedule is due while a :workflow: is running (or retrying),
  then it will skip that schedule. To start one late run when a run overruns its
  schedule, set a [cron overlap policy](#cron-overlap-policy).
* Cron :workflow:workflows: will not stop until they are terminated or cancelled.

Cadence supports the standard cron spec:

```go
// CronSchedule - Optional cron schedule for workflow. If a cron schedule is specified, the workflow will run
// as a cron based on the schedule. The scheduling will be based on UTC time. The schedule for next run only happen
// after the current run is completed/failed/timeout. If a RetryPolicy is also supplied, and the workflow failed
// or timed out, the workflow will be retried based on the retry policy. While the workflow is retrying, it won't
// schedule its next run. If next schedule is due while the workflow is running (or retrying), then it will skip that
// schedule. Cron workflow will not stop until it is terminated or cancelled (by returning cadence.CanceledError).
// The cron spec is as following:
// ┌───────────── minute (0 - 59)
// │ ┌───────────── hour (0 - 23)
// │ │ ┌───────────── day of the month (1 - 31)
// │ │ │ ┌───────────── month (1 - 12)
// │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
// │ │ │ │ │
// │ │ │ │ │
// * * * * *
CronSchedule string
```

Cadence also supports more [advanced cron expressions](https://pkg.go.dev/github.com/robfig/cron#hdr-CRON_Expression_Format).

The [crontab guru site](https://crontab.guru/) is useful for testing your cron expressions.

## Cron overlap policy

The `CronOverlapPolicy` field of `StartWorkflowOptions` decides what the server does when a run closes after its next scheduled time has already passed, because the run took longer than the interval between two scheduled times.

| Policy | What happens when a run closes after its next scheduled time |
|--------|----------------------------------------------------------------|
| `shared.CronOverlapPolicySkipped` | The scheduled times that passed while the run was open are skipped. The next run starts at the first scheduled time after the run closes. This is the default. |
| `shared.CronOverlapPolicyBufferone` | The next run starts as soon as the run closes. The server starts only one late run, however many scheduled times passed. |

For example, take the cron schedule `*/5 * * * *` and a run that starts at 10:00 and closes at 10:08. With `CronOverlapPolicySkipped`, the 10:05 run is skipped and the next run starts at 10:10. With `CronOverlapPolicyBufferone`, the next run starts at 10:08, and the run after that is due at 10:10. If the 10:08 run is still open at 10:10, the next run starts as soon as it closes. If the first run instead closed at 10:17, `CronOverlapPolicyBufferone` would still start just one run at 10:17 and then continue at 10:20.

```go
import (
    "time"

    "go.uber.org/cadence/.gen/go/shared"
    "go.uber.org/cadence/client"
)

workflowOptions := client.StartWorkflowOptions{
    ID:                           "inventory-sync",
    TaskList:                     "inventory-tasklist",
    ExecutionStartToCloseTimeout: time.Hour,
    CronSchedule:                 "*/5 * * * *",
    CronOverlapPolicy:            shared.CronOverlapPolicyBufferone,
}
we, err := cadenceClient.StartWorkflow(ctx, workflowOptions, InventorySyncWorkflow)
```

With the :CLI:, pass `--cron_overlap_policy` (or `--cop`) to `cadence workflow start` or `cadence workflow run`. Use `0` for skip and `1` for buffer one:

```bash
cadence --domain samples-domain workflow start --tl inventory-tasklist --wt main.InventorySyncWorkflow --et 3600 --cron "*/5 * * * *" --cron_overlap_policy 1
```

Keep the following in mind:

* The policy is set when the cron :workflow: starts. The server passes it on to every later run, including retries.
* For a run that was retried, the next scheduled time is the first one after the last attempt started. Scheduled times that passed during earlier attempts are skipped with either policy.
* `JitterStart` still applies. With `CronOverlapPolicyBufferone`, the late run starts at a random point up to `JitterStart` after the previous run closes.
* `SignalWithStartWorkflow` accepts the same option. `ChildWorkflowOptions` has no `CronOverlapPolicy` field, so cron child :workflow:workflows: use the default.
* The policy needs Go client v1.3.1 or later and server v1.3.3 or later.

## Convert existing cron workflow

Before CronSchedule was available, the previous approach to implementing cron
:workflow:workflows: was to use a delay timer as the last step and then return
`ContinueAsNew`. One problem with that implementation is that if the :workflow:
fails or times out, the cron would stop.

To convert those :workflow:workflows: to make use of Cadence CronSchedule, all you need is to
remove the delay timer and return without using
`ContinueAsNew`. Then start the :workflow: with the desired CronSchedule.


## Retrieve last successful result

Sometimes it is useful to obtain the progress of previous successful runs.
This is supported by two new APIs in the client library:
`HasLastCompletionResult` and `GetLastCompletionResult`. Below is an example of how
to use this in Go:

```go
func CronWorkflow(ctx workflow.Context) (CronResult, error) {
    startTimestamp := time.Time{} // By default start from 0 time.
    if workflow.HasLastCompletionResult(ctx) {
        var progress CronResult
        if err := workflow.GetLastCompletionResult(ctx, &progress); err == nil {
            startTimestamp = progress.LastSyncTimestamp
        }
    }
    endTimestamp := workflow.Now(ctx)

    // Process work between startTimestamp (exclusive), endTimestamp (inclusive).
    // Business logic implementation goes here.

    result := CronResult{LastSyncTimestamp: endTimestamp}
    return result, nil
}
```

Note that this works even if one of the cron schedule runs failed. The
next schedule will still get the last successful result if it ever successfully
completed at least once. For example, for a daily cron :workflow:, if the first day
run succeeds and the second day fails, then the third day run will still get
the result from first day's run using these APIs.
