
# DR sync point start up with Cylc 8.3

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
