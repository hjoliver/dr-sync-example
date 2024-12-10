# DR sync point start up with Cylc 8.3

## Background

I need to (re)start my workflow on a DR platform, at a recent sync point.

With Cylc 8 you can start a flow anywhere in the graph. There's no need to go back
to the start of a cycle or back to a workflow database checkpoint (in fact Cylc 8
doesn't even have DB checkpoints - they're not needed).

### Sync Points

A "sync point" is a point in the graph where you can successfully trigger a flow
because critical input data for the tasks at that point have been pre-copied to
the platform.

The more sync points, the fewer tasks have to be re-run during DR fail-over.

When a sync point is reached the associated data should be copied to the DR platform,
and a DB (or similar) updated to say that point is ready to go if needed, along with
the task IDs that should be triggered to start the flow there.

The ideal sync point is a bottleneck in the graph:
```console
a & b => c => d & e  # sync point c
# When c starts running, copy its input data update the DB.
# If needed, trigger the flow on the DR platform at c.
```

If the graph has multiple concurrent streams, sync points can be spread out:
```console
a => b => c
x => y => z  # sync point (b, y)
# After b and y have both started, copy their input data and update the DB.
# If needed, trigger the flow on the DR platform at b and y.
```

Or you could have independent sync points on the different graph streams, if your
DR start-up logic can handle that:
```console
a => b => c  # sync points a, c
x => y => z  # sync points x, z
```

### Points to note

Choose sync points where it's easy to trigger a flow that traverses the entire
graph downstream.

This may require some care in a cycling graph with inter-cycle dependence and
some parentless tasks that must automatically spawn to the top of subsequent cycles.

To make this easier, start the flow with `cylc play --start-task` (with multiple start
tasks if necessary) and Cylc will automatically ignore any dependence on cycle points
prior to the first start-task. Note that this "pre-initial ignore" feature is global.
If you have multiple graph streams with different cycling intervals, or if you trigger
the flow manually with `cylc trigger` instead of with start-tasks, you may need to
use `cylc set` to set off-flow prerequisites or the upstream outputs that satisfy them,
to avoid stalling the workflow.


## The example

**N.B. this example is non-trivial but does not require any manual use of `cylc set`**
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

2. unpause the workflow after holding the start tasks
```console
$ cylc hold 'ecx//*'
$ cylc play ecx
```

3. run the start-up task without flow-on
```console
$ cylc trigger --flow=none ecx//1/prep
```

4. unhold the waiting start tasks, to start the flow

```console
$ cylc release "ecx//*"
```
