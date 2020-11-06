[TOC]



# 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）

请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：

-   在本实验的执行过程中，创建且运行了几个内核线程？
-   语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

完成代码编写后，编译并运行代码：make qemu

如果可以得到如 附录A所示的显示内容（仅供参考，不是标准答案输出），则基本正确。



上代码

```c
/* *
 * load_esp0 - change the ESP0 in default task state segment,
 * so that we can use different kernel stack when we trap frame
 * user to kernel.
 * 修改TSS的esp0
 * */
void
load_esp0(uintptr_t esp0) {
    ts.ts_esp0 = esp0;
}

switch_to:                      # switch_to(from, to)

    # save from's registers     保存from的context
    movl 4(%esp), %eax          # eax points to from	4(%esp)就是*(%esp+4)
    							# from向上4个字节是参数亦即from.con text的地址
    popl 0(%eax)                # save eip !popl	通过特殊手段把from的eip保存到swap_context里
    movl %esp, 4(%eax)          # save esp::context of from	把esp的内容存到from的context里
    movl %ebx, 8(%eax)          # save ebx::context of from 下面是参数存到from的context里面
    movl %ecx, 12(%eax)         # save ecx::context of from
    movl %edx, 16(%eax)         # save edx::context of from
    movl %esi, 20(%eax)         # save esi::context of from
    movl %edi, 24(%eax)         # save edi::context of from
    movl %ebp, 28(%eax)         # save ebp::context of from

    # restore to's registers    恢复to的context
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to 取到to的context指针
    movl 28(%eax), %ebp         # restore ebp::context of to  把context信息逐一导入对应寄存器完成寄存器恢复
    movl 24(%eax), %edi         # restore edi::context of to
    movl 20(%eax), %esi         # restore esi::context of to
    movl 16(%eax), %edx         # restore edx::context of to
    movl 12(%eax), %ecx         # restore ecx::context of to
    movl 8(%eax), %ebx          # restore ebx::context of to
    movl 4(%eax), %esp          # restore esp::context of to

    pushl 0(%eax)               # push eip	为什么push eip？ 因为下面要ret 直接ret返回的就是这个eip存的forkret的地址

    ret

// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {			// 如果proc不是当前运行的进程
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);	// 关中断
        {
            current = proc;			// 把当前的进程变为proc
            load_esp0(next->kstack + KSTACKSIZE);	// 载入esp0
            lcr3(next->cr3);						// 载入proc的页目录表,lab4貌似用处不大因为都是内核线程都一样
            switch_to(&(prev->context), &(next->context));	
            //switch_to(struct context *from, struct context *to)被c语言调用的一个函数,本身用汇编实现在switch.S
            //切换上下文
        }
        local_intr_restore(intr_flag);	// 开中断
    }
}

```

### **问题回答**

创建两个并运行两个内核线程

一个关中断,一个开中断

理由看代码

```c
#define FL_IF           0x00000200  // Interrupt Flag
/* intr_enable - enable irq interrupt */
void
intr_enable(void) {
    sti();
}

/* intr_disable - disable irq interrupt */
void
intr_disable(void) {
    cli();
}

static inline bool
__intr_save(void) {
    if (read_eflags() & FL_IF) {	// 如果是开中断
        intr_disable();		// 关中断 cli()
        return 1;
    }
    return 0;
}

static inline void
__intr_restore(bool flag) {
    if (flag) {	
        intr_enable();		// 开中断 sti()
    }
}

#define local_intr_save(x)      do { x = __intr_save(); } while (0)
#define local_intr_restore(x)   __intr_restore(x);

```

