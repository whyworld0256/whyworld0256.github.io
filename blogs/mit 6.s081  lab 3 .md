# task 1 - Print a page table

任务要求： 在我们开机时，在屏幕以下面的形式上打印出页表。

### 解答

```c
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

- `freewalk`,  我们先来研究以下`freewalk`函数。

```c
// kernel/vm.c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.

void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

可以看出，`freewalk` 做的就是递归地遍历页表，知道走到叶子节点(注意：使用`freewalk` 时，我们需要提前解除虚拟地址和物理地址的映射，否则会PTE)， 然后释放页表的空间。

```c
void
_vmprint(pagetable_t pagetable)
{
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V)){
        for(int j=0,j<=level,j++){
            if (j) printf("%s", " ");
            print("%s","..");
        }
      uint64 child = PTE2PA(pte);
       printf("%d: pte %p pa %p\n", i, pte, child);
    } if((pte & (PTE_R|PTE_W|PTE_X)) == 0){ //除了最后一层，我们都递归
        _vmprint((pagetable_t)child, level + 1); //递归
      }
  }
}
void
vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  _vmprint(pagetable, 0);
}
```

然后我们需要在`kernel/defs/h` 中声明:

```c
// kernel/defs.h
//vm.c
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
void           vmprint(pagetable_t pagetable); // 添加函数声明
```

最后把这个函数出入到`exec.c`中，使得`exec`在返回前，会执行这个函数

```c
// exec.c

int
exec(char *path, char **argv)
{
  // ......

  vmprint(p->pagetable); // 按照实验要求，在 exec 返回之前打印一下页表。
  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```

# task 2 - A Kernel page table per process

### 题目背景

`Xv6` 有一个被所有进程共享的`kernel page table`,  每个进程有一个属于自己的`user page table`  .  其中`kernle page table` 的映射与 `user page table` 的映射不同。

现在考虑一种情况： 当一个进程调用`system call` 的时候，其需要传入一个指针参数，现在这个指针参数被传递给了`kernel`,  但是`kernel`在使用这个指针的时候会存在一定的问题。在`user space` 中，这个指针或者说地址通过页表直接被翻译为物理地址，但是`kernel` 并没有`user space`的页表，因此单独凭借`kernel`的能力，它是无法获取这个指针的引用的。因此`kernel` 需要使用`user`的`page table` 去完成地址的翻译(可以查看函数`copyin`)。被任务即是要求我们为每个进程创建一个独属于其的`kernel page table`, 并且当这个进程进入`supervivor mode` 时， 不是使用真正的`kerel page table`, 而是使用我们为其创建的那个。在下一个任务中，我们会把`user page table` 映射到其对应的 `kernel page table` , 从而达到最终目的。

### 解答

我们按照官方的提示一步一步来。

-  首先，我们需要在进程结构体`proc` 加入我们创建的新的页表, 我们将其命名为`kernelpt`.

```c
//kernel/proc.h

// Per-process state
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
  pagetable_t kernelpt;      // 进程的内核页表
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

- 现在我们需要写一个函数来完成`kernelpt` 的创建和初始化。在`kernel/vm.c` 中， 函数`kvminit` 实现了对全局`kernel page table` 的初始化，其中`kvmmap`实现了对各个区域的映射。 函数`uvmcreate` 的作用为创建一个空的页表。

模仿上述的这些函数，我们可以写出函数`uvmmap`, 和 `proc_kpt_init` ：

```c
void
uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pagetable, va, sz, pa, perm) != 0)
    panic("uvmmap");
}

