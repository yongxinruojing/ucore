# **练习0：填写已有实验**

本实验依赖实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意：为了能够正确执行lab5的测试应用程序，可能需对已完成的实验1/2/3/4的代码进行进一步改进。

```
根据提示修改了部分内容
alloc_proc:
	proc->wait_state = 0; 		
	proc->cptr = NULL;		children and proc is parent
	proc->yptr = NULL;		younger sibling and proc is younger sibling
	proc->optr = NULL;		older sibling and proc is younger sibling
do_fork:
	if ((proc = alloc_proc()) == NULL) {
        goto fork_out;
    }
    proc->parent = current;
	assert(current->wait_state == 0);	//**********

    if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    copy_thread(proc, stack, tf);

    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
		set_links(proc);				//**********
        nr_process ++;
    }
    local_intr_restore(intr_flag);
trap.c
	idt_init:
		// 设置给用户态用的中断门 让用户态能够进行系统调用
   		 SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
   		 
    trap_dispatch:
    	  ticks ++;
        if (ticks % TICK_NUM == 0) {
            print_ticks();
	   		assert( current != NULL );
            current->need_resched = 1;****************
        }
```

