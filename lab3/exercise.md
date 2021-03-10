# lab3 实验要求

- 本节难度： **并不那么简单了！早点动手** 

## 编程作业

### stride 调度算法

lab3中我们引入了任务调度的概念，可以在不同任务之间切换，目前我们实现的调度算法十分简单，存在一些问题且不存在优先级。现在我们要为我们的 os 实现一种带优先级的调度算法：stide 调度算法。

算法描述如下:
1. 为每个进程设置一个当前 stride，表示该进程当前已经运行的“长度”。另外设置其对应的 pass 值（只与进程的优先权有关系），表示对应进程在调度后，stride 需要进行的累加值。
1. 每次需要调度时，从当前 runnable 态的进程中选择 stride 最小的进程调度。对于获得调度的进程 P，将对应的 stride 加上其对应的步长 pass。
1. 一个时间片，回到 2.步骤，重新调度当前 stride 最小的进程。

可以证明，如果令 P.pass = BigStride / P.priority 其中 P.priority 表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。证明过程我们在这里略去，有兴趣的同学可以在网上查找相关资料。

其他实验细节：
1. stride 调度要求进程优先级 >= 2，所以设定进程优先级 <= 1 会导致错误。
1. 进程初始 stride 设置为 0 即可。
1. 进程初始优先级设置为 16。

tips: 使用优先级队列是实现 stride 算法的不错选择。但是我们的实验不要求效率和优雅，可能直接遍历选找更加省事。

### 实验要求

- 实现分支：ch3。
- 完成实验指导书中的内容，实现 sys_yield，实现协作式和抢占式的调度。
- 实现 stride 调度算法，实现 sys_gettime, sys_set_priority 两个系统调用并通过[C测例](https://github.com/DeathWish5/riscvos-c-tests) 中 chapter3 对应的所有测例，测例详情见对应仓库，系统调用具体要求参考[这里](https://github.com/DeathWish5/riscvos-c-tests/blob/main/guide.md#lab3)。

需要说明的是 lab3 有3类测例，`ch3_0_*` 用来检查基本 syscall 的实现，`ch3_1_*` 基于 yield 来检测基本的调度，`ch3_2_*` 基于时钟中断来测试 stride 调度算法实现的正确性。测试时可以分别测试 3 组测例，使得输出更加可控、更加清晰。

特别的，我们有一个死循环测例 `ch3t_deadloop` 用来保证大家真的实现了始终中断。这一章中我们人为限制一个程序执行的最大时间（必须很大），超过就杀死，这样，我们的程序更不容易被恶意程序伤害。这一规定可以在实验4开始删除，仅仅为通过 lab3 测例设置。

challenge: 实现多核，可以并行调度。

### 实验检查

- 实验目录要求

    目录要求不变(参考lab1目录或者示例代码目录结构)。同样在 os 目录下 `make run` 之后可以正确加载用户程序并执行。

    加载的用户测例位置： `../user/target/bin`。

- 检查

    可以正确 `make run` 执行，可以正确执行目标用户测例，并得到预期输出（详见测例注释）。

## 问答作业

考虑在一个完整的 os 中，随时可能有新进程产生，新进程在调度池中的位置见[chapter5相关代码](https://github.com/DeathWish5/ucore-Tutorial/blob/ch5/kernel/proc.c#L90-L98)。

1. 请简要描述[chapter３示例代码](https://github.com/DeathWish5/ucore-Tutorial/blob/ch3/kernel/proc.c#L60-L74)调度策略，尽可能多的指出该策略存在的缺点。

2. 该调度策略在公平性上存在比较大的问题，请找到一个进程产生和结束的时间序列，使得在该调度算法下发生：先创建的进程后执行的现象。你需要给出类似下面例子的信息（有更详细的分析描述更好，但尽量精简）。同时指出该序列在你实现的 stride 调度算法下顺序是怎样的？

   例子：

   |   时间   |        0        |  1   |   2    |   3    |   4    |   5    |   6    |  7   |
   | :------: | :-------------: | :--: | :----: | :----: | :----: | :----: | :----: | :--: |
   | 运行进程 |        -        |  p1  |   p2   |   p3   |   p4   |   p1   |   p3   |  -   |
   |   事件   | p1、p2、p3 产生 |      | p2结束 | p4产生 | p4结束 | p1结束 | p3结束 |  -   |

   产生顺序：p1、p2、p3、p4。第一次执行顺序: p1、p2、p3、p4。

   其他细节：允许进程在其他进程执行时产生（也就是被当前进程产生）/结束（也就是被当前进程杀死）。

3. stride 算法深入

    stride算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride， p1.stride = 255, p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

    - 实际情况是轮到 p1 执行吗？为什么？

    我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，**如果不考虑溢出**，在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。

    - 为什么？尝试简单说明（传达思想即可，不要求严格证明）。

    已知以上结论，**在考虑溢出的情况下**，假设我们通过逐个比较得到 Stride 最小的进程，请设计一个合适的比较函数，用来正确比较两个 Stride 的真正大小：

    ```c++
    typedef unsigned long long Stride_t;
    const Stride_t BIGSTRIDE = 0xffffffffffffffffULL;
    bool Less(Stride_t, Stride_t) {
        // ...
    }

    ```

    例子：假设使用 8 bits 储存 stride, BigStride = 255。那么：
    * `Less(125, 255) == false`
    * `Less(129, 255) == true`

## 报告要求

* 简单总结本次实验与上个实验相比你增加的东西。（控制在5行以内，不要贴代码）
* 完成问答问题
* (optional) 你对本次实验设计及难度的看法。

## 参考信息
如果有兴趣进一步了解　stride　调度相关内容，可以尝试看看：

- [作者 Carl A. Waldspurger 写这个调度算法的原论文](https://people.cs.umass.edu/~mcorner/courses/691J/papers/PS/waldspurger_stride/waldspurger95stride.pdf)
- [作者 Carl A. Waldspurger 的博士生答辩slide](http://www.waldspurger.org/carl/papers/phd-mit-slides.pdf)
- [南开大学实验指导中对Stride算法的部分介绍](https://nankai.gitbook.io/ucore-os-on-risc-v64/lab6/tiao-du-suan-fa-kuang-jia#stride-suan-fa)
- [NYU OS课关于Stride Scheduling的Slide](https://cs.nyu.edu/rgrimm/teaching/sp08-os/stride.pdf)

如果有兴趣进一步了解用户态线程实现的相关内容，可以尝试看看：

- [user-multitask in rv64](https://github.com/chyyuu/os_kernel_lab/tree/v4-user-std-multitask)
- [绿色线程 in x86](https://github.com/cfsamson/example-greenthreads)
- [x86版绿色线程的设计实现](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
- [用户级多线程的切换原理](https://blog.csdn.net/qq_31601743/article/details/97514081?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control>)