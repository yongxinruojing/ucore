[TOC]



# 练习1：分配并初始化一个进程控制块（需要编码）

alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。

>   【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

-   请说明proc_struct中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

```c
// alloc_proc - alloc a proc_struct and init all fields of proc_struct
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    //LAB4:EXERCISE1 YOUR CODE
    /*
     * below fields in proc_struct need to be initialized
     *       enum proc_state state;                      // Process state
     *       int pid;                                    // Process ID
     *       int runs;                                   // the running times of Proces
     *       uintptr_t kstack;                           // Process kernel stack
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
     *       struct proc_struct *parent;                 // the parent process
     *       struct mm_struct *mm;                       // Process's memory management field
     *       struct context context;                     // Switch here to run process
     *       struct trapframe *tf;                       // Trap frame for current interrupt
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
     *       uint32_t flags;                             // Process flag
     *       char name[PROC_NAME_LEN + 1];               // Process name
     */
    proc->state = PROC_UNINIT;
    proc->pid = -1;										// 我写的是get_pid(),答案是-1,我理解错了初始化的含义
    proc->runs = 0;
    // 我写的是while( setup_kstack(proc) != 0); 如果分配失败就循环
    proc->kstack = 0;
    proc->need_resched = 0;
    proc->parent = NULL;
    //我写成了current
    proc->mm = NULL;	
    //在实际OS中，内核线程常驻内存，不需要考虑swap page问题，在lab5中涉及到了用户进程，才考虑进程用户内存空间的swap page问题，mm才会发挥作用。所以在lab4中mm对于内核线程就没有用了，这样内核线程的proc_struct的成员变量*mm=0是合理的 
    //我写的是proc->cr3 = &pde_t;
    //pmm.h有：extern pde_t *boot_pgdir;;这个是虚拟地址,cr3存的应该是物理地址因此是boot_cr3,他是通过PADDR(boot_pgdir)得到的
    proc->cr3 = boot_cr3;
    //mm里有个很重要的项pgdir，记录的是该进程使用的一级页表的物理地址。由于*mm=NULL，所以在proc_struct数据结构中需要有一个代替pgdir项来记录页表起始地址，这就是proc_struct数据结构中的cr3成员变量。
    
    //proc->context = NULL; 自己写的,看了答案之后这个发现明显错误
    memset(&(proc->context), 0, sizeof(struct context));
    //初始化全置0
    proc->tf = NULL;    
    proc->flags = 0;
    //proc->name = 0;和上面context错误的原因一样
    //memset(proc->name, 0, PROC_NAME_LEN)    //答案是不是错了,因为长度不是PROC_NAME_LEN+1吗
    memset(proc->name, 0, PROC_NAME_LEN+1);
    }
    return proc;
}


//其他分析
// copy_thread - setup the trapframe on the  process's kernel stack top and
//             - setup the kernel entry point and stack of process
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;	
    //首先proc->kstack为0然后大小4B,+ KSTACKSIZE为8KB,于是proc->kstack + KSTACKSIZE就为内核堆栈的栈顶,把这个地址转换类型为trapframe *地址然后减去其一个类型的大小(76B)的地址赋值给proc->tf,类似压栈？
	//测试了struct trapframe占用内存76个字节,把这个tf放到了此线程内核堆栈的栈底(高地址)的76个字节	
    *(proc->tf) = *tf;			//把结构体tf里面的值!赋值给proc->tf结构体
    proc->tf->tf_regs.reg_eax = 0;
    proc->tf->tf_esp = esp;
    proc->tf->tf_eflags |= FL_IF;
    proc->context.eip = (uintptr_t)forkret;		
    // 存储forkret的地址,其完成中断的恢复执行过程,forkret之后才是内核线程入口地址kernel_thread_entry,然后是fn完成函数功能
    proc->context.esp = (uintptr_t)(proc->tf);  // 存储trapframe的地址
}
```

### **问题回答**

context：对于ucore而言是进程or线程的上下文,当执行内核线程的时候切换完地址空间之后切换context上下文

tf：被中断or异常打断的进程or线程当前的状态包括硬件保存的堆栈的信息,用于特权级转换的信息,和用软件保存的段寄存器的信息和通用寄存器的信息以及中断号。此外uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore 在内核栈上维护了 tf 的链(没找到。。。。)。