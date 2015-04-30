#实验五 用户进程管理

##练习零 填写已有实验
[练习0]本实验一览实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有LAB1/LAB2/LAB3/LAB4的注释相应部分。注意：为了能够正确执行LAB5的测试应用程序，可能需对已完成的实验1/2/3/4的代码进行进一步的改进。
>见代码

##练习一 加载应用程序并执行
>设计实现过程：load_icode()主要是加载用户elf文件并执行，我们所要做的就是设置trapframe中相应的一些值，以便从内核态跳到用户态去执行，需要设置的大致有tf_cs,tf_ds,tf_ss,tf_esp,tf_eip,tf_eflags.设置的值根据代码注释获得

[练习1]请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（running态）到具体执行应用程序第一条指令的整个经过。
```
在load_icode中我们加载了用户程序并且设置了trapframe中相关的一些参数，进行SYS_exec系统调用之后，使用iret指令返回继续执行trapframe中tf_eip指向的地址，而这里我们设置的就是用户程序的入口地址，这样就可以执行用户程序了。
```
##练习二 父进程复制自己的内存空间给子进程 
>设计实现过程：填写copy_range函数，将父进程的内存地址空间中合法内容复制到子进程。首先获得源地址和目的地址的内核虚地址，然后利用memcpy函数实现拷贝，最后建立映射关系

[练习2]简要说明如何设计实现“Copy On Write”机制
```
在dup_mmap中利用vma指针进行访问，而当某使用者需要对该资源进行写操作时，新建一个vma，并调用copy_range复制数据，对该结构体进行写操作，代码如下：
```
int dup_mmap(struct mm_struct *to, struct mm_struct *from) {
    assert(to != NULL && from != NULL);
    list_entry_t *list = &(from->mmap_list), *le = list;
    while ((le = list_prev(le)) != list) {
        struct vma_struct *vma, *nvma;
        vma = le2vma(le, list_link);
        nvma = vma_create(vma->vm_start, vma->vm_end, vma->vm_flags);
        if (nvma == NULL) 
            return -E_NO_MEM;
        insert_vma_struct(to, nvma);
        bool share = 0;
        if(copy_range(to->pgdir,from->pgdir,vma->vm_start,vma->vm_end,share)!= 0)
            return -E_NO_MEM;
    }
    return 0;
}
```

##练习三 阅读分析代码，理解进程执行fork/exec/wait/exit的实现，以及系统调用的实现

[练习3]请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的
```
fork创建子进程，并复制mm等数据，通过wakeup_proc()变为RUNNABLE,exec时，进程一直为RUNNING状态；wait时，若存在子进程，则变为SLEEPING状态，等待被唤醒；exit时，变为ZOMBIE状态
```

