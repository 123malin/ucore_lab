#实验六 调度器

##练习零 填写已有实验
[练习0]本实验一览实验1/2/3/4/5。请把你做的实验1/2/3/4/5的代码填入本实验中代码中有LAB1/LAB2/LAB3/LAB4/LAB5的注释相应部分。并确保编译通过。并注意：为了能够正确执行LAB6的测试应用程序，可能需对已完成的实验1/2/3/4/5的代码进行进一步的改进。
```
使用meld已完成前5次lab的移植工作，部分需要更新如下所示：
(1). LAB1:EXERCISE3的kern/trap/trap.c中需要更新：
case IRQ_OFFSET + IRQ_TIMER:
    ticks++;
	assert(current!=NULL);
	sched_class_proc_tick(current);
    break;
然后修改 sched.c / sched.h 将 sched_class_proc_tick() 前面的 void 删去，因为在 trap.c 里要使用。
(2). LAB4:EXERCISE1的kern/process/proc.c中的alloc_proc需要更新：
//LAB6 2012011281 : (update LAB5 steps)
    /*
     * below fields(add in LAB6) in proc_struct need to be initialized
     *     struct run_queue *rq;                       // running queue contains Process
     *     list_entry_t run_link;                      // the entry linked in run queue
     *     int time_slice;                             // time slice for occupying the CPU
     *     skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
     *     uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process
     *     uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
     */
	proc->rq = NULL;
    list_init(&(proc->run_link));
    proc->time_slice = 0;
    proc->lab6_run_pool.left = proc->lab6_run_pool.parent = proc->lab6_run_pool.right = NULL;
    proc->lab6_stride = 0;
    proc->lab6_priority = 0;
```

##练习一 使用 Round Robin 调度算法（不需要编码）

[练习1.1]请理解并分析sched_class中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程
```
sched_class如下所示：
struct sched_class {
    const char *name;
    void (*init)(struct run_queue *rq);
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
name 为调度器的名字
(*init) 用于初始化 run_queue ；
(*enqueue) 用于向 run_queue 中添加一个就绪态的进程；
(*dequeue) 用于从 run_queue 中移除一个进程；
(*pick_next) 用于从 run_queue 中选取一个进程运行；
(*proc_tick) 作用就是在每次时钟中断到来时将当前进程的 time_slice 减一。

ucore的调度执行过程：在 do_exit() ，do_wait() ，init_main() ，cpu_idle() 中都会调用 schedule() ，schedule() 作用是将 current 进程放入 run_queue ，并从 run_queue 中选出 next 进程，并使用 proc_run() 运行 current 。
```

[练习1.2]请在实验报告中简要说明如何设计实现“多级反馈队列调度算法”，给出概要设计，鼓励给出详细设计
```
多级反馈队列调度算法：就是维护多条不同优先级的 run_queue 。新进程加入最顶层的 queue 尾部；队列头部的进程分配 CPU 运行；若进程在时间片用完之前退出，那么移出队列；若进程主动放弃CPU，移出队列。当进程再次就绪，放回到离开时的队列的队尾；若一个进程用完了时间片，它的优先级降低。将其插入低一级的队列的队尾；在最低级，进程按照 RR 算法调度，直至退出离开队列。
```
##练习二 实现 Stride Scheduling 调度算法（需要编码） 
```
参考注释与伪代码完成Stride Scheduling调度算法：
(1)stride_init(rq):初始化rq->run_list并且将rq->lab6_run_pool初始化为NULL，rq->proc_num初始化为0
static void
stride_init(struct run_queue *rq) {
     /* LAB6: 2012011281 
      * (1) init the ready process list: rq->run_list
      * (2) init the run pool: rq->lab6_run_pool
      * (3) set number of process: rq->proc_num to 0       
      */
	list_init(&(rq -> run_list));
	rq -> lab6_run_pool = NULL;
	rq -> proc_num = 0;
}
(2)stride_enqueue(rq, proc):初始化刚进入 run_queue 的进程 proc 的 stride 域，将 proc 插入 run_queue
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2012011281 
      * (1) insert the proc into rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_insert: insert a entry into skew_heap
      *         list_add_before: insert  a entry into the last of list   
      * (2) recalculate proc->time_slice
      * (3) set proc->rq pointer to rq
      * (4) increase rq->proc_num
      */
	assert(USE_SKEW_HEAP == 1);
	rq -> lab6_run_pool = skew_heap_insert(rq->lab6_run_pool,&(proc -> lab6_run_pool), proc_stride_comp_f);
	if(proc->time_slice == 0 || proc->time_slice > rq->max_time_slice)
	{
		proc->time_slice = rq->max_time_slice;
	}
	proc->rq = rq;
	rq->proc_num ++;
}
(3)stride_dequeue(rq, proc)：从 run_queue 中删除相应的元素
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2012011281 
      * (1) remove the proc from rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_remove: remove a entry from skew_heap
      *         list_del_init: remove a entry from the  list
      */
	assert(USE_SKEW_HEAP == 1);
	rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool,&(proc->lab6_run_pool),proc_stride_comp_f);
	rq->proc_num--;
}
(4)stride_pick_next(rq)：返回 run_queue 中 stride 最小的进程，更新对应进程的 stride
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: 2012011281 
      * (1) get a  proc_struct pointer p  with the minimum value of stride
             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
             (1.2) If using list, we have to search list to find the p with minimum stride value
      * (2) update p;s stride value: p->lab6_stride
      * (3) return p
      */
	assert(USE_SKEW_HEAP == 1);
	if(rq->lab6_run_pool == NULL){
		return NULL;
	}
	struct proc_struct *p = le2proc(rq->lab6_run_pool,lab6_run_pool);
	if(p->lab6_priority == 0){
		p->lab6_stride = p->lab6_stride + BIG_STRIDE;
	}else{
		p->lab6_stride = p->lab6_stride + BIG_STRIDE/p->lab6_priority;
	}
	return p;
}

(5)stride_proc_tick(rq, proc)：减小time_slice域。检测当前进程是否用完了分配的时间片，如果时间片用完，则需要设置标记引起进程切换。
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: 2012011281 */
	if(proc->time_slice > 0){
		proc->time_slice --;
	}
	if(proc->time_slice == 0){
		proc->need_resched = 1;
	}
}

```

