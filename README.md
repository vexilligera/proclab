# OS Lab 2

### 说明

在minix 3.3.0操作系统系统PCB中添加计时器，到时间后发送信号终止进程。将调用过chrt的进程优先级记为7，每次挑选最早结束的进程执行。

### 应用层

1.添加函数声明 /usr/src/include/unistd.h

```c
int    chrt(long);
```

2.添加函数实现 /usr/src/minix/lib/libc/sys/chrt.c

```c
#include <sys/cdefs.h>
#include "namespace.h"
#include <lib.h>
#include <string.h>
#include <unistd.h>

int chrt(long deadline)
{
	message m;
	memset(&m, 0, sizeof(m));
	m.m2_l1 = deadline;
	return _syscall(PM_PROC_NR, PM_CHRT, &m);
}
```

3.修改Makefile文件 /usr/src/minix/lib/libc/sys/Makefile.inc

```makefile
SRCS+= ... fork.c chrt.c\ ...
```

### 服务层

1.进程管理服务原型 /usr/src/minix/servers/pm/proto.h

```c
...
/* misc.c */
...
int do_chrt(void);
...
```

2.服务实现 /usr/src/minix/servers/pm/misc.c

```C
int do_chrt(void)
{
  return sys_chrt(who_p, m_in.m2_l1);
}
```

3.定义服务号 /usr/src/minix/include/minix/callnr.h

```c
...
#define PM_CHRT				(PM_BASE + 48)

#define NR_PM_CALLS		49	/* highest number from base plus one */
...
```

4.映射服务到服务号  /usr/src/minix/servers/pm/table.c

```c
...
int (* const call_vec[NR_PM_CALLS])(void) = {
    ...
    CALL(PM_GETSYSINFO)	= do_getsysinfo,	/* getsysinfo(2) */
	CALL(PM_CHRT) = do_chrt
};
...
```

5.声明系统调用  /usr/src/minix/include/minix/syslib.h

```c
int sys_chrt(endpoint_t proc_ep, long deadline);
```

6.实现系统调用  /usr/src/minix/lib/libsys/sys_chrt.c

```c
#include "syslib.h"

int sys_chrt(endpoint_t proc_ep, long deadline)
{
	message m;
	m.m2_l1 = deadline;
	m.m2_i1 = proc_ep;
	return (_kernel_call(SYS_CHRT, &m));
}
```

7.修改Makefile文件

```makefile
SRC+=  \
	...
	sys_chrt.c \
	...
```

### 内核层

1.声明do_chrt  /usr/src/minix/kernel/system.h

```c
...
int do_chrt(struct proc * caller, message *m_ptr);

#endif	/* SYSTEM_H */
```

2.实现do_chrt /usr/src/minix/kernel/system/do_chrt.c

```c
#include "kernel/system.h"
#include <minix/timers.h>
#include "kernel/clock.h"
#include <minix/endpoint.h>
#include <signal.h>

static void chrt_watchdog(minix_timer_t *tp)
{
	struct proc *rp;
	int proc_nr;
	rp = proc_addr(tp->tmr_arg.ta_int);
	/* if process is alive, abort */
	if (isokendpt(rp->p_endpoint, &proc_nr)) {
		rp = proc_addr(proc_nr);	
		cause_sig(rp->p_nr, SIGABRT);
	}
}

int do_chrt(struct proc *caller_ptr, message *m_ptr)
{
	struct proc *rp;
	minix_timer_t *tp;
	long deadline;
	deadline = m_ptr->m2_l1;
	rp = proc_addr(m_ptr->m2_i1);
	tp = &rp->ddl_timer;

	if (deadline < 0)
		return -1;
	/* dequeue to reschedule */
	RTS_SET(rp, RTS_NO_QUANTUM);
	/* chrt already called */
	if (tp->tmr_exp_time != 0)
	{
		reset_kernel_timer(tp);
		tp->tmr_exp_time = 0;
	}
	/* set timer */
	tp->tmr_arg.ta_int = m_ptr->m2_i1;
	tp->tmr_exp_time = system_hz * deadline + get_monotonic();
	tp->tmr_func = (tmr_func_t)chrt_watchdog;
	set_kernel_timer(tp, tp->tmr_exp_time, chrt_watchdog); 
	/* enqueue the process*/
	RTS_UNSET(rp, RTS_NO_QUANTUM);
	return (OK);
}
```

3.修改do_exit实现，回收退出进程的计时器 /usr/src/minix/kernel/system/do_exit.c

