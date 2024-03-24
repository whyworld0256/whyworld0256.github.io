# Lab 2

## task 1 System call tracing

> 研究了一会终于搞懂了system call的流程
> 以trace命令为例，trace.c的main函数调用了函数 int trace(int) ，其declaration在user.h中，而其implementation则在由脚本文件usys.pl产生的汇编文件usys.S当中。usys.s把宏SYS_trace保存在寄存器a7当中，然后执行ecall进入kernel mode并跳转到syscall.c的syscall函数。syscall从a7读出宏SYS_trace至变量num，syscalls【num】调了函数sys_trace(sysproc.c里面)，并把sys_trace的返回值保存在寄存器a0并返回，usys.S在ecall后顺序执行ret返回，至此trace函数得到返回值，一次system call结束

- `user/trace.c`已经被官方实现， 但是在`trace.c`中调用了函数`trace`, 这个还没被实现
- 我们需要在`user/user.h`中添加`trace` ,  这样我们在`trace.c`中调用`trace`是，程序知道这个函数已经被定义了。
- 在`user/usys.pl` 中添加`trace`, 我也不知道为什么要这样。
- `Makefile` 会调用脚本`user/usys.pl`, 这样会生成`RISC-V` 的汇编文件`user/user.S`, 这里是真正的`trace` 的实现，此时系统会调用`ecall` 进入内核。
- 内核执行`syscall.c`, 这个函数会根据函数调用的编号调用函数指针`syscalls` 的某个位置的真正的系统调用(`sys_trace`)，这个系统调用是定义在`sysproc.c`中， 然后把返回值放到`a0`寄存器中去。

大概就是上面这个流程，虽然有些细节我也不懂。

### 具体实施

- 在`user/user.h`中加入: 

```c
int trace(int);
```

- 在`user/usys.pl` 中加入

```c
entry("trace");
```

- 在`kernel/syscall.h`中加入

```c
#define SYS_trace  22
```

- 在`kernel/syscall.c`中加入

```c
extern uint64 sys_trace(void);

[SYS_trace]   sys_trace,
```

- 此时我们需要在`kernel/sysproc.c`中具体定义`sys_trace` 函数了.

```c
uint64 sys_trace(void) {
    int mask;
    if (argint(0, &mask) < 0)
        return -1;
    myproc()->mask = mask;
    return 0;
}
```

- 为了使主进程和子进程可以传递`mask`，我们在结构体`proc`中加入`mask`变量。

```c
int mask;
```

- 修改`kernel/proc.c`中的 `fork`,使得父进程的`mask`会被传递给子进程。

```c
 np->sz = p->sz;
 np->mask = p->mask; // 复制父进程的mask数据到子进程
 np->parent = p;
```

- 修改`syscall.c`

```c
void syscall(void) {
    int num;
    struct proc *p = myproc();
    char *names[] = {"fork", "exit", "wait", "pipe", "read", "kill", "exec", "fstat", "chdir", "dup", "getpid",
                     "sbrk", "sleep", "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close",
                     "trace"};
    num = p->trapframe->a7;
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
        if ((1 << num) & myproc()->mask) {
            printf("%d: syscall %s -> %d\n", p->pid, names[num - 1], p->trapframe->a0);
            // 按照格式打印信息
        }
    } else {
        printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}
```

## task -2 Sysinfo

与上一题类似

- 在`user/user.h`中加上

```c
struct sysinfo;
int sysinfo(struct sysinfo *);
```

- 在`user/usys.pl` 中加上

```c
entry("sysinfo");
```

- 在`syscall.h` 中加上

```c
#defsine SYS_sysinfo  23
```

- 在`syscall.c` 中加上

```c
// ...
extern uint64 sys_sysinfo(void);
// ...
[SYS_sysinfo] sys_sysinfo,
// ...
```

```c
//1.获取指向sysinfo结构体的指针
//2.计算空闲的memory，并将数据放入sysinfo结构体相应的数据段中
//3.计算unused的process，并将数据放入sysinfo结构体相应的数据段中
//4.将内核中处理好的sysinfo传到用户空间
```

