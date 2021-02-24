# lab1 RV64裸机应用

该章节我们将完成一个能够在屏幕输出的小 os，也就是从 bootloader 手中接过程序执行的接力棒，这是后续所有实验的基础。

同学们可以通过

```shell
git checkout ch1 
```
来查看本章对应代码。

## 基本项目结构

``` 
├── kernel
│       ├── entry.S
│       ├── kernel.ld
│       ├── main.c
│       ├── Makefile
│       └── ...
├── bootloader
│       └── rustsbi-qemu.bin
└── readme.md
```

## Makefile 与 qemu

我们希望在 kernel 目录下可以通过 `make run` 运行，简单看一下 Makefile 中 `make run` 之后发生了什么。

```makefile
SRCS = $(wildcard *.S *.c)
OBJS = $(addsuffix .o, $(basename $(SRCS)))

kernel: $(OBJS) kernel.ld
	$(LD) $(LDFLAGS) -T kernel.ld -o kernel $(OBJS)

QEMU = qemu-system-riscv64
QEMUOPTS = \
	-nographic \
	-smp 1 \
	-machine virt \
	-bios $(BOOTLOADER) \
	-device loader,addr=0x80200000,file=kernel

run: kernel
	$(QEMU) $(QEMUOPTS)
```

可以看到执行流程为：
* 编译所有 .c .S 文件并按照 kernel.ld 链接，得到 elf 文件 kernel
* 运行 qemu，参数含义（[详细参考](https://qemu.readthedocs.io/en/latest/system/invocation.html#)）：
    * -nographic: 无图形界面
    * -smp 1: 单核
    * -machine virt: 模拟硬件 RISC-V VirtIO Board
    * -bios ...: 使用制定 bios
    * -device loader ...: 增加 loader 类型的 device,　这里其实就是把我们的 os 文件放到制定位置。

那么，为什么要把 os 起始位置放到 0x80200000 这个位置呢？这其实是我们使用的 bootloader 也就是 rustsbi 的要求。

## riscv 与 RustSBI

我们都知道 riscv 硬件加点之后位于 M 态，但我们编写的 os 运行在 S 态，是谁帮我们完成了 M -> S 的过度呢？正是我们使用的 [rustsbi](https://github.com/luojia65/rustsbi)。
目前阶段我们只需要知道 rustsbi 帮我们干了两件事情
* 完成 M 态的初始化，进行 S 态中断委托，进入 S 态同时跳转到 S 态软件初始位置。
* 当 S 态发出 ｀ecall｀ 请求时完成该请求并返回。

而 rustsbi 指定的 S 态初始位置也就是 0x80200000，这也是我们的第一行代码执行的位置。那么这个位置放的是什么代码？

[sbi文档](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc)。

## 链接脚本

请先阅读[内存布局参考](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter1/4understand-prog.html#id8)。

还记得我们的 os 是使用 kernel.ld 链接的。链接脚本决定了 elf 程序的内存空间布局(严格的讲是虚存映射，注意程序中的各种绝对地址就在链接的时候确定)，由于刚进入 S 态的时候我们尚未激活虚存机制，我们必须把 os 置于物理内存的 0x80200000 处。

```
// linker.ld 

BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)   # 第一行代码
        *(.text .text.*)
    }

	...
}
```

从链接脚本可知，os 的第一个 section (也就是内存中的第一个段) 是 text 段（代码段），而 text 段　由不同文件中的 text 段组成，我们没有规定这些 text 段的具体顺序，但是我们规定了一个特殊的 text 段：.text.entry 段，该 text 段是 BASE_ADDRESS 后的第一个段，该段的第一行代码就在 0x80200000 处。这个特殊的段不是编译生成的，它在 entry.S 中人为设定。

## 第一行代码与 main()

```assembly
# entry.S

    .section .text.entry
    .globl _entry
_entry:
    la sp, boot_stack
    call main

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:

```

.text.entry 段中只有一个函数 _entry，它干的事情也十分简单，设置好 os 运行的堆栈（bootloader 并没有好心的设置好这些），然后调用 main 函数。main 函数位于 main.c 中，从此开始我们就基本进入了 C 的世界。

```c
void clean_bss() {
    char* p;
    for(p = sbss; p < ebss; ++p)
        *p = 0;
}

void main() {
    clean_bss();
    printf("\n");
    printf("hello wrold!\n");
    printf("stext: %p\n", stext);
    printf("etext: %p\n", etext);
    printf("sroda: %p\n", srodata);
    printf("eroda: %p\n", erodata);
    printf("sdata: %p\n", sdata);
    printf("edata: %p\n", edata);
    printf("sbss : %p\n", sbss);
    printf("ebss : %p\n", ebss);
    printf("\n");
    shutdown();
}
```

main 函数也十分简单，首先初始化了 .bss 段（正常来说 os 会负责 .bss 段的清空，但现在我们只能自己来了），然后输出了一些内存布局，最后关机。问题在于，在没有 libc 库甚至没有 os 的情况下，输出和关机是如何实现的？这就是 sbi 给我们带来的便利了。

## 串口输出与关机

```c
// sbi.c
const uint64 SBI_CONSOLE_PUTCHAR = 1;
const uint64 SBI_SHUTDOWN = 8;

int inline sbi_call(uint64 which, uint64 arg0, uint64 arg1, uint64 arg2) {
    register uint64 a0 asm("a0") = arg0;
    register uint64 a1 asm("a1") = arg1;
    register uint64 a2 asm("a2") = arg2;
    register uint64 a7 asm("a7") = which;
    asm volatile("ecall"
                 : "=r"(a0)
                 : "r"(a0), "r"(a1), "r"(a2), "r"(a7)
                 : "memory");
    return a0;
}

void console_putchar(int c) {
    sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}

void shutdown() {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic("shutdown");
}

```

在 sbi.c 中，我们使用嵌入式汇编，调用 ecall 指令向 M 态，也就是向 rustsbi 发出了服务请求，rustsbi 会帮助我们完成串口输出和关机，这极大的方便了我们的开发。

printf 的实现见 `printf.c`，这是一个十分简化的实现（甚至可以注入一些恶意代码），但基本能满足我们的需求。

## 展望

至此我们搭建了一个十分基本的开发环境，接下来的 exercise 中，我们将为我们的 os 增加更加炫酷的输出功能，而 lab2 中我们将引入用户态，并实现第一个系统调用 sys_write。