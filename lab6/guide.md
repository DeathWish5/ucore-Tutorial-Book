# 进程间通信

废话不说了，这一章写进程间通信，具体实现是 pipe。我们来看看与 pipe 相关的系统调用长什么样子。

```c++
/// 功能：为当前进程打开一个管道。
/// 参数：pipe 表示应用地址空间中的一个长度为 2 的 long 数组的起始地址，内核需要按顺序将管道读端和写端的文件描述符写入到数组中。
/// 返回值：如果出现了错误则返回 -1，否则返回 0 。可能的错误原因是：传入的地址不合法。
/// syscall ID：59
long sys_pipe(long fd[2]);

// ...
long sys_read(int fd, void* buf, size_t size);

// ...
long sys_write(int fd, void* buf, size_t size);

/// 功能：当前进程关闭一个文件。
/// 参数：fd 表示要关闭的文件的文件描述符。
/// 返回值：如果成功关闭则返回 0 ，否则返回 -1 。可能的出错原因：传入的文件描述符并不对应一个打开的文件。
/// syscall ID：57
long sys_close(int fd);
```

此外还有读写管道的系统调用，其实就是 sys_write 和 sys_read，这两个是我们的老朋友了，此外还需要实现 sys_close 用来关闭管道。

秉承越简单越好、能跑就行就方针，ucore-tutorial 对于 pipe 的设计也十分简单: 找一块空闲内存作为 pipe 的 data buffer，对 pipe 的读写就转化为了对这块内存的读写。想一想是不是特别简单？但是再仔细想想，sys_write 还需要完成屏幕输出，一个程序还可以拥有多个 pipe，而且 pipe 还要能够使得其他程序可见来完成进程通讯的功能，对每个 pipe 还要维护一些状态来记录上一次读写到的位置和 pipe 实际可读的 size等。其实，这里所有的麻烦点大多来自于文件系统接口对我们的要求，考虑到 lab7 就要实现 file system 了，在 lab6 我们将实现文件系统的雏形。

## 文件系统初步

首先考虑刚才说到的全局可见，没错，一个需要文件能够被多个进程打开，文件不是一个进程的属性，而是整个 os 的属性。为此，我们需要一个全局文件表（虽然这个时候里面只有 pipe...)

见 `file.h`:

```c++
// pipe.h
#define PIPESIZE 512

struct pipe {
    char data[PIPESIZE];    
    uint nread;     // number of bytes read
    uint nwrite;    // number of bytes written
    int readopen;   // read fd is still open
    int writeopen;  // write fd is still open
};

// file.h
struct file {
    enum { FD_NONE = 0, FD_PIPE} type;  // FD_NODE means this file is null.
    int ref;           // reference count
    char readable;
    char writable;
    struct pipe *pipe; // FD_PIPE
};

// 全局文件池
extern struct file filepool[128 * 16];
```

这里我们可以看到全局文件池和 `struct file` 的定义，注意文件是使用引用计数来管理的。目前，我们不认为 `stdin`、`stdout`、`stderr` 是真正的文件，而是直接和串口接在一起，这是一种临时的实现，在后续实验中会改掉，所以文件类型中只有 `FD_PIPE` 有效。

如下是我们对于文件的分配和关闭(注意我们还没有实现 `sys_open`)，分配很简单，遍历进程池找 `ref == 0` 的，关闭就是 `ref--`，如果为 0 就真的关闭，对于 pipe 来说就是执行 `pipe_close`。

```c++
// kernel/file.c
struct file* filealloc() {
    for(int i = 0; i < FILE_MAX; ++i) {
        if(filepool[i].ref == 0) {
            filepool[i].ref = 1;
            return &filepool[i];
        }
    }
    return 0;
}

void
fileclose(struct file *f)
{
    if(--f->ref > 0) {
        return;
    }
    if(f->type == FD_PIPE){
        pipeclose(f->pipe, f->writable);
    }
    memset(f, 0, sizeof(struct file));
}
```

然后，进程也需要增加文件相关支持。进程控制块增加文件指针数组。

