---
layout: posts
title: shared memory
date: 2024-11-12 00:42:18
tags:
  - C
  - sys
category:
  - 编程
mathjax: true
---

最近在嗯看 shared memory（共享内存）, 感觉要似了。这是一种 ipc 手段（不是 cuda 那个啦），具体来讲是通过将不同进程的虚拟地址映射到同一块物理内存，从而让这块内存能够被跨进程访问。由于通信实际上就是直接 memcpy, 速度上会比较快，但是需要一些跨进程同步手段（比如信号量之类），这方面会比较难写。下面具体讲讲罢。

<!--more-->

## shared memory 初探

我们可以通过 `free` 命令来看用了多少共享内存：

```sh
$ free
               total        used        free      shared  buff/cache   available
Mem:        16297496     4288184    11109888        3712      899424    11673012
Swap:        4194304           0     4194304
$ 
```

当然你也可以通过 `ipcs` 来查看。这玩应不仅能看共享内存还能看别的 ipc 信息：

```sh
$ ipcs

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      

------ Semaphore Arrays --------
key        semid      owner      perms      nsems 

$ 
```

不过这个方式会有些问题，下文会讲到。

我们可以通过一些手段控制系统层面的共享内存数目和大小。主要有下面三个配置项：

- SHMMAX: 共享内存的最大段大小。你可以通过查看或者写入文件 `/proc/sys/kernel/shmmax` 来控制它，也可以用 `sysctl` 来控制它。`sysctl` 是这么控制滴：`sysctl -w kernel.shmmax=2147483648 >> /etc/sysctl.conf`。
- SHMMNI: 系统中最多的共享内存数目。控制方法同上。
- SHMALL：共享内存的总量。控制方法同上。

## shared memory 的 API

linux 提供了两种风格的 API, 分别是 System-V API 和 POSIX API。这两种 API 长得很不一样。（实际上对于其他的 IPC 手段，这两种 API 的接口都不太一样）

### System-V API

SysV API 主要使用 SysV 独有的 IPC key 来标识一个 ipc 对象，共享内存也是一样。我们首先需要一个 API 生成这个 key:

```c
#include <sys/types.h>
#include <sys/ipc.h>

key_t ftok (const char *pathname, int proj_id);
```

`pathname` 需要是个已存在的文件（因为 key 的生成主要用该文件的 inode id），`proj_id` 随便怎样都行啦，但是不能是 0. 它返回一个 SysV IPC key。

然后我们需要一个 API 来通过 key 获取 shared memory:

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int shmget (key_t key, size_t size, int shmflg);
```

`key` 就是刚刚获取的 key；当然你也可以把它设置为 `IPC_PRIVATE` ，这样他就会完全忽略后面的 `shmflg` 创建一段新的。`shmflg` 用来控制你申请到的共享内存是怎么样的。有这么一些选项：

- `IPC_CREAT` 创建一段新的共享内存。如果跟 `IPC_EXCL` 合用，那么 key 已经有共享内存对应的情况下会报错返回 -1 。
- `IPC_EXCL` 见上。
- `SHM_HUGETLB` 表明这段内存是大页的。
- `SHM_NORESERVE` 跟 `MAP_NORESERVE` 差不多，就是告诉 os 不要给我这段内存保留 swap 空间啦。

这个 API 的返回值就是这段内存的 id 啦。

> 这里实际上有个问题，就是你用 `IPC_PRIVATE` 获取的共享内存怎么能够让其他进程访问到呢？
> 一般的解决办法是进程之间通过别的手段交换 shmid，或者干脆就是一对父子进程，继承了 shmid。

由于返回的是 id 而不是具体的虚拟地址，我们还需要一个能够返回虚拟地址的 API:

```c
#include <sys/types.h>
#include <sys/shm.h>

