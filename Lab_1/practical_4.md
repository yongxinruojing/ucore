[TOC]



# 练习4

#### **练习4：分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）**

通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS，

- bootloader如何读取硬盘扇区的？
- bootloader是如何加载ELF格式的OS？

提示：可阅读“硬盘访问概述”，“ELF执行文件格式概述”这两小节。



```c
#define SECTSIZE        512								//扇区大小
#define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space 暂存空间？地址为0x10000

/* waitdisk - wait for disk'磁盘' ready */
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)  /*inb向IO端口读取一个字节*/
                                        /* 0x1F7表示0号硬盘(第一个硬盘扇区)的状态和命令寄存器，当其的最高两位是01时，
                                        表示空闲状态.nb(0x1F7) & 0xC0 表示将0x1F7端口所代表的寄存器的值和0xC0做与操作 
                                        观察0x1F7的最高两位是否是01如果是01,表示空闲,跳出循环,如果不是，则继续循环。*/
		/* do nothing */;
}

/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();
/*outb为向IO端口写入一个字节*/
    outb(0x1F2, 1);                         // count = 1  设置读写扇区数为1
    outb(0x1F3, secno & 0xFF);				// 设置读取扇区编号，向其输入secno的后8位(7-0)
    outb(0x1F4, (secno >> 8) & 0xFF);		// 15-8——9-16位
    outb(0x1F5, (secno >> 16) & 0xFF);		// 23-16——17-24位
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);//31-24，25-32位，全0的高4位与1110逻辑或→前三位1，第四位0
    							//答案补充：		上面四条指令联合制定了扇区号，在这4个字节线联合构成的32位参数中
    							//				29-31位强制设为1，28位( = 0)表示Disk 0，0-27位是28位的偏移量
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors写入读扇区命令

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);//从IO地址0x1F0读取一个磁盘扇区数据到dst
}

/* *
 * readseg - read @count bytes at @offset from kernel into virtual address @va,
 * might copy more than asked.
 * 把数据读到这个虚拟地址(此时没有映射)
 * */
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary，向下舍入到扇区边界/起始，下面从扇区的起始读
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    // 答案补充：加1是因为0扇区被引导占用，因此ELF文件从1扇区开始
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}

void
bootmain(void) {
    // read the 1st page off disk，read eight SECTSIZE(一个page是4k) to ELFHDR(header of elf)，为什么是4k？
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF? 
    // must equal to ELF_MAGIC
    // 通过幻数判断
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph; 
	// create program header
    
    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);// e_phoff is file position of program 																	 // header table(程序段数组的偏移值)
    															 
    eph = ph + ELFHDR->e_phnum;// e_phnum为程序段的个数
    for (; ph < eph; ph ++) {  // 按照描述表将ELF文件中的程序段载入内存
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
    // memsz: size of segment in memory
    // ELFHDR:0x10000,从0x10000开始读入 
   	// ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000
    
    // call the entry point from the ELF header
    // ((void (*)(void))addr将addr转化为无参数无返回值的函数指针然后调用此函数指针执行地址addr处的代码
    // 答案补充：根据ELF头部存储的入口信息，找到内核的入口
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
    // 如果返回就是出错，无限循环
bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}


```

#### **bootloader如何读取硬盘扇区的？**

通过readsect函数实现：

1. 循环判断磁盘是否空闲
2. 向I/O端口写入读取扇区的命令
3. 循环判断磁盘是否空闲
4. 将磁盘扇区的数据读到指定内存dst

#### **bootloader如何加载ELF格式的OS？**

根据bootmain函数：

1. 读取4k大小的磁盘的第一个页到ELFHDR(elfheader)
2. 通过ELFHDR->e_magic判断ELF是否有效
3. 创建程序头
4. 依次加载每一个程序段(忽视ph标志)
5. 在程序入口的虚拟地址启动ucore并交给他控制权