```diff
// proc.h
// Per-process state
struct proc {
    // ...

+   struct file* files[16];
};
```

每一个文件指针与对应 fd 关联，fd分配很简单，遍历寻找空闲 fd。

```c++
// kernel/proc.c
int fdalloc(struct file* f) {
    struct proc* p = curr_proc();
    // fd = 0,1,2 is reserved for stdio/stdout/stderr
    for(int i = 3; i < FD_MAX; ++i) {
        if(p->files[i] == 0) {
            p->files[i] = f;
            return i;
        }
    }
    return -1;
}
```

## pipe 函数

现在我们来看看 `pipe_close` 等是咋做的（`kernel/pipe.c`）。首先是 pipe 的分配：

```c++
int
pipealloc(struct file *f0, struct file *f1)
{
    // 这里没有用预分配，由于 pipe 比较大，直接拿一个页过来，也不算太浪费
    struct pipe *pi = (struct pipe*)kalloc();
    // 一开始 pipe 可读可写，但是已读和已写内容为 0
    pi->readopen = 1;
    pi->writeopen = 1;
    pi->nwrite = 0;
    pi->nread = 0;

    // 两个参数分别通过 filealloc 得到，把该 pipe 和这两个文件关连，一端可读，一端可写。读写端控制是 sys_pipe 的要求。
    f0->type = FD_PIPE;
    f0->readable = 1;
    f0->writable = 0;
    f0->pipe = pi;

    f1->type = FD_PIPE;
    f1->readable = 0;
    f1->writable = 1;
    f1->pipe = pi;
    return 0;
}
```

pipe 的关闭：

```c++
// 该函数其实只关闭了读写端中的一个，如果两个都被关闭，释放 pipe。
void
pipeclose(struct pipe *pi, int writable)
{
    if(writable){
        pi->writeopen = 0;
    } else {
        pi->readopen = 0;
    }
    if(pi->readopen == 0 && pi->writeopen == 0){
        kfree((char*)pi);
    }
}
```

pipe 的读写：（注意，pipe 是使用 ring buffer 管理的）

```c++
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
    // w 记录已经写的字节数
    int w = 0;
    struct proc *p = curr_proc();
    while(w < n){
        // 若不可读，写也没有意义
        if(pi->readopen == 0){
            return -1;
        }

        if(pi->nwrite == pi->nread + PIPESIZE){
            // pipe write 端已满，阻塞
            yield();
        } else {
            // 一次读的 size 为 min(用户buffer剩余，pipe 剩余写容量，pipe 剩余线性容量)
            uint64 size = MIN(
                n - w, 
                pi->nread + PIPESIZE - pi->nwrite, 
                PIPESIZE - (pi->nwrite % PIPESIZE
            );
            // 使用 copyin 读入用户 buffer 内容 
            copyin(p->pagetable, &pi->data[pi->nwrite % PIPESIZE], addr + w, size);
            pi->nwrite += size;
            w += size;
        }
    }
    return w;
}

int
piperead(struct pipe *pi, uint64 addr, int n)
{
    // r 记录已经写的字节数
    int r = 0;
    struct proc *p = curr_proc();
    // 若 pipe 可读内容为空，阻塞或者报错
    while(pi->nread == pi->nwrite) {
        if(pi->writeopen)
            yield();
        else
            return -1;
    }
    while(r < n && size != 0) {
        // pipe 可读内容为空，返回
        if(pi->nread == pi->nwrite)
            break;
        // 一次写的 size 为：min(用户buffer剩余，可读内容，pipe剩余线性容量)
        uint64 size = MIN(
            n - r, 
            pi->nwrite - pi->nread, 
            PIPESIZE - (pi->nread % PIPESIZE)
        );
        // 使用 copyout 写用户内存
        copyout(p->pagetable, addr + r, &pi->data[pi->nread % PIPESIZE], size);
        pi->nread += size;
        r += size;
    }
    return r;
}
```

好了，到现在，我们可以来实现那几个系统调用了。

