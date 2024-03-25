# Lab 4

### task -1  RISC-V assembly

本任务的目的是帮助我们熟悉`RISC-V` 汇编，我们需要回答以下几个问题:

- > Which registers contain arguments to functions?  For example, which    register holds 13 in main's call to `printf`?  

- > Where is the call to function `f`    in the assembly code for main? Where is the call to `g`?  (Hint: the    compiler may inline functions.)  

- > At what address is the   function `printf` located?  

- > What value is in the register `ra` just after the `jalr` to `printf` in `main`?  

- ```c
  Run the following code.
  
  	unsigned int i = 0x00646c72;
  	printf("H%x Wo%s", 57616, &i);
        
  
  What is the output? Here's an ASCII table that maps bytes to characters.
  
  The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?
  ```

### 解答

- 在`RISC-V`里面，`a0-a7` 为参数寄存器，其中`13` 放在了`a2`
- 由于采用了函数内联化，`main` 并没有调用`f`和`g`, 直接就算出了结果`12`
- 由`0000000000000628 <printf>:` 知，其地址为`0x628`
- `jalr` 指令会把`PC+4`放到 `ra`,  因此`ra = 0x38`.
- 我们运行这段代码

```c++
#include<bits/stdc++.h>
using namespace std;
int main(){
	unsigned int i = 0x00646c72;
	printf("H%x Wo%s", 57616, &i);
	return 0;
}
```

结果为

```c
HE110 World
```

首先`%x` 表明我们需要输出`57616` 的16进制。

57616的16进制为`e110`

`00646c72` 的小端编码为`72 6c 64 00` 其表示字符串`r l d \0`



### task -  2 Backtrace

任务要求: 打印把栈上的所有`return address`

### 解答

我们首先可以通过函数`f_tp()` 拿到当前`frame pointer` 的值。 

栈上的结构:

```python
fp
RA
previous fp
variables
```

上图是一个函数的`stack frame` 上面是高地址，下面是低地址。

因此`*(fp-8) = RA`,  `*(fp-16) = previous fp`

这样我们可以循环遍历`fp`, 并在每步打印出`RA`, 但是怎么样判断循环的结束呢？

首先我们知道，`stack` 是一个`page`,  然后 它的起始地址和结束地址都是`page-aligned` ， 即起始地址和结束地址都是`4096` 的倍数。

给定一个地址`fp`, 函数`PGROUNDDOWN` 可以求出 `fp` 所在`page` 的结束地址，`PGROUNDUP`  求出`fa` 所在`page`的首地址

```c++
void backtrace(){
	uint64 fp  = r_fp();
	uint64 bottom = PGROUNDDOWN(fp);
	while (fp>=bottom){
		printf("%p\n", * ((uint64 *)(fp-8)) );
		fp = * ((uint64 *)(fp-16));
	}
}
```

### task - 3 Alarm

#### test0

- 修改`Makefile` 使得 `alarmtest.c` 可以被编译为`xv6` 用户程序。

```c
// makefile
$U/_alarmtest\
```

- 在`user/user.h` 中声明下面两个函数:

```c
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
```

- 修改`user/usys.pl`, `kernel/syscall.h`, `kernel/syscall.c`

- 在`kernel/sysproc.c` 中添加`sys_sigalarm,sys_sigreturn `

- 在`proc` 结构体里面定义`alarm interval ` 和 `pointer to handler`

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  int ticks;
  uint64 handler;
};
```

- `sys_sigalarm()` 把 上述两个信息（其接收的参数) 写入`proc` 结构体

- 我们需要记录从上一次执行`handler` 到现在所经过的时间，再在`proc` 中加一个变量。

- 在`kernnel/pro.c` 中的函数`allocproc` 中添加新的初始化一个进程的逻辑。

- 修改`trap.c` 使得，每过一个固定 数目的始终周期，就会执行一次`sigalarm`

```c
  if(which_dev == 2){
    if (p->ticks>0){
      p->tick_cnt++;

      if (p->tick_cnt == p->ticks){
      p->tick_cnt = 0;
      p->trapframe->epc = p->handler;
   		}
    }
    yield();
  }
```

- 这样我们就可以通过`test0` 了。

#### test1/2

我们需要在进入`alarm` 之前保存所有的寄存器，这样才能保证`alarm` 结束后

程序仍然从之前被时钟周期打断的位置继续执行。

我们来回顾整个过程：

- 首先，在某个时刻我们会调用`sigalarm`,  这会改变我们的进程`proc`,  我们会去设置`proc` 中的一些变量，包括`ticks, handler`
- 每个了固定的时钟周期，我们会把`proc->trapframe-epc`设置为 `handler`,  因此当时钟周期引发的`trap` 结束后，进程回到了用户态，此时它的`pc` 被设置为了`proc->trapframe-epc`,  因此我们的进程直接跑去执行`handler`了。
- `handler` 执行的最后，它会调用`sigreturn` ，如果`sigreturn` 直接 `return 0` 的话，我们的进程接下来会去执行什么东西呢？

为了让进程顺着其被时钟周期打断的那个断点接着往下执行，我们需要在`sigreturn` 的 `return 0` 之前，把我们真正想回去的地址写进`proc-trapframe` 中去，那么在`sigreturn` 结束后，我们的进程将自己`trapframe` 的数据调出来，并执行下去。除此之外，我们还要避免重复进入`handler`, 也就是说，在我们执行`handler` 的时候，时钟周期还是在增加，如果此时我们的handler还在执行，但是时钟周期由达到了`ticks` 呢？这样可能导致`handler` 还没执行完，然后又被从头执行。



##### 解决方案

我们可以在`proc` 结构体定义一个新的`alarmframe` , 其负责记录我们在调用`handler` 之前的`trapframe`,  也就是我们在`yield` 之前的`trapframe`,  也就是进程在被时钟打断之前的状态。然后我们可以定义一个变量`is_alarming`,  来表示我们的`sigreturn` 执行没有。

- 首先，我们先定义新的`proc`

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  int ticks;
  uint64 handler;
  int tick_cnt;
  int inalarm;
  struct trapframe *alarmframe; // data page for trampoline.S
};
```



- 修改`allocpro` 和 `freeproc`

```c
//allocproc

static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

    if((p->alarmframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }
  

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  p->is_alarming = 0;
  p->tick_cnt = 0;

  return p;
}
```

```c
//freeproc

static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;

    if(p->alarmframe)
    kfree((void*)p->alarmframe);
  p->alarmframe = 0;


  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);

  p->is_alarming = 0;
  p->tick_cnt = 0;
  p->ticks = 0;
  p->handler = 0;
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

- 修改`usertrap`

```c
// trap.c
if(which_dev == 2) {
  if(p->alarm_interval != 0 && ++p->ticks_count == p->alarm_interval && p->is_alarming == 0) {
    memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
    p->trapframe->epc = (uint64)p->alarm_handler;
    p->ticks_count = 0;
    p->is_alarming = 1;
  }
  yield();
}
```

- 修改`sys_sigreturn`

```c
uint64
sys_sigreturn(){
  memmove(myproc()->trapframe, myproc()->alarmframe, sizeof(struct trapframe));
  myproc()->is_alarming = 0;
  return 0;
}
```



