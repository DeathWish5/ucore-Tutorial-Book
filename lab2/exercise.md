# lab2 实验要求

- 本节难度： **低** 

## 编程作业

### 简单安全检查

lab2 中，我们实现了第一个系统调用 `sys_write`，这使得我们可以在用户态输出信息。但是 os 在提供服务的同时，还有保护 os 本身以及其他用户程序不受错误或者恶意程序破坏的功能。

由于还没有实现虚拟内存，我们可以在用户程序中指定一个属于其他程序字符串，并将它输出，这显然是不合理的，因此我们要对 sys_write 做检查：

- 传入的 **fd** 是否合法（目前仅支持 stdout，也就是 1）
- 传入缓冲区是否位于用户地址之外（需要检查 .text .data .bss 各段以及用户栈，如果是 bin 格式会简单很多）

### 实验要求

- 实现分支 ch2。
- 完成实验指导书中的内容，能运行用户态程序并执行 sys_write 和 sys_exit 系统调用。
- 增加对 sys_write 的安全检查，通过[C测例](https://github.com/DeathWish5/riscvos-c-tests) 中 chapter2 对应的所有测例，测例详情见对应仓库，系统调用具体要求参考[这里](https://github.com/DeathWish5/riscvos-c-tests/blob/main/guide.md#lab2)。

challenge: 实现多核，可以并行执行用户程序。

### 实验约定

在第二章的测试中，我们对于内核有如下仅仅为了测试方便的要求，请调整你的内核代码来符合这些要求。

* 用户栈大小必须为 4096，且按照 4096 字节对其。

### 实验检查

- 实验目录要求

    目录要求不变(参考lab1目录或者示例代码目录结构)。同样在 os 目录下 `make run` 之后可以正确加载用户程序并执行。

    加载的用户测例位置： `../user/target/bin`。

    可以先参考示例代码 [pack.py](https://github.com/DeathWish5/ucore-Tutorial/blob/ch2/kernel/pack.py)

- 检查

    ```console
    $ cd os
    $ git checkout ch2
    $ make run
    ```
    可以正确执行正确执行目标用户测例，并得到预期输出（详见测例注释）。

    注意：如果设置默认 log 等级，从 lab2 开始关闭所有 log 输出。

## 问答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。目前由于一些其他原因，这些问题不太好测试，请同学们可以自行测试这些内容（参考[前三个测例](https://github.com/DeathWish5/rCore_tutorial_tests/tree/master/user/src/bin))，描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

2. 请结合用例理解 [trampoline.S](https://github.com/DeathWish5/ucore-Tutorial/blob/ch2/kernel/trampoline.S) 中两个函数 `userret` 和 `uservec` 的作用，并回答如下几个问题:

    1. L79: 刚进入 `userret`时，`a0`、`a1`分别代表了什么值。 

    1. L87-L88: `sfence` 指令有何作用？为什么要执行该指令，当前章节中，删掉该指令会导致错误吗？
        ```
        csrw satp, a1
        sfence.vma zero, zero
        ```

    1. L96-L125: 为何注释中说要除去 `a0`？哪一个地址代表 `a0`？现在 `a0` 的值存在何处？
        ```assembly
        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)
        ```

    1. `userret`：中发生状态切换在哪一条指令？为何执行之后会进入用户态？

    1. L29： 执行之后，a0 和 sscratch 中各是什么值，为什么？
        ```assembly
        csrrw a0, sscratch, a0
        ```

    1. L32-L61: 从 trapframe 第几项开始保存？为什么？是否从该项开始保存了所有的值，如果不是，为什么？
        ```assembly
        sd ra, 40(a0)
        sd sp, 48(a0)
        ...
        sd t5, 272(a0)
        sd t6, 280(a0)
        ```

    1. 进入 S 态是哪一条指令发生的？

    1. L75-L76: `ld t0, 16(a0)` 执行之后，`t0`中的值是什么，解释该值的由来？
        ```assembly
        ld t0, 16(a0)
        jr t0
        ```

3. 描述程序陷入内核的两大原因是中断和异常，请问 riscv64 支持那些中断／异常？如何判断进入内核是由于中断还是异常？描述陷入内核时的几个重要寄存器及其值。

4. 对于任何中断， `uservec` 中都需要保存所有寄存器吗？你有没有想到一些加速 `uservec` 的方法？简单描述你的想法。


## 报告要求

* 简单总结本次实验与上个实验相比你增加的东西。（控制在5行以内，不要贴代码）
* 完成问答问题
* (optional) 你对本次实验设计及难度的看法。
