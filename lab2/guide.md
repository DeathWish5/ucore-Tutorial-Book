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