void *shmat (int shmid, const void *shmaddr, int shmflg);
```

`shmid` 就是 `shmget()` 的返回值； `shmflg` 是一些操控选项，不过跟 `shmget()` 里面的不太一样；如果我们想指定它映射到哪个虚拟地址，我们可以用 `shmaddr` 指定这个虚拟地址。

这里还是给一下 `shmflg` 的选项：

- `SHM_RND` 在 `shmaddr` 被指定时，用来给这个地址作对齐的；如果没有的话， `shmaddr` 就必须是对齐的。
- `SHM_RDONLY` 指定这一段内存对于现在的进程是只读的。
- `SHM_REMAP` 直接 remap `shmaddr` 原先指向的内存。这个只有 Linux 有。

另外提一嘴，这段内存在 fork 之后是会被子进程继承的。

如果想要控制这段共享内存，怎么办呢？我们用 `shmctl()`:

```c
#include <sys/ipc.h>
#include <sys/shm.h>

int shmctl(int shmid, int cmd, struct shmid_ds *buf); 
```

这个 `shmid_ds` 是个什么东西呢？

```c
struct shmid_ds {
    struct ipc_perm shm_perm;    /* Ownership and permissions */
    size_t          shm_segsz;   /* Size of segment (bytes) */
    time_t          shm_atime;   /* Last attach time */
    time_t          shm_dtime;   /* Last detach time */
    time_t          shm_ctime;   /* Last change time */
    pid_t           shm_cpid;    /* PID of creator */
    pid_t           shm_lpid;    /* PID of last shmat(2)/shmdt(2) */
    shmatt_t        shm_nattch;  /* No. of current attaches */
    ...
};

struct ipc_perm {
    key_t          __key;    /* Key supplied to shmget(2) */
    uid_t          uid;      /* Effective UID of owner */
    gid_t          gid;      /* Effective GID of owner */
    uid_t          cuid;     /* Effective UID of creator */
    gid_t          cgid;     /* Effective GID of creator */
    unsigned short mode;     /* Permissions + SHM_DEST and
                                SHM_LOCKED flags */
    unsigned short __seq;    /* Sequence number */
};
```

可以说是一个用来管理共享内存的数据结构。

`cmd` 有哪些我累了不想讲了，只要知道 `IPC_STAT` 用来获取 `shmid_ds`，`IPC_SET` 用来设置 `shmid_ds`，`IPC_RMID` 用来删掉这块共享内存即可。

如果不想要这块内存了怎么办？使用 `shmdt()` 罢：

```c
#include <sys/types.h>
#include <sys/shm.h>

int shmdt(const void *shmaddr);
```

可以看到，这个 SysV API 是比较麻烦的，而下面要讲到的 POSIX API 更加契合 Unix “万物皆文件”的思想。

### POSIX API

POSIX API 是比 SysV API 更加晚近的一种 API，它更加依靠“名字”而非 key 来对共享内存进行标识。这个设施是通过 tmpfs 来提供的。这里只有两个 API 是共享内存独有的，其他的操作可以通过一些已有的设施来进行。

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int shm_open (const char *name, int oflag, mode_t mode);
```

这玩应就是一个共享内存版的 `open()` ，这意味着它返回的就是一个文件描述符。不过名字有些差别，一定得是 `/{name}` 的形式。这并不意味着它会在根目录给你拉屎，只是一个名字而已。实际上，通过这种方法创建的共享内存文件都是在 /dev/shm/ 下。

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int shm_unlink(const char *name); 
```

这个 API 就是根据名字删掉那个共享内存文件了。

既然是文件，我们就可以通过一些文件的手段来操控这段共享内存。你可以用 `ftruncate()` 来控制共享内存大小，用 `close()` 关掉共享内存的文件。不仅如此，我们还可以用（而且必须用） `mmap()` 映射到这块共享内存上，用法也是跟平常的文件映射一模一样：

```c
// Get shared memory 
if ((fd_shm = shm_open (SHARED_MEM_NAME, O_RDWR, 0)) == -1)
    error ("shm_open");

