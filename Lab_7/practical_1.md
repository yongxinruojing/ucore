[TOC]



# **练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）**

完成练习0后，建议大家比较一下（可用meld等文件diff比较软件）个人完成的lab6和练习0完成后的刚修改的lab7之间的区别，分析了解lab7采用信号量的执行过程。执行`make grade`，大部分测试用例应该通过。

请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程。

请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。



上代码

```c
/***********************************内核级信号量描述****************************************/
typedef struct {
    int value;		// 类比原理课的sem
    wait_queue_t wait_queue;	// 等待队列
} semaphore_t;

void
sem_init(semaphore_t *sem, int value) {		//信号量init,等待队列init
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}

void
up(semaphore_t *sem) {		// 信号量的V操作
    __up(sem, WT_KSEM);
}
#define WT_KSEM                      0x00000100                    // wait kernel semaphore

// wait_state是等待的原因,此处为原因是内核信号量
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);	// 屏蔽中断
    {
        /* 1.申请一个等待项
           2.判断等待队列是不是为空,如果为空就仅仅sem++
****************************************若不为空就此处实现是assert等待原因和此时的原因是否相同(lab7是相同的？)然后唤醒这个进程
           类似于信号量原理的if(sem<=0)就跳出一个进程唤醒(若有)
        */
        wait_t *wait;		
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}


void
down(semaphore_t *sem) {
    uint32_t flags = __down(sem, WT_KSEM);	
    assert(flags == 0);	// lab7肯定返回0,不会出现不一样的等待原因
}
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {		// 代表sem > 0，也就是可以进入,直接sem--返回0即可
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    // 此时代表sem <= 0也就是不可以进入临界区执行
    /*
    1. 申请一个等待项,挂到等待队列里
    2. 使当前进程与wait连接,并且将当前进程的wait进入等待队列,当前进程sleeping
    */
    wait_t __wait, *wait = &__wait;	 
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);
	// 3. 执行调度程序选取一个新的进程执行
    schedule();

    /*
     此时代表等待的进程被唤醒,
     1. 首先把此进程对应的等待项删除,并从等待队列移除
     2. 判断唤醒我的是不是我之前等待的原因,如果是就返回0如果不是就返回这个唤醒我的原因交给高层进行下一步的判断执行
    */
    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);	
    local_intr_restore(intr_flag);
	
    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}

bool
try_down(semaphore_t *sem) {
    bool intr_flag, ret = 0;
    // 尝试sem--,如果成功就返回1,如果失败就返回0,即使失败也不等待/阻塞！！
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --, ret = 1;
    }
    local_intr_restore(intr_flag);
    return ret;
}
/* 执行流程就是
	1. sem_init
	2.       P操作：down --> __down
			 V操作：up --> __up

/**********************************************信号量哲学家************************************/
#define N 5 /* 哲学家数目 */
#define LEFT (i-1+N)%N /* i的左邻号码 */
#define RIGHT (i+1)%N /* i的右邻号码 */
#define THINKING 0 /* 哲学家正在思考 */
#define HUNGRY 1 /* 哲学家想取得叉子 */
#define EATING 2 /* 哲学家正在吃面 */
#define TIMES  4 /* 吃4次饭 */
#define SLEEP_TIME 10	// 睡眠/等待时间

//---------- philosophers problem using semaphore ----------------------
int state_sema[N]; /* 记录每个人状态的数组 */
/* 信号量是一个特殊的整型变量 */
semaphore_t mutex; /* 临界区互斥 */
semaphore_t s[N]; /* 每个哲学家一个信号量 */

struct proc_struct *philosopher_proc_sema[N];	// 哲学家指针数组,表示5个哲学家的proc

void phi_test_sema(i) /* i：哲学家号码从0到N-1 */
{ 
    // 如果我本身饥饿 and 左边不在吃 and 右边不在吃  
    //               ---->  我可以吃了,更改我的状态,执行up自己的信号量变为'1'(check_sync初始化为0了)
    /*			
    			sem.value>0，表示共享资源的空闲数
				sem.vlaue<0，绝对值表示该信号量的等待队列里的进程数
				sem.value=0，表示等待队列为空
				因为拿筷子的时候后续有个down操作,因此这里如果拿到了up之后后面down就不会阻塞
	*/
    if(state_sema[i]==HUNGRY&&state_sema[LEFT]!=EATING
            &&state_sema[RIGHT]!=EATING)
    {
        state_sema[i]=EATING;
        up(&s[i]);
    }
}

void phi_take_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{ 
        down(&mutex); /* 进入临界区 */
        state_sema[i]=HUNGRY; /* 记录下哲学家i饥饿的事实 */
        phi_test_sema(i); /* 试图得到两只叉子 */	// 所以是一次拿两个叉子！！
        up(&mutex); /* 离开临界区 */
        down(&s[i]); /* 如果得不到叉子就阻塞 */	  // 初始为0,在down的实现中这种情况是直接转为等待状态(不会减1)
}

void phi_put_forks_sema(int i) /* i：哲学家号码从0到N-1 */
{ 
        down(&mutex); /* 进入临界区 */
        state_sema[i]=THINKING; /* 哲学家进餐结束 */
        phi_test_sema(LEFT); /* 看一下左邻居现在是否能进餐 */ 	// 也就是让邻居测试可否进餐,可以的话就直接进餐！
        phi_test_sema(RIGHT); /* 看一下右邻居现在是否能进餐 */
        up(&mutex); /* 离开临界区 */
}

/*void指针：void *
	可以用其指代任何类型的指针。
	但不能用void指针直接进行操作；只能转换成对应类型指针后，才能操作
*/
int philosopher_using_semaphore(void * arg) /* i：哲学家号码，从0到N-1 */
{
    int i, iter=0;
    i=(int)arg;	// 转为int
    cprintf("I am No.%d philosopher_sema\n",i);
    while(iter++<TIMES)// 一个哲学家4次
    { /* 无限循环 */
        cprintf("Iter %d, No.%d philosopher_sema is thinking\n",iter,i); /* 哲学家正在思考 */
        do_sleep(SLEEP_TIME);	// 睡一会也就是思考一会
        phi_take_forks_sema(i); // 试一下哪两个叉子 失败就阻塞了
        /* 需要两只叉子，或者阻塞 */
        // 此时代表被唤醒了
        cprintf("Iter %d, No.%d philosopher_sema is eating\n",iter,i); /* 进餐 */
        do_sleep(SLEEP_TIME);	// 等待模拟进餐时间？
        phi_put_forks_sema(i); 
        /* 把两把叉子同时放回桌子 */
    }
    cprintf("No.%d philosopher_sema quit\n",i);
    return 0;    
}

void check_sync(void){

    int i;

    //check semaphore
    // 初始化mutex为1
    sem_init(&mutex, 1);	
    for(i=0;i<N;i++){
        sem_init(&s[i], 0); // 各个哲学家的sem初始化为0
        // 创建内核线程,philosopher_using_semaphore是加载的内核线程功能函数,i是传参,0代表分配新的空间
        // 注意kernel_thread之后这个内核线程就是就绪态了,随时可被调度！！
        // 是通过每次时钟中断run_timer_list() -> sched_class_proc_tick(current) -> proc_tick选取下一个要运行的进程
        // 这是前面lab6的内存,一时间没想起来 我就去查了查。。。。。。。
        // 然后创建好的哲学家就开始philosopher_using_semaphore了
        int pid = kernel_thread(philosopher_using_semaphore, (void *)i, 0);
        if (pid <= 0) {	// 创建失败了pid不可能<=0,只有idleproc的pid是0
            panic("create No.%d philosopher_using_semaphore failed.\n");
        }
        
        // 根据pid从hash_list找到proc
        philosopher_proc_sema[i] = find_proc(pid);
        // 设置名字
        set_proc_name(philosopher_proc_sema[i], "philosopher_sema_proc");
    }
}





```

