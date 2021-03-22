# 物理内存管理与虚实映射

这一章我们会首先把物理内存更加科学的管理起来，然后激活页表，启用虚存机制，这样我们就终于不用 0x80400000 这些奇奇怪怪的地址和 lab3 烦人的地址等差数列了。

## 物理内存管理：kalloc

出于简单考虑，我们使用链表来管理内存：

```c
// kernel/kalloc.c
struct linklist {
    struct linklist *next;
};

struct {
    struct linklist *freelist;
} kmem;
```

我们使用一个链表来维护当前空闲的页面，当申请一个页时，将链表头分配出去，当一个页被释放时，加到链表头。注意，我们的管理仅仅在页这个粒度进行，所以所有的地址必须是 PAGE_SIZE 对齐的。

```c
// kernel/kalloc.c: 页面分配
void *
kalloc(void)
{
    struct linklist *l;
    l = kmem.freelist;
    kmem.freelist = l->next;
    return (void*)l;
}

// kernel/kalloc.c: 页面释放
void *
kfree(void *pa)
{
    struct linklist *l;
    l = (struct linklist*)pa;
    l->next = kmem.freelist;
    kmem.freelist = l;
}
```

可能你会疑惑这个链表为啥和常见的长得不一样，为啥只有指针，没有数据，那有啥用？其实，这里链表指针本身就蕴含了数据，由于我们管理的是空闲页面，所以其实我们可以在这些页面中储存一些东西，不会破坏数据。所以，我们在空闲页面的开头存了一个指针，指向下一个空闲页面地址。

那么我们的内核有那些空闲内存需要管理呢？事实上，qemu 已经规定了内核需要管理的内存范围，可以参考[这里](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter1/3-2-mini-rt-baremetal.html#id4)，具体来说，需要软件管理的内存为 [0x80000000, 0x88000000)，其中，rustsbi 使用了 [0x80000000, 0x80200000) 的范围，其余都是内核使用，来看看 `kmem` 的初始化：

```c
// kernel/kalloc.c

// 物理内存管理初始化
// ekernel 为链接脚本定义的内核代码结束地址，PHYSTOP = 0x88000000
void
kinit()
{
    freerange(ekernel, (void*)PHYSTOP);
}

// kfree [pa_start, pa_end)
void
freerange(void *pa_start, void *pa_end)
{
    char *p;
    p = (char*)PGROUNDUP((uint64)pa_start);
    for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
        kfree(p);
}
```

可见，我们把内核暂时没有用到的 `ekernel` 到 0x88000000 的范围 `kfree` 掉，也就是加入了空闲内存链表。这十分简单合理。以后，我们需要一个新的内存页面时，不会采用预分配的方案，也不会直接指定一个地址就开始用，而是使用 `kalloc` 获得一个空闲页，使用完成后通过 `kfree` 释放。这就完成了物理内存的管理，页式管理很简单对吧！

## RV39 页表

这部分理论知识请参见[RV39页表](https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter4/3sv39-implementation-1.html)，可直接跳过其中语言相关的内容。

在示例代码中，对于页表的操作位于 `kernel/vm.c` 文件，主要接口有：

```c
// kernel/vm.c

// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t* walk(pagetable_t pagetable, uint64 va, int alloc);

// Look up a virtual address, return the physical page,
// or 0 if not mapped.
// Can only be used to look up user pages.
// Use `walk`
uint64 walkaddr(pagetable_t pagetable, uint64 va);

// Look up a virtual address, return the physical address.
// Use `walkaddr`
uint64 useraddr(pagetable_t pagetable, uint64 va);

```

这三个是查页表相关函数，`useraddr` 将 `pagetable` 页表下的虚拟地址 `va` 转化为物理地址，`walkaddr`、`walk` 是更底层的函数。

以下是建立新映射和取消映射的函数，`mappages` 在 `pagetable` 中建立 `[va, va + size)` 到 `[pa, pa + size)` 的映射，页表属性为`perm`，`uvmunmap` 则取消一段映射，`do_free` 控制是否 `kfree` 对应的物理内存（比如这是一个共享内存，那么第一次 unmap 就不 free，最后一个 unmap 肯定要 free）。

```c
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm);

// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free);
```

然后有三个实用的跨页表操作函数：

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len);

// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);
```
用于与指定页表进行数据交换，`copyout` 可以向页表中写东西，后续用于 `sys_read`，也就是给用户传数据，`copyin` 用户接受用户的 buffer，也就是从用户哪里读数据。**注意**，用户在启用了虚拟内存之后，用户 syscall 给出的指针是不能直接用的，因为与内核的映射不一样，会读取错误的物理地址，使用指针必须通过 `useraddr` 转化，当然，更加推荐的是 `copyin/out` 接口，否则很可能损坏内存数据，同时，`copyin/out` 接口处理了虚存跨页的情况，`useraddr` 则需要手动判断并处理。

> 注意虚拟内存上的连续页面，物理内存（或者说内核视角）不一定是连续的，所以如果传递数据碰到跨页现象，必须特判处理！否则很容易导致读写错误内存，无线玄学 bug! 用户 buffer 要么使用 copyin/out 接口，要么一定要判断跨页！

相关数据类型定义如下：

```c
typedef uint64 pte_t;
typedef uint64* pagetable_t;// 512 PTEs
```

## 内核页表

知道了页表操作相关函数，那么就来看看具体建立页表的过程吧，首先是内核页表，对应函数也在 `vm.c`。一般来说，由于内核有频繁的操作不同进程内存的需求，内核需要能够方便的访问所有的物理内存，所以内核映射往往十分简单，很多时候往往是线性映射，也就是内核 `va = pa + KERNEL_OFFSET`，在该示例代码中，为了更加的简单，取 `KERNEL_OFFSET = 0`，也就是内核虚拟地址完全等于实际物理地址。内核页表建立过程如下：

```c
// kernel/vm.c

