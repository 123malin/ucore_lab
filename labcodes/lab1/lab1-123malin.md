# Lab1 report

## 练习一 理解通过make生成执行文件的过程

[练习1.1] 操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释Makefile中每一条相关命令和命令参数的含
义,以及说明命令导致的结果)

```
	ucore.img生成后位于bin目录下，由makefile文件看出，要生成ucore.img文件，首先需要生成kernal和bootblock这两个文件，makefile中相关文件如下：
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
	为生成bootblock文件，需先生成bootmain.o,bootasm.o，sign文件，即位于目录/boot下的makefile中相关代码如下：
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
由上我们可以看出，对于bootfiles中要生成的文件，即bootmain.o bootasm.o生成语句为:$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc)),其中生成bootasm.o需要bootasm.S。实际命令为：gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \-c boot/bootasm.S -o obj/boot/bootasm.o 生成bootmain.o需要bootmain.c，实际命令为：gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \-fno-stack-protector -Ilibs/ -Os -nostdinc \-c boot/bootmain.c -o obj/boot/bootmain.o
而对于sign文件，生成它的makefile代码为
		$(call add_files_host,tools/sign.c,sign,sign)
		$(call create_target_host,sign,sign)
实际命令为：gcc -Itools/ -g -Wall -O2 -c tools/sign.c \-o obj/sign/tools/sign.o
		   gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
在这三个文件的基础上，首先利用bootmain.o和bootasm.o文件生成bootblock.o
		ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
		obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
然后复制二进制代码bootblock.o到bootblock.out实际命令为：objcopy -S -O binary obj/bootblock.o obj/bootblock.out
最后利用sign工具处理bootblock.out，生成bootblock
bin/sign obj/bootblock.out bin/bootblock
	为生成kernel文件,相关makefile代码为
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
可以看出为了生成kernel，首先需要生成 kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.otrapentry.o vectors.o pmm.o  printfmt.o string.o 其中kernel.ld已存在，而对于obj/kern/*/*.o这些.o文件的相关makefile代码为
		$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
		$(KCFLAGS))
实际指令基本一致，如init.o文件生成的实际指令为：gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \-gstabs -nostdinc  -fno-stack-protector \-Ilibs/ -Ikern/debug/ -Ikern/driver/ \-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \-o obj/kern/init/init.o
从而生成kernal的指令为：
	ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
	obj/kern/init/init.o obj/kern/libs/readline.o \
	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
	obj/kern/driver/clock.o obj/kern/driver/console.o \
	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
	obj/libs/printfmt.o obj/libs/string.o
接下来执行dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc   
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
意思即为生成一个有10000个块的文件，每个块默认512字节，用0填充，把bootblock中的内容写到第一个块，从第二个块开始写kernel中的内容

```

