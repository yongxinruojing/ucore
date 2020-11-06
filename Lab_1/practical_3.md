[TOC]



# 练习3

#### **练习3：分析bootloader进入保护模式的过程。（要求在报告中写出分析）**

BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

提示：需要阅读**小节“保护模式和分段机制”**和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：

- 为何开启A20，以及如何开启A20
- 如何初始化GDT表
- 如何使能和进入保护模式

通过修改A20地址线可以完成从实模式到保护模式的转换。有关A20的进一步信息可参考附录“关于A20 Gate”

## **分析**

**分段地址转换**：段选择子selector和段偏移offset组成

1. 段选择子的内容作为段描述符的索引找到表中对应的段描述符
2. 段描述符中的段基址加上offset形成Linear Address(若不启动分页存储管理，线性地址等于物理地址)
3. 补充：CS:IP可以看做PC，此时PC只是概念的说法。CS:IP指向的内容为指令执行

```makefile
# 通过对bootblock.asm文件中的注解对bootasm.S代码进行分析，并添加练习三的相关注解(中文部分)
# include <asm.h>

# Start the CPU: switch to 32-bit protected mode, jump into C.# C为c语言
# Th1e BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.set PROT_MODE_CSEG,        0x8                 # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                # kernel data segment selector
.set CR0_PE_ON,             0x1                 # protected mode enable flag

# start address should be 0:7c00, in real mode, the beginning address of the running bootloader
.globl start
start:											#从`%cs=0 $pc=0x7c00`，进入后
.code16                                         # Assemble for 16-bit mode
    cli                                         # Disable interrupts
    cld                                         # String operations increment
    											# cli：修改IF，禁止中断——为了禁止键盘操作后序要修改A20
    											# 	IF：中断允许标志位
    											# cld：清除DF：置0，地址从低到高(字串处理)
    											# 	DF(方向/)向量标志位：为1 地址从高到低、STD置1 CLD置0

    # Set up the important data segment registers (DS, ES, SS).（段选择子）
    xorw %ax, %ax                               # Segment number zero
    movw %ax, %ds                               # -> Data Segment
    movw %ax, %es                               # -> Extra Segment
    movw %ax, %ss                               # -> Stack Segment
    											# 因为之前BIOS必须通过实模式启动，因此引入kernel时从实模式转换到保护
    											# 模式之前需要清除之前16位实模式遗留下的寄存器的值，这里先异或把ax清0
    											# 然后把0拷贝到ds,es,ss三个段寄存器
    											# 总结：上面是做实模式的收尾工作

    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
        										
seta20.1:										# 读64h端口读出Status Register的一个字节
    inb $0x64, %al                              # (%al AND 0x0010)也就是测试bit1是否为1，把结果设置到flags寄存器
    testb $0x2, %al								# 若为1表示input register有数据
    jnz seta20.1								# jump if not zero跳转到seta20.1亦即循环判断等待bit1是0

    movb $0xd1, %al                             # 此时bit1是0了，代表input Register没有数据
    											# movb是移动8位0xd1给al寄存器
    											
    											
    outb %al, $0x64								# 将al中存储的8位0xd1 -> port 0x64   
    											# 也就是向64h发送0d1h命令，表明要写Output Port 
    											# 发送Write 8042 Output Port命令到8042 Input buffer


seta20.2:										# 读64h端口读出Status Register的一个字节
    inb $0x64, %al                              # 同上
    testb $0x2, %al								# 循环等待8042 Input buffer为空
    jnz seta20.2

    movb $0xdf, %al                             # 0xdf -> %al
    outb %al, $0x60                             # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
												# 向0x60h写入0xdf：第二位置1 
												# 置8042芯片的输出端口（64h）的bit 1为1，打开A20 Gate

    # Switch from real to protected mode, using a bootstrap GDT(全局描述符表)
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    # 保护模式下才能使用分段机制，且ucore实现的整个内存地址都是一个段4G，因此线性地址空间和物理地址空间对等
    lgdt gdtdesc								# 加载gdt	
    											# 答案：一个简单的GDT表和描述符已经静态存储在引导区，载入即可
    											# 共三个表项解释在105行
    										
    movl %cr0, %eax			# cr0 → eax									
    orl $CR0_PE_ON, %eax    # '0x1' 逻辑或 eax → 将eax置为1(最后一位)
    movl %eax, %cr0			# cr0最后一位变为1，使能和进入保护模式

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
    						# 长跳转到kernel代码段选择子0x8,偏移量为protcseg的地址
    						# 0x8——0x8=1000去掉特权级两位和TI表指示位一位之后1就是Index(or 1左移三位为8)
    						# 同时设置了CS和IP
    						# ljmp a，b 类似于movl a,%CS   movl b,%IP
                        
                        	# 之前设置段的时候有：0为空段，1为内核代码段，2为内核数据段，3为用户代码段，4用户数据段，5是								# TSS。他们的索引乘以8(左移三位)分别为：0x8,0x10,0x18,0x20,0x28

.code32                                         # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    # 设置保护模式下的段寄存器(段选择子)，全部为0x10第三个段描述符的(index,TI,RPL)
    movw $PROT_MODE_DSEG, %ax                   # kernel data segment selector写入到ax寄存器(EAX的低16位)
    movw %ax, %ds                               # -> DS: Data Segment
    movw %ax, %es                               # -> ES: Extra Segment
    movw %ax, %fs                               # -> FS
    movw %ax, %gs                               # -> GS
    movw %ax, %ss                               # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    # 初始化栈——低地址栈顶，高地址栈底
    movl $0x0, %ebp								# 0x0作为基址
    movl $start, %esp							# start的地址作为栈顶(esp为栈顶指针寄存器)
    call bootmain								# 调用bootmain函数,此时没有压栈？

    # If bootmain returns (it shouldn't), loop.
spin:
    jmp spin									# 一般情况不会回来，若回来就无限spin

# Bootstrap GDT
.p2align 2                                      # force 4 byte alignment
gdt:
    SEG_NULLASM                                 # null seg	，GDT的第一项，当做空段描述符
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)       # code seg for bootloader and kernel，大小为4G，可执行可读
    SEG_ASM(STA_W, 0x0, 0xffffffff)             # data seg for bootloader and kernel，可写不可执行，4G
    											# 把整个4G分成一个简单的段
    											# tips:设置0开始保证之前的代码可以运行，亦即地址不变

gdtdesc:										# GDTR长48位，高32位是基地址
												# 每个段描述符是8个字节大小
    .word 0x17                                  # .word就是在当前gdtdesc处放一个值0x17
    											# 因此三个描述符大小就是sizeof(gdt) - 1 = 3*8 - 1（从0开始）
    .long gdt                                   # .long定义一个长整型32位！岂不就是gdt的基地址
```

