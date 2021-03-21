# lab7 实验要求

- 本节难度： **难度较大** 

## 编程作业

### 硬链接

你的电脑桌面是咋样的？是放满了图标吗？反正我的 windows 是这样的。显然很少人会真的吧可执行文件放到桌面上，桌面图标其实都是一些快捷方式。或者用 unix 的术语来说：软链接。为了减少工作量，我们今天来实现软链接的兄弟：[硬链接](https://en.wikipedia.org/wiki/Hard_link)。

硬链接要求两个不同的目录项指向同一个文件，在我们的文件系统中也就是两个不同名称目录项指向同一个磁盘块。本节要求实现三个系统调用 sys_linkat、sys_unlinkat、sys_stat。

link：
  * syscall ID: 37
  * 功能：创建一个文件的一个硬链接，[具体含义](https://linux.die.net/man/2/linkat)。
  * Ｃ接口： `int linkat(int olddirfd, char* oldpath, int newdirfd, char* newpath, unsigned int flags)`
  * Rust 接口： `fn linkat(olddirfd: i32, oldpath: *const u8, newdirfd: i32, newpath: *const u8, flags: u32) -> i32`
  * 参数：
    * olddirfd，newdirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。
    * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
    * oldpath：原有文件路径
    * newpath: 新的链接文件路径。
  * 说明：
    * 为了方便，不考虑新文件路径已经存在的情况（属于未定义行为），除非链接同名文件。
  * 返回值：如果出现了错误则返回 -1，否则返回 0。
  * 可能的错误
    * 链接同名文件。

unlink:
  * syscall ID: 35
  * 功能：取消一个文件路径到文件的链接,[具体含义](https://linux.die.net/man/2/unlinkat)。
  * Ｃ接口： `int unlinkat(int dirfd, char* path, unsigned int flags)`
  * Rust 接口： `fn unlinkat(dirfd: i32, path: *const u8, flags: u32) -> i32`
  * 参数：
    * dirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。
    * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
    * path：文件路径。
  * 说明：
    * 为了方便，不考虑使用 unlink 彻底删除文件的情况。
  * 返回值：如果出现了错误则返回 -1，否则返回 0。
  * 可能的错误
    * 文件不存在。

fstat:
  * syscall ID: 80
  * 功能：获取文件状态。
  * Ｃ接口： `int fstat(int fd, struct Stat* st)`
  * Rust 接口： `fn fstat(fd: i32, st: *mut Stat) -> i32`
  * 参数：
    * fd: 文件描述符
    * st: 文件状态结构体
      ```c
      struct Stat {
      	uint64 dev,     // 文件所在磁盘驱动器号
      	uint64 ino,     // inode 文件所在 inode 编号
      	uint32 mode,    // 文件类型
      	uint32 nlink,   // 硬链接数量，初始为1
      	uint64 pad[7],  // 无需考虑，为了兼容性设计
      }
      
      // 文件类型只需要考虑:
      ＃define DIR 0o040000		// directory
      ＃define FILE 0o100000		// ordinary regular file
      ```
    * 返回值：如果出现了错误则返回 -1，否则返回 0。
    * 可能的错误
      * fd 无效。
      * st 地址非法。

### 实验要求

- 实现分支：ch7。
- 完成实验指导书中的内容，实现基本的文件操作。
- 实现硬链接及相关系统调用，并通过 [C测例](https://github.com/DeathWish5/riscvos-c-tests)中 chapter7 对应的所有测例。

challenge: 支持多核。

### 实验检查

- 实验目录要求

    目录要求不变(参考lab1目录或者示例代码目录结构)。同样在 os 目录下 `make run` 之后可以正确加载用户程序并执行。

    加载的用户测例位置： `../user/target/bin`。

- 检查

    可以正确 `make run` 执行，可以正确执行目标用户测例，并得到预期输出（详见测例注释）。

## 问答作业

1. 目前的文件系统只有单级目录，假设想要支持多级文件目录，请描述你设想的实现方式，描述合理即可。

2. 在有了多级目录之后，我们就也可以为一个目录增加硬链接了。在这种情况下，文件树中是否可能出现环路？你认为应该如何解决？请在你喜欢的系统上实现一个环路(软硬链接都可以，鼓励多尝试)，描述你的实现方式以及系统提示、实际测试结果。

## 报告要求

* 简单总结本次实验与上个实验相比你增加的东西。（控制在5行以内，不要贴代码）
* 完成问答问题
* (optional) 你对本次实验设计及难度的看法。