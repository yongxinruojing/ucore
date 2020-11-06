[TOC]

# 练习3

**练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）**

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解page_remove_pte函数中的注释。为此，需要补全在 kern/mm/pmm.c中的page_remove_pte函数。page_remove_pte函数的调用关系图如下所示：

![img](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2_figs/image002.png)

图2 page_remove_pte函数的调用关系图

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

-   数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
-   如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ **鼓励通过编程来具体完成这个问题**

```c
if(*ptep & PTE_P){
    struct Page *page = pte2page(*ptep);
    page_ref_dec(page);
    if(page->ref == 0)				//答案是二者合到了一起if(page_ref_dec(page) == 0),之前看代码没发现返回值就是ref
    {	
        free_page(page);
       // *ptep = (*ptep) & (~PTE_P);  自己写的测试通过了,但是和答案不一致,为了防止后面的实验导致某些错误 以答案为主把
    }
    *ptep = 0;						//直接清0简单粗暴
    tlb_invalidate(pgdir, la);

}
```

问题回答：

1.  pages为页结构的物理起始地址，page存储的对应物理页的信息,从物理地址空间分布是从page管理信息再到实际物理页的。通过page-pages获取物理页号然后左移12位得到对应page的实际物理地址,pte中存储的是物理页的物理地址，这个物理地址可以通过pte2page宏获得，这个物理地址和通过page-pages然后左移12位得到的页的物理地址一样

2.  首先更改KERNBASE,0xc0000000 → 0x00000000，然后更改链接脚本tools/kernel.ld，将虚拟地址改为0x100000，最后注释掉取消0~4M区域内存页映射的代码(为什么我的entry.S没有这一段？？我失败了)

    ```c
    //disable the map of virtual_addr 0~4M
    // boot_pgdir[0] = 0;
    ```

#### 实验中的重点总结：

1.  PTE和PDE存的地址都是物理地址！！从PDE中获取页表项的物理地址后通过KADDR()将其转为虚拟地址才可以去访问页表！！！
2.  \*, &, memset等操作的都是虚拟地址！！
3.  alloc_page()获取的是物理page对应的Page结构体，不是物理page，需要通过page2pa()才可以获得Page结构体对应的物理page的物理地址，然后才可以获得虚拟地址