# lab5 实验要求

- 本节难度： **一定比lab4简单** 

## 编程作业

### 进程创建

大家一定好奇过为啥进程创建要用 fork + execve 这么一个奇怪的系统调用，就不能直接搞一个新进程吗？思而不学则殆，我们就来试一试！这章的编程练习请大家实现一个完全 DIY 的系统调用 spawn，用以创建一个新进程。

spawn ([标准spawn看这里](https://man7.org/linux/man-pages/man3/posix_spawn.3.html>))
- syscall ID: 400
- C 接口：`int spawn(char *filename)`
- Rust 接口：`fn spawn(file: *const u8) -> isize`
- 功能：相当于 fork + exec，新建子进程并执行目标程序。
- 说明：成功返回子进程id，否则返回 -1。
- 可能的错误：
    - 无效的文件名
    - 进程池满等资源错误。

### 实验要求

- 实现分支：ch5。
- 完成实验指导书中的内容，实现进程控制，可以运行 usershell。
- 实现自定义系统调用 spawn，并通过 [C测例](https://github.com/DeathWish5/riscvos-c-tests)中chapter5对应的所有测例。

challenge: 支持多核。

### 实验检查

- 实验目录要求

    目录要求不变(参考lab1目录或者示例代码目录结构)。同样在 os 目录下 `make run` 之后可以正确加载用户程序并执行。

    加载的用户测例位置： `../user/target/bin`。

- 检查

    可以正确 `make run` 执行，可以正确执行目标用户测例，并得到预期输出（详见测例注释）。

## 问答作业

1. fork + exec 的一个比较大的问题是 fork 之后的内存页/文件等资源完全没有使用就废弃了，针对这一点，有什么改进策略？

2. 其实使用了题1的策略之后，fork + exec 所带来的无效资源的问题已经基本被解决了，但是今年来 fork 还是在被不断的批判，那么到底是什么正在"杀死"fork？可以参考[论文](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf)，**注意**：回答无明显错误就给满分，出这题只是想引发大家的思考，完全不要求看论文，球球了，别卷了。

3. fork 当年被设计并称道肯定是有其好处的。请使用**带初始参数**的 spawn 重写如下 fork 程序，然后描述 fork 有那些好处。注意:使用"伪代码"传达意思即可，spawn接口可以自定义。可以写多个文件。

    ```c
    int main() {
        int a = get_a();
        if(fork() == 0) {
            int b = get_b();
            printf("a + b = %d", a + b);
            exit(0);
        }
        printf("a = %d", a);
        return 0;
    }

    ```

4. 描述进程执行的几种状态，以及 fork/exec/wait/exit 对与状态的影响。

## 报告要求

* 简单总结本次实验与上个实验相比你增加的东西。（控制在5行以内，不要贴代码）
* 完成问答问题
* (optional) 你对本次实验设计及难度的看法。