- ##### 获取指向sysinfo结构体的指针

```c
uint64 pSysinfo;
if(argaddr(0, &pSysinfo) < 0)
    return -1;
```

- ##### 计算空闲的memory

打开`kalloc.c`

```c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r; // 声明一个单链表（头结点指针）

  acquire(&kmem.lock); // 因为内存是临界资源，所以在分配内存时需要上锁
  r = kmem.freelist; // 令单链表头结点指针指向内存链表的头结点
  if(r) // 若内存结点存在，则从链表中摘下
    kmem.freelist = r->next;
  release(&kmem.lock); // 释放锁

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

可以确定，`kmem.freelist`链表中保存了空闲的内存，一个结点表示一页free memory，通过遍历链表即可计算出空闲内存的大小：

```c
// kernel/kalloc.c
uint64 countFreememory() {
    struct run *r;
    uint64 freememory = 0;
    acquire(&kmem.lock);
    r = kmem.freelist;
    while (r) {
        ++freememory;
        r = r->next;
    }
    release(&kmem.lock);
    return PGSIZE * freememory;
}
```

- 计算没被使用的process

根据提示，打开`proc.c`文件，注意到`allocproc()`函数中有关于unused proc的描述：

```c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.static struct proc*
allocproc(void)
{
  struct proc *p;
    
  for(p = proc; p < &proc[NPROC]; p++) { // 遍历proc数组（进程表）
    acquire(&p->lock);
    if(p->state == UNUSED) { // 找到state为UNUSED的进程
      goto found;
    } else {
      release(&p->lock);
    }     
  }
// ...
}
```

于是，通过遍历proc数组统计state为UNUSED的进程数量：

```c
uint64 countUnusedproc() {
    struct proc *p;
    uint64 unusedproc = 0;
    for (p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if (p->state != UNUSED) { 
            ++unusedproc;
        }
        release(&p->lock);
    }
    return unusedproc;
}
```

- ##### 将内核中得到数据的sysinfo传到用户空间

根据提示，观察`sys_fstat() (kernel/sysfile.c)`和`filestat() (kernel/file.c)`，学习如何使用`copyout()`：

```c
uint64
sys_fstat(void)
{
  // ...
  if(argfd(0, 0, &f) < 0 || argaddr(1, &st) < 0) // 获取用户传来的指针参数
  // ...
}

// Get metadata about file f.
// addr is a user virtual address, pointing to a struct stat.
int
filestat(struct file *f, uint64 addr)
{
  // ...
  struct stat st;
  // ...
  if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
  // 将从&st开始的sizeof(st)长的数据复制到p->pagetable的addr处
  // ...
}
```

模仿可得:

```c
// 从内核态拷贝到用户态
// 拷贝len字节数的数据, 从src指向的内核地址开始, 到由pagetable下的dstv用户地址
// 成功则返回 0, 失败返回 -1
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
```

```c
copyout(p->pagetable, addr, (char *)&sysinfo, sizeof(sysinfo));
```

- ##### 添加sysinfo系统调用的定义

因为`countFreememory()`和`countUnusedpro()`都是新定义的函数，所以需要在`defs.h`中声明函数接口。

> kernel中含有多个.c文件，若想要使用其他.c文件中的函数或者变量，就需要在defs.h中设置好接口，defs.h就像是所有函数的头文件。为了便于管理，defs.h中的函数声明是按照函数所处的不同.c文件进行分块放置的，比如，kalloc.c文件中的函数就放在// kalloc.c的注释下方。
>

```c
// kernel/defs.h
// kalloc.c
uint64          countFreememory(void);
// proc.c
uint64          countUnusedproc(void);
```

在`sysproc.c`中给出系统调用的定义：

```c
// kernel/sysproc.c
uint64
sys_sysinfo(void) {
    struct sysinfo info;
    uint64 addr;

    info.freemem = countFreememory();
    info.nproc = countUnusedproc();

    if (argaddr(0, &addr) < 0)
        return -1;
    if (copyout(myproc()->pagetable, addr, (char *) &info, sizeof(info)) < 0)
        return -1;
    return 0;
}
```