[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

```
	从sign.c的代码来看，一个磁盘主引导扇区只有512字节。且最后两个分别为0x55和0xAA，相关代码为：buf[510] = 0x55;buf[511] = 0xAA;

```


## 练习二 使用qemu执行并调试lab1中的软件

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

```
	修改makefile文件，增加如下内容到# files for grade script位置
lab1-mon: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"
debug-mon: $(UCOREIMG)
#	$(V)$(QEMU) -S -s -monitor stdio -hda $< -serial null &
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/moninit"
其主要是方便我们实验一的调试；在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中，同时为方便之后能区分出我每个实验做的事情，我将lab1中我的gdbinit另外写到一个文件中，命名为lab1init，这样当执行make lab1-mon命令时，就会执行上述命令
然后为实现单步跟踪BIOS，我们需更改lab1init中内容，具体lab1init中的内容为：
set architecture i8086
target remote :1234
然后在命令行执行make lab1-mon就可以进入到gdb的调试界面(gdb)，输入si之后就可以单步跟踪BIOS了，如果想查看指令的话，可以利用x /2i $pc这条指令，其中2为你想查看的指令个数。

```

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。

```
这道题和练习2.1基本一致，不同之处在于lab1init中的内容有所区别，本题中lab1init中的内容为
file bin/kernel
target remote :1234
set architecture i8086
b *0x7c00
continue
x /2i $pc
接下来同上题

```

[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

```
	lab1init中内容同练习2.2中，但为了对比更具说服力，可以多列出一些指令进行比较，这主要时更改x /2i $pc中的数字，运行完之后q.log中相关部分文件为：
----------------
	IN: 
	0x00007c00:  cli    
	
	----------------
	IN: 
	0x00007c01:  cld    
	0x00007c02:  xor    %ax,%ax
	0x00007c04:  mov    %ax,%ds
	0x00007c06:  mov    %ax,%es
	0x00007c08:  mov    %ax,%ss
	
	----------------
	IN: 
	0x00007c0a:  in     $0x64,%al
	
	----------------
	IN: 
	0x00007c0c:  test   $0x2,%al
	0x00007c0e:  jne    0x7c0a
	
	----------------
	IN: 
	0x00007c10:  mov    $0xd1,%al
	0x00007c12:  out    %al,$0x64
	0x00007c14:  in     $0x64,%al
	0x00007c16:  test   $0x2,%al
	0x00007c18:  jne    0x7c14
	
	----------------
	IN: 
	0x00007c1a:  mov    $0xdf,%al
	0x00007c1c:  out    %al,$0x60
	0x00007c1e:  lgdtw  0x7c6c
	0x00007c23:  mov    %cr0,%eax
	0x00007c26:  or     $0x1,%eax
	0x00007c2a:  mov    %eax,%cr0
	
	----------------
	IN: 
	0x00007c2d:  ljmp   $0x8,$0x7c32
	
	----------------
	IN: 
	0x00007c32:  mov    $0x10,%ax
	0x00007c36:  mov    %eax,%ds
	
	----------------
	IN: 
	0x00007c38:  mov    %eax,%es
	
	----------------
	IN: 
	0x00007c3a:  mov    %eax,%fs
	0x00007c3c:  mov    %eax,%gs
	0x00007c3e:  mov    %eax,%ss
	
	----------------
	IN: 
	0x00007c40:  mov    $0x0,%ebp
	
	----------------
	IN: 
	0x00007c45:  mov    $0x7c00,%esp
	0x00007c4a:  call   0x7d0d
	
	----------------
	IN: 
	0x00007d0d:  push   %ebp
对比bootblock.asm和bootasm.S文件，可以发现是一致的。

```

[练习2.4]自己找一个bootloader或内核中的代码位置,设置断点并进行测试。

```
	可以在bootmain函数入口设置断点，即将练习2.2中lab1init文件改为
file obj/bootblock.o
target remote :1234
set architecture i8086
break bootmain
continue
x /10i $pc
执行指令make lab1-mon可以查看运行结果如下：
Breakpoint 1, bootmain () at boot/bootmain.c:87
87	bootmain(void) {
=> 0x7cd1 <bootmain>:	push   %bp
   0x7cd2 <bootmain+1>:	mov    %sp,%bp
   0x7cd4 <bootmain+3>:	push   %di
   0x7cd5 <bootmain+4>:	push   %si
   0x7cd6 <bootmain+5>:	push   %bx
   0x7cd7 <bootmain+6>:	mov    $0x1,%bx
   0x7cda <bootmain+9>:	add    %al,(%bx,%si)
   0x7cdc <bootmain+11>:	sub    $0x1c,%sp
   0x7cdf <bootmain+14>:	lea    0x7f(%bp,%di),%ax
   0x7ce2 <bootmain+17>:	mov    %bx,%dx
(gdb)

```

## 练习三 分析bootloader 进入保护模式的过程。

```
1.首先初始化段寄存器，关闭中断
.code16                                    # Assemble for 16-bit mode
    cli                                    # Disable interrupts
    cld                                    # String operations increment
    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                          # Segment number zero
    movw %ax, %ds                          # -> Data Segment
    movw %ax, %es                          # -> Extra Segment
    movw %ax, %ss                          # -> Stack Segment
    
 2.开启A20，原因是要进入保护模式，即访问32位地址线，如果A20一直是0，则智能访问一些奇数1M段，即1M,3M,5M…，也就是00000-FFFFF,200000-2FFFFF,300000-3FFFFF…。如果A20 Gate被打开，则可以访问的内存则是连续的。A20gate关闭的原因是当时系统升级的时候为了与以前的8086/8088相兼容，将A20置为0后，当程序员给出超过1M（100000H-10FFEFH）的地址时，系统并不认为其访问越界而产生异常，而是自动从重新0开始计算，也就是说系统计算实际地址的时候是按照对1M求模的方式进行的，即wrap-around。
 开启的方式就是通过设置8042芯片输出端口（64h）的2nd-bit，但事实上，当你向8042芯片输出端口进行写操作的时候，在键盘缓冲区中，或许还有别的数据尚未处理，因此你必须首先处理这些数据。具体流程为：禁止中断；等待，直到8042 Inputbuffer为空为止；发送禁止键盘操作命令到8042Input buffer；等待，直到8042 Inputbuffer为空为止；发送读取8042 OutputPort命令；等待，直到8042 Outputbuffer有数据为止；读取8042 Outputbuffer，并保存得到的字节；等待，直到8042 Inputbuffer为空为止；发送Write 8042Output Port命令到8042 Input buffer；等待，直到8042 Inputbuffer为空为止；将从8042 OutputPort得到的字节的第2位置1（OR 2），然后写入8042 Input buffer；等待，直到8042 Inputbuffer为空为止；发送允许键盘操作命令到8042Input buffer；打开中断。代码为：
 seta20.1:
    inb $0x64, %al         # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al        # 0xd1 -> port 0x64
    outb %al, $0x64        # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al         # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al        # 0xdf -> port 0x60
    outb %al, $0x60	    # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
其中0x64和0x60为8042芯片的两个I/O端口，其中0x64h为命令和状态口，0x60h为数据口，可同时做读写操作。

3.初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可
	lgdt gdtdesc
    
4.开启保护模式，通过将cr0寄存器中PE位置为1即可开启保护模式,开启保护模式既可以使用32位寻址空间，同时也可以避免实模式下访存任意的风险
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    
5.通过长跳转指令进入保护模式：长跳转指令ljmp $PROT_MODE_CSEG, $protcseg跳转到下一条代码，目的是跳过剩余的16位指令。此处我们看到新的GDT已经发挥作用，seg= PROT_MODE_CSEG, offset=protcseg，因为CSEG的基地址为0，则程序跳转到了代码段的protcseg偏移处

6.设置段寄存器，并建立堆栈
	movw $PROT_MODE_DSEG, %ax               # Our data segment selector
    movw %ax, %ds                           # -> DS: Data Segment
    movw %ax, %es                           # -> ES: Extra Segment
    movw %ax, %fs                           # -> FS
    movw %ax, %gs                           # -> GS
    movw %ax, %ss                           # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    
7.转到保护模式完成，进入boot主方法
	    call bootmain
	 
```

##练习四 分析bootloader加载ELF格式的OS的过程

[练习4.1]bootloader如何读取硬盘扇区的?

```
读扇区函数为
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();            //等到磁盘可用

    outb(0x1F2, 1);                         // count = 1，即要读取一个扇区
    outb(0x1F3, secno & 0xFF);              //要读取的扇区编号
    outb(0x1F4, (secno >> 8) & 0xFF);       //用来存放读写柱面的低8位字节
    outb(0x1F5, (secno >> 16) & 0xFF);      //用来存放读写柱面的高两位字节
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);//用来存放要读/写的磁盘号和磁头号
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4); //获取数据到dst，除以4的原因时我们一次读入32位，即4byte
}
对硬盘进行操作的常用端口是1f0h~1f7h号端口，各端口含义如下：
端口号   读还是写 	 具体含义
1F0H 	读/写	 用来传送读/写的数据(其内容是正在传输的一个字节的数据)
1F1H 	读 	   用来读取错误码
1F2H 	读/写 	用来放入要读写的扇区数量
1F3H 	读/写 	用来放入要读写的扇区号码
1F4H 	读/写 	用来存放读写柱面的低8位字节
1F5H 	读/写 	用来存放读写柱面的高2位字节(其高6位恒为0)
1F6H 	读/写 	用来存放要读/写的磁盘号及磁头号
1f7H 	读        用来存放读操作后的状态
第7位 控制器忙碌
第6位 磁盘驱动器准备好了
第5位 写入错误
第4位 搜索完成
第3位 为1时扇区缓冲区没有准备好
第2位 是否正确读取磁盘数据
第1位 磁盘每转一周将此位设为1,
第0位 之前的命令因发生错误而结束
写 该位端口为命令端口,用来发出指定命令
为50h 格式化磁道
为20h 尝试读取扇区
为21h 无须验证扇区是否准备好而直接读扇区
为22h 尝试读取长扇区(用于早期的硬盘,每扇可能不是512字节,而是128字节到1024之间的值)
为23h 无须验证扇区是否准备好而直接读长扇区
为30h 尝试写扇区
为31h 无须验证扇区是否准备好而直接写扇区
为32h 尝试写长扇区
为33h 无须验证扇区是否准备好而直接写长扇区  
```

[练习4.2]bootloader是如何加载ELF格式的OS？

```
练习4.1主要了解了bootloader如何读一个扇区（512字节），而bootmain.c文件中另外一个函数readseg利用readsect实现任意位置任意字节的读取，函数为：
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;              //上界

    // round down to sector boundary
    va -= offset % SECTSIZE;         //下界，刚好使读取的区域为512的整数倍

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;   //初始扇区号，kernal起始于第一个扇区，第0个扇区为加载程序

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);             //利用读取扇区函数开始读取
    }
}

接下来据此来分析bootmain函数
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);  //读取kernal（elf文件）

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {          //判断是否是合法的ELF文件
        goto bad;
    }

    struct proghdr *ph, *eph;

    //利用elf文件头部有描述文件应该加载内存什么位置的描述表，利用ELFHDR->e_phoff找到描述表头位置
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;  //利用ELFHDR -> e_phnum和表头得到结束位置
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);        //将ELF文件读到内存中相应的位置
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();         //利用ELF文件表头储存的入口信息，跳到ucore入口

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

##练习五 实现函数调用堆栈跟踪函数

```
实现的函数如下：
void 
print_stackframe(void) {
     /* lab1 2012011281 : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp = read_ebp(), eip = read_eip();

    int i, j;
    for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i ++) {
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        uint32_t *args = (uint32_t *)ebp + 2;
        for (j = 0; j < 4; j ++) {
            cprintf("0x%08x ", args[j]);
        }
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }
}
ps：不能写成for(int i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++),定义只能放在循环体外面。
输出中，最后一行内容为
ebp:0x00007bf8 eip:0x00007d64 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d63 --
分析可知其对应的是第一个使用堆栈的函数，bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。call指令压栈，所以bootmain中ebp为0x7bf8。

```

#练习六 完善中断初始化和处理

[练习6.1]中断描述符表(也可简称为保护模式下的中断向量表)中一个表项占多少字节?其中哪几位代表中断处理代码的入口?

```
中断向量表一个表项占用8字节，其中2-3字节是段选择子（selector），利用段选择子查找GDT获取基址，0-1字节和6-7字节拼成offset，两者联合便是中断处理程序的入口地址。

```

[练习6.2]请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中,依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏,填充idt数组内容。每个中断的入口由tools/vectors.c生成,使用trap.c中声明的vectors数组即可。

```
填充后的kern/trap.c/init函数为
void
idt_init(void) {
     /* 2012011281 : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */
      extern uintptr_t __vectors[];
      int i;
      for(i = 0; i < sizeof(idt)/sizeof(struct gatedesc); i++)
      {
      	SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL)           // SETGATE(gate, istrap, sel, off, dpl)
      }
      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);    //set switch from user to kernel T_SWITCH_TOK = 121
	  lidt(&idt_pd);
}
利用函数体最前面的注释，我们可以获取所有中断服务例程entry的地址，存放于vectors数组中，然后利用SETGATE函数初始化IDT，经过查找可以发现SETGATE函数的原型，为：
/* *
 * Set up a normal interrupt/trap gate descriptor
 *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 *   - sel: Code segment selector for interrupt/trap handler
 *   - off: Offset in code segment for interrupt/trap handler
 *   - dpl: Descriptor Privilege Level - the privilege level required
 *          for software to invoke this interrupt/trap gate explicitly
 *          using an int instruction.
 * */
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
对于上述的调用函数，GD_KTEXT为GDT的基址，值为0x1000，___vectors[]为偏移，DPL_KERNEL为内核态

```

[练习6.3]请编程完善trap.c中的中断处理函数trap,在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分,使操作系统每遇到100次时钟中断后,调用print_ticks子程序,向屏幕上打印一行文字”100	ticks”。

```
在trap.c文件的trap_dispatch函数里，在lab1 your code处补充，相关代填充如下 
case IRQ_OFFSET + IRQ_TIMER:
        /* lab1 2012011281 : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
         ticks ++;
         if(ticks % TICK_NUM == 0)
         {
         	print_ticks();
         }
        break;

```