------

## **回答**

#### **为何开启A20**

1. 实模式状态为16位模式，此时软件可访问的物理内存空间不超过1MB，无法发挥80386以上的32位CPU的4G内存管理能力。

2. 实模式下整个物理内存看成分段的区域，程序代码和数据位于不同区域，操作系统和用户程序没有区别对待！！每一个指针都指向实际物理地址，如果应用程序的指针指向了os区域并修改了内容，那毁灭就是灾难性的。

3. 之前实模式下使用键盘控制器剩余的输出线视作第21根地址线(地址的位数从右到左是从0开始数，第21根对应着秩20)；由于实模式下地址的访问0ffff0h+0ffffh = 10ffefh大于了1024KB实际上是1088KB多了64KB，因此通过关闭A20实现即使是10ffeh我访问的还是00ffeh。如果转换到保护模式的话使用32位地址线，如果A20关闭那么第20位始终为0，导致访问0-1M，2-3M...奇数兆的内存，因此需要打开A20.（A20的实现是通过和第21位亦即20进行AND实现，A20关闭就是0——实模式的时候就是0，AND之后必为0，开启之后置1，AND之后还是本身）

4. 补充：
    如何计算20位：把20个地址线可以理解为20个引脚，相当于给引脚复值

    

#### **如何开启A20**

