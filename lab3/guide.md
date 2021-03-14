# 多道程序与分时多任务

这一章我们主要进行任务切换，任务的交替执行对于现代计算机是极端重要的，否则一旦开始跑一个程序，你的电脑就会“死机”、“无响应”，这是不能忍受的。

具体而言，我们将要完成 [sys_yield]() 系统调用，这样用户程序就可以在空闲的时候暂停执行并把执行权交给其他进程，这就是协作式调度。但是期盼用户程序都是那么的好心是不现实的，我们还将为内核加入时钟中断，同时以时钟中断为节点，不断的在进程间切换，也就是抢占式调度。

[rust指导书](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter3/0intro.html#id5) 中对这一章结果很好的展示。

## 现场保存与恢复

任务切换要求我们对于用户的执行现场进行保存和恢复。欸，现场的保存与恢复？我们似乎已经干过类似的工作！在 lab2 中，我们已经完成了用户现场的保存，就在 `trapframe` 里面，那么是不是说我们只要在返回的时候指定另一个任务的 `trapframe` 就可以直接开始运行另一个任务呢？答案是肯定的！其实 [rCore](https://github.com/rcore-os/rCore) 和 [zCore](https://github.com/rcore-os/zCore) 就是这种设计。但是如果想要这么实现需要更加复杂的 `trapframe` 管理，这里我们采用一种复古的实现方式。

> 在传统的模型中，每有一个正在运行的用户进程，就要有一个内核线程与之对应，不同进程的最显著区别在于地址空间，也就是页表位置。事实上，在过去的 [ucore 框架](https://github.com/LearningOS/ucore_os_lab) 中，进入内核是不换页表的，内核态和用户态共用页表。但是这样其实会导致一些[侧信道攻击](TODO: google崩了)，所以内核与用户态的页表隔离是必须的。

> 在上述模型中，内核完成进程切换需要完成内核态的切换，我们可以认为不同进程的内核态是执行同一段代码的不同线程，这些线程有不同的页表、内核堆栈、寄存器信息。所以它们之间的切换必须将这三样全部切换。这种切换必须使用汇编完成，否则将十分危险！因为编译器并不能安全的处理这种操作。

我们基本上还是使用原有的进程模型，不过由于我们在内核态-用户态切换的时候处理了页表，所以切换时不需要处理 `satp`，但是需要完成寄存器的切换（也就是寄存器的保存和恢复）已经堆栈切换（本质是 `sp` 寄存器的切换）。

要在不同内核线程之间切换首先需要有不同的内核线程。在新增的 `proc.h`、`proc.c` 文件中，可以看到进程的定义和初始化。

```c
// kernel/trap.h
struct proc {
    enum procstate state;   // 进程状态
    int pid;                // 进程ID
    uint64 ustack;
    uint64 kstack;
    struct trapframe *trapframe; 
    struct context context; // 用于保存进程内核态的寄存器信息，进程切换时使用
};
```

`struct proc` 一般称作进程控制块，因为它包含了进程几乎所有的信息（例如保存内核栈上的信息可以通过 `trapframe`访问）。 `trapframe` 定义不变，`context`结构体定义如下：

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

进程池定义以及进程池初始化如下所示：

```c
// kernel/trap.c
struct proc pool[NPROC];
struct proc idle;           // boot proc，始终存在
struct proc* current_proc;  // 指示当前进程

char kstack[NPROC][PAGE_SIZE];
char ustack[NPROC][PAGE_SIZE];
char trapframe[NPROC][PAGE_SIZE];
extern char boot_stack_top[];   // bootstack，用作 idle proc kernel stack
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

被标记为 `UNUSED` 表明进程池该位置没有被使用。

## 用户进程加载

然后我们来看看用户进程的加载与 lab2 有何异同。加载同样在 `batch.c` 完成。

```c
// kernel/batch.c

int load_app(int n, uint64* info) {
    uint64 start = info[n], end = info[n+1], length = end - start;
    memset((void*)BASE_ADDRESS + n * MAX_APP_SIZE, 0, MAX_APP_SIZE);
    memmove((void*)BASE_ADDRESS + n * MAX_APP_SIZE, (void*)start, length);
    return length;
}

int run_all_app() {
    for(int i = 0; i < app_num; ++i) {
        struct proc* p = allocproc();    // 分配一个进程控制块
        struct trapframe* trapframe = p->trapframe;
        printf("run app %d\n", i);
        load_app(i, app_info_ptr);
        uint64 entry = BASE_ADDRESS + i*MAX_APP_SIZE;
        trapframe->epc = entry;
        trapframe->sp = (uint64) p->ustack + PAGE_SIZE;
        p->state = RUNNABLE;
    }
    return 0;
}
```

可以看到，进程 load 的逻辑其实没有变化，但是：

* 一次性加载所有进程，而不是运行完一个再加载一个。
* 每个进程加载的位置不同，而不是加载于同一位置。

这个我们认为设定每个进程所使用的空间是 `[0x80400000 + id*0x20000, 0x80400000 + (id+1)*0x20000)`，每个进程的最大 size 为 0x20000，id 即为进程编号。这个变化是平凡的，因为我们要同时运行多个进程，自然需要所有进程同时存在于内存中。

> 用户态的进程编译也需要按照这个要求编译，也就是第 i 个程序的起始地址必须为 `0x80400000 + i*0x20000`, 示例代码/测例代码用户态都已经实现这一点。

`allocproc()` 函数的作用是从进程池中找到一个 `UNUSED` 的进程控制块，进行基本初始化并返回其指针，具体如下：

```c
// kernel/proc.c
struct proc* allocproc(void)
{
    struct proc *p;
    for(p = pool; p < &pool[NPROC]; p++) {
        if(p->state == UNUSED) {
            goto found;
        }
    }
    return 0;

    found:
    p->pid = allocpid();    // 分配一个没有被使用过的 id
    p->state = USED;        // 标记该控制块被使用
    memset(&p->context, 0, sizeof(p->context));
    memset(p->trapframe, 0, PAGE_SIZE);
    memset((void*)p->kstack, 0, PAGE_SIZE);
    // 初始化第一次运行的上下文信息
    p->context.ra = (uint64)usertrapret;    
    p->context.sp = p->kstack + PAGE_SIZE;
    return p;
}
```

其中，对于 `p->context` 的初始化和 lab2 中我们对于 `trapframe` 的初始化有异曲同工之妙。开始运行一个进程，在这里复用了程序 yield 之后重新开始运行的流程。进程切换的整体流程如下：

```c

1.程序调用 yield 暂停执行 -> 2.进入内核态 -> 3.内核态切换切到其他进程 -> ... 4.其他进程执行 ... -> 5.其他进程通过相同流程切回到该进程 -> 6.返回用户态 -> 7. 用户态继续执行。

```

这其中，`1` `2` `6` `7` 几个步骤都是 lab2 中的标准步骤，进程切换相当于一个特殊的中断处理。而开始运行一个进程从 lab2 中的 `6` `7` 变成了 `5` `6` `7` 。

这一章核心的内容就是 `3` `5` 两个新增步骤了，我们来看看这两个步骤的核心流程。

## 进程切换核心函数

当一个内核线程判断自己要切换出去的时候，它会调用 `sched`函数，并最终通过`swtch` 函数完成切换：

```c
// kernel/trap.c
void
sched(void)
{
    struct proc *p = curr_proc();
    swtch(&p->context, &idle.context);
}
```

`swtch` 函数的两个参数就是进程控制块的 `context` 结构体的指针，分别为当前进程的和目标进程的。`swtch` 函数设计编译器不能控制的行为，必须汇编实现，**但是编译器还是帮助我们干了一些事情，如保存调用者保存寄存器，设定 ra**。`swtch` 实现如下：

```c
# Context switch
#
#   void swtch(struct context *old, struct context *new);
#
# Save current registers in old. Load from new.


.globl swtch

# a0 = &old_context, a1 = &new_context

swtch:
    sd ra, 0(a0)        # save `ra`
    sd sp, 8(a0)        # save `sp`
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    ld ra, 0(a1)        # restore `ra`
    ld sp, 8(a1)        # restore `sp`
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)

    ret                 # return to new `ra`
```

这个函数的汇编实现看似简单，但其实干了一些常规函数绝对不敢干的事情：

* 改变了 `ra`，使得函数 ret 时，不会返回调用它的函数（这里不妨成为父函数），而是另一个地方（储存在 `0(a1)`中）。事实上，返回的地方是其他因为调用 `swtch` 函数而没有继续执行的父函数。
* 改变了 `sp`，该函数没有任何对堆栈的直接操作，所以直接修改 sp 并没有破坏该函数的执行。

该函数成功的切换了：
* 执行流：通过切换 `ra`
* 堆栈：通过切换 `sp`
* 寄存器：通过保存和恢复被调用者保存寄存器。调用者保存寄存器由编译器生成的代码负责保存和恢复。

接下来，我们来看看进程切换的整体流程。

## 进程切换整体代码框架

你可能已经注意到了，示例代码中，`sched` 函数并没有如我们所设想的进程切换模型：找到下一个执行的进程，然后 `swtch` 过去。而是直接切到了 `idle` 进程。那么 `idle` 进程会从那里继续执行呢？思考这个问题之前，我们需要先搞明白，到底是谁开始执行了第一个进程？没错，就是 `idle` 进程，`idle` 进程是第一个进程(boot进程)，也是唯一一个永远会存在的进程，它还有一个大家更熟悉的面孔，它就是 os 的 `main` 函数。

是时候从头开始梳理从机器 boot 到多个用户进程相互切换到底发生了什么了。

```c
void main() {
    clean_bss();    // 清空 bss 段
    trapinit();     // 开启中断
    batchinit();    // 初始化 app_info_ptr 指针
    procinit();     // 初始化线程池
    // timerinit();    // 开启时钟中断，现在还没有
    run_all_app();  // 加载所有用户程序
    scheduler();    // 开始调度
}
```

从 main 函数可以看出，`idle` 线程在完成一系列初始化之后，开始运行 `scheduler` 函数，然后就再也没有回来...

```c
void
scheduler(void)
{
    struct proc *p;
    for(;;){
        for(p = pool; p < &pool[NPROC]; p++) {
            if(p->state == RUNNABLE) {
                p->state = RUNNING;
                current_proc = p;
                swtch(&idle.context, &p->context);
            }
        }
    }
}
```

可以看到 `idle` 线程死循环在了一件事情上：寻找一个 `RUNNABLE` 的进程，然后切换到它开始执行。当这个进程调用 `sched` 后，执行流会回到 `idle` 线程，然后继续开始寻找，如此往复。直到所有进程执行完毕，在 sys_exit 系统调用中有统计计数，一旦 exit 的进程达到用户程序数量就关机。

也就是说，所有进程间切换都需要通过 `idle` 中转一下。那么可不可以一步到位呢？答案是肯定的，其实 [rust版代码](https://github.com/rcore-os/rCore-Tutorial-v3) 就是采取这种实现：在一个进程退出时，直接寻找下一个就绪进程，然后直接切换过去，没有 idle 的中转。两种实现都是可行的。

在了解这些之后，我们就可以实现协作式调度了，主要是 `sys_yeild` 系统调用，其实现十分简单，请同学们自行查看 `kernel/syscall.c`。

## 时钟中断与抢占式调度

没一个程序员都应该干过的一件事情就是提交一个死循环，看看系统会不会真的被卡死。目前，我们虽然有了基于 `sys_yield` 的协作式调度，但只要用户进程不愿意放弃执行权，我们的 os 是没有办法切换到其他进程的。这样的 os 低效且不公平，因此我们需要强制的进程切换手段，这就需要时钟中断的介入。

`timer.c` 中包含了相关函数，功能分别为：打开了时钟中断使能，设置下一次中断间隔，读取当前的机器 cycle 数：

```c
// kernel/timer.c

/// Enable timer interrupt
void timerinit() {
    // Enable supervisor timer interrupt
    w_sie(r_sie() | SIE_STIE);
    set_next_timer();
}
/// Set the next timer interrupt
void set_next_timer() {
    uint64 timebase = 125000;
    set_timer(get_cycle() + timebase);
}

uint64 get_cycle() {
    return r_time();
}
```

qemu 模拟的时钟频率大致为 12_500_000Hz（未找到官方数据，属于个人测算），这里我们选择的时钟中断间隔为 10ms。

利用 `get_cycle` 函数还可以实现 gettime 函数（注意测例要求的接口），原理比较简单不做赘述。

那么时钟中断如何处理呢？按照设计，需要在发生时钟中断时干两件事：设置下一次时钟中断和切换当前进程。相关逻辑在 `kernel/trap.c` 中：

```c
void usertrap() {
    // ...
    uint64 cause = r_scause();
    if(cause & (1ULL << 63)) {
        cause &= ~(1ULL << 63);
        switch(cause) {
        case SupervisorTimer:
            set_next_timer();
            yield();
            break;
        default:
            unknown_trap();
            break;
        }
    } else {
        // ....
    }
    usertrapret();
}
```

按照 riscv 标准，通过 `scause` 最高位区分中断与异常，然后再细分处理。

## 其他

目前，如果内核发生异常，比如访问非法指令、时钟中断，我们是不处理的（以后可能会处理），这可以从 `kernel_trap` 的设计中看出：

```c
void kerneltrap() {
    if((r_sstatus() & SSTATUS_SPP) == 0)
        panic("kerneltrap: not from supervisor mode");
    panic("trap from kernel\n");
}

void set_kerneltrap(void) {
    w_stvec((uint64)kerneltrap & ~0x3); // DIRECT
}
```

一旦从用户态进入内核态，我们就改变 `stvec`，防止内核中断错误的跳转到用户中断处理例程。在返回用户态时切换回来:

```c
void usertrap() {
    set_kerneltrap();
    // ...
}

void usertrapret() {
    set_usertrap();
    // ...
}
```

最后，进程结束的 `sys_exit` 系统调用需要调整：

```c
void exit(int code) {
    struct proc *p = curr_proc();
    p->state = UNUSED;              // 空出进程池位置
    sched();                        // 运行下一个进程
}
```

`main` 函数需要进行新添加的初始化：

```c
void main() {
    clean_bss();
    trapinit();
    batchinit();
    procinit();
    timerinit();            // 增加时钟初始化
    run_all_app();
    printf("start scheduler!\n");
    scheduler();
}
```

## 展望

下一节就是页表了，页表极大的方便了用户态的开发，比如我们终于不用在一 0x80400000 这个奇怪的人为地址了，从 lab4 开始，所有的用户程序将 从 0x1000 开始，虽然这也是人为规定的，但比 0x80400000 要舒服很多。

但是，困难旺旺是不会消失的，那么用户态的困难转嫁到哪里去了呢？准备好迎接硬骨头吧！