## pipe 相关系统调用

首先是 `sys_pipe`，记不记得 一开始描述的 `sys_pipe` 接口？

```c++
// kernel/syscall.c
uint64
sys_pipe(uint64 fdarray) {
    struct proc *p = curr_proc();
    // 申请两个空 file
    struct file* f0 = filealloc();
    struct file* f1 = filealloc();
    // 实际分配一个 pipe，与两个文件关联
    pipealloc(f0, f1);
    // 分配两个 fd，并将之与 文件指针关联
    fd0 = fdalloc(f0);
    fd1 = fdalloc(f1);
    size_t PSIZE = sizeof(fd0);
    copyout(p->pagetable, fdarray, &fd0, PSIZE);
    copyout(p->pagetable, fdarray + PSIZE, &fd1, PSIZE);
    return 0;
}
```

sys_close 比较简单：

```c++
uint64 sys_close(int fd) {
    // stdio/stdout/stderr can't be closed for now
    if(fd <= 2)
        return 0;
    struct proc *p = curr_proc();
    fileclose(p->files[fd]);
    p->files[fd] = 0;
    return 0;
}
```

原来的 `sys_write` 更名为 `console_write`，新 `sys_write` 根据文件类型分别调用 `console_write` 和 `pipe_write`。`sys_read` 同理。

```c++
uint64 sys_write(int fd, uint64 va, uint64 len) {
    if(fd <= 2) {
        return console_write(va, len);
    }
    struct proc *p = curr_proc();
    struct file *f = p->files[fd];
    if(f->type == FD_PIPE) {
        return pipewrite(f->pipe, va, len);
    }
    error("unknown file type %d\n", f->type);
    return -1;
}

uint64 sys_read(int fd, uint64 va, uint64 len) {
    if(fd <= 2) {
        return console_read(va, len);
    }
    struct proc *p = curr_proc();
    struct file *f = p->files[fd];
    if(f->type == FD_PIPE) {
        return piperead(f->pipe, va, len);
    }
    error("unknown file type %d\n", f->type);
    return -1;
}
```

以上。

## 进程通讯与 fork

fork 为什么是毒瘤呢？因为你总是要在新增加一个东西以后考虑要不要为新功能增加 fork 支持。这一章的文件就是第一个例子，那么在 fork 语境下，文件应该如何处理呢？我们来看看 fork 在这一个 chapter 的实现：

```diff
int fork() {
    // ...
+   for(int i = 3; i < FD_MAX; ++i)
+       if(p->files[i] != 0 && p->files[i]->type != FD_NONE) {
+           p->files[i]->ref++;
+           np->files[i] = p->files[i];
+       }
    // ...
}
```

没错，只需要拷贝文件指针并增加文件引用计数就好了，不需要重新新建一个文件。那么，`exec` 呢？你会发现 `exec` 的实现竟然没有修改，注意 `exec` 仅仅重新加载进程执行的文件镜像，不会改变其他属性，比如文件。也就是说，`fork` 出的子进程打开了与父进程相同的文件，但是 exec 并不会把打开的文件刷掉，基于这一点，我们可以利用 pipe 进行进程间通信。在用户态有一个简单例子：

见 `user/pipetest.c`:

```c++
char STR[] = "hello pipe!";

int main() {
    uint64 pipe_fd[2];
    int ret = pipe(&pipe_fd);
    if (fork() == 0) {
        // 子进程，从 pipe 读，和 STR 比较。
        char buffer[32 + 1];
        read(pipe_fd[0], buffer, 32);
        assert(strncmp(buffer, STR, strlen(STR) == 0);
        exit(0);
    } else {
        // 父进程，写 pipe
        write(pipe_fd[1], STR, strlen(STR));
        int exit_code = 0;
        wait(&exit_code);
        assert(exit_code == 0);
    }
    return 0;
}
```

## 展望

这一章已经给 lab7 打好一定的底子了，lab7 我们将拥有文件系统！我们的 os 将拥有操纵持久化存储的能力，wow!