// Create a kernel page table for the process
pagetable_t
proc_kpt_init(){
  pagetable_t kernelpt = uvmcreate();
  if (kernelpt == 0) return 0;
  uvmmap(kernelpt, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  uvmmap(kernelpt, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  uvmmap(kernelpt, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  uvmmap(kernelpt, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  uvmmap(kernelpt, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return kernelpt;
}
```

在文件`kernel/proc.c`中， 函数`allocproc` 实现了创建一个新的进程，此时一个进程里面的各个元素被创建和初始化，因此我们需要添加创建`kernelpt` 的部分。

```c
//kernel/proc.c
// Init the kernal page table
p->kernelpt = proc_kpt_init();
if(p->kernelpt == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}
```

- 在本步中，我们需要创建此进程的`kernel stack	`, 并将其与`kernelpt` 的对应位置相映射，具体可查看`kernel virtual space` 的那张图。我们将`kernel/proc.c` 中函数`procinit` 的对应部分移到`allocproc` 中即可。

```c
//kernel/proc.c

// Allocate a page for the process's kernel stack.
// Map it high in memory, followed by an invalid
// guard page.
char *pa = kalloc();
if(pa == 0)
  panic("kalloc");
uint64 va = KSTACK((int) (p - proc));
uvmmap(p->kernelpt, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
p->kstack = va;
```

- 在本步中，我们需要修改`scheduler()`, 使得当本进程`trap` 时，写入寄存器`satp` 的是`kernelpt` 而不是原来的`kernel page table` , 并且在没有进程运行时，`satp` 中的应该是`kernel page table`.

在文件`kernel/vm.c` 中，函数`kvminithart` 实现的功能正是把`kernel_pagetable` 写入寄存器`satp`, 我们模仿即可。

```c
//kernel/vm.c

// Store kernel page table to SATP register
void
proc_inithart(pagetable_t kpt){
  w_satp(MAKE_SATP(kpt));
  sfence_vma();
}
```

然后修改`scheduler()`:

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        proc_inithart(p->kernelpt);
        swtch(&c->context, &p->context);
        kvminithart();

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

- 在本步中，我们要在函数`freeproc` 中加上部分代码，去释放`kernelpt` 所带来的内存，但不释放其对应的物理内存。

首先我们要清理为其分配的`kernel stack`, 其次按照函数`freewalk` 去释放整个页表。

```c
// kernel/proc.c

static void
freeproc(struct proc *p)
{
  uvmunmap(p->kernelpt, p->kstack, 1, 1);
  p->kstack = 0;
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  if (p->kernelpt)
    proc_freekernelpt(p->kernelpt);
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

其中`proc_freekernelpt` 为：

```c
// kernel/vm.c
void
proc_freekernelpt(pagetable_t kernelpt)
{
  // similar to the freewalk method
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = kernelpt[i];
    if(pte & PTE_V){
      kernelpt[i] = 0;
      if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        uint64 child = PTE2PA(pte);
        proc_freekernelpt((pagetable_t)child);
      }
    }
  }
  kfree((void*)kernelpt);
}
```

- 最后一步，把上述我们定义的所有函数都加到`kernel/defs.h` 中去。

# task 3 - Simplify `copyin/copyinstr`

任务要求：在上一个任务中，我们已经为每一个进程分配了一个属于它自己的`kernelpt`, 接下来，我们需要将我们的`user page table` 映射给`kernelpt`, 并且改善`system call` ： `copyin/copyinstr`,  使得我们的`kernel`, 无需通过`user page table` 去引用指针，而是直接使用`kernelpt`.

###  解答

- 首先，我们需要实现将`user page table` 映射给`kernelpt`, 也就是说： 如果在`user page table` 当中， 虚拟地址`va` 对应的物理地址为`pa`, 那么在`kernelpt` 中，虚拟地址`va` 也应该对应`pa`.  方法比较简单，我们任给一个虚拟地址`va`, 使用`walk` 函数找到其`PTE`, 在`user page table` 中， 我们使用`PTE` 翻译出物理地址， 然后赋给`kernelpt` 的 `PTE`, 除此之外，我们还需注意权限的问题。

```c
// kernel/vm.c

void
u2kvmcopy(pagetable_t pagetable, pagetable_t kernelpt, uint64 oldsz, uint64 newsz){
  pte_t *pte_from, *pte_to;
  oldsz = PGROUNDUP(oldsz);
  for (uint64 i = oldsz; i < newsz; i += PGSIZE){
    if((pte_from = walk(pagetable, i, 0)) == 0)
      panic("u2kvmcopy: src pte does not exist");
    if((pte_to = walk(kernelpt, i, 1)) == 0)
      panic("u2kvmcopy: pte walk failed");
    uint64 pa = PTE2PA(*pte_from);
    uint flags = (PTE_FLAGS(*pte_from)) & (~PTE_U);
    *pte_to = PA2PTE(pa) | flags;
  }
}
```

- 接下来，当我们的`user address space` 的映射关系发生改变时，我们需要将这些改变复制给`kernelpt`,  这种情况可能发生在函数`fork(), exec(), sbrk()` 中。
- `fork()`

```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  u2kvmcopy(np->pagetable, np->kernelpt, 0, np->sz); //here!!

  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}

```

- `exec()`

```c
// kernel/exec.c

int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  u2kvmcopy(pagetable, p->kernelpt, 0, sz); //here!!


  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);
  vmprint(p->pagetable);

  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```

- `sbrk()`

根据教材上的`figure 3.3 , figure 3.4`, 我们可以知道，`user address space` 的范围为$ 0 \sim MAXVA$,  也就是说`kernelpt` 的  $0 \sim MAXVA$ 都会被我们占用，但是有可能这会破坏`kernelpt`原本的空间，因此题目要求我们用户空间不能超过地址` 0xC000000`

```c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    // 加上PLIC限制
    if (PGROUNDUP(sz + n) >= PLIC){
      return -1;
    }
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    // 复制一份到内核页表
    u2kvmcopy(p->pagetable, p->kernelpt, sz - n, sz);
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
```

- 修改`userinit`

```c
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  u2kvmcopy(p->pagetable, p->kernelpt, 0, p->sz); // here!!


  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}

```

