#实验八

##练习零 填写已有实验
[练习0]本实验一览实验1/2/3/4/5/6/7。请把你做的实验1/2/3/4/5/6/7的代码填入本实验中代码中有LAB1/LAB2/LAB3/LAB4/LAB5/LAB6/LAB7的注释相应部分。并确保编译通过。注意：为了能够正确执行LAB8的测试应用程序，可能需对已完成的实验1/2/3/4/5/6/7的代码进行进一步的改进。
```
使用meld已完成前7次lab的移植工作，部分需要更新如下所示：
(1). kern/process/proc.c中LAB4:EXERCISE4需增加如下代码：
//LAB8:EXERCISE2 2012011281 HINT:need add some code to init fs in proc_struct, ...
	proc->filesp = NULL;
(2). kern/process/proc.c中LAB4:EXERCIES2需增加如下代码：
//LAB8
	if(copy_files(clone_flags,proc)!=0)
	{
		goto bad_fork_cleanup_kstack;
	}
```

##练习一 完成读文件操作的实现

>实现过程：sfs_io_nolock函数本质是在内存和磁盘上传输内容，即完成SFS层面上的文件读写
>第一步判断offset是否和第一块对齐，如果不对齐就要先把offset到第一块末尾中的内容读出来，利用sfs_bmap_load_nolock和sfs_buf_op函>数，前者是通过路径和inode中的逻辑块index找到磁盘上对应的块，后者是对buf进行对应的读写操作；
>第二步对对齐的块进行读写
>第三步判断最后一块是否对齐，不对齐同1一样，进行读写。具体实现见下：

```
if ((blkoff = offset % SFS_BLKSIZE) != 0) {                  //1 如果offset和第一块不对齐，则一直从offset先读到第一块末尾
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
            goto out;
        }
        alen += size;
        if (nblks == 0) {
            goto out;
        }
        buf += size, blkno ++, nblks --;
    }
    size = SFS_BLKSIZE;                  //2 读对齐的块
    while (nblks != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) {
            goto out;
        }
        alen += size, buf += size, blkno ++, nblks --;
    }

    if ((size = endpos % SFS_BLKSIZE) != 0) {         //如果结束位置不与最后一个块对齐，则按第一种不对齐的方式读
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
            goto out;
        }
        alen += size;
    }
```

>UNIX的pipe机制：管道是UNIX向应用软件提供的的进程间通信手段的一种，其中父进程与子进程，或者两个兄弟进程之间，可以通过系统调用建立起一个单向的通信管道。
>但是，这种管道只能由父进程来建立，所以对于子进程来说是静态的，与生俱来的。管道两端的进程各自将该管道视作一个文件。
>一个进程往通道中写的内容由另一个进程从通道读出，通过通道传递的内容遵循“先入先出”（FIFO）的规则。每个通道都是单向的，需要双向通信时要建立起两个通道。
>管道机制的主体是系统调用pipe()，但是由pipe（）所建立的管道的两端都在同一个进程中，这样的管道起不到进程间通信的作用。
>所以必须在fork（）的配合下，才能在父子进程间或者两个子进程之间建立起进程间的通信管道。下面就介绍一下怎样将管道用于进程间通信：
>（1）进程A创建了一个管道，创建完成时代表管道两端的两个已打开文件都在进程A中。
>(2)进程A通过frok（）创建出进程B，在fork（）的过程中进程A的打开文件表按原样复制到进程B中。
>(3)进程A关闭管道的读端，而进程B关闭管道的写段。于是，管道的写段在进程A中而读端在进程B中，成为了父子进程之间的通信管道。
>(4)进程A又通过frok（）创建进程C，而后关闭其管道写段而与管道脱离关系，使得管道的写段在进程C中而读端在进程B中，成为两个兄弟进程之间的管道。
>由于管道是一种“无名”，“无形”的文件，它可以通过fork（）的过程创建于“近亲” 的进程之间，而不能成为可以在任意两个进程之间建立通信的机制，
>更不可能成为一种一般的，通用的进程间通信模型，同时，管道机制的这种缺点本身强烈的暗示着人们，只要用“有名”，“有形”的文件来实现管道，就能克服这种缺点。
>所以有了管道之后，“命名管道”的出现时必然的。为了实现“命名管道”，在“普通文件”，“块设备文件”，“字符设备文件”之外，又设立了一种文件类型，称为FIFO文件。
>对这种文件的访问严格遵循“先进先出”的原则。这样就可以像在磁盘上建立一个文件一样建立一个命名管道

