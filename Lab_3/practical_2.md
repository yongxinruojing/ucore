[TOC]



# 练习2：补充完成基于FIFO的页面替换算法（需要编程）

完成vmm.c中的do_pgfault函数，并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_victim函数。通过对swap的测试。注意：在LAB3 EXERCISE 2处填写代码。执行

```
make　qemu
```

后，如果通过check_swap函数的测试后，会有“check_swap() succeeded!”的输出，表示练习2基本正确。

请在实验报告中简要说明你的设计实现过程。

请在实验报告中回答如下问题：

-   如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题
    -   需要被换出的页的特征是什么？
    -   在ucore中如何判断具有这样特征的页？
    -   何时进行换入和换出操作？



1.  硬盘扇区和虚拟页对应关系是PTE,虚拟页和物理页帧也为PTE,如何区分？一个PTE有两个功能？

    区分方法：PTE的存在位为0且高24位不问0即为硬盘扇区和虚拟页对应的关系
    
2.  vmm.c中的部分

```c
    //如果不全为0但是缺页了,那就代表页在磁盘中,需要置换进来
    else {
    /*LAB3 EXERCISE 2: YOUR CODE
    * Now we think this pte is a  swap entry, we should load data from disk to a page with phy addr,
    * and map the phy addr with logical addr, trigger swap manager to record the access situation of this page.
    *
    *  Some Useful MACROs and DEFINEs, you can use them in below implementation.
    *  MACROs or Functions:
    *    swap_in(mm, addr, &page) : alloc a memory page, then according to the swap entry in PTE for addr,
    *                               find the addr of disk page, read the content of disk page into this memroy page
    *    page_insert ： build the map of phy addr of an Page with the linear addr la
    *    swap_map_swappable ： set the page swappable
    */
        if(swap_init_ok) {			//swap_init_ok是硬盘交换分区初始化是否完成的标志
            struct Page *page = NULL;
            //(1）According to the mm AND addr, try to load the content of right disk page 
            // 	 into the memory which page managed.
            if( swap_in(mm, addr, &page) != 0) goto failed;				//源代码是如果加载成功就返回0
            if( !page ) goto failed;
            //(2) According to the mm, addr AND page, setup the map of phy addr <---> logical addr
            if( page_insert(mm->pgdir, page, addr, perm) ) goto failed;													//page_insert加载完成的返回值也是0
            //page_insert有个标注note: PT is changed, so the TLB need to be invalidate
            //因此
           // tlb_invalidate(mm->pgdir, addr);但是答案没有这句！！！还是以参考答案为准把
            //(3) make the page swappable.
            swap_map_swappable(mm, addr, page, 1);	//swap_in什么意思 是0吗？
            page->pra_vaddr = addr;		//*****此处的代码：我少了这一句一直没通过测试。。。然后加上去通过测试了*****
            //pra_vaddr用来记录此物理页对应的虚拟页起始地址——线性地址
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }
   ret = 0;
failed:
    return ret;
}

```

3.  swap_fifo.c中的部分

