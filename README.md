# DR sync point start up with Cylc 8.3

## The Problem

I need to be able to (re)start my cycling workflow mid-run on a remote Disaster
Recovery (DR) platform. The run directory must contain the right files to allow
the worklow to start at the chosen point.

## The Solution

With Cylc 8 you can start a flow anywhere in the graph. There's no need to
refer to a workflow database checkpoint or go back to the start of a cycle.

You just have to ensure that the critical data is present on disk to allow
the flow to be started at planned sync points in the graph. 

The more sync points, the fewer tasks have to be re-run during DR fail-over.

A "sync workflow" could watch for marker tasks in your main workflow(s), copy
the associated data and then update a database to say that the sync point is
ready if needed, and the IDs of tasks to trigger to start the flow there.

(Note if data volumes are large and/or latency is significant it may not be
possible to rely on low-level global disk-sync to stay up to date with the
workflow states. Deliberate workflow-driven data copy guarantees that the
right data will be present on disk at designated sync points in the graph.) 

### Sync Points

The ideal sync point is a bottleneck in the graph:
```console
a & b => c => d & e  # sync point c
# When c starts running, copy its input data update the DB.
# If needed, trigger the flow on the DR platform at c.
```

If bottlenecks aren't available sync points can be spread over the graph:
```console
a => b => c
x => y => z  # sync point (b, y)
# After b and y have both started, copy their input data and update the DB.
# If needed, trigger the flow on the DR platform at b and y.
```

With appropriate start-up logic, you could even have independent sync points on
different graph paths:
```console
a => b => c  # graph path 1: sync points a, c
x => y => z  # graph path 2: sync points x, z
# If needed, trigger the flow at the most recent sync point on each path.
```

### Things to watch out for

Choose sync points with an understanding of the graph.

#### Off-flow prerequisites

Off-flow prerequisites reflect dependence on tasks that are not downstream of
the trigger point(s) - usually dependence on tasks in the previous cycle.
They have to be artificially satisfied to avoid stalling the workflow.

If you start the flow with `cylc play --start-task=ID1 --start-task=ID2 ...`
any dependence on cycle points prior to the first start-task will be ignored
automatically.

However this "pre-initial ignore" feature picks a single cycle point. If you
have multiple graph streams with different cycling intervals you may still need
to use `cylc set` to satisfy some off-flow prerequisites (or to complete the
upstream outputs that satisfy them) to avoid stalling the workflow.

Or, if you trigger the flow manually after start-up with `cylc trigger` instead
of using start tasks, you will need to use `cylc set` for all off-flow
prerequisites. (*Group trigger - coming in Cylc 8.5 - will make this easier.*)


#### Parentless tasks

Cycling workflows typically have clock-triggered or xtriggered parentless tasks
at the top of each cycle. These don't have parents to spawn them in the flow,
so the scheduler spawns them automatically into future cycle points.

If a sync point is at the start of a cycle if may be sufficient to simply
trigger the parentless task(s) to get the flow going. Otherwise you'll need to
include next-cycle parentless tasks as start-tasks to avoid a stall there
(once started they will spawn forward automatically).

If not included as start-tasks use `cylc set --pre=all` to spawn parentless
tasks into the active window and start checking their clock and xtriggers.

#### Dependence on initial prep tasks

If your workflow graph begins with tasks that prep the workspace and (e.g.)
build or deploy code for the other tasks, you may need to run these again
before starting the flow at the sync point.

To do this:
 1. start the workflow paused
 2. hold the waiting start-tasks, then resume (unpause) the workflow
 3. manually trigger the initial prep tasks with `--flow=none` to avoid
    flow-on, and wait for them to finish
 4. release (unhold) the start tasks

If you have a sub-graph of prep tasks with internal dependencies, group
trigger (8.5) will make that easier. For now, trigger them manually in the
right order with `--flow=none`, or trigger a new flow to traverse them
after pre-hold follow-on tasks to block flow-on beyond the prep tasks
(remove the held blocking tasks when prep is done).

## An example

**TODO: expand to several examples, both simpler and more difficult.**

Sync point (see `cgraph.png` with cycle point 2 -> 3):
- I can start from `3/a`, `3/x`, and `4/data`
- I need the start-up task 1/prep to run first
- No stall due to off-flow or inter-cycle dependencies

NOTE: note `cylc play --start-task` uses pre-initial ignore on intercycle dependencies.

1. start paused, with the start tasks in the pool waiting
```console
$ cylc vip --pause ecx -t //3/a -t //3/x -t //4/data
```

2. resume the workflow after holding the start tasks
```console
$ cylc hold 'ecx//*'
$ cylc play ecx
```

3. run the start-up task without flow-on
```console
$ cylc trigger --flow=none ecx//1/prep
```

4. release the waiting start tasks, to start the flow

```console
$ cylc release "ecx//*"
```
