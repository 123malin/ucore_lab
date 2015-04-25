#实验四 内核线程管理

##练习一 分配并初始化一个进程控制块
>设计实现过程：根据注释对相关变量进行初始化即可

[练习1] 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？
```
context是进程的上下文，用于进程切换，在ucore中，所有进程在内核中也是相互独立的（独立的堆栈和上下文等等），使用context保存寄存器的目的就在于在内核态中能够进行上下文之间的切换。
tf是中断帧的指针，总是只想内核栈的某个位置，当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态，当进程需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。
```
##练习二 为新创建的内核线程分配资源
>设计实现过程：根据注释给新内核线程分配资源，并且复制原进程的状态，大致过程为：调用alloc_proc，获得一块用户信息块；为进程分配一个内核栈；复制原进程的内存管理信息到新进程；复制原进程上下文到新进程；将新进程添加到进程列表；唤醒新进程；返回新进程号

[练习2]请说明ucore是否做到给每一个新fork的线程一个唯一的id？请说明你的分析和理由
```
是的，分配id的函数为get_pid()，我们可以先来看一个此函数
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}	
其中last_pid为准备分配的id值，next_safe为下一个可以分配的值，即最小的大于last_pid的可用于分配的id值，通过遍历proc_list可以做到unique。
```

##练习三 阅读代码，理解proc_run函数和它调用的函数如何完成进程切换的
>load_esp0()函数将栈顶指向新进程的栈顶；lcr3()函数将页目录表的基址设成新进程的页目录表的基址；switch_to()将进程的context里面的保存值放到运行的栈中，并把被切换的进程的寄存器的值保存到context中。
[练习3.1]在本实验的执行过程中，创建且运行了几个内核线程？
```
2个内核线程，idleproc和initproc
```
[练习3.2]语句local_intr_save(intr_flag);...local_intr_restore(intr_flag);在这里有何作用？请说明理由
```
这2条语句用于关闭中断和恢复中断，在lab4中，ucore在向终端输出字符、分配物理页帧、释放页、进程调度时都会先关闭中断，操作完成后再开启中断。
```
