# 批处理系统

这一章我们将进入用户态的世界，完成一个能够顺序执行用户任务的 os。 这主要是模拟了早期硬件内存十分有限的场景，内存小到我们在执行第二个任务的时候需要把第一个任务从内存中丢掉。

你可以在 ch2 分支中看到文档对应的代码。

## 用户程序的载入

由于我们还没有文件系统，我们采取直接将用户程序打包到内核镜像的方法装载用户程序。

首先生成用户程序，由于要编译到我们自己的 os 上，我们不能使用 libc 库，而是链接我们自定义的程序库。cmake 是强大的程序生成工具，我们使用它来完成用户程序的构建。你可以在新增的 `user/` 目录下找到用户程序和自定义程序库。

```c
// user/lib/syscall.c
ssize_t write(int fd, const void *buf, size_t len) {
    return syscall(SYS_write, fd, buf, len);
}

void exit(int code) {
    syscall(SYS_exit, code);
}
```

自定义库(`user/lib`)中看到我们给自己定的小目标，`sys_write` 系统调用和 `sys_exit` 系统调用。详细接口见[这里](https://github.com/DeathWish5/riscvos-c-tests/blob/main/guide.md#lab2)。

```c
// user/src/hello.c
#include <stdio.h>
#include <unistd.h>

int main() {
    puts("hello wrold!");
    return 0;
}
```

`user/src`目录下是我们要运行的用户程序，lab2的还十分简单，就是简单的计算和输出。

```makefile
# user/Makefile
elf:
	@mkdir -p build
	@cd build && cmake $(cmake_build_args) .. && make -j

bin: elf
	@mkdir -p asm
	@$(CP) build/asm/* asm
	@mkdir -p $(out_dir)
	@$(CP) build/target/* target
```

`user/Makefile`中可以看到用户程序生成的过程，我们首先使用 camke 得到 elf/bin 程序已经汇编代码(具体内容参见`CMakeLists.txt`文件，不做赘述)，然后将对应文件拷贝到特定目录（`user/target/bin`）。

然后我们使用一个 python 脚本 `pack.py` 生成 `link_app.S`，后者大致内容如下：

```assembly
    .align 4
    .section .data
    .global _app_num
_app_num:
    .quad 2
    .quad app_0_start
    .quad app_1_start
    .quad app_1_end

    .global _app_names
_app_names:
   .string "hello.bin"
   .string "matrix.bin"

    .section .data.app0
    .global app_0_start
app_0_start:
    .incbin "../user/target/hello.bin"

    .section .data.app1
    .global app_1_start
app_1_start:
    .incbin "../user/target/matrix.bin"
app_1_end:
```

可以看到，这个汇编文件使用 [incbin](https://www.keil.com/support/man/docs/armasm/armasm_dom1361290017052.htm) 将目标用户程序包含入 `link_app.S`中，同时记录了这些程序的地址和名称信息。最后，我们在 `Makefile` 中会将内核与 `link_app.S` 一同编译并链接。这样，我们在内核中就可以通过 `extern` 指令访问到用户程序的所有信息。

由于 riscv 要求程序指令必须是对齐的，我们对内核链接脚本也作出修改，保证用户程序链接时的指令对齐，这些内容见 `kernel/kernelld.py`。最终修改后的脚本中多了如下对齐要求：

```diff
    .data : {
        *(.data)
+       . = ALIGN(0x1000);
+       *(.data.app0)
+       . = ALIGN(0x1000);
+       *(.data.app1)
        *(.data.*)
    }
```

## 内核 relocation

内核中通过访问 `link_app.S` 中定义的 `_app_num`、`app_0_start` 等符号来获得用户程序位置。

```c
// kernel/batch.c
extern char _app_num[];
void batchinit() {
    app_info_ptr = (uint64*) _app_num;
    app_num = *app_info_ptr;
    app_info_ptr++;
    // from now on:
    // app_n_start = app_info_ptr[n]
    // app_n_end = app_info_ptr[n+1]
}
```

然而我们并不能直接跳转到 `app_n_start` 直接运行，因为用户程序在编译的时候，会假定程序处在虚存的特定位置，而由于我们还没有虚存机制，因此我们在运行之前还需要将用户程序加载到规定的物理内存位置。

> 虽然现在有很多相对寻址的技术，但是为了更好的支持诸如动态链接等技术，其实我们编译出的程序并不是完全相对寻址的。观察我们编译出的程序就可以发现，程序对与 .data 段的访问是基于 GOT 表的间接寻址。理论上 os 需要在加载的时候修改 GOT 表来完成 relocation，但这样反而更加复杂。

为此我们规定了用户的链接脚本，并在内核完成程序的 "搬运"：

```linker.ld
# user/lib/arch/riscv/user.ld
SECTIONS {
    . = 0x80400000;                 #　规定了内存加载位置

    .startup : {
        *crt.S.o(.text)             #　确保程序入口在程序开头
    }

    .text : { *(.text) }
    .data : { *(.data .rodata) }

    /DISCARD/ : { *(.eh_*) }
}
```
```c
// kernel/batch.c
const uint64 BASE_ADDRESS = 0x80400000, MAX_APP_SIZE = 0x20000;
int load_app(uint64* info) {
    uint64 start = info[0], end = info[1], length = end - start;
    memset((void*)BASE_ADDRESS, 0, MAX_APP_SIZE);
    memmove((void*)BASE_ADDRESS, (void*)start, length);
    return length;
}

```

## 用户程序启动与中断返回

现在我们们可以直接跳转到程序开头开始运行吗？显然没这么简单，os为了保护自己，需要与用户程序进行特权级的隔离（U态和Ｓ态），在两个状态之间切换不能通过 function call，事实上编译器是没有这个能力的，我们需要设计一段代码进行这个过程。

首先，在执行流之间切换需要进行状态保存与恢复，按照 riscv 标准，`trap.h` 中的 `trapframe` 结构定义了需要保存和回复的内容。

```c
// kernel/trap.h
struct trapframe {
    /*   0 */ uint64 kernel_satp;   // kernel page table
    /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
    /*  16 */ uint64 kernel_trap;   // usertrap entry
    /*  24 */ uint64 epc;           // saved user program counter
    /*  32 */ uint64 kernel_hartid; // saved kernel tp， unused in our project
    /*  40 */ uint64 ra;
    /*  48 */ uint64 sp;
    /* ... */ ....
    /* 272 */ uint64 t5;
    /* 280 */ uint64 t6;
};
```

这其中 40-280 的项保存用户通用寄存器信息，0-32 保存一些内核信息。

`trap.c` 的 `usertrapret()` 和 `trampoline.S` 中的 `userret` 函数展示了从S态返回/进入U态的过程，也就是当中断处理完毕后返回用户态的过程。这两个函数请同学们仔细理解。

```c
void usertrapret(struct trapframe* trapframe, uint64 kstack)
{
    // 这两个有啥用？往后看！
    trapframe->kernel_sp = kstack + PGSIZE; 
    trapframe->kernel_trap = (uint64)usertrap;

    // 设定返回地址
    w_sepc(trapframe->epc);
    
    // set up the registers that trampoline.S's sret will use
    // to get to user space.
    // set S Previous Privilege mode to User.
    uint64 x = r_sstatus();
    x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
    x |= SSTATUS_SPIE; // enable interrupts in user mode
    w_sstatus(x);

    userret((uint64)trapframe);
}
```

```assembly
.globl userret
userret:
        # a0 = bottom of trapframe
        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        # ... 48-272
        ld t6, 280(a0)

	    # restore user a0, and **save TRAPFRAME in sscratch**
        csrrw a0, sscratch, a0

        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret

```

注意 userret 最终在 scratch 中储存了 trapframe 的位置。 

用户程序的创建后的第一次运行也是通过 `usertrapret` 完成的，我们会人为构造程序的 `trapframe`，设定返回地址和用户栈顶：

```c
// kernel/batch.c
__attribute__ ((aligned (4096))) char user_stack[4096];       // 预定义的用户 stack
__attribute__ ((aligned (4096))) char trap_page[4096];        // 预定义的中断页，用来存放 trapframe
// run_first_app 也使用这个函数
int run_next_app() {
    struct trapframe* trapframe = (struct trapframe*)trap_page;
    app_info_ptr++;     // go to the location of next user-app
    load_app(app_info_ptr);
    memset(trapframe, 0, 4096);
    // 设置 trapframe
    trapframe->epc = BASE_ADDRESS;                  // epc 也就是返回地址，设置为用户程序起始地址
    trapframe->sp = (uint64) user_stack + 4096;     // 设置为用户栈顶
    usertrapret(trapframe, (uint64)boot_stack);     // 调用 usertrapret 启动用户程序
    return 0;
}
```

通过巧妙的设置 `trapframe`（其实只是设置了 `epc` 和 `sp`，因为程序开始运行时对其他寄存器没有要求），我们复用了用户态中断处理完成后返回的函数，使得使用该 `trapframe` 返回后，用户程序刚好可以开始执行。用户进程中断处理的全流程图示如下：

```c

1.程序进行系统调用或者出现异常 -> 2.进入内核态 ->  ... 3.内核态完成系统调用 ...  ->  4. 返回用户态 -> 5.用户态继续执行

```

而开始运行一个进程就是其中的 `4` `5` 两步骤。

此外当一个应用退出后，也会调用 `run_next_app`开始运行下一个应用，直到全部应用结束。

## 用户中断处理

如何返回 U 态我们已经知道了，那么 U 态发生错误之后如何正确处理呢？也就是如何从 U 态进入 S 态呢？首先，U 进入 S 都是因为中断或者异常，我们首先需要作出如下配置：

```c
// set up to take exceptions and traps while in the kernel.
void trapinit(void)
{
    w_stvec((uint64)uservec & ~0x3);   // 写 stvec, 最后两位表明跳转模式，该实验始终为 0
    intr_on();      // 开启中断使能
}
```

这个函数填写 `stvec` 寄存器，用于保存中断发生时跳转到的地址，也就是中断处理函数入口地址。当 U 态发生中断（系统调用）或者异常时，硬件会完成一些中断寄存器的保存，最重要的就是 `sepc` 寄存器，表明了中断发生的地址，它将帮助我们正确的返回。此外还有 sstatus、scause、stval，含义请查阅手册。我们来看看 `stvec` 的值，也就是 `uservec` 函数干了那些事情。

```assembly
.globl uservec
uservec:
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #

	    # swap a0 and sscratch
        # so that **a0 is TRAPFRAME**
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        # ... 48-272
        sd t6, 280(a0)

	    # save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        csrr t1, sepc
        sd t1, 24(a0)

        ld sp, 8(a0)        // 想想看这是在干啥
        ld tp, 32(a0)
        ld t1, 0(a0)
        ld t0, 16(a0)
        jr t0
```

可以看到 uservec 在 trapframe 中保存了基础寄存器，然后就跳转到了我们早先设定在 `trapframe->kernel_trap` 中的地址，也就是 `usertrap` 函数，该函数完成中断处理与返回：

```c
// kernel/trap.c

//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void usertrap(struct trapframe *trapframe)
{
    if((r_sstatus() & SSTATUS_SPP) != 0)
        panic("usertrap: not from user mode");

    uint64 cause = r_scause();
    // 如果是一个系统调用，处理并返回
    if(cause == UserEnvCall) {
        trapframe->epc += 4;
        syscall();
        return usertrapret(trapframe, (uint64)boot_stack);
    }
    // 否则报错并杀死进程，目前我们只处理系统调用
    switch(cause) {
        case StoreFault:
        // ... 报错信息
        default:
            printf("unknown trap: %p, stval = %p sepc = %p\n", r_scause(), r_stval(), r_sepc());
            break;
    }
    printf("switch to next app\n");
    run_next_app();
}

```

至此，我们只需要完成系统调用的处理，也就是 `syscall` 函数就完成第二章的基础功能了，这部分逻辑在 `syscall.c` 中，由于十分简单，不做赘述。

## 总体执行流程



## 展望

第二章，我们成功建立了用户态执行环境，可以运行用户程序。但是目前我们只能现行的运行，这有很大的缺陷。第三章，我们的内存就变大到足以同时容纳多个用户程序了，我们将实现多任务的调度与切换。