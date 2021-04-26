# 文件系统

这一章实现文件系统，也就是管理磁盘模块。首先，一切的基础，需要有读写磁盘的能力。

## 内存缓存与 virtio 磁盘驱动

注意：这一部分代码不需要同学们详细了解细节。

首先，我们使用 qemu 的虚拟磁盘 virtio 作为我们的磁盘。注意 Makefile 中的 qemu 参数增加了两行：

```diff
QEMU = qemu-system-riscv64
QEMUOPTS = \
	-nographic \
	-smp $(CPUS) \
	-machine virt \
	-bios $(BOOTLOADER) \
	-kernel kernel	\
+	-drive file=$(U)/fs-copy.img,if=none,format=raw,id=x0 \       # 以 user/fs-copy.img 作为磁盘镜像
+   -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0      # 虚拟 virtio 磁盘设备
```

在 ucore-tutorial 中磁盘块的读写是通过中断处理的。在 `virtio.h` 和 `virtio-disk.c` 中我们按照 qemu 对 virtio 的定义，实现了 `virtio_disk_init` 和 `virtio_disk_rw` 两个函数，前者完成磁盘设备的初始化和对其管理的初始化。`virtio_disk_rw` 实际完成磁盘IO，当设定好读写信息后会通过 MMIO 的方式通知磁盘开始写。然后，os 会开启中断并开始死等磁盘读写完成。当磁盘完成 IO 后，磁盘会触发一个外部中断，在中断处理中会把死循环条件解除。

```c++
virtio_disk_rw(struct buf *b, int write) {
	/// ... set IO config
	*R(VIRTIO_MMIO_QUEUE_NOTIFY) = 0; 		// notify the disk to carry out IO
    struct buf volatile * _b = b;   		// Make sure complier will load 'b' form memory
    intr_on();
    while(_b->disk == 1);	// _b->disk == 0 means that this IO is done 
    intr_off();
}
```

中断处理流程调整有：<1>增加 kernel trap 的处理;<2>增加外部中断的响应;<3> kernel 态处理时钟中断是，不切换进程。

##### trap form kernel

riscv 是支持内核发生中断在内核处理的，也就是所谓的中断嵌套，为了支持这一点，首先需要增加区别于用户中断入口的内核中断处理入口 `kernelvec`:

```c
// trap.c
extern char kernelvec[];
void set_kerneltrap(void) {
	// 进入内核后设置 stvec 为 kernelvec
    w_stvec((uint64) kernelvec & ~0x3);
}
```

`kernelvec.S` 的就具体内容十分简单，由于没有发生特权级的切换，所以这段汇编很类似于函数调用的寄存器v保存恢复，只不过处理了所有通用寄存器。甚至比 switch 还简单。`x0` 为常量不需要保存，tp 寄存器单核情况下不起作用，可以不关心。值得注意的是，在 restore 的时候没有特殊处理 `sp` 寄存器，同学们可以思考一下这是为什么，直接 `ld sp, 8(sp)` 不会有问题吗？（放心，由于 rust 处理方式不同，所以其实期末不会考）。

```asm
# kernelvec.S

kernelvec:
        // make room to save registers.
        addi sp, sp, -256
        // save the registers expect x0
        sd ra, 0(sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        // ...
        sd t4, 224(sp)
        sd t5, 232(sp)
        sd t6, 240(sp)

        call kerneltrap

kernelret:
        // restore registers.
        ld ra, 0(sp)
        ld sp, 8(sp)
        ld gp, 16(sp)
        // restore all registers expect x0
        ld t4, 224(sp)
        ld t5, 232(sp)
        ld t6, 240(sp)

        addi sp, sp, 256

        sret
```

`kernelvec.S` 中调用了 `kerneltrap` 函数，如下：

```c++
void kerneltrap() {
	// 老三样，不过在这里把处理放到了 C 代码中
    uint64 sepc = r_sepc();
    uint64 sstatus = r_sstatus();
    uint64 scause = r_scause();

    if ((sstatus & SSTATUS_SPP) == 0)
        panic("kerneltrap: not from supervisor mode");

    if (scause & (1ULL << 63)) {
		// 可能发生时钟中断和外部中断，我们的主要目标是处理外部中断
        devintr(scause & 0xff);
    } else {
		// kernel 发生异常就挣扎了，肯定出问题了，杀掉用户线程跑路
        error("invalid trap from kernel: %p, stval = %p sepc = %p\n", scause, r_stval(), sepc);
        exit(-1);
    }
}
```

外部中断处理函数 `devintr` 如下，主要处理时钟和外部中断：

```c++
void devintr(uint64 cause) {
    int irq;
    switch (cause) {
        case SupervisorTimer:
            set_next_timer();
            // 时钟中断如果发生在内核态，不切换进程，原因分析在下面
			// 如果发生在用户态，照常处理
            if((r_sstatus() & SSTATUS_SPP) == 0) {
                yield();
            }
            break;
        case SupervisorExternal:
            irq = plic_claim();
            if (irq == UART0_IRQ) {		// UART 串口的终端不需要处理，这个 rustsbi 替我们处理好了
                // do nothing
            } else if (irq == VIRTIO0_IRQ) {	// 我们等的就是这个中断
                virtio_disk_intr();
            }
            if (irq)
                plic_complete(irq);		// 表明中断已经处理完毕
            break;
    }
}
```

