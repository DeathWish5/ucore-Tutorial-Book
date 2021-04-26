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

需要特别注意 brelse 函数：

```c++
void brelse(struct buf *b) {
    b->refcnt--;
    if (b->refcnt == 0) {
        b->next->prev = b->prev;
        b->prev->next = b->next;
        b->next = bcache.head.next;
        b->prev = &bcache.head;
        bcache.head.next->prev = b;
        bcache.head.next = b;
    }
}
```

**需要特别注意**的是 `brelse` 不会真的如字面意思释放一个 buf，。它的准确含义是暂时不操作该 buf 了，buf 的真正释放会被推迟到 buf 池满，无法分配的时候，就会把最近最久未使用的 buf 释放掉（释放 = 写回 + 清空）。这是为了仅可能保留内存缓存，因为读写磁盘真的太太太太慢了。

此外，brelse 的数量必须和 bget 相同，因为 bget 会是的引用计数加一。如果没有相匹配的 brelse，就好比 new 了之后没有 delete。千万注意。

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

那么什么是 inode 和 data block 呢？以下是 superblock, dinode，inode, dirent 三个结构体定义（务必基本理解）：

```c++
// 超级块位置固定，用来指示文件系统的一些元数据，这里最重要的是 inodestart 和 bmapstart
struct superblock {
    uint magic;     // Must be FSMAGIC
    uint size;      // Size of file system image (blocks)
    uint nblocks;   // Number of data blocks
    uint ninodes;   // Number of inodes.
    uint inodestart;// Block number of first inode block
    uint bmapstart; // Block number of first free map block
};

// On-disk inode structure
// 储存磁盘 inode 信息，主要是文件类型和数据块的索引，其大小影响磁盘布局，不要乱改，可以用 pad
struct dinode {
    short type;             // File type
    short pad[3];
    uint size;              // Size of file (bytes)
    uint addrs[NDIRECT + 1];// Data block addresses
};

// in-memory copy of an inode
// dinode 的内存缓存，为了方便，增加了 dev, inum, ref, valid 四项管理信息，大小无所谓，可以随便改。
struct inode {
    uint dev;           // Device number
    uint inum;          // Inode number
    int ref;            // Reference count
    int valid;          // inode has been read from disk?
    short type;         // copy of disk inode
    uint size;
    uint addrs[NDIRECT+1];  // data block num
};

// 目录对应的数据块的内容本质是 filename 到 file inode_num 的一个 map，这里为了简单，就存为一个 `dirent` 数组，查找的时候遍历对比
struct dirent {
    ushort inum;
    char name[DIRSIZ];
};
```

注意几个量的概念: 

* block num: 表示某一个磁盘块的编号。
* inode num: 表示某一个 inode 在所有 inode 项里的编号。注意 inode blocks 其实就是一个 inode 的大数组。

同时，目录本身是一个 filename 到 file inode　num　的　map，可以完成 filename 到 inode_num 的转化。

注意：为了简单，我们的文件系统只有单级目录，也就是只有根目录一个目录，所有文件都在这一个目录里，这个目录里也再没有目录。

那么，在没有任何内存缓存的情形下，如何从磁盘中读取一个文件呢？更加具体来说，`filewrite(filename, "hello world")` 这句伪代码是如何执行的？

首先，我们需要找到该文件对应的 inode，这个过程是：首先找到根目录的 inode，其 inode_num 固定为 0，可以在第一个 inode block 的第一项找到，root_inode 会指示数据块的位置，这些　block 是一个 filename 到 file inode_num 的一个 map，从而依据 filename 得到 inode_num。再次回到 inode blocks，找到第　inode_num 个，就得到了文件对应的 inode。在该　inode 中，可以找到文件的类型、大小和对应数据块的编号(block num)，最后就可以依据　block num　找到对应的数据块，最终完成读写。

inode blocks 的位置在 superblock 中有指示。

理解了 superblock / inode / data-block 的套路之后，我们来看看如何如何完成文件操作。

##### 文件系统实现

还是从 syscall 入手讲起, lab8 进一步更新了 sys_write / sys_read:

