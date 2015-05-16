#实验七

##练习零 填写已有实验
[练习0]本实验一览实验1/2/3/4/5/6。请把你做的实验1/2/3/4/5/6的代码填入本实验中代码中有LAB1/LAB2/LAB3/LAB4/LAB5/LAB6的注释相应部分。并确保编译通过。注意：为了能够正确执行LAB7的测试应用程序，可能需对已完成的实验1/2/3/4/5/6的代码进行进一步的改进。
```
使用meld已完成前6次lab的移植工作，部分需要更新如下所示：
(1). LAB1:EXERCISE3的kern/trap/trap.c中需要更新：
case IRQ_OFFSET + IRQ_TIMER:
    ticks++;
    assert(current!=NULL);
    run_timer_list();
    break;
去掉kern/schedule/sched.c中sched_class_proc_tick函数前的static
```

##练习一 理解内核级信号量的实现和基于内核级条件变量的哲学家就餐问题
```
信号量是一种同步互斥机制的实现，通过操作系统来保证对资源申请和释放的原子性，从而达到互斥访问的目的，在kern/sync/sem.h中可以看到实现信号量的结构体
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
其中value即为可用资源数目，wait_queue即为当前未能申请到资源的等待进程序列。对于信号量而言，最重要的操作应该就是P/V操作了，P为申请资源，V为释放资源，观察可以发现，在ucore中，P对应的即为down(semaphore_t *sem)函数，V对应的即为up(semaphore_t *sem)函数，而这两个函数的具体实现是在_down(semaphore_t *sem,uint32_t wait_state)函数和_up(semaphore_t *sem,uint32_t wait_state)函数
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
首先关掉中断，以此来保证操作的原子性；然后判断当前信号量的value值是否大于0,如果大于0,则value值减1后打开中断并返回；如果不大于0,则表明无法获得信号量，故需要将当前进程加入到等待队列中，并打开中断，然后运行调度器选择另外一个进程执行。其中wait_current_set函数是让wait与进程关联，且让当前进程关联的wait进入等待队列queue，当前进程睡眠。之后如果进程被V操作唤醒，则把自身关联的wait从该信号量的等待队列中删除，且该操作应该先关中断。
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
该函数实现V操作，首先也是关中断，如果信号量对应的wait_queue中没有进程在等待，直接把信号量的值加1,然后打开中断并返回；否则如果有进程在等待且让进程等待的原因是该信号量设置的，这调用wakeup_wait函数将waitqueue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。

在哲学家就餐问题中
int state_sema[N]; /* 记录每个人状态的数组 */
/* 信号量是一个特殊的整型变量 */
semaphore_t mutex; /* 临界区互斥 */
semaphore_t s[N]; /* 每个哲学家一个信号量 */
临界区用来保证只有一个哲学家在吃饭，主体过程为哲学家先思考，然后试图拿起两个筷子（以临界区的形式访问），如果能拿到则吃东西，然后退出临界区，放下两根筷子（也是以临界区形式访问），继续思考。
```
##练习二 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题 
```
ucore中管程的数据结构定义如下：
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
其中mutex是一个二进制信号量，是实现每次只允许一个进程进入管程的关键元素，确保了互斥访问性质。管程的条件变量cv通过执行wait_cv，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件C为真并执行signal_cv时，能够让等待条件C为真的睡眠进程被唤醒，从而继续在管程中执行。信号量next和整形变量next_count是配合进程对条件变量cv的操作而设置的，这是由于发出signal_cv的进程A会唤醒睡眠进程B，进程B的执行会导致A睡眠，直到B离开管程，进程A才能继续执行，这个同步过程是通过信号量next完成的，next_count表示了由于发出了signal_cv而睡眠的进程个数。
其中在ucore中，wait_cv和signal_cv的具体实现函数为cond_wait和cond_signal函数，具体实现如下：
cond_wait (condvar_t *cvp) {
	cvp->count++;
	if(cvp->owner->next_count>0)
	{
		up(&(cvp->owner->next));
	}else
	{
		up(&(cvp->owner->mutex));
	}
	down(&(cvp->sem));
	cvp->count--;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
首先在该条件变量上睡眠的进程数count加1,然后判断因为该条件满足而睡眠的进程数，如果大于0,这唤醒它们继续执行，否则唤醒因为互斥条件而无法进入管程的进程。然后阻塞到该条件变量上，等到执行完后将条件变量的等待进程数减1后返回。
void 
cond_signal (condvar_t *cvp) {
	if(cvp->count > 0)              //表明挂在该条件变量的等待队列不为空
	{
		cvp->owner->next_count++;
		up(&(cvp->sem));           //唤醒因为该条件而等待的进程
		down(&(cvp->owner->next));
		cvp->owner->next_count--;
	}
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
如果有进程执行cond_signal并且等待在该条件变量上的进程数大于0,则先将当前进程挂在monitor的next信号量上，然后唤醒挂在该条件变量上的进程，阻塞到monitor的next上，直到它执行了之后，将monitor的next_count数目减1后返回。

哲学家就餐问题中，通过管程只有一个进程能访问来限制拿筷子过程和放筷子过程都是互斥的，具体实现代码如下：
void phi_take_forks_condvar(int i) {
     down(&(mtp->mutex));
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: 2012011281
     // I am hungry
     // try to get fork
	state_condvar[i]=HUNGRY;
	phi_test_condvar(i); 
    while (state_condvar[i] != EATING) {
          cprintf("phi_take_forks_condvar: %d didn't get fork and will wait\n",i);
          cond_wait(&mtp->cv[i]);
    }
//--------leave routine in monitor--------------
      if(mtp->next_count>0)
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}
void phi_put_forks_condvar(int i) {
     down(&(mtp->mutex));
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: 2012011281
     // I ate over
     // test left and right neighbors
	state_condvar[i]=THINKING;
      // test left and right neighbors
      phi_test_condvar(LEFT);
      phi_test_condvar(RIGHT);
//--------leave routine in monitor--------------
     if(mtp->next_count>0)
        up(&(mtp->next));
     else
        up(&(mtp->mutex));
}
```

