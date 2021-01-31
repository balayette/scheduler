% Replacing the Linux Scheduler
% Mathieu Nativel, Nicolas Manichon, Thibault Vivies, Tom Rollet

## Why?

* ... FIXME ...
* Why not?

## The "generic" scheduler interface

Scheduler classes list

```c
#define SCHED_DATA				\
	STRUCT_ALIGN();				\
	__begin_sched_classes = .;		\
	*(__idle_sched_class)			\
	*(__fair_sched_class)			\
	*(__rt_sched_class)			\
	*(__dl_sched_class)			\
	*(__stop_sched_class)			\
	__end_sched_classes = .;
```

## The "generic" scheduler interface

```c
void enqueue_task(struct rq *, struct task_struct *, int);
void dequeue_task(struct rq *, struct task_struct *, int);
void check_preempt_curr(struct rq *, struct task_struct *, int);
struct task_struct *pick_next_task(struct rq *);
void put_prev_task(struct rq *, struct task_struct *);
void set_next_task(struct rq *, struct task_struct *, bool);
void task_tick(struct rq *, struct task_struct *, int);
void prio_changed(struct rq *, struct task_struct *, int);
```
