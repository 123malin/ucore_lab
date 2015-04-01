## 第二次实验报告

##练习一 实现first-fit连续物理内存分配算法
>default_init:初始话free_list，并且将nr_free置为0，其中free_list是一个空闲块的双向链表，按物理地址排序，nr_free为空闲页个数
>default_init_memmap：此函数的功能就是用一个页基址和页个数初始话一个空闲块，首先初始化每个页，设置一些相关参数，然后将页加入free_list中，同时设置nr_free
>default_alloc_pages:此函数功能为用first_fit方法分配一块大小为n的空闲块，大致流程为查找free_list知道得到一块的大小>n，然后依次设置这些页的一些标志位，同时将超出的部分作为一个新块，更新其相关参数，最后nr_free减n，并返回分配内存的低地址
>default_free_pages：此函数为释放页，并将其重新加入free_list中，大致做法为根据释放地址，查询free_list，找到合适的插入点后插入释放页，并将临近可以合并的小块合并为一个大块。

[问题]你的first-fit算法是否有进一步的改进空间？

```
在分配空间的时候，比如要分配一个大小为n的块，现在在链表中查询，首次发现一个大于n的块，立即分配，并将大于n的部分重新作为一个新块，但这时候如果剩余块太小的话，可能会造成链表前段很多基本没用的小块，延长大块分配的查找时间，所以可以设定一个阈值，如果超出部分小于该值则做新的处理，或者直接都分配掉，或者加入新的链表，等到有块释放的时候，判断其是否可以merge回去。
```
## 练习二 实现寻找虚拟地址对应的页表项
>此功能主要在get_pte(pde_t *pgdir,uintptr_t la, bool create)函数中实现，其中pgdir为PDT的内核虚地址，la为需要匹配的虚地址，create为创建标志位
>首先找到待map虚地址的页目录表项，检查该项是否是present，如果不是，检查是否create，如果为0，则分配page，同时设置page reference，得到其虚地址后，清空page内容，设置PDT。
>返回PDT项

[练习2.1]请描述页目录项（Page Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

```
PTE每个占32位，其中20位为物理页号，剩余12位为一些标识位，例如有效位，修改位，读写/只读位，系统/用户位
PDE每个占32位，其中20位为该虚拟地址对应的页表基址，剩余12位也为一些标志位，基本同PTE。
可以找到在ucore的代码中有如下一些标识位信息：
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
                                                // hardware, so user processes are allowed to set them arbitrarily.
潜在用处：查询代码可以在mmu.h中找到对应PTE各个位的标识符号，其中write_to_cache和dirty等关键字表明，ucore可以进一步实现分页机制和cache功能的接合。
```
[练习2.2]如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

```
访问内存发生页访问异常时，硬件首先要通过CR0~CR3寄存器保存现场，设置相应位，然后跳转到该中断服务例程异常处理入口，之后交给操作系统，异常处理完成后恢复线程，继续刚才的指令向下运行
```
##练习三 释放某虚地址所在的页并取消对应二级页表项的映射
>检查ptep对应页表项是否present，是的话找到其中的页，将其引用计数减1。若减至0，则将该页释放。然>后清空该页表项，并设置tlb_invalidate。

[练习3.1]数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

```
Page的全局变量就表示的是物理内存中的页，也就是页表和页目录表中的表项指示的物理地址页，其中还可以通过以下函数转换成page
static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}

static inline struct Page *
pde2page(pde_t pde) {
    return pa2page(PDE_ADDR(pde));
}
```
[练习3.2]如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题。

```
即变成lab1的情形即可
```