`virtio_disk_intr()` 会把 buf->disk 置零，这样中断返回后死循环条件接触，程序可以继续运行。具体代码在 `virtio-disk.c` 中。

这里还需要注意的一点是，为什么始终不允许内核发生进程切换呢？只是由于我们的内核并没有并发的支持，相关的数据结构没有锁或者其他机制保护。考虑这样一种情况，一个进程读写一个文件，内核处理等待磁盘相应时，发生时钟中断切换到了其他进程，然而另一个进程也要读写同一个文件，这就可能发生数据访问上的冲突，甚至导致磁盘出现错误的行为。这也是为什么内核态一直不处理时钟中断，我们必须保证每一次内核的操作都是原子的，不能被打断。

事实上可以看到，我们在内核中是全程关中断的（注意 lab2 不应该开启中断，这个操作是导致 ch7-bug-fix 这个分支的元凶），只有在不得已的时候，也就是等待磁盘中断的时候短暂的开启中断，完成之后马上关闭。大家可以想一想，如果内核可以随时切换，当前有那些数据结构可能被破坏。提示：想想 kalloc 分配到一半，进程 switch 切换到一半之类的。

##### 磁盘缓存

为了加快磁盘访问的速度，在内核中设置了磁盘缓存 `struct buf`，一个 buf 对应一个磁盘 block，这一部分代码也不要求同学们深入掌握。大致的作用机制是，对磁盘的读写都会被转化为对 `buf` 的读写，当 buf 有效时，读写 buf，buf 无效时（类似页表缺页和 TLB 缺失），就实际读写磁盘，将 buf 变得有效，然后继续读写 buf。详细的内容在 `buf.h` 和 `bio.c` 中。

buf 写回的时机是 buf 池满需要替换的时候(类似内存的 swap 策略) 手动写回。如果 buf 没有写回，一但掉电就 GG 了，所以手动写回还是挺重要的。

重要函数功能描述如下，基本了解作用机制即可。

```c++
struct buf {
	uint dev;		// buf 缓存的磁盘 block 对应的块号和设备号
    uint blockno;
    int valid;   // 表明 data 是否有效
    int disk;    // 用作磁盘读写时等待
    uint refcnt;		// 引用计数，为 0 时执行写回操作
    struct buf *prev;   // buf 池链表
    struct buf *next;
    uchar data[BSIZE];	// 数据块
};

// 全局初始化函数，建立 buf 池的链表链接
void binit(void); 

// Helper function.
// Look through the buf for block on device dev.
// If not found, allocate a buffer.
static struct buf * bget(uint dev, uint blockno);

// Return a buf with the contents of the indicated block.
// use `bget` and `virtio_disk_rw`
struct buf * bread(uint dev, uint blockno);

// Write back b's contents to disk.
// use `virtio_disk_rw`
void bwrite(struct buf *b);

// Release a buffer.
void brelse(struct buf *b);

void bpin(struct buf *b) {
    b->refcnt++;
}

void bunpin(struct buf *b) {
    b->refcnt--;
}
```

可以通过 `bread` 来读取特定磁盘 block 的内容，可以通过 `bwrite` 写回。

## nfs 文件系统

在 `fs.h` 和 `fs.c` 中，我们实现了 naive fs 的主要逻辑，注意这部分逻辑要和另一个目录中的 `nfs/fs.c` 相匹配。

##### 文件系统磁盘布局与文件读取流程

在 `nfs/fs.c` 中揭示了磁盘数据的布局：

```c++
// 基本信息：块大小 BSIZE = 1024B，总容量 FSSIZE = 1000 个 block = 1000 * 1024 B。
// Layout:
// 0号块目前不起作用，可以忽略。superblock 固定为 1 号块，size 固定为一个块。
// 其后是储存 inode 的若干个块，占用块数 = inode 上限 / 每个块上可以容纳的 inode 数量，
// 其中 inode 上限固定为 200，每个块的容量 = BSIZE / sizeof(struct disk_inode)
// 再之后是数据块相关内容，包含一个 储存空闲块位置的 bitmap 和 实际的数据块，bitmap 块
// 数量固定为 NBITMAP = FSSIZE / (BSIZE * 8) + 1 = 1000 / 8 + 1 = 126 块。
// [ boot block | sb block | inode blocks | free bit map | data blocks ]
```

注意：不推荐同学们修改该布局，除非你完全看懂了 fs 的逻辑，所以最好不要改变 `disk_inode` 这个结构的大小，如果想要增删字段，一定使用 pad。

那么什么是 inode 和 data block 呢？

// TODO: 课堂上似乎讲了，先跳过？
// TODO: 有其他事情需要搞，今晚一定补完，QAQ，满地打滚求放过。。。。。