if ((shared_mem_ptr = mmap (NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd_shm, 0)) == MAP_FAILED)
    error ("mmap");
```

你只需要指定 `MAP_SHARED` 就行了。是不是很简单？

读到这里，相信有聪明的读者会问既然这玩应又新又好，为什么 SysV API 还活着呢？这个主要是兼容性的问题，起码 macOS 并没有支持完全。还有一个问题是另一些老设施是没法看到 POSIX shared memory 的。比如说 `ipcs`:

> ipcs shows information on System V inter-process communication
> facilities. By default it shows information about all three
> resources: shared memory segments, message queues, and semaphore
> arrays.

还有人会问既然如此为什么不直接用常规文件和 `mmap()` 啊，这个确实。有人读了 glibc 的源码，发现 `shm_open()` 相比于 `open()` 只不过是给文件名加了前缀而已，要说有什么区别的话还是 `/dev/shm` 是个 tmpfs 而已（（（

> 不过好像在一些别的系统上这两者也会有差别

> 你还可以用 `memfd_create()` 创建一个内存上的文件

## shared memory 的使用

shared memory 的使用比较麻烦，需要担心跨进程同步的问题，一般搭配信号量来使用。

这里给一个使用 POSIX API 的示例程序，它总体上是生产者模式的，producer 会给 consumer 按行发送里尔克的第一首哀歌，但是 buf 只有一块共享内存（（（

`producer.cpp`:

```cpp
#include <cstdio>
#include <cstdlib>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/types.h>
#include <unistd.h>
#include <cstring>
#include <semaphore.h>

void exit_on_error(const char *msg) {
    perror(msg);
    exit(EXIT_FAILURE);
}

const char *contents = R"(
Wer, wenn ich schriee, hörte mich denn aus der Engel
Ordnungen? und gesetzt selbst, es nähme
einer mich plötzlich ans Herz: ich verginge von seinem
stärkeren Dasein. Denn das Schöne ist nichts
als des Schrecklichen Anfang, den wir noch grade ertragen,
und wir bewundern es so, weil es gelassen verschmäht,
uns zu zerstören. Ein jeder Engel ist schrecklich.
exit
)";

int main() {
    int fd = shm_open("/shmem_0", O_CREAT | O_RDWR, S_IRUSR | S_IWUSR);
    if (fd == -1)
        exit_on_error("shm_open");

    printf("contents size: %ld\n", strlen(contents));

    ftruncate(fd, 4096);
    char *addr = (char *)mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)
        exit_on_error("mmap");

    fprintf(stderr, "addr: %p [0~%d]\n", addr, 4095);
    fprintf(stderr, "file: /dev/shm%s\n", "/shmem_0");

    sem_t *sem = sem_open("/sem_0", O_CREAT, S_IRUSR | S_IWUSR, 1);
    if (sem == SEM_FAILED)
        exit_on_error("sem_open");
    sem_t *sem_empty_buf = sem_open("/sem_empty_buf", O_CREAT, S_IRUSR | S_IWUSR, 1);
    if (sem_empty_buf == SEM_FAILED)
        exit_on_error("sem_open");
    sem_t *sem_full_buf = sem_open("/sem_full_buf", O_CREAT, S_IRUSR | S_IWUSR, 0);
    if (sem_full_buf == SEM_FAILED)
        exit_on_error("sem_open");


    fprintf(stderr, "process %d wrote into addr %p\n", getpid(), addr);

    char *addr2 = addr;
    const char *p = contents;
    while (*p != '\0') {
        sem_wait(sem_empty_buf);
        sem_wait(sem);
        while (*p != '\n') {
            *addr2 = *p;
            ++addr2;
            ++p;
        }
        *addr2 = '\n';
        ++addr2;
        ++p;
        sem_post(sem);
        sem_post(sem_full_buf);
    }

    if (munmap(addr, 4096) == -1)
        exit_on_error("munmap");

    if (close(fd) == -1)
        exit_on_error("close");

    if (sem_close(sem) == -1)
        exit_on_error("sem_close");
    if (sem_close(sem_empty_buf) == -1)
        exit_on_error("sem_close");
    if (sem_close(sem_full_buf) == -1)
        exit_on_error("sem_close");

    if (shm_unlink("/shmem_0") == -1)
        exit_on_error("shm_unlink");

    if (sem_unlink("/sem_0") == -1)
        exit_on_error("sem_unlink");
    if (sem_unlink("/sem_empty_buf") == -1)
        exit_on_error("sem_unlink");
    if (sem_unlink("/sem_full_buf") == -1)
        exit_on_error("sem_unlink");

    return 0;
}
```

`consumer.cpp`:

```cpp
// 前面的不复制了...

