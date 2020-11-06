[TOC]



# 练习1: 使用 Round Robin 调度算法（不需要编码）

完成练习0后，建议大家比较一下（可用kdiff3等文件比较软件）个人完成的lab5和练习0完成后的刚修改的lab6之间的区别，分析了解lab6采用RR调度算法后的执行过程。执行make grade，大部分测试用例应该通过。但执行priority.c应该过不去。

请在实验报告中完成：

-   请理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描ucore的调度执行过程
-   请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

```c

// The introduction of scheduling classes is borrrowed from Linux, and makes the 
// core scheduler quite extensible. These classes (the scheduler modules) encapsulate 
// the scheduling policies. 
// 通过函数指针模仿实现调度类
struct sched_class {
    // the name of sched_class
    const char *name;
    
    // Init the run queue
    void (*init)(struct run_queue *rq);
    
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    
    // choose the next runnable task/process
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    
    // dealer of the time-tick
    // 让调度算法感知到时钟中断的发生,然后执行schedule
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
    /* for SMP support in the future
     *  load_balance
     *     void (*load_balance)(struct rq* rq);
     *  get some proc from this rq, used in load_balance,
     *  return value is the num of gotten proc
     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
     */
};

// 先创建调度类,然后通过下面这个把调度类和具体调度算法连接起来
static struct sched_class *sched_class;
extern struct sched_class default_sched_class;
struct sched_class default_sched_class = {
    .name = "RR_scheduler",
    .init = RR_init,
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};

// Round Robin 结合schedule函数分析

static inline void
sched_class_enqueue(struct proc_struct *proc) {
    if (proc != idleproc) {			// 如果proc不是idleproc就将其入队
        sched_class->enqueue(rq, proc);
    }
}
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));					// 首先断点proc不再就绪队列里
    list_add_before(&(rq->run_list), &(proc->run_link));	// 加入到head的before
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;				// 如果时间片为0了 or 时间片过大了 就校准时间片
    }
    proc->rq = rq;											// 记录就绪队列
    rq->proc_num ++;										// num加1
}

static inline struct proc_struct *
sched_class_pick_next(void) {
    return sched_class->pick_next(rq);						// 返回选好的proc
}
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
    list_entry_t *le = list_next(&(rq->run_list));			// 直接选取就绪队列尾部！！的项
    if (le != &(rq->run_list)) {							// 如果是自己代表就绪队列不为空那就返回
        return le2proc(le, run_link);						
    }
    return NULL;											// 否则没找到下一个proc返回NULL
}

static inline void
sched_class_dequeue(struct proc_struct *proc) {
    sched_class->dequeue(rq, proc);
}
static void
RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);// 就绪队列不空且proc在就绪队列里
    list_del_init(&(proc->run_link));						 // 就绪队列删去proc并且就绪队列重新初始化
    rq->proc_num --;										 // 就绪队列数目减一
}

void
sched_class_proc_tick(struct proc_struct *proc) {			
    // 这个是在时钟中断中trap_dispatch调用run_timer_list再调用sched_class_proc_tick(current)使用的
    if (proc != idleproc) {									// 如果不是idleproc就对其时间片减1然后判0→schedule
        sched_class->proc_tick(rq, proc);					// 调用RR_proc_tick
    }
    else {
        proc->need_resched = 1;								
        // 如果是的话就置1然后接下来的中断处理例程会判断若1就仍然执行schedule
    }
}
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}


void
schedule(void) {
    bool intr_flag;
    struct proc_struct *next;
    local_intr_save(intr_flag);		// 关中断
    {
        current->need_resched = 0;	// 修改当前proc的need_resched为0
        if (current->state == PROC_RUNNABLE) {	// 如果当前进程是就绪的
            sched_class_enqueue(current);		// 将当前进程加入就绪队列的头(如果是idleproc就不入队)
        }
        if ((next = sched_class_pick_next()) != NULL) {	// 从就绪队列找下一个proc找不到就返回NULL
            sched_class_dequeue(next);					// 找到了就出队
        }
        if (next == NULL) {
            next = idleproc;							// 若找不到下一个就还是idleproc把
        }
        next->runs ++;									// 运行时间++
        if (next != current) {
            proc_run(next);								// 如果当前proc和下一个proc不是同一个那就运行下一个proc
        }
    }
    local_intr_restore(intr_flag);
}


```

### **多级反馈队列调度算法**

**参考**：https://www.cnblogs.com/ECJTUACM-873284962/p/11282800.html

```makefile
#1. 在 proc_struct 中添加总共 N 个多级反馈队列的入口，每个队列都有着各自的优先级，编号越大的队列优先级越低，并且优先级越低的队列上时间片的长度越大，为其上一个优先级队列的两倍；并且在 PCB 中记录当前进程所处的队列的优先级；
#2. 处理调度算法初始化的时候需要同时对 N 个队列进行初始化；
#3. 在处理将进程加入到就绪进程集合的时候，观察这个进程的时间片有没有使用完，如果使用完了，就将所在队列的优先级调低，加入到优先级低 1 级的队列中去，如果没有使用完时间片，则还是加入到当前优先级的队列中去；
#4. 在同一个优先级的队列内使用时间片轮转算法 OR FIFO也行；
#5. 在选择下一个执行的进程的时候，先看高优先级的队列中是否存在任务，如果不存在才转而向较低优先级的队列寻找；（有可能导致饥饿）
#	为了防止饥饿可以把进程运行时间也考虑到优先级判断中,运行时间越长优先级越低
#6. 从就绪进程集合中删除某一个进程就只需要在对应队列中删除即可；处理时间中断的函数不需要改变；
```

