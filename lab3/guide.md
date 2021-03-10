# 多道程序与分时多任务

这一章我们主要进行任务切换，任务的交替执行对于现代计算机是极端重要的，否则一旦开始跑一个程序，你的电脑就会“死机”、“无响应”，这是不能忍受的。

具体而言，我们将要完成 [sys_yield]() 系统调用，这样用户程序就可以在空闲的时候暂停执行并把执行权交给其他进程，这就是协作式调度。但是期盼用户程序都是那么的好心是不现实的，我们还将为内核加入时钟中断，同时以时钟中断为节点，不断的在进程间切换，也就是抢占式调度。

[rust指导书](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter3/0intro.html#id5) 中对这一章结果很好的展示。

## 现场保存与恢复

任务切换要求我们对于用户的执行现场进行保存和恢复。欸，现场的保存与恢复？我们似乎已经干过类似的工作！在 lab2 中，我们已经完成了用户现场的保存，就在 `trapframe` 里面，那么是不是说我们只要在返回的时候指定另一个任务的 `trapframe` 就可以直接开始运行另一个任务呢？答案是肯定的！其实 [rCore](https://github.com/rcore-os/rCore) 和 [zCore](https://github.com/rcore-os/zCore) 就是这种设计。但是如果想要这么实现需要更加复杂的 `trapframe` 管理，这里我们采用一种复古的实现方式。

> 在传统的模型中，每有一个正在运行的用户进程，就要有一个内核线程与之对应，不同进程的最显著区别在于地址空间，也就是页表位置。事实上，在过去的 [ucore 框架](https://github.com/LearningOS/ucore_os_lab) 中，进入内核是不换页表的，内核态和用户态共用页表。但是这样其实会导致一些[侧信道攻击](TODO: google崩了)，所以内核与用户态的页表隔离是必须的。

> 在上述模型中，内核完成进程切换需要完成内核态的切换，我们可以认为不同进程的内核态是执行同一段代码的不同线程，这些线程有不同的页表、内核堆栈、寄存器信息。所以它们之间的切换必须将这三样全部切换。这种切换必须使用汇编完成，否则将十分危险！因为编译器并不能安全的处理这种操作。

我们基本上还是使用原有的进程模型，不过由于我们在内核态-用户态切换的时候处理了页表，所以切换时不需要处理 `satp`，但是需要完成寄存器的切换（也就是寄存器的保存和恢复）已经堆栈切换（本质是 `sp` 寄存器的切换）。

要在不同内核线程之间切换首先需要有不同的内核线程。在新增的 `proc.h`、`proc.c` 文件种，可以看到进程的定义和初始化。

```c
// kernel/trap.h
struct proc {
    enum procstate state;   // 进程状态
    int pid;                // 进程ID
    uint64 ustack;
    uint64 kstack;
    struct trapframe *trapframe; 
    struct context context; // 用于保存进程寄存器信息，用于切换
};
```

其中 `trapframe` 定义不变，其余结构体定义如下：

```c
// kernel/trap.h

// Saved registers for kernel context switches.
struct context {
    uint64 ra;
    uint64 sp;
    // callee-saved
    uint64 s0;
    uint64 s1;
    uint64 s2;
    uint64 s3;
    uint64 s4;
    uint64 s5;
    uint64 s6;
    uint64 s7;
    uint64 s8;
    uint64 s9;
    uint64 s10;
    uint64 s11;
};
```
```c
// kernel/trap.h
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```
目前我们只使用这些状态中的一小部分。

进程池定义以及初始化如下：

```c
// kernel/trap.c
struct proc pool[NPROC];
struct proc idle;
struct proc* current_proc;

char kstack[NPROC][PAGE_SIZE];
char ustack[NPROC][PAGE_SIZE];
char trapframe[NPROC][PAGE_SIZE];
extern char boot_stack_top[];
```
```c
// kernel/trap.c
void
procinit(void)
{
    struct proc *p;
    for(p = pool; p < &pool[NPROC]; p++) {
        p->state = UNUSED;
        p->kstack = (uint64)kstack[p - pool];
        p->ustack = (uint64)ustack[p - pool];
        p->trapframe = (struct trapframe*)trapframe[p - pool];
    }
    idle.kstack = (uint64)boot_stack_top;
    idle.pid = 0;
}
```