**前言：**早期的8086CPU提供20根地址线，可寻址空间为0~2^20(00000H~FFFFFH)亦即1MB的内存空间。但是8086数据处理位宽只有16位，无法直接寻址1MB内存空间，因此8086提供了段地址+偏移地址的地址转换机制：PC机的寻址结构是segment:offset，segment和offset都是16位的寄存器最大值是0fffffh，换算成物理地址：segment左移4位+offset因此其所能表达最大的寻址空间为0ffff0h+0ffffh = 10ffefh，大约是1088KB，它超过了20位地址线的物理寻址能力(1MB)，因此当寻找超过1MB的内存时→“回卷”(不会发生异常)。下一代的80286CPU提供了24根地址线，此时CPU的寻址范围变为2^24 = 16M，同时提供了保护模式可访问1MB以上的内存，但如果此时寻找超过1MB的情况与之前的“回卷”就不兼容了，为了保持完全向下兼容，IBM就决定在PC AT计算机系统上加了个硬件逻辑来模仿以上的回绕特征，于是就出现了A20。

**开启A20：**

1. 'cli'禁止中断——禁止键盘操作
2. 等待8042 Input buffer为空：
   + 发送Write 8042 Output Port(p2)命令到8042 Input buffer，告诉他我要写数据
3. 等待8042 Input buffer为空：
   + 将上面8042 Output Port(p2)得到的字节第二个bit置1，然后写入8042 Input buffer

#### **如何初始化GDT表(pmm.c相关代码)**

```c
 gdt_init(void) {
    // Setup a TSS so that we can get the right stack when we trap from
    // user to the kernel. But not safe here, it's only a temporary value,
    // it will be set to KSTACKTOP in lab2.
    //trap:陷阱，用户态的应用程序通过陷阱把控制权交给操作系统来实现调用内核函数和使用硬件
    ts.ts_esp0 = (uint32_t)&stack0 + sizeof(stack0);
    ts.ts_ss0 = KERNEL_DS;

    // initialize the TSS filed of the gdt
    gdt[SEG_TSS] = SEG16(STS_T32A, (uint32_t)&ts, sizeof(ts), DPL_KERNEL);
    gdt[SEG_TSS].sd_s = 0;

    // reload all segment registers
    lgdt(&gdt_pd);

    // load the TSS
    ltr(GD_TSS);
}
```

1. 设置一个TSS(任务状态段)，以便于当我们陷阱来自于用户态到内核态时可以得到正确的堆栈
2. 初始化gdt的TSS段
3. 重新加载所有的段寄存器
   + 设置GDT第一项为空段，代码段和数据段都设置为简单的全部4G
4. 加载TSS

#### **如何使能和进入保护模式**

```assembly
 movl %cr0, %eax			# 1. 将cr0的值传入eax累加器
 orl $CR0_PE_ON, %eax    	# 2. %eax=(0x1逻辑或0)将其置为1(最后一位)
 movl %eax, %cr0			# 3. cr0使最后一位(PE位)为1，使能和进入保护模式了

# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
ljmp $PROT_MODE_CSEG, $protcseg					# 4. 无条件长跳转到内核代码段选择子+protcseg的偏移
												#答案补充：更新cs的基地址！！
.code32                                       
protcseg:
    # Set up the protected-mode data segment registers
    # 5. 设置保护模式下的数据段寄存器
    movw $PROT_MODE_DSEG, %ax                   # kernel data segment selector写入到ax寄存器(EAX的低16位)
    movw %ax, %ds                               # -> DS: Data Segment
    movw %ax, %es                               # -> ES: Extra Segment
    movw %ax, %fs                               # -> FS
    movw %ax, %gs                               # -> GS
    movw %ax, %ss                               # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    # 6. 设置(初始化)栈区域0--start(0x7c00)以便于c语言能运行，并调用bootmain
    movl $0x0, %ebp								# ebp初始化为0x0
    movl $start, %esp							# start的地址作为栈顶(esp为栈顶指针寄存器)
    call bootmain								# 转到保护模式完成，调用bootmain

```

