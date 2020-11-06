[TOC]



# **练习1: 加载应用程序并执行（需要编码）**

**do_execv**函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。

请在实验报告中简要说明你的设计实现过程。

请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

```c

/* load_icode - load the content of binary program(ELF format) as the new content of current process
 * @binary:  the memory addr of the content of binary program
 * @size:  the size of the content of binary program
 */
static int
load_icode(unsigned char *binary, size_t size) {
    if (current->mm != NULL) {				// do_execve调用load_icode之前对mm进行了清空,所以这里一定要保证是空的
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;					// 内存空间不足错误代码
    struct mm_struct *mm;
    //(1) create a new mm for current process
    if ((mm = mm_create()) == NULL) {		// 如果失败直接返回内存不足的错误代码'-E_NO_MEM'
        goto bad_mm;
    }
    //(2) create a new PDT, and mm->pgdir= kernel virtual addr of PDT
    if (setup_pgdir(mm) != 0) {				
        // 如果分配page作为PDT失败,直接跳到mm_destroy - free mm and mm internal fields,然后还是返回内存不足的错误代码
        // PDT里面存的是PDE,PT里面存的是PTE
        goto bad_pgdir_cleanup_mm;
    }
    //(3) copy TEXT/DATA section, build BSS parts in binary to memory space of process
    //BSS:Block Started bySymbol,BSS段的变量只有名称和大小却没有值。此名后来被许多文件格式使用，包括PE。“以符号开始的块”指的是编译器处理未初始化数据的地方。BSS节不包含任何数据，只是简单的维护开始和结束的地址，以便内存区能在运行时被有效地清零。BSS节在应用程序的二进制映象文件中并不存在。bss段（Block Started by Symbolsegment）通常是指用来存放程序中未初始化的全局变量的一块内存区域，一般在初始化时bss段部分将会清零。bss段属于静态内存分配，即程序一开始就将其清零了。
    struct Page *page;
    //(3.1) get the file header of the bianry program (ELF format)
    struct elfhdr *elf = (struct elfhdr *)binary;	//binary是elf的起始地址
    //(3.2) get the entry of the program section headers of the bianry program (ELF format)
    struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);	//e_phoff是程序header的位置
    //(3.3) This program is valid?
    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;	
        //如果无效就直接跳到后面按顺序释放之前的建好的PDT和mm然后返回错误代码'-E_INVAL_ELF'
    }

    uint32_t vm_flags, perm;
    struct proghdr *ph_end = ph + elf->e_phnum;	//获取程序header的结束地址(可能多个program header)
    for (; ph < ph_end; ph ++) {
    //(3.4) find every program section headers
        if (ph->p_type != ELF_PT_LOAD) {
            continue ;		// 如果是不可加载类型直接跳到下一轮for循环
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
            // 此时还没有bss段,如果文件中段大小大于了分配的内存的段大小就直接跳到exit_mmap(mm)删除用户进程的合法内存空间然后就是类似的释放PDT和mm,返回错误代码'-E_INVAL_ELF'
        }
        if (ph->p_filesz == 0) {
            continue ;
            // 如果文件中段大小为0,那直接跳过进行下一个program header的for循环
        }
    //(3.5) call mm_map fun to setup the new vma ( ph->p_va, ph->p_memsz)
        vm_flags = 0, perm = PTE_U;							// 判断ELF的program flags赋予相应的vm的flags
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;	// 代码段可执行
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;	// 数据段可读写
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        if (vm_flags & VM_WRITE) perm |= PTE_W;				// 如果vm可写就置PTE对应标志位PTE_W
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            // 设定用户进程的合法内存空间,返回值为0代表成功！否则直接跳到exit_mmap(mm)按照一样的流程删除用户进程的合法内存空间、清除之前的mm、PDT
            goto bad_cleanup_mmap;
        }
        unsigned char *from = binary + ph->p_offset;	// file offset of segment
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);
		// p_va去映射段的虚拟地址,la为虚拟地址的起始地址(4KB对其)
        ret = -E_NO_MEM;

     //(3.6) alloc memory, and  copy the contents of every program section (from, from+end) to process's memory (la, la+end)
        end = ph->p_va + ph->p_filesz;	
     //(3.6.1) copy TEXT/DATA section of bianry program	建立虚地址和物理地址映射关系 把section内容拷贝进来
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memcpy(page2kva(page) + off, from, size);
            start += size, from += size;
        }

      //(3.6.2) build BSS section of binary program
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
    //(4) build user stack memory
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    
    //(5) set current process's mm, sr3, and set CR3 reg = physical addr of Page Directory
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    //(6) setup trapframe for user environment
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    /* LAB5:EXERCISE1 YOUR CODE
     * should set tf_cs,tf_ds,tf_es,tf_ss,tf_esp,tf_eip,tf_eflags
     * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. So
     *          tf_cs should be USER_CS segment (see memlayout.h)
     *          tf_ds=tf_es=tf_ss should be USER_DS segment
     *          tf_esp should be the top addr of user stack (USTACKTOP)
     *          tf_eip should be the entry point of this binary program (elf->e_entry)
     *          tf_eflags should be set to enable computer to produce Interrupt
     */
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;		// 置中断标志1
    ret = 0;
out:
    return ret;
bad_cleanup_mmap:
    exit_mmap(mm);
bad_elf_cleanup_pgdir:
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    goto out;
}
```

### **问题回答**

​      分析在创建了用户态进程并且加载了应用程序之后，其占用 CPU 执行到具体执行应用程序的整个经过：

**参考来自：**https://www.cnblogs.com/ECJTUACM-873284962/p/11282776.html

1.  在经过调度器占用了 CPU 的资源之后，用户态进程调用了 exec 系统调用，从而转入到了系统调用的处理例程
2.  在经过了正常的中断处理例程之后，最终控制权转移到了 syscall.c 中的 syscall 函数，然后根据系统调用号转移给了 sys_exec 函数，在该函数中调用了上文中提及的 do_execve 函数来完成指定应用程序的加载；
3.  在do_execve中进行了若干设置，包括推出当前进程的页表，换用 kernel 的 PDT 之后，使用 load_icode 函数，完成了对整个用户线程内存空间的初始化，包括堆栈的设置以及将 ELF 可执行文件的加载，之后通过 current->tf 指针修改了当前系统调用的 trapframe，使得最终中断返回的时候能够切换到用户态，并且同时可以正确地将控制权转移到应用程序的入口处(forkret→iret跳到ring3的执行地址)；
4.  在完成了 do_execve 函数之后，进行正常的中断返回的流程，**(load_icode修改了栈信息SS对应的RPL也为3,EIP为执行地址的offset,ESP为堆栈的offset)**由于中断处理例程的栈上面的 eip 已经被修改成了应用程序的入口处，而 CS 上的 RPL 是用户态，因此 iret 进行中断返回的时候会将堆栈切换到用户的栈，并且完成特权级的切换，并且跳转到要求的应用程序的入口处；
5.  接下来开始具体执行应用程序的第一条指令；

syscall $\rightarrow$ sys_exec $\rightarrow$ do_execve $\rightarrow$ load_icode $\rightarrow$ do_execve $\rightarrow$ iret $\rightarrow$ entry-user-start