# task - 1 Eliminate allocation from sbrk()

在本任务中，我们需要去修改函数`sys_sbrk()` 使得其只扩大进程的大小，但是并不为其分配内存。本任务比较简单，只需去掉分配内存的部分即可：

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  myproc()->sz+=n;
  return addr;
}
```

# task - 2 Lazy allocation

我们先阐述一下`lazy allocation` 的思想：

首先，如果我们调用`sbrk` ， 这个函数扩张我们的`page table`,  具体来说就是，增大虚拟地址，申请物理地址，构建映射。

但是由于某些情况，这样的方法是低效的，比如说： 程序往往会申请比它们真正需要使用的内存更大的内存；程序往往会提前申请它们的内存，这些内存可能在很多时候都是空闲的。

基于以上种种原因，我们采用`lazy allocation` 策略： 在调用`sbrk()` 时，我们仅仅增大进程内存空间的大小，但是并不实际分配其对应的物理内存。如果此时程序请求了虚拟地址`va`, 而`va` 并没有映射，那么我们就申请一块物理内存，并构建映射即可。

- 我们可以通过判断`r_sause() = 13 or 15` 来判断是否发生`page fault`
- `r_stval()` 返回引起`page fault` 的虚拟地址。

我们去修改`kernel/trap.c` ， 使得其能在发生`page fault` 时，能够申请物理地址，并构建映射。

```c
else if(cause == 13 || cause == 15) {
    // 处理页面错误
    uint64 fault_va = r_stval();  // 产生页面错误的虚拟地址
    char* pa;                     // 分配的物理地址
    if(PGROUNDUP(p->trapframe->sp) - 1 < fault_va && fault_va < p->sz &&
      (pa = kalloc()) != 0) {
        memset(pa, 0, PGSIZE);
        if(mappages(p->pagetable, PGROUNDDOWN(fault_va), PGSIZE, (uint64)pa, PTE_R | PTE_W | PTE_X | PTE_U) != 0) {
          kfree(pa);
          p->killed = 1;
        }
    } else {
      // printf("usertrap(): out of memory!\n");
      p->killed = 1;
    }
  }
```

- 我们需要修改函数`uvmunmap`,  使得其在某一页并没有映射时不`panic`

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
        continue;
      // panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

# task - 3 Lazytests and Usertests

- 首先我们需要处理当`sbrk()` 的参数为负数时， 使用`uvmdealloc` 即可

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;

  struct proc* p = myproc();
  addr = p->sz;
  uint64 sz = p->sz;

  if(n > 0) {
    // lazy allocation
    p->sz += n;
  } else if(sz + n > 0) {
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    p->sz = sz;
  } else {
    return -1;
  }
  return addr;
}
```

- 我们需要修改`fork()`, 使其能够正确运行。

在`fork()` 函数中，我们调用了`uvmcopy`, 这个函数会把父进程的页表和物理地址均复制一份给子进程，因此我们需要处理当页表还不存在时的情况，直接跳过就好了。

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0) //页表不存在
      continue;
      // panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0) //页表存在，但是没映射
      continue;
      // panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

除此之外我们需要修改函数`uvmunmap`, 因为我们会在`uvmdealloc` 中调用它。

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0) 
      continue;

    if((*pte & PTE_V) == 0)  
      continue;

    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

- 最后，我们需要考虑如果我们再使用了`sbrk()` 后，在调用某些系统调用时，需要传入指针参数，而这个地址并没有分配物理内存。

```c
int argaddr(int n, uint64 *ip) // 进行sbrk后继续执行系统调用，进入另一个trap分支
{                              // 会导致虚拟地址为分配内存 所以在这里需要判断一下
  *ip = argraw(n);
  struct proc *p = myproc();
  if (walkaddr(p->pagetable, *ip) == 0) // 如果页表不存在或者未分配物理块则进行分配和映射
  {
    // 如果对应的虚拟地址并没有分配内存时 给他分配并建立映射
    uint64 pa;
    if (*ip > PGROUNDUP(p->trapframe->sp) - 1 && *ip < p->sz &&
        (pa = (uint64)kalloc()) != 0) // 地址是否合法
    {
      memset((void *)pa, 0, PGSIZE);
      // 建立映射
      *ip = PGROUNDDOWN(*ip); // *ip 所在页表的首地址
      if (mappages(p->pagetable, *ip, PGSIZE, pa, PTE_W | PTE_R | PTE_U | PTE_X) != 0)
      {
        kfree((void *)pa);
        return -1;
      }
    }
    else
    {
      return -1;
    }
  }
  return 0;
}
```

