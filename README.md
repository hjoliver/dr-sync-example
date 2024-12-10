# DR sync point start up with Cylc 8.3

## Principles

I need to start my workflow up on the DR platform, at the most recent previous
sync point (that is, previous with respect to the point that the workflow got
to on the main platform before the disaster occurred).

With Cylc 8 you can start a flow anywhere in the graph, not just at the start.

### Sync Points

A "sync point" is a point in the graph where we can start from because the
critical input data for the tasks there has been pre-copied to the DR platform.

The more sync points, the fewer tasks have to be re-run during DR fail-over.

When a sync point is reached the critical data should be copied to the DR platform,
and a DB (or similar) there updated to say that point is ready to go if needed.

The ideal sync point is a bottleneck in the graph:
```
a & b => c => d & e  # sync point c
# E.g. when c starts running, copy its input data update the DB
```

But if the graph has multiple concurrent streams, fixed sync points can be spread out:
```
a => b => c
x => y => z  # sync point (b, y)
# after b and y have both started, copy their input data and update the DB
```

Or you could have independent sync points on the different graph streams, if your
DR start-up logic can handle that:
```
a => b => c  # sync points a, c
x => y => z  # sync points x, z
```

### Bootstrapping off-flow dependence

If possbile, sync points should be chosen such that the triggered flow:
- will continue downstream to traverse the whole graph
- will not stall due to unsatisfied off-flow prerequisites 

To help with this, start the flow with `cylc play --start-task` (with multiple start tasks
if necessary) - Cylc will automatically ignore dependence on earlier cycle points
("pre-initial ignore").

But note you may need to:
- include upstream parentless tasks in the list of start-tasks 
- manually set off-flow prerequisites or outputs where pre-initial ignore doesn't help


## The example

Sync point (see `cgraph.png` with cycle point 2 -> 3):
- I can start from `3/a`, `3/x`, and `4/data`
- I need the start-up task 1/prep to run first
- No stall due to off-flow or inter-cycle dependencies

NOTE: note `cylc play --start-task` uses pre-initial ignore on intercycle dependencies.

1. start paused, with the start tasks in the pool waiting
```
$ cylc vip --pause ecx -t //3/a -t //3/x -t //4/data
```

2. unpause the workflow after holding the start tasks
```
$ cylc hold 'ecx//*'
$ cylc play ecx
```

3. run the start-up task without flow-on
```
$ cylc trigger --flow=none ecx//1/prep
```

4. unhold the waiting start tasks, to start the flow

```
$ cylc release "ecx//*"
```
