# Copy-on-Write Fork for xv6

### 问题背景

我们知道系统调用`fork()`,  会创建一个新的子进程，并且会把父进程的内存拷贝给子进程。

在`fork()`中， 函数`uvmcopy` 会被调用。

```c
if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
```

而在函数`uvmcopy` 中， 首先对于父进程的每一页，我们找到其对应的物理地址，然后新建一块物理页，把父进程的物理页复制给新建的物理页，最后建立映射。

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
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

因此，如果父进程的内存很大，那么`fork()` 将需要花费大量的时间。除此之外，`exec()` 常常在`fork()` 之后被使用，这又会让子进程丢弃之前在`fork()` 中复制的内存，转而加载其他的内存，这更显得我们在`fork()` 复制的内存有些多余。

综合以上几种因素，我们提出`Copy-on-Write Fork` 的策略。在`fork()` 时，我们先不为子进程分配物理内存，而仅仅为其创建页表，但页表的映射指向父进程的物理地址，也就是说此时子进程和父进程共享同样的物理地址。并且我们还需要把子进程和父进程的`PTE` 均设置为不可写。

当某个进程需要写入某块物理页时，它会引起`page fault`, 此时我们需要一个新的`handler`,  它可以把物理地址复制一份，修改引起`page fault` 的进程 的`PTE`,  使其指向新复制的物理页，并且我们还要同时修改子进程和父进程的`PTE`  为可写。

### 实验步骤

- 修改`uvmcopy` 使得其不为子进程分配物理页，仅仅只是把子进程的虚拟页和父进程的物理页映射，除此之外我们还需把子进程和父进程的`PTE` 修改为不可写。

我们先考虑把`uvmcopy` 创建物理页的部分删去。

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
	pte_t *pte;
	uint64 pa, i;
	uint flags;
	
	for(i = 0; i < sz; i += PGSIZE){
		if((pte = walk(old, i, 0)) == 0)
			panic("uvmcopy: pte should exist");
		if((*pte & PTE_V) == 0)
			panic("uvmcopy: page not present");
		pa = PTE2PA(*pte);
		flags = PTE_FLAGS(*pte);

		if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
			goto err;
		}
	}
	return 0;
	
	err:
		uvmunmap(new, 0, i / PGSIZE, 1);
		return -1;
}
```

加上把父进程和子进程的`PTE` 设置为不可写的逻辑。

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
	pte_t *pte;
	uint64 pa, i;
	uint flags;
	
	for(i = 0; i < sz; i += PGSIZE){
		if((pte = walk(old, i, 0)) == 0)
			panic("uvmcopy: pte should exist");
		if((*pte & PTE_V) == 0)
			panic("uvmcopy: page not present");
		if(*pte & PTE_W){
			*pte = *pte&(~PTE_W);
		}
		
		pa = PTE2PA(*pte);
		flags = PTE_FLAGS(*pte);

		if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
			goto err;
		}
	}
	return 0;
	
	err:
		uvmunmap(new, 0, i / PGSIZE, 1);
		return -1;
}
```

- 接下来，我们考虑写入的问题。如果此时父进程需要写入虚拟地址，而这个虚拟地址对应的物理地址是被子进程所共享的，那么我们需要把这块物理地址复制一份，然后修改子进程的`PTE`, 使其映射到当前这块物理页，并且我们还需修改父进程和子进程为可写。

那么问题是，当发生`page fault` 时，我们怎么知道这块虚拟地址是子进程和父进程共享的？我们可以在`uvmcopy` 的时候，为父进程和子进程的`PTE` 打上一个标记。因此我们在`kernenl/riscv.h` 中加入：

```c
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access

#define PTE_F (1L << 8) // copy on write
```