```c
/* The kernel call implemented in this file:
 *   m_type:	SYS_EXIT
 */
#include "kernel/system.h"
#include "kernel/clock.h"
#include <signal.h>

#if USE_EXIT

/*===========================================================================*
 *				 do_exit				     *
 *===========================================================================*/
int do_exit(struct proc * caller, message * m_ptr)
{
/* Handle sys_exit. A system process has requested to exit. Generate a
 * self-termination signal.
 */
  int sig_nr = SIGABRT;

  cause_sig(caller->p_nr, sig_nr);      /* send a signal to the caller */
  reset_kernel_timer(&caller->ddl_timer);	/* clear chrt timer */
  caller->ddl_timer.tmr_exp_time = 0;
  return(EDONTREPLY);			/* don't reply */
}

#endif /* USE_EXIT */
```

4.修改Makefile文件 /usr/src/minix/kernel/system/Makefile.inc

```makefile
SRCS+= 	\
	...
	do_chrt.c
```

5.定义内核调用号  /usr/src/minix/include/minix/com.h

```c
...
#  define SYS_CHRT (KERNEL_CALL + 58) /* sys_chrt() */
/* Total */
#define NR_SYS_CALLS	59	/* number of kernel calls */
...
```

6.映射内核调用到内核调用号  /usr/src/minix/kernel/system.c

```c
  ...
  /* Process management. */
  ...
  map(SYS_CHRT, do_chrt);       /* chrt */
  ...
```

7.映射内核调用到system_tab /usr/src/minix/commands/service/parse.c

```c
...
struct
{
	char *label;
	int call_nr;
} system_tab[]=
{
	...
	{ "CHRT",			SYS_CHRT},
	{ NULL,		0 }
};
...
```

### 计时器与调度算法

1.修改进程结构体添加计时器 /usr/src/minix/kernel/proc.h

```c
struct proc {
  ...
  minix_timer_t ddl_timer;		/* timer for chrt deadline */
  ...
};
```

2.修改入队操作，将chrt进程优先级记作7 /usr/src/minix/kernel/proc.c

```c
...
void enqueue(
  register struct proc *rp	/* this process is now runnable */
)
{
  if (rp->ddl_timer.tmr_exp_time > 0)
  	rp->p_priority = 7;
  ...
}
...
static void enqueue_head(struct proc *rp)
{
  if (rp->ddl_timer.tmr_exp_time > 0)
  	rp->p_priority = 7;
  ...
}
...
```

3.修改调度算法，实现近似EDF /usr/src/minix/kernel/proc.c

```c
...
static struct proc * pick_proc(void)
{
/* Decide who to run now.  A new process is selected an returned.
 * When a billable process is selected, record it in 'bill_ptr', so that the 
 * clock task can tell who to bill for system time.
 *
 * This function always uses the run queues of the local cpu!
 */
  register struct proc *rp;			/* process to run */
  struct proc **rdy_head;
  struct proc *tmp;
  int q;				/* iterate over queues */

  /* Check each of the scheduling queues for ready processes. The number of
   * queues is defined in proc.h, and priorities are set in the task table.
   * If there are no processes ready to run, return NULL.
   */
  rdy_head = get_cpulocal_var(run_q_head);
  for (q=0; q < NR_SCHED_QUEUES; q++) {	
	if(!(rp = rdy_head[q])) {
		TRACE(VF_PICKPROC, printf("cpu %d queue %d empty\n", cpuid, q););
		continue;
	}
	if (q == 7) {
		// loop out the earliest deadline process and pick it
		rp = rdy_head[q];
		tmp = rp->p_nextready;
		while (tmp) {
			if (tmp->ddl_timer.tmr_exp_time > 0 && (rp->ddl_timer.tmr_exp_time == 0
					|| (rp->ddl_timer.tmr_exp_time > 0
					&& tmp->ddl_timer.tmr_exp_time < rp->ddl_timer.tmr_exp_time)))
				rp = tmp;
			tmp = tmp->p_nextready;
		}
	}
	assert(proc_is_runnable(rp));
	if (priv(rp)->s_flags & BILLABLE)	 	
		get_cpulocal_var(bill_ptr) = rp; /* bill for system time */
	return rp;
  }
  return NULL;
}
...
```

### 测试

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t pid;
    chrt(5);
    if (!(pid = fork())) {
        chrt(5);
        sleep(1);
        exit(0);
    }
    sleep(10);
    return 0;
}
```

正常运行。父进程被终止，子进程正常退出。
