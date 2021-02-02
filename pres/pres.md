% Replacing the Linux Scheduler
% Mathieu Nativel, Nicolas Manichon, Thibault Vivies, Tom Rollet

## Why?

* ... FIXME ...
* Why not?

## Definitions

* runqueue: A data structure that contains the processes that are ready to run.

# Kernel data structures

## struct rq

Per-CPU runqueue. Actually just a wrapper around the scheduler-specific
runqueues.

`rq` doesn't actually implement a runqueue, and leaves that to the schedulers.

```c
struct rq {
	...
	unsigned int		nr_running;
	...
	struct cfs_rq		cfs;
	struct rt_rq		rt;
	struct dl_rq		dl;
	...
	struct task_struct __rcu	*curr;
	...
};
```

## struct task_struct

Huge structure that contains the entire state of the process.

```c
struct task_struct {
	...
	int on_rq;
	int prio;
	...
	const struct sched_class* sched_class;
	...
	struct sched_entity		se;
	struct sched_rt_entity		rt;
	struct sched_dl_entity		dl;
	...
};
```

## `sched_entities`

Per-scheduler metadata is stored in separate structures.

All schedulers need to implement the actual runqueue, `rq` is just a wrapper.

```c
struct sched_rt_entity {
	...
	struct list_head		run_list;
	unsigned long			timeout;
	unsigned int			time_slice;
	...
};
```

## Scheduler classes

Each scheduler has a `sched_class`.

```c
struct sched_class {
	void (*enqueue_task) (struct rq *rq, ...);
	void (*dequeue_task) (struct rq *rq, ...);
	void (*yield_task)   (struct rq *rq, ...);
	void (*check_preempt_curr)(struct rq *rq, ...);
	struct task_struct *(*pick_next_task)(struct rq *rq);
	...
};
```

The functions are called through `task_struct->sched_class`.

```c
void put_prev_task(struct rq *rq, struct task_struct *p)
{
	p->sched_class->put_prev_task(rq, p);
}
```

## The `sched_class` array

The classes used to be linked with a `next` pointer, but that was too simple.

Each class is stored in a different section, and the address of that section
is written to an array in a linker script.

```c
const struct sched_class fair_sched_class
	__section("__fair_sched_class") = {
	...
};

#define SCHED_DATA				\
	__begin_sched_classes = .;		\
	...					\
	*(__fair_sched_class)			\
	...					\
	__end_sched_classes = .;
```

# Scheduler functions

## `enqueue_task()`

```c
void enqueue_task(struct rq *rq, struct task_struct *p,
	int flags);
```

`enqueue_task` is called when:

* A task is created
* A task that was waiting/blocked is now ready

`enqueue_task` adds the task to the scheduler's runqueue.

## `dequeue_task()`

```c
void dequeue_task(struct rq *rq, struct task_struct *p,
	int flags);
```

`dequeue_task` is called when:

* A task that is running / was ready is now blocked/waiting.

`dequeue_task` removes the task from the scheduler's runqueue.

## `pick_next_task()`

```c
struct task_struct *pick_next_task(struct rq *rq);
```

`pick_next_task` is the core of a scheduler implementation.

It returns the task that should replace the task that is currently running, or
the current task if it can keep running.

If the scheduler has nothing to schedule, and the current task should not keep
running (it is blocked, for example), then `pick_next_task` returns `NULL` and
the kernel will ask another scheduler class for a process to run.

## `put_prev_task()`

```c
void put_prev_task(struct rq *rq,
	struct task_struct *prev);
```

`put_prev_task` is called when a task that was on the CPU is kicked out.

There are two cases that must be handled:

* `pick_next_task` decided that `prev` must stop running, even if it is not
blocked/waiting. Add `prev` to the runqueue
* `prev` is blocked/waiting, do not add it to the runqueue.

 
## `task_tick()`

```c
void task_tick(struct rq *rq, struct task_struct *p,
	int queued);
```

`task_tick` is called by the kernel and allows the scheduler to compute how
long a task has been running.

The scheduler can use this to ask the kernel to reschedule the current task
if it has been running for too long, using `resched_curr(rq)`.

## `check_preempt_curr()`

```c
static void check_preempt_curr(struct rq *rq,
	struct task_struct *p, int flags);
```

`check_preempt_curr` is called when a task is ready to run.

This allows the scheduler to ask the kernel to reschedule the current task
if its priority is lower than the new task.

## `prio_changed()`

```c
void prio_changed(struct rq *rq, struct task_struct *p,
	int oldprio);
```

`prio_changed` is called when a task changes priority.

If the task is ready to run, and its priority increased, the scheduler can
ask the kernel to reschedule.

# Implementation

## Scheduler entity

```diff
--- a/include/linux/sched.h
+++ b/include/linux/sched.h
@@ -581,6 +581,11 @@ struct sched_dl_entity {
+struct sched_fe_entity {
+	struct list_head list;
+	int time;
+};
```

`sched_fe_entity` contains a list node (`prev` and `next` point to other
tasks in the runqueue).
The `time` field is incremented during each call of `task_tick`.

```diff
@@ -695,6 +700,7 @@ struct task_struct {
 	struct sched_dl_entity		dl;
+	struct sched_fe_entity		fe;
```

## Remove `SCHED_NORMAL`

The default scheduler policy is `SCHED_NORMAL`, and is hardcoded in many places
in the kernel.

* `init_task.c`: Modify the initialization of `init_task`, a static structure
that represents the first task that runs.
* `kthread.c`: When a kernel thread is created, its policy is reset to
`SCHED_NORMAL`
* `sched/core.c`: On `fork`, the policy might be reset to `SCHED_NORMAL`.
* `sched/core.c`: Fair scheduler's `pick_next_task` implementation is called
directly in some cases, instead of going through
`task_struct->sched_class->pick_next_task`, as an optimization.

## Runqueue initialization

The function `sched_init` is called very early during the boot. It initializes
everything that the scheduler will need.

Add a call to our initialization routine.

```diff
@@ -7144,6 +7151,7 @@ void __init sched_init(void)
 		init_cfs_rq(&rq->cfs);
 		init_rt_rq(&rq->rt);
 		init_dl_rq(&rq->dl);
+		init_fe_rq(&rq->fe);
```

## Scheduler implementation

```diff
--- a/kernel/sched/Makefile
+++ b/kernel/sched/Makefile
@@ -23,7 +23,7
 obj-y += core.o loadavg.o clock.o cputime.o
-obj-y += idle.o fair.o rt.o deadline.o
+obj-y += idle.o fair.o rt.o deadline.o fair_enuf.o
 obj-y += wait.o wait_bit.o swait.o completion.o
```

* Add it to the scheduler Makefile
* Implement all the functions we talked about
* ???
* Profit

## ???

* The only documentation is the code, and nobody agrees on the internet.
* We had to discover basically everything we explained earlier
* Debugging a scheduler is not easy
* When your scheduler is broken, everything looks like a race condition

## Weird behaviors

Some things that are not mentionned anywhere:

* `enqueue_task` can be called with the currently running task.
* `pick_next_task` needs to call `put_prev_task`, because the kernel won't do it.
* `put_prev_task` is called by the kernel even when the task is not runnable.

Most of those make sense in hindsight, but can cause subtle bugs.