// 页表项定义如下：
#define PTE_V (1L << 0)     // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4)     // 1 -> user can access

#define KERNBASE (0x80200000)
extern char e_text[];     // kernel.ld sets this to end of kernel code.
extern char trampoline[];

pagetable_t kvmmake(void) {
    pagetable_t kpgtbl;
    kpgtbl = (pagetable_t) kalloc();
    memset(kpgtbl, 0, PGSIZE);
    mappages(kpgtbl, KERNBASE, KERNBASE, (uint64) e_text - KERNBASE, PTE_R | PTE_X);
    mappages(kpgtbl, (uint64) e_text, (uint64) e_text, PHYSTOP - (uint64) e_text, PTE_R | PTE_W);
    mappages(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
    return kpgtbl;
}
```

其中，内核页表地址随机取一个页就可以。内核映射有三段，一段为代码段:[KERNBASE, e_text)，可读可执行（注意页表项）。一段为数据段：[e_text, 0x88000000),可读可写。关于 `TRAMPOLINE` 段含义稍后解释。

激活页表机制在 `kinit` 函数：

```c
// kernel/vm.c

void kvminit(void) {
    kernel_pagetable = kvmmake();
    w_satp(MAKE_SATP(kernel_pagetable));    // 写 satp 寄存器，正式激活页表机制
    sfence_vma();                           // 更换页表，必须刷新 TLB
}
```

## 用户地址加载

用户的加载逻辑在 `loader.c` 中（也就是原来的 `batch.c`，改名了），其中唯一逻辑变化较大的就是 `bin_loader` 函数：

```c
// kernel/vm.c

// kernel/loader.c
pagetable_t bin_loader(uint64 start, uint64 end, struct proc *p) {
    pagetable_t pg = (pagetable_t) kalloc();
    memset(pg, 0, PGSIZE);
    // trampoline 就是 uservec userret 两个函数的位置
    mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0);
    // trapframe 之前是预分配的，现在我们用 kalloc 得到。
    p->trapframe = (struct trapframe*)kalloc();
    memset(p->trapframe, 0, PGSIZE);
    // map trapframe，位置稍后解释
    mappages(pg, TRAPFRAME, PGSIZE, (uint64)p->trapframe, PTE_R | PTE_W);
    // 这部分就是 bin 程序的实际 map, 我们把 [BASE_ADDRESS, APP_SIZE) map 到 [app_start, app_end)
    // 替代了之前的拷贝
    uint64 s = PGROUNDDOWN(start), e = PGROUNDUP(end);
    if (mappages(pg, BASE_ADDRESS, e - s, s, PTE_U | PTE_R | PTE_W | PTE_X) != 0) {
        panic("wrong loader 1\n");
    }
    p->pagetable = pg;
    p->trapframe->epc = BASE_ADDRESS;
    // map user stack
    mappages(pg, USTACK_BOTTOM, USTACK_SIZE, (uint64) kalloc(), PTE_U | PTE_R | PTE_W | PTE_X);
    p->ustack = USTACK_BOTTOM;
    p->trapframe->sp = p->ustack + USTACK_SIZE;
    return pg;
}
```

其中 `trapframe` 和 `trampoline`代码（也就是 `userret`、`uservec`函数）比较特殊，这两块内存用户特权级切换，必须用户态和内核态都能访问。所以它们在内核和用户页表中都有 map，注意所有 kalloc() 分配的内存内核都能访问。他们设定的虚拟地址为:

```c
// one beyond the highest possible virtual address.
// MAXVA is actually one bit less than the max allowed by
// Sv39, to avoid having to sign-extend virtual addresses
// that have the high bit set.
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))

#define USER_TOP (MAXVA)

#define TRAMPOLINE (USER_TOP - PGSIZE)

#define TRAPFRAME (TRAMPOLINE - PGSIZE)
```

这与为何要这么设定，留给读者思考（不是很关键，感兴趣可以在群里讨论或者直接找助教）。

`bin_loader` 主要变化就是利用 map 代替了原来的拷贝，这可以节约时间和内存，而且 `BASE_ADDRESS` 可以设置为 0x1000 这个看起来更统一和舒服的地址，不再是 `0x80400000`。

## 其他

由于采用了原地映射（也就是 `KERNEL_OFFSET = 0`），内核大部分代码不需要调整，除了 `trampoline` 中的 `userret` 函数，需要调整 `trap.c` 中的 `userrettrap` 函数最后几行跳转到 `userret` 的逻辑：

```c
// kernel/trap.c

void usertrapret() {
    // ... 

    // tell trampoline.S the user page table to switch to.
    uint64 satp = MAKE_SATP(curr_proc()->pagetable);
    uint64 fn = TRAMPOLINE + (userret - trampoline);
    ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```

`vm.c` 等文件中还新增了一些函数，这些函数在 lab4 不是很重要，在 lab5 我们会对他们进行了解。也就是说，lab5 会对内存模型进行一定的调整，之后我们的内存模型就稳定了。

## 展望

习题课上话说太满了。。。。好像 lab4 也不是很难的样子哈 (O.O)，不过好像 rust 很难的样子诶(偷笑。下一章主要是 fork 和 exec，就酱紫。