### **问题1中的疑问**

为什么__up中需要assert(wait->proc->wait_state == wait_state);？

​	因为这个lab里面原因都一样,所以不可能wait_state不一样？

### **问题2的回答**

**参考**：https://www.cnblogs.com/ECJTUACM-873284962/p/11282818.html

将内核信号量机制迁移到用户态的最大麻烦在于，用于保证操作原子性的禁用中断机制、以及 CPU 提供的 Test and Set 指令机制都只能在内核态下运行，而使用软件方法的同步互斥又相当复杂，这就使得没法在用户态下直接实现信号量机制；于是，为了方便起见，可以将信号量机制的实现放在 OS 中来提供，然后使用系统调用的方法统一提供出若干个管理信号量的系统调用(这样时间消耗大！！)，分别如下所示：

-   申请创建一个信号量的系统调用，可以指定初始值，返回一个信号量描述符(类似文件描述符)；
-   将指定信号量执行 P 操作；
-   将指定信号量执行 V 操作；
-   将指定信号量释放掉；

给内核级线程提供信号量机制和给用户态进程/线程提供信号量机制的异同点在于：

-   相同点：
    -   提供信号量机制的代码实现逻辑是相同的；
-   不同点：
    -   由于实现原子操作的中断禁用、Test and Set 指令等均需要在内核态下运行，因此提供给用户态进程的信号量机制是通过系统调用来实现的，而内核级线程只需要直接调用相应的函数就可以了,**用户态由于频繁系统调用时间开销会较大**