```c
#include <defs.h>
#include <x86.h>
#include <stdio.h>
#include <string.h>
#include <swap.h>
#include <swap_fifo.h>
#include <list.h>

/* [wikipedia]The simplest Page Replacement Algorithm(PRA) is a FIFO algorithm. The first-in, first-out
 * page replacement algorithm is a low-overhead algorithm that requires little book-keeping on
 * the part of the operating system. The idea is obvious from the name - the operating system
 * keeps track of all the pages in memory in a queue, with the most recent arrival at the back,
 * and the earliest arrival in front. When a page needs to be replaced, the page at the front
 * of the queue (the oldest page) is selected. While FIFO is cheap and intuitive, it performs
 * poorly in practical application. Thus, it is rarely used in its unmodified form. This
 * algorithm experiences Belady's anomaly.
 *
 * Details of FIFO PRA
 * (1) Prepare: In order to implement FIFO PRA, we should manage all swappable pages, so we can
 *              link these pages into pra_list_head according the time order. At first you should
 *              be familiar to the struct list in list.h. struct list is a simple doubly linked list
 *              implementation. You should know howto USE: list_init, list_add(list_add_after),
 *              list_add_before, list_del, list_next, list_prev. Another tricky method is to transform
 *              a general list struct to a special struct (such as struct page). You can find some MACRO:
 *              le2page (in memlayout.h), (in future labs: le2vma (in vmm.h), le2proc (in proc.h),etc.
 */

list_entry_t pra_list_head;
/*
 * (2) _fifo_init_mm: init pra_list_head and let  mm->sm_priv point to the addr of pra_list_head.
 *              Now, From the memory control struct mm_struct, we can access FIFO PRA
 */
static int
_fifo_init_mm(struct mm_struct *mm)
{     
     list_init(&pra_list_head);
     mm->sm_priv = &pra_list_head;
     //cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);
     return 0;
}
/*
 * (3)_fifo_map_swappable: According FIFO PRA, we should link the most recent arrival page at the back of pra_list_head qeueue
 */
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{	
    list_entry_t *head=(list_entry_t*) mm->sm_priv;		// 获取pra_page_link构造的按时间排序的链表头
    list_entry_t *entry=&(page->pra_page_link);			// 获取这个page的pra_page_link准备加入链表
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 2: YOUR CODE*/ 
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add_before(head, entry);						// 链接到头前面就是末尾
    //入口参数：list_add_before(list_entry_t *listelm, list_entry_t *elm)
    return 0;
}
/*
 *  (4)_fifo_swap_out_victim: 
 		According FIFO PRA, we should unlink the  earliest arrival page in front of pra_list_head qeueue,
 *    then assign the value of *ptr_page to the addr of this page.
 */
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
     assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     /*LAB3 EXERCISE 2: YOUR CODE*/ 
     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
     list_entry_t *victim = list_next(head);
     
     //(2)  assign the value of *ptr_page to the addr of this page
     struct Page *p = le2page(victim, pra_page_link);	//*****之前自己写的是page_link*****
     list_del(le);
     //p->pra_vaddr = 0;							//自己写的,答案没有这句,为什么不用清0?
     //p = *ptr_page;								//*****一直没通过测试：答案是*ptr_page = p*****
     *ptr_page = p; 
    return 0;
}

static int
_fifo_check_swap(void) {
    cprintf("write Virt Page c in fifo_check_swap\n");
    *(unsigned char *)0x3000 = 0x0c;
    assert(pgfault_num==4);
    cprintf("write Virt Page a in fifo_check_swap\n");
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==4);
    cprintf("write Virt Page d in fifo_check_swap\n");
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==4);
    cprintf("write Virt Page b in fifo_check_swap\n");
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==4);
    cprintf("write Virt Page e in fifo_check_swap\n");
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==5);
    cprintf("write Virt Page b in fifo_check_swap\n");
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==5);
    cprintf("write Virt Page a in fifo_check_swap\n");
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==6);
    cprintf("write Virt Page b in fifo_check_swap\n");
    *(unsigned char *)0x2000 = 0x0b;
    assert(pgfault_num==7);
    cprintf("write Virt Page c in fifo_check_swap\n");
    *(unsigned char *)0x3000 = 0x0c;
    assert(pgfault_num==8);
    cprintf("write Virt Page d in fifo_check_swap\n");
    *(unsigned char *)0x4000 = 0x0d;
    assert(pgfault_num==9);
    cprintf("write Virt Page e in fifo_check_swap\n");
    *(unsigned char *)0x5000 = 0x0e;
    assert(pgfault_num==10);
    cprintf("write Virt Page a in fifo_check_swap\n");
    assert(*(unsigned char *)0x1000 == 0x0a);
    *(unsigned char *)0x1000 = 0x0a;
    assert(pgfault_num==11);
    return 0;
}


static int
_fifo_init(void)
{
    return 0;
}

static int
_fifo_set_unswappable(struct mm_struct *mm, uintptr_t addr)
{
    return 0;
}

static int
_fifo_tick_event(struct mm_struct *mm)
{ return 0; }


struct swap_manager swap_manager_fifo =
{
     .name            = "fifo swap manager",
     .init            = &_fifo_init,
     .init_mm         = &_fifo_init_mm,
     .tick_event      = &_fifo_tick_event,
     .map_swappable   = &_fifo_map_swappable,
     .set_unswappable = &_fifo_set_unswappable,
     .swap_out_victim = &_fifo_swap_out_victim,
     .check_swap      = &_fifo_check_swap,
};

```

#### **问题回答：**

**可以**：(简书的答案：https://www.jianshu.com/p/8d6ce61ac678?utm_campaign=hugo)

根据上文中提及到的PTE的组成部分可知，PTE中包含了dirty位和访问位，因此可以确定某一个虚拟页是否被访问过以及写过，但是，考虑到在替换算法的时候是将物理页面进行换出，而可能存在着多个虚拟页面映射到同一个物理页面这种情况，也就是说某一个物理页面是否dirty和是否被访问过是有这些所有的虚拟页面共同决定的，而在原先的实验框架中，物理页的描述信息Page结构中默认只包括了一个对应的虚拟页的地址，应当采用链表的方式，**在Page中扩充一个成员，把物理页对应的所有虚拟页都给保存下来；而物理页的dirty位和访问位均为只需要某一个对应的虚拟页对应位被置成1即可置成1；**

完成了上述对物理页描述信息的拓展之后，考虑对FIFO算法的框架进行修改得到拓展时钟算法的框架，由于这两种算法都是将所有可以换出的物理页面均按照进入内存的顺序连成一个环形链表，因此初始化时，将某个页面置为可以/不可以换出这些函数均不需要进行大的修改(小的修改包括在初始化当前指针等)，唯一需要进行重写的函数是选择换出物理页的函数swap_out_victim，对该函数的修改如下：

-   **从当前指针开始，对环形链表进行扫描，根据指针指向的物理页的状态（表示为(access, dirty)）来确定应当进行何种修改：**
    -   如果状态是(0, 0)，则将该物理页面从链表上去下，该物理页面记为换出页面，但是由于这个时候这个页面不是dirty的，因此事实上不需要将其写入swap分区；
    -   如果状态是(0, 1)，则将该物理页对应的虚拟页的PTE中的dirty位都改成0，**并且将该物理页写入到外存中**，然后指针跳转到下一个物理页；
    -   如果状态是(1, 0), 将该物理页对应的虚拟页的PTE中的访问位都置成0，然后指针跳转到下一个物理页面；
    -   如果状态是(1, 1)，则该物理页的所有对应虚拟页的PTE中的访问为置成0，然后指针跳转到下一个物理页面；

被换出页特征：

-   该物理页在当前指针上一次扫过之前没有被访问过；
-   该物理页的内容与其在外存中保存的数据是一致的, 即没有被修改过;
-   (0,0)

如何判断：

-   -   假如某物理页对应的所有虚拟页中**只要存在一个**dirty的页，则认为这个物理页为dirty，否则不这么认为；
    -   假如某物理页对应的所有虚拟页中**只要存在一个**被访问过的页，则认为这个物理页为被访问过的，否则不这么认为；

何时换入和换出：

-   在产生page fault的时候进行换入操作；
-   **换出操作源于在算法中将物理页的dirty从1修改成0的时候，**因此这个时候如果不进行写出到外存，就会造成数据的不一致，具体写出内存的时机是比较细节的问题, 可以在修改dirty的时候**写入外存**，或者是在这个物理页面上打一个需要写出的标记，到了最终删除这个物理页面(OR置换)的时候，如果发现了这个写出的标记，则在这个时候再写入外存；后者使用一个写**延迟标记**，有利于多个写操作的合并，从而降低缺页的代价；

### **实验中遇到的疑惑：**

1.  vmm.c中为什么答案需要page->pra_vaddr = addr ？

    +    pra_vaddr存储的是物理页对应的虚拟页起始地址——线性地址

2.  为什么page_insert之后答案没有tlb_invalidate(mm->pgdir, addr) ？

    +   page_insert的提示有,加上去也能通过测试

3.  _fifo_swap_out_victim中为什么struct Page *p = le2page(victim, pra_page_link)而不是使用page_link ? 

    +   因为pra_page_link参与的是按照时间顺序排列的链表,而此处FIFO的实现需要这个

4.  _fifo_swap_out_victim中答案是*ptr_page = p而不是p = *ptr_page ？

    +   ```makefile
        # 令*ptr_page存储被挑选出来的page(victim),而不是*ptr_page赋给page
        # 感觉对提示理解错了：then assign the value of *ptr_page to the addr of this page.
        # 应该是：把这个page的addr存储到*ptr_page
        # 为什么？没时间了,不想了,找了半天也不知道**ptr_page是什么，只找到了@ptr:a struct pointer of member
        #                          类似于page里面的page_link是meber的name,对应ptr就是list_entry_t *le = &free_list;
        # 					    然后list_next(le)找到对应page的那个项list_entry_t *le 或者说直接 &(p->page_link)
        ```