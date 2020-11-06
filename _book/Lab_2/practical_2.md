[TOC]

# 练习2

**练习2：实现寻找虚拟地址对应的页表项（需要编程）**

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全get_pte函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解get_pte函数中的注释。get_pte函数的调用关系图如下所示：

![img](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2_figs/image001.png) 图1 get_pte函数的调用关系图

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

-   请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对ucore而言的潜在用处。
-   如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
-   如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？



```c
//get_pte - get pte and return the kernel virtual address of this pte for la(虚拟地址是段，线性地址是页)
//        - if the PT contians this pte didn't exist, alloc a page for PT找虚拟地址对应的二级页表项的虚拟地址
// parameter:
//  pgdir:  the kernel virtual base address of PDT
//  la:     the linear address need to map
//  create: a logical value to decide if alloc a page for PT
// return vaule: the kernel virtual address of this pte
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    /* LAB2 EXERCISE 2: YOUR CODE
     *
     * If you need to visit a physical address, please use KADDR()
     * please read pmm.h for useful macros
     *
     * Maybe you want help comment, BELOW comments can help you finish the code
     *
     * Some Useful MACROs and DEFINEs, you can use them in below implementation.
     * MACROs or Functions:
     *   PDX(la) = the index of page directory entry of VIRTUAL ADDRESS la.
     *   KADDR(pa) : takes a physical address and returns the corresponding kernel virtual address.
     *   也就是+0xc0000000?
     *   set_page_ref(page,1) : means the page be referenced by one time
     *   page2pa(page): get the physical address of memory which this (struct Page *) page  manages
     *   struct Page * alloc_page() : allocation a page
     *   memset(void *s, char c, size_t n) : sets the first n bytes of the memory area pointed by s
     *                                       to the specified value c.将s指向的前n个字节用c替换
     * DEFINEs:
     *   PTE_P           0x001                   // page table/directory entry flags bit : Present
     *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
     *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
     */
#define PTE_ADDR(pte)   ((uintptr_t)(pte) & ~0xFFF)
#define PDE_ADDR(pde)   PTE_ADDR(pde)
#if 0					  // #if 0..#endif为让编译器屏蔽代码，把0改为1即可
     // (1) find page directory entry
             // (2) check if entry is not present
    		// (3) check if creating is needed, then alloc page for page table

                          // CAUTION: this page is used for page table, not for common data page
                          // (4) set page reference
      // (5) get linear address of page
                          // (6) clear page content using memset
                          // (7) set page directory entry's permission

             // (8) return page table entry
#endif
    pde_t *pdep = 0;
          //pdep = (pde_t *)(((uint32_t)pgdir & ~0x0FFF) | PDX(la));   
	pdep = &pgdir[PDX(la)];    
if (!(*pdep & PTE_P)) {    
	struct Page *page_n;          
       if(create && (page_n = alloc_page()) != NULL){			
           set_page_ref(page_n, 1);
           uintptr_t page_n_pa = page2pa(page_n);	//通过结构体获得对应的物理page的物理地址
           memset(KADDR(page_n_pa), 0x0, PGSIZE);
           *pdep = page_n_pa | PTE_P | PTE_W | PTE_U;
           return &((pte_t*)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
           /*	pgdir[(PDX(la))] = ((page2pa(page_n)) & 0xFFFFF000) | (pgdir[(PDX(la))] & 0x00000FFF);
           		pgdir[(PDX(la))] = pgdir[(PDX(la))] | PTE_P | PTE_W | PTE_U;
           		return &KADDR(page2pa(page_n))[(PTX(la))];
           		页目录表存的是页表项的物理地址！！物理地址要转换成虚拟地址
           */
       }
        else return NULL;
	}
    else return &((pte_t*)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
    //前二十位物理地址转换为内核虚拟地址，然后加上线性地址中间的十位*4作为偏移形成pte(因为pte_t是32位=4*8)
    为什么不是直接拼接
    //计算机内存地址以字节为单位！0x1和0x2相差了8个bit也为1个字节

}
```

问题回答：

1.  虚拟地址or线性地址通过页目录表访问到页表项然后访问到物理地址，同一个物理空间可以实现重用！

    ```makefile
    页目录项内容 = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
    # 前20位是页表项的起始物理地址，后面三位是用户态软件是否可读，物理内存是否可写，物理内存页是否存在
    页表项内容 = (pa & ~0x0FFF) | PTE_P | PTE_W
    # 前20位是页的起始物理地址，后面同上
    ```

2.  ```undefined
    - 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
        - 当ucore执行过程中出现了页访问异常，硬件需要完成的事情分别如下：
            - 将发生错误的线性地址保存在cr2寄存器中;
            - 在中断栈中依次压入EFLAGS，CS, EIP，以及页访问异常码error code(若有)，如果page fault是发生在用户态，则还需要先压入ss和esp，并且切换到内核栈；
            - 根据中断描述符表查询到对应page fault的ISR，跳转到对应的ISR处执行，接下来将由软件进行page fault处理；
    ```

3.  关中断

    保护现场。包括：将页访问异常的错误码压入内核栈的栈顶、将导致页访问异常的虚拟地址记录在cr2寄存器中、保存状态寄存器PSW及断点等。

    根据中断源，跳转到缺页服务例程