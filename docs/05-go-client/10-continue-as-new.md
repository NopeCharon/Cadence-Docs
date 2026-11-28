---
layout: default
title: Continue as new
description: This page explains how to use ContinueAsNew in Cadence to restart a workflow with a fresh history, preventing event history from growing too large in long-running periodic workflows, and how to keep a cron schedule across ContinueAsNew with WithCronSchedule.
keywords:
  - cadence continue as new
  - ContinueAsNew
  - workflow history limit
  - periodic workflow
  - long running workflow
  - ContinueAsNewError
  - WithCronSchedule
  - cron workflow continue as new
  - go client
  - cadence go continue as new tutorial
permalink: /docs/go-client/continue-as-new
---

:workflow:Workflows: that need to rerun periodically could naively be implemented as a big **for** loop with
a sleep where the entire logic of the :workflow: is inside the body of the **for** loop. The problem
with this approach is that the history for that :workflow: will keep growing to a point where it
reaches the maximum size enforced by the service.

**ContinueAsNew** is the low level construct that enables implementing such :workflow:workflows: without the
risk of failures down the road. The operation atomically completes the current execution and starts
a new execution of the :workflow: with the same **:workflow_ID:**. The new execution will not carry
over any history from the old execution. To trigger this behavior, the :workflow: function should
terminate by returning the special **ContinueAsNewError** error:

```go
func SimpleWorkflow(workflow.Context ctx, value string) error {
    ...
    return workflow.NewContinueAsNewError(ctx, SimpleWorkflow, value)
}
```

## Samples

Samples that use continue-as-new:

| Sample | Description | Code |
|--------|-------------|------|
| **Child workflow loop** | Child workflow that runs its iterations through continue-as-new | [childworkflow](https://github.com/cadence-workflow/cadence-samples/tree/master/new_samples/childworkflow) |
| **Long optimization loop** | Particle swarm optimization that resets history between iterations | [pso](https://github.com/cadence-workflow/cadence-samples/tree/master/cmd/samples/pso) |

## Continue as new from a cron workflow

Continue-As-New does not carry over the cron schedule. When a :workflow: that was started with a `CronSchedule` returns a `ContinueAsNewError`, the new run starts immediately instead of at the next scheduled time, and it is not a cron run. After it closes, the server does not start another run, so the cron :workflow: stops. The new run also starts without a last completion result, so `HasLastCompletionResult` returns false in it.

Starting with Go client v1.4.0, `workflow.WithCronSchedule` sets the cron schedule of the new run. Pass the current schedule to keep it, or a different spec to change it. For example, this :workflow: moves itself to a new schedule when its configuration changes:

```go
func ReportWorkflow(ctx workflow.Context, reportID string) error {
    ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        ScheduleToStartTimeout: time.Minute,
        StartToCloseTimeout:    10 * time.Minute,
    })

    var config ReportConfig
    if err := workflow.ExecuteActivity(ctx, LoadReportConfig, reportID).Get(ctx, &config); err != nil {
        return err
    }

    info := workflow.GetInfo(ctx)
    if info.CronSchedule != nil && *info.CronSchedule != config.CronSchedule {
        ctx = workflow.WithCronSchedule(ctx, config.CronSchedule)
        return workflow.NewContinueAsNewError(ctx, ReportWorkflow, reportID)
    }

    return workflow.ExecuteActivity(ctx, GenerateReport, reportID, config).Get(ctx, nil)
}
```

The run that Continue-As-New starts still begins immediately. After it closes, the server starts the following runs on the schedule passed to `WithCronSchedule`. Continue-As-New does not carry over `JitterStart` or `CronOverlapPolicy` from the start options either, so these runs have no jitter and use the default overlap policy. `NewContinueAsNewError` panics if the cron spec is invalid.

A cron :workflow: that only needs to run on its schedule does not need Continue-As-New. Return from each run and let the server start the next one, as described in [Distributed CRON](/docs/go-client/distributed-cron).

`WithCronSchedule` also sets the cron schedule of child :workflow:workflows: started with the returned context. A non-empty `ChildWorkflowOptions.CronSchedule` replaces it, and `WithCronSchedule(ctx, "")` clears it.
