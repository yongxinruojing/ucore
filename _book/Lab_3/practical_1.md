[TOC]



# 练习1：给未被映射的地址映射上物理页（需要编程）

完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。注意：在LAB3 EXERCISE 1处填写代码。执行

```
make　qemu
```

后，如果通过check_pgfault函数的测试后，会有“check_pgfault() succeeded!”的输出，表示练习1基本正确。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

-   请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。
-   如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

直接上源代码(从上到下依次阅读代码)

```c
//page fault number
volatile unsigned int pgfault_num=0;

/* do_pgfault - interrupt handler to process the page fault execption
 * @mm         : the control struct for a set of vma using the same PDT
 * @error_code : the error code recorded in trapframe->tf_err which is setted by x86 hardware
 * @addr       : the addr which causes a memory access exception, (the contents of the CR2 register)
 *
 * CALL GRAPH: trap--> trap_dispatch-->pgfault_handler-->do_pgfault
 * The processor provides ucore's do_pgfault function with two items of information to aid in diagnosing
 * the exception and recovering from it.
 *   (1) The contents of the CR2 register. The processor loads the CR2 register with the
 *       32-bit linear address that generated the exception. The do_pgfault fun can
 *       use this address to locate the corresponding page directory and page-table
 *       entries.
 *   (2) An error code on the kernel stack. The error code for a page fault has a format different from
 *       that for other exceptions. The error code tells the exception handler three things:
 *         -- The P flag   (bit 0) indicates whether the exception was due to a not-present page (0)
 *            or to either an access rights violation or the use of a reserved bit (1).
 *         -- The W/R flag (bit 1) indicates whether the memory access that caused the exception
 *            was a read (0) or write (1).
 *         -- The U/S flag (bit 2) indicates whether the processor was executing at user mode (1)
 *            or supervisor mode (0) at the time of the exception.
 */
int
do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
    int ret = -E_INVAL;					//#define E_INVAL   3   /Invalid parameter
    //try to find a vma which include addr
    struct vma_struct *vma = find_vma(mm, addr);

    pgfault_num++;
    //If the addr is in the range of a mm's vma?
    if (vma == NULL || vma->vm_start > addr) {
        //地址非法直接返回
        cprintf("not valid addr %x, and  can not find it in vma\n", addr);
        goto failed;
    }
    //check the error_code
    switch (error_code & 3) {		//先判断引起的原因然后和对应地址的vma的标志位进行比较得出结果
            
            
    default:
            /* error code flag : default is 3 ( W/R=1, P=1): write, present */
            //疑问1：为什么写一个存在的页异常就不管了？？？
            //自己的解答：此时映射页表项存在且发生写异常,说明发生了缺页异常！！！然后下面进行的就是缺页的两种处理情况
            //存在但是不在内存！所以要读到内存中
    case 2: /* error code flag : (W/R=1, P=0): write, not present */
        if (!(vma->vm_flags & VM_WRITE)) {
            //此时由写一个不存在的页引起,然后if判断是否vma可写如果不可写那就是异常,下面类似
            cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write\n");
            goto failed;
        }
        break;
    case 1: /* error code flag : (W/R=0, P=1): read, present */
            //此时由读一个存在的页引起,那肯定是用户态读内核态or特权级不够,直接异常
        cprintf("do_pgfault failed: error code flag = read AND present\n");
        goto failed;
    case 0: /* error code flag : (W/R=0, P=0): read, not present */
        if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {
            //此时由读一个不存在的页引起,然后if判断是否vma可读or可执行如果有一个不可那就是异常
            cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
            goto failed;
        }
    }
    /* IF (write an existed addr 此时已经是可写的了？) OR
     *    (write an non_existed addr && addr is writable) OR
     *    (read  an non_existed addr && addr is readable)
     * THEN
     *    continue process
     * 此时通过了上面的判断已经是合法的了,此时有两种处理方法,一种是该页还没建立,一种是在硬盘中需要置换到内存
     * 练习1对应第一种,练习二对应第二种
     */
    uint32_t perm = PTE_U;					//#define PTE_U   0x004  User
    if (vma->vm_flags & VM_WRITE) {			//如果可写
        perm |= PTE_W;						//把PTE_W和PTE_U合并到一起		
    }
    addr = ROUNDDOWN(addr, PGSIZE);			//求出页号

    ret = -E_NO_MEM;						//#define E_NO_MEM   4 /Request failed due to memory shortage

    pte_t *ptep=NULL;						//创建一个NULL的pte页表项
    /*LAB3 EXERCISE 1: YOUR CODE
    * Maybe you want help comment, BELOW comments can help you finish the code
    *
    * Some Useful MACROs and DEFINEs, you can use them in below implementation.
    * MACROs or Functions:
    *   get_pte : get an pte and return the kernel virtual address of this pte for la
    *             if the PT contians this pte didn't exist, alloc a page for PT (notice the 3th parameter '1')
    *   pgdir_alloc_page : call alloc_page & page_insert functions to allocate a page size memory & setup
    *             an addr map pa<--->la with linear address la and the PDT pgdir
    * DEFINES:
    *   VM_WRITE  : If vma->vm_flags & VM_WRITE == 1/0, then the vma is writable/non writable
    *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
    *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
    * VARIABLES:
    *   mm->pgdir : the PDT of these vma   tips:PDT和PDE是不是类似的,不是！！
    *
    */
#if 1
    /*LAB3 EXERCISE 1: YOUR CODE*/
    /*MY CODE
   	    	//(1) try to find a pte, if pte's PT(Page Table) isn't existed, then create a PT.
    ptep = get_pte(mm->pgdir, addr, 1);		
    		//mm->pgdir就是vma所属的PDT,addr是CR2寄存器的内容,而CR2存的就是线性地址,最后create置1
    
        	//(2) if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr 
    if (*ptep == 0) {
       struct Page *page = pgdir_alloc_page(mm->pgdir, addr, perm);
	   tlb_invalidate(mm->pgdir, addr);	//这一句实验指导书说了需要刷新TLB,但是答案不包括这一句,练习2有这一句！！
       return 0;			//不知道返回什么,然后测试发现trap.c的182行错误 → 正确处理的情况下应该返回0
    }
    通过了测试,但是为了防止后续有某些错误,代码写老师的答案
    第二个完全自己写出的答案,ヾ(o◕∀◕)ﾉヾ
    */
    if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
        cprintf("get_pte in do_pgfault failed\n");
        goto failed;
    }
    if (*ptep == 0) { 		//全为0代表此前不存在,需要新分配一个物理页
        // if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
            goto failed;
        }
    } 
    //以上是练习1老师给出的参考答案


```

### **问题回答：**

+   怎么和lab2的一样,直接去lab2看吧，，，，

### **实验中遇到的疑惑：**

1.  疑问1：bit0和bit1均为1的时候是default
    解答：此时映射页表项存在且发生写异常,说明发生了缺页异常！！！然后下面进行的就是缺页的两种处理情况.此处的存在是指这个页表项是否存在,即使页表项存在但是也发生了异常那就是页不在内存