```c++
uint64 sys_write(int fd, uint64 va, uint64 len) {
    if (fd <= 2) {
        return console_write(va, len);
    }
    struct proc *p = curr_proc();
    struct file *f = p->files[fd];
    if (f->type == FD_PIPE) {
        return pipewrite(f->pipe, va, len);
    } else if (f->type == FD_INODE) {   // 如果是一个磁盘文件
        return filewrite(f, va, len);
    }
    return -1;
}

uint64 sys_read(int fd, uint64 va, uint64 len) {
    if (fd <= 2) {
        return console_read(va, len);
    }
    struct proc *p = curr_proc();
    struct file *f = p->files[fd];
    if (f->type == FD_PIPE) {
        return piperead(f->pipe, va, len);
    } else if (f->type == FD_INODE) {   // 如果是一个磁盘文件
        return fileread(f, va, len);
    }
    return -1;
}
```

现在我们来完成　`filewrite` 和 `fileread`,首先需要更新 lab6 中的　file 结构体定义，现在我们多了一种文件类型，除了预留给 0、1、2　的标准输入输出文件和 pipe 文件，多了一种 inode 文件，也就是磁盘文件：

```diff
// file.h
struct file {
-   enum { FD_NONE = 0, FD_PIPE} type;
+   enum { FD_NONE = 0, FD_PIPE, FD_INODE} type;
    int ref; // reference count
    char readable;
    char writable;
    struct pipe *pipe; // FD_PIPE
+   struct inode *ip;  // FD_INODE
    uint off;
};
```

对于 inode，为了解决共享问题（不同进程可以打开同一个磁盘文件），也有一个全局的 inode table，每当新打开一个文件的时候，会把一个空闲的　inode 绑定为对应 dinode 的缓存，这一步通过 `iget`　完成:

```c++
// 找到 inum 号 dinode 绑定的 inode，如果不存在新绑定一个
static struct inode *
iget(uint dev, uint inum) {
    struct inode *ip, *empty;
    // 遍历查找 inode table
    for (ip = &itable.inode[0]; ip < &itable.inode[NINODE]; ip++) {
        // 如果有对应的，引用计数 +1并返回
        if (ip->ref > 0 && ip->dev == dev && ip->inum == inum) {
            ip->ref++;
            return ip;
        }
    }
    // 如果没有对于的，找一个空闲 inode 完成绑定
    empty = find_empty()
    // GG，inode 表满了，果断自杀
    if (empty == 0)
        panic("iget: no inodes");
    // 注意这里仅仅是写了元数据，没有实际读取，实际读取推迟到后面
    ip = empty;
    ip->dev = dev;
    ip->inum = inum;
    ip->ref = 1;
    ip->valid = 0;  // 没有实际读取，valid = 0
    return ip;
}
```

当已经得到一个文件对应的 inode 后，可以通过 ivalid 函数确保其是有效的：

```c++
// Reads the inode from disk if necessary.
void ivalid(struct inode *ip) {
    struct buf *bp;
    struct dinode *dip;
    if (ip->valid == 0) {
        // bread　可以完成一个块的读取，这个在将 buf 的时候说过了
        // IBLOCK 可以计算 inum 在几个 block
        bp = bread(ip->dev, IBLOCK(ip->inum, sb));
        // 得到 dinode 内容
        dip = (struct dinode *) bp->data + ip->inum % IPB;
        // 完成实际读取
        ip->type = dip->type;
        ip->size = dip->size;
        memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
        // buf 暂时没用了
        brelse(bp);
        // 现在有效了
        ip->valid = 1;
    }
}
```

在 inode 有效之后，可以通过 writei, readi 完成读写：

```c++
// 从 ip 对应文件读取 [off, off+n) 这一段数据到 dst
int readi(struct inode *ip, char* dst, uint off, uint n) {
    uint tot, m;
    // 还记得 buf 吗？
    struct buf *bp;
    for (tot = 0; tot < n; tot += m, off += m, dst += m) {
        // bmap 完成 off 到 block num 的对应，见下
        bp = bread(ip->dev, bmap(ip, off / BSIZE));
        // 一次最多读一个块，实际读取长度为 m
        m = MIN(n - tot, BSIZE - off % BSIZE);
        memmove(dst, (char*)bp->data + (off % BSIZE), m);
        brelse(bp);
    }
    return tot;
}

// 同 readi
int writei(struct inode *ip, char* src, uint off, uint n) {
    uint tot, m;
    struct buf *bp;

    for (tot = 0; tot < n; tot += m, off += m, src += m) {
        bp = bread(ip->dev, bmap(ip, off / BSIZE));
        m = MIN(n - tot, BSIZE - off % BSIZE);
        memmove(src, (char*)bp->data + (off % BSIZE), m);
        bwrite(bp);
        brelse(bp);
    }

    // 文件长度变长，需要更新 inode 里的 size 字段
    if (off > ip->size)
        ip->size = off;

    // 有可能 inode 信息被更新了，写回
    iupdate(ip);

    return tot;
}
```