##练习二 完成基于文件系统的执行程序机制的实现
>实现过程：在练习0中已经补充了跟本次实验有关的其他函数的实现，所以在这只需要重写proc.c中的load_icode函数；
>首先利用mm_creat函数和setup_pgdir(mm)函数创建mm和页表项；
>然后利用load_icode_read函数加载elf文件，并判断格式是否正确；
>接着利用mm_map创建一个新的vma，利用pgdir_alloc_page为TEXT/DATA/BSS/stack分配内存；
>最后使用mm_map建立用户栈，并将参数压入栈中，更改cr3,切换页表，设置trapframe。

```
assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);

    if (current->mm != NULL) {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
    if ((mm = mm_create()) == NULL) {
        goto bad_mm;
    }
    if (setup_pgdir(mm) != 0) {
        goto bad_pgdir_cleanup_mm;
    }

    struct Page *page;

    struct elfhdr __elf, *elf = &__elf;
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {
        goto bad_elf_cleanup_pgdir;
    }

    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }

    struct proghdr __ph, *ph = &__ph;
    uint32_t vm_flags, perm, phnum;
    for (phnum = 0; phnum < elf->e_phnum; phnum ++) {
        off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;
        if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0) {
            goto bad_cleanup_mmap;
        }
        if (ph->p_type != ELF_PT_LOAD) {
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0) {
            continue ;
        }
        vm_flags = 0, perm = PTE_U;
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        if (vm_flags & VM_WRITE) perm |= PTE_W;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            goto bad_cleanup_mmap;
        }
        off_t offset = ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

        ret = -E_NO_MEM;

        end = ph->p_va + ph->p_filesz;
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0) {
                goto bad_cleanup_mmap;
            }
            start += size, offset += size;
        }
        end = ph->p_va + ph->p_memsz;

        if (start < la) {
            /* ph->p_memsz == ph->p_filesz */
            if (start == end) {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
    }
    sysfile_close(fd);

    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    //setup argc, argv
    uint32_t argv_size=0, i;
    for (i = 0; i < argc; i ++) {
        argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }

    uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
    char** uargv=(char **)(stacktop  - argc * sizeof(char *));
    
    argv_size = 0;
    for (i = 0; i < argc; i ++) {
        uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
        argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }
    
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;
    
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = stacktop;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
    ret = 0;
```
>链接就是可以指向文件系统中其他位置的一个快捷方式，它可以避免键入很长的路径名或cd深入到多个文件夹中，在UNIX中链接形式分为两种，分别为硬链接和软链接；
>硬链接的原文件和链接文件公用一个inode号，说明他们是同一个文件，实际上就是源文件的一个指针，不足之处在于不可以在不同文件系统的文件间建立链接，而且也只有超级用户才可以为目录建立硬链接。
>而软链接源文件和链接文件拥有不同的inode号，表明他们是两个不同的文件；在文件属性上软链接明确写出了是链接文件，而硬链接没有写出来，因为在本质上硬链接文件和原文见是完全平等关系；链接数目是一样的，软链接的链接数目不会增加；文件大小是不一样的；不足之处在于链接文件包含原文见的路径信息，如果原文见从一个目录下移到其他目录下，链接文件就找不到原文见了，而硬链接文件就没有这个问题。


