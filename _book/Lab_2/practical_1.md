#  练习1

**练习1：实现 first-fit 连续物理内存分配算法（需要编程）**

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default_pmm.c中的default_init，default_init_memmap，default_alloc_pages， default_free_pages等相关函数。请仔细查看和理解default_pmm.c中的注释。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

-   你的first fit算法是否有进一步的改进空间





自己刚开始写的错误答案

```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {		// 类似轮循机制	
        struct Page *p = le2page(le, page_link);		// 获取此page_link所在的page结构
        if (p->property >= n) {							// 连续内存块(页)的个数>=n直接分配
            page = p;
            break;
        }
    }
    if (page != NULL) {									// 如果分配成功
        //ClearPageProperty(page);	
        list_del(&(page->page_link));					// 双向链表中删除这个页的link
        if (page->property > n) {						// 如果页的个数大于需要的个数就切分
            struct Page *p = page + n;
            p->property = page->property - n;
          //page->property = n;
            list_add(&free_list, &(p->page_link));		// 并把剩下的加入到free_list
           //list_add(page->page_link),&(p->page_link));//插在原先的后面然后删除原先的
           //list_del(&(page->page_link));
    }
        nr_free -= n;									// 别忘了 nr_free需要减少n(在分配成功的情况下)
        ClearPageProperty(page);						// 同时置page的flags的bit1为0,表示连续的一些页已经被分配了
    }
    return page;
}

static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {						// 对于每一个page
        assert(!PageReserved(p) && !PageProperty(p));	// 不被保留且被分配
        p->flags = 0;									// 直接置flags两位均为0,若非第一页p->property也要置0
        set_page_ref(p, 0);								// 设置对应的ref被页表的引用次数也为0
    }
    base->property = n;
    SetPageProperty(base);								// 设置此连续页的头一页为free
    list_entry_t *le = list_next(&free_list);
    while (le != &free_list) {							// 整理and合并空闲空间
        p = le2page(le, page_link);						// 暂存此page
        le = list_next(le);								// 获取下一个page
        if (base + base->property == p) {				// 如果此空闲块空间恰好就是此p之前的(base结尾与p开头紧挨)
            base->property += p->property;				// 		二者数量加起来 → base->property
            ClearPageProperty(p);						//		page置为非free,注释的规定。。
            list_del(&(p->page_link));					//		free_list中删除这个page
            //list_add_before(&(p->page_link),&(base->page_link));
            //list_del(&(p->page_link));
        }
        else if (p + p->property == base) {				// 如果p恰好在我这个base之前(p结尾与base开头紧挨)
            p->property += base->property;				//		仍为二者数量相加 → p->property
            ClearPageProperty(base); 					// 		base置为非free
             //1.为什么要删除？不需要删除！！下面两句都不需要了吧，我把这个单独测试了一下 行不通，，，，，！！！
            base = p;									//		同时把p作为新的基址
            list_del(&(p->page_link));					//		别忘了删除free_list中的这个p
        }
        /* 2.这个是对的，但是即使加了这个之后前面两个的判断还是需要while，得不偿失，还不如直接省去然后一起add
        else if(base + base->property < p){
        	list_add_before(&(p->page_link),&(base->page_link));
        	break; 
        }
        */
    }
    nr_free += n;										// 最后不要忘记增加nr_free
    list_add(&free_list, &(base->page_link));			// 然后重新把合并(若有)后的base->page_link加到free_list中
	//把删除移到里面方便
}

```

错误答案总结：

1.  不知道为什么当base恰好在p之后的时候直接修改p->property不行吗
2.  while的时候加个判断base + base->property < p不正好找到了插入点吗，测试通过了，但是如果第一点不能满足好像不能优化多少，，，，



正确答案(参考了老师给的参考答案，不过我写到代码里面写的是老师的答案——我怕自己的可能有问题虽然测试通过了)

```c
struct Page {
    int ref;        // page frame's reference counter，亦即虚拟页的映射
    uint32_t flags; // array of flags that describe the status of the page frame
    unsigned int property;// the num of free block, used in first fit pm manager
    list_entry_t page_link;// free list link，空闲列表链接
};
//根据提示
//1. 先看'default_init'
	//首先初始化每个页
	//1.设置'p->flags'的'PG_property'
	//	如果是空闲页且不是一个空闲块的第一个页，置'p->property'为0
	//	如果是空闲页且是一个空闲块的第一个页，置'p->property'为此空闲块包含的页的数量
	//	'p->ref'置为0，因为此时'p'是空闲的，没有引用
	//2.用'p->page_link'连接页到'free_list'(此记录空闲内存块)
static void
default_init(void) {
    list_init(&free_list);				//初始化
    nr_free = 0;						//置0
}

static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);						//若页数量为0，报错
    struct Page *p = base;	
    for (; p != base + n; p ++) {
        assert(PageReserved(p));		//若是未Reserved就报错,因为之前pmm_init已经把这个bit置1了
//而调用路径是CALL GRAPH: 
 //`kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` --> `pmm_manager` --> `init_memmap`.
        
        p->flags = p->property = 0;		//除了第一个其他的都设置为0
        set_page_ref(p, 0);				//页引用为0
    }
    base->property = n;					//第一个设置为n
    SetPageProperty(base);				//设置PG_property为1，空闲
	nr_free += n;						//更新空闲内存块的页的数量
	list_add(&free_list, &(base->page_link));	//把空闲块(页为最小单位)增加到'free_list'的后面
}

//2.看'default_alloc_pages'
	//根据要求对源代码就那些了修改
	//总的逻辑就是
		//先判断剩下的空闲块够不够分，若可以分----然后判断是不是大了，进行切割一下并调整一下'free_list'的结构
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);//将le的page_link转化到page结构
        if (p->property >= n) {					//先判断是否可以分配
            page = p;							//把free_list地址传给page
            break;
        }
    }
    if (page != NULL) {
        if (page->property > n) {				//若大于n	
            struct Page *p = page + n;			//p相当于分完后剩下的起始页
            p->property = page->property - n;	//p的大小为原大小减去分配走的
            SetPageProperty(p);					//设置PG_property为1，表示空闲
            list_add_after(&(page->page_link),&(p->page_link));	//再把分配后的空闲页连接到之前位置的后面
            														//(前面也可)
    }
		list_del(&(page->page_link));			//将page_link删除，p顺利成章接替其位置
        nr_free -= n;							//空闲块总量(页的个数)也相应减小
        ClearPageProperty(page);				//设置PG_property为0，不空闲
    }
    return page;								//返回分配的页
}

//3.'default_free_pages'
//'p->ref'置0
//置PG_property为1
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);	
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));	//未保留和非空闲状态
        p->flags = 0;									//与初始化类似置0
        set_page_ref(p, 0);								//置引用为0
    }
    base->property = n;									//设置初始为n
    SetPageProperty(base);								//置为空闲
    list_entry_t *le = list_next(&free_list);			//获取'free_list'的后一个元素
    int i = 0;
    while (le != &free_list) {							//循环
        p = le2page(le, page_link);		
		le = list_next(le);					
        if (base + base->property == p) {				//base恰好在p之前
            base->property += p->property;				//设置为自身的个数+p的个数
            ClearPageProperty(p);  						//清除p的状态，此时base为起始
			list_del(&(p->page_link));					//移除p
        }
        else if (p + p->property == base) {				//若base正好在p之后
            p->property += base->property;
            ClearPageProperty(base);					//清除base的状态，p为起始
            base = p;									
            list_del(&(p->page_link));					//移除p
        }	
        else if(base + base->property < p){				//找到插入的直接提前跳出循环
        	list_add_before(list_prev(le),&(base->page_link));
            i = 1;
        	break; 
        }	
    }
	nr_free += n;										//空闲页数+n
    if( i == 0)
    {
        le = list_next(&free_list);							//
        while(le != &free_list){							//找插入位置
            p = le2page(le, page_link);
            if(base + base->property <=p){					//第一个地址超过or等于base+n的空闲块为插入地址
                assert(base + base->property != p);
                break;
            }
            le = list_next(le);	
        }//找到插入位置
        list_add_before(le, &(base->page_link));			//插在其前面
    }
}

```