bmap 完成的功能很简单，但是我们支持了间接索引，同时还设计到文件大小的改变，所以也拉出来看看:

```c++
// bn = off / BSIZE
uint bmap(struct inode *ip, uint bn) {
    uint addr, *a;
    struct buf *bp;
    // 如果 bn < 12，属于直接索引, block num = ip->addr[bn]
    if (bn < NDIRECT) {
        // 如果对应的 addr, 也就是　block num = 0，表明文件大小增加，需要给文件分配新的 data block
        // 这是通过 balloc 实现的，具体做法是在 bitmap 中找一个空闲 block，置位后返回其编号
        if ((addr = ip->addrs[bn]) == 0)    
            ip->addrs[bn] = addr = balloc(ip->dev);
        return addr;
    }
    bn -= NDIRECT;
    // 间接索引块，那么对应的数据块就是一个大　addr 数组。
    if (bn < NINDIRECT) {
        // Load indirect block, allocating if necessary.
        if ((addr = ip->addrs[NDIRECT]) == 0)
            ip->addrs[NDIRECT] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a = (uint *) bp->data;
        if ((addr = a[bn]) == 0) {
            a[bn] = addr = balloc(ip->dev);
            bwrite(bp);
        }
        brelse(bp);
        return addr;
    }

    panic("bmap: out of range");
    return 0;
}
```

iupdate, balloc 等比较简单，同学们可以自行查看。值得一提的是，是的 `writei` 和 `readi` 考虑了数据来源是内核还是用户，多调用了一层 copyin/copyout，没有本质改变。

现在我们终于可以看看 filewrite 长啥样了：

```c++
uint64 filewrite(struct file* f, uint64 va, uint64 len) {
    int r;
    // 获得文件对应的 inode，不一定有效，但必须有元数据，元数据在 open 的时候写入
    ivalid(f->ip);
    // 注意，由于 writei 处理了 copyin，这里可以直接传用户 va 进去
    if ((r = writei(f->ip, 1, va, f->off, len)) > 0)
        f->off += r;    // 注意这里移动了文件指针
    return r;
}
```

`fileread` 同理,不在赘述。

你可能已经发现一个 bug，为啥莫名奇妙就有了 `inum`(作为 iget 的参数)？其实 `inum` 是在 open 一个 file 的时候获得的，open 其实非常复杂，要注意 open 有创建的语义，创建文件是比较复杂的内容，中涉及到了对 inode blocks 的改动。其实在 `writei` 的时候，有可能是的文件大小增加，这时就已经设计到了磁盘的修改。

还是从 syscall 开始：

```c++
#define O_RDONLY  0x000     // 只读
#define O_WRONLY  0x001     // 只写
#define O_RDWR    0x002     // 可读可写
#define O_CREATE  0x200 　　//　如果不存在，创建
#define O_TRUNC   0x400 　　// 舍弃原有内容，从头开始写

uint64 sys_openat(uint64 va, uint64 omode, uint64 _flags) {
    struct proc *p = curr_proc();
    char path[200];
    copyinstr(p->pagetable, path, va, 200);
    return fileopen(path, omode);
}
```

``` c++
int fileopen(char *path, uint64 omode) {
    int fd;
    struct file *f;
    struct inode *ip;
    if (omode & O_CREATE) {
        // 新常见一个路径为 path 的文件
        ip = create(path, T_FILE);
    } else {
        // 尝试寻找一个路径为 path 的文件
        ip = namei(path);
        ivalid(ip);
    }
    // 还记得吗？从全局文件池和进程 fd 池中找一个空闲的出来，参考 lab6
    f = filealloc();
    fd = fdalloc(f);
    // 初始化
    f->type = FD_INODE;
    f->off = 0;
    f->ip = ip;
    f->readable = !(omode & O_WRONLY);
    f->writable = (omode & O_WRONLY) || (omode & O_RDWR);
    if ((omode & O_TRUNC) && ip->type == T_FILE) {
        itrunc(ip);
    }
    return fd;
}
```