int main() {
    int fd = shm_open("/shmem_0", O_CREAT | O_RDWR, S_IRUSR | S_IWUSR);
    if (fd == -1)
        exit_on_error("shm_open");

    ftruncate(fd, 4096);

    char *addr = (char *)mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)
        exit_on_error("mmap");

    sem_t *sem = sem_open("/sem_0", O_EXCL);
    if (sem == SEM_FAILED)
        exit_on_error("sem_open");
    sem_t *sem_empty_buf = sem_open("/sem_empty_buf", O_EXCL);
    if (sem_empty_buf == SEM_FAILED)
        exit_on_error("sem_open");
    sem_t *sem_full_buf = sem_open("/sem_full_buf", O_EXCL);
    if (sem_full_buf == SEM_FAILED)
        exit_on_error("sem_open");

    fprintf(stderr, "process %d read from addr: %p\n", getpid(), addr);

    char *addr2 = addr;
    char line_buf[256];
    char *line_buf_ptr = line_buf;

    while (true) {
        sem_wait(sem_full_buf);
        sem_wait(sem);
        char *p = addr2;
        while (*p != '\n') {
            *line_buf_ptr = *p;
            ++line_buf_ptr;
            ++p;
        }
        *line_buf_ptr = '\n';
        ++line_buf_ptr;
        *line_buf_ptr = '\0';
        line_buf_ptr = line_buf;
        addr2 = p + 1;
        sem_post(sem);
        sem_post(sem_empty_buf);
        if (strcmp(line_buf, "exit\n") == 0)
            break;
        printf("%s", line_buf);
    }

    if (munmap(addr, 4096) == -1)
        exit_on_error("munmap");

    if (close(fd) == -1)
        exit_on_error("close");

    if (sem_close(sem) == -1)
        exit_on_error("sem_close");
    if (sem_close(sem_empty_buf) == -1)
        exit_on_error("sem_close");
    if (sem_close(sem_full_buf) == -1)
        exit_on_error("sem_close");

    return 0;
}
```

值得吐槽的是这个同步写死我了。。。写的时候老是报 bus error

## benchmark

shared memory 不是说比别的 IPC 快嘛，我们来看看是真的吗。这里有一个 [benchmark](https://github.com/goldsborough/ipc-bench), 大家可以看看。shared memory 似乎跟 mmap 普通文件没啥差别，甚至其实要慢一些。


## references

1. [http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/](http://ranler.github.io/2013/07/01/System-V-and-POSIX-IPC/)
2. [https://www.softprayog.in/programming/interprocess-communication-using-system-v-shared-memory-in-linux](https://www.softprayog.in/programming/interprocess-communication-using-system-v-shared-memory-in-linux)
3. [https://man7.org/linux/man-pages/man2/memfd_create.2.html](https://man7.org/linux/man-pages/man2/memfd_create.2.html)
4. [https://man7.org](https://man7.org)
5. [https://linux.die.net](https://linux.die.net)
6. [https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc](https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc)