并修改`uvmcopy`

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
	pte_t *pte;
	uint64 pa, i;
	uint flags;
	
	for(i = 0; i < sz; i += PGSIZE){
		if((pte = walk(old, i, 0)) == 0)
			panic("uvmcopy: pte should exist");
		if((*pte & PTE_V) == 0)
			panic("uvmcopy: page not present");
		if(*pte & PTE_W){
			*pte = *pte&(~PTE_W);
		}
		
		*pte = *pte | PTR_F;
		
		pa = PTE2PA(*pte);
		flags = PTE_FLAGS(*pte);

		if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
			goto err;
		}
	}
	return 0;
	
	err:
		uvmunmap(new, 0, i / PGSIZE, 1);
		return -1;
}
```

- 我们还需要考虑如果我们需要释放某块物理地址该怎么操作。

由于内存是共享的，如果此时父进程请求释放这块内存，我们只能解除其与这块内存的映射，而不能真正释放物理页。当没有任何一个进程需要这块内存时，我们便将其释放。因此我们需要定义一个全局数组，用来统计当前映射了这块物理页的进程数量。

```c
// kernel/kalloc.c
struct ref_stru {
  struct spinlock lock;
  int cnt[PHYSTOP / PGSIZE];  // 引用计数
} ref;
```

- 在`kinit` 中初始化 `ref` 的自旋锁

```c
// kernel/kalloc.c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&ref.lock, "ref");
  freerange(end, (void*)PHYSTOP);
}
```

- 修改`kfree`,  当请求释放某块物理页时，我们现在数组`cnt` 中对其减1，如果减为了0，我们将其释放。

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&ref.lock);
  if(--ref.cnt[(uint64)pa/PGSIZE] == 0){
    release(&ref.lock);
    r = (struct run*)pa;
    memset(pa, 1, PGSIZE);
    acquire(&kmem.lock);
    r->next = keme.freelist;
    keme.freelist = r;
    release(&keme.lock);
  }
  else{
    release(&ref.lock);
  }
```

- 修改`kalloc`, 分配一页物理内存时，初始化其`cnt=1`.

```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    acquire(&ref.lock);
    ref.cnt[(uint64)r / PGSIZE] = 1;  // 将引用计数初始化为1
    release(&ref.lock);
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

- 下面我们定义一些辅助函数。
- `cowpage`: 判断一个给定的虚拟地址所在的页是否为共享页

```c
int checkpage(pagetable_t pagetable, uint64 va) {
  if(va >= MAXVA)
    return -1;
  pte_t* pte = walk(pagetable, va, 0);
  if(pte == 0)
    return -1;
  if((*pte & PTE_V) == 0)
    return -1;
  return (*pte & PTE_F ? 0 : -1);
}
```

- `krefcnt`: 返回物理地址对应页表的`cnt`

```c
int krefcnt(void* pa) {
  return ref.cnt[(uint64)pa / PGSIZE];
}
```

- `kaddrefcnt`: 增加物理地址对应页表的`cnt`

```c
int kaddrefcnt(void* pa) {
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    return -1;
  acquire(&ref.lock);
  ++ref.cnt[(uint64)pa / PGSIZE];
  release(&ref.lock);
  return 0;
}
```

- `cowcopy`: 当发生`page fault` 时，如果当前这块物理地址的`cnt=1`,  那么我们直接清除它的`PTE_w`， 如果`cnt!=1`, 我们需要新申请一块物理页并构建映射，不要忘记清除`PTE_W` 标记。

```c
void* cowalloc(pagetable_t pagetable,uint64 va){
	if (va%PGSIZE!=0)
		return 0;
	
	uint64 pa = walkaddr(pagetable,va);
	if(pa==0) return 0;
	
	pre_t* pte = walk(pagetable,va,0);
	if (krefcnt((char *)pa) == 1){
		*pte|=PTE_W;
		*pte&=~PTE_F;
		reutrn (void*)pa;
	}
	
	else{
		char* mem = kalloc();
		if(mem==0) return 0;
		memmove(mem,(char*)pa,PGSIZE);
		
		*pte &=~PTE_V;
		
		if(mappages(pagetable,va,PGSIZE,(uint64)mem,(PTE_FLAGS(*pte) | PTE_W) & ~PTE_F) != 0){
			kfree(mem);
			*pte |= PTE_V;
			return 0;
		}
		
		kfree((char*)PGROUNDDOWN(pa));
		return mem;
		
	}
	
}
```

- 再次修改`uvmcopy`,  我们需要给每个物理页的`cnt`+1

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    // 仅对可写页面设置COW标记
    if(flags & PTE_W) {
      // 禁用写并设置COW Fork标记
      flags = (flags | PTE_F) & ~PTE_W;
      *pte = PA2PTE(pa) | flags;
    }

    if(mappages(new, i, PGSIZE, pa, flags) != 0) {
      uvmunmap(new, 0, i / PGSIZE, 1);
      return -1;
    }
    // 增加内存的引用计数
    kaddrefcnt((char*)pa);
  }
  return 0;
}
```

- 修改`uertrap` 使其能够处理`page fault`

```c
// kernel/trap.c
  else if(cause == 13 || cause == 15) {
  uint64 fault_va = r_stval();  // 获取出错的虚拟地址
  if(fault_va >= p->sz
    || cowpage(p->pagetable, fault_va) != 0
    || cowalloc(p->pagetable, PGROUNDDOWN(fault_va)) == 0)
    p->killed = 1;
} 
```

- 修改`copyout`

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(cowpage(pagetable, va0) == 0) {
    // 更换目标物理地址
    pa0 = (uint64)cowalloc(pagetable, va0);
  }
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