可见，核心函数其实是 `create` 和 `namei`, 后者比较简单，先来研究一下:

```c++
// namei = 获得根目录，然后在其中遍历查找 path
struct inode *namei(char *path) {
    struct inode *dp = root_dir();
    return dirlookup(dp, path, 0);
}

// root_dir 位置固定
struct inode *root_dir() {
    struct inode* r = iget(ROOTDEV, ROOTINO);
    ivalid(r);
    return r;
}

// 便利根目录所有的 dirent，找到 name 一样的 inode
struct inode *
dirlookup(struct inode *dp, char *name, uint *poff) {
    uint off, inum;
    struct dirent de;
    // 每次迭代处理一个 block，注意根目录可能有多个 data block
    for (off = 0; off < dp->size; off += sizeof(de)) {
        readi(dp, 0, (uint64) &de, off, sizeof(de));
        if (strncmp(name, de.name, DIRSIZ) == 0) {
            if (poff)
                *poff = off;
            inum = de.inum;
            // 找到之后，绑定一个内存 inode 然后返回
            return iget(dp->dev, inum);
        }
    }

    return 0;
}
```

create 比较复杂，它长这样：

```c++
static struct inode *
create(char *path, short type) {
    struct inode *ip, *dp;
    if(ip = namei(path) != 0) {
        // 已经存在，直接返回
        return ip;
    }
    // 创建一个文件,首先分配一个空闲的 disk inode, 绑定内存 inode 之后返回
    ip = ialloc(dp->dev, type);
    // 注意 ialloc 不会执行实际读取，必须有 ivalid
    ivalid(ip);
    // 在根目录创建一个 dirent 指向刚才创建的 inode 
    dirlink(dp, path, ip->inum);
    // dp 不用了，iput 就是释放内存 inode，和 iget 正好相反。
    iput(dp);
    return ip;
}
```

ialloc 干的事情：便利 inode blocks 找到一个空闲的，初始化并返回。dirlink 干的事情，便利根目录数据块，找到一个空的 dirent，设置 dirent = {inum, filename} 然后返回，注意这一步可能找不到空位，这是需要找一个新的数据块，并扩大 root_dir size，这是由　bmap 自动完成的。这两个函数就不做代码展示。

fileopen 还可能会导致文件 truncate，也就是截断，具体做法是舍弃全部现有内容，释放所有 data block 并添加到 free bitmap 里。这也是目前 nfs 中唯一的文件变短方式。

最后一个剩余的操作是　fclose，其实 inode 文件的关闭只需要调用 iput 就好了，iput 的实现简单到让人感觉迷惑，就是 inode 引用计数减一。诶？为什么没有计数为 0 就写回然后释放 inode 的操作？和 buf 的释放同理，这里会等 inode 池满了之后自行被替换出去，重新读磁盘实在太太太太慢了。对了，千万记得 iput 和 iget 数量相同，一定要一一对应，否则你懂的。C 编程实在太危险了 QAQ，我感觉框架里打概率有泄漏。。没导致错误而已。

```c++
void
fileclose(struct file *f)
{
    if(--f->ref > 0) {
        return;
    }
    // 暂时不支持标准输入输出文件的关闭
    if(f->type == FD_PIPE){
        pipeclose(f->pipe, f->writable);
    } else if(f->type == FD_INODE) {
        iput(f->ip);
    }

    f->off = 0;
    f->readable = 0;
    f->writable = 0;
    f->ref = 0;
    f->type = FD_NONE;
}
```
```c++
void iput(struct inode *ip) {
    ip->ref--;
}
```

// TODO 总感觉讲完了，但又没有完全讲完，大家没搞懂的地方可以发个 issue 然后最后 wechat 再 cue 我一下

## 展望

lab8 马上就位，在 lab8 中，我们将实现带参数的 exec, 实现从磁盘 load 文件来丢掉丑陋的 pack.py，支持 elf 解析来摆脱人为规定的地址，此外，还将支持标准文件的关闭和 sys_dup 来支持 IO 重定向。我们还将拥有一批用户态程序如 ls, echo, cat 等，是不是有点唬人了？虽然这些和 lab8 的要求并没有什么联系，emm，到时候就知道了。