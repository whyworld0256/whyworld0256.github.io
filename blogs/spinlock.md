# spinlock

一个`spinlock` 结构体的内容如下：

```c
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
```

其中，`locked = 0` 表示这个锁处于未上锁状态， `locked = 1` 表示这个锁

处于上锁状态， `name` 和 `cpu` 主要用于debug。

# 重要的函数

- `acquire`
- `release`
- `initlock`
- `holding`

# acquire

当一个进程想要给某个东西上锁时，此时这个锁应该是未上锁的，然后这个进程拿到锁，随后执行自己的代码。

```c
void acquire(struct spinlock *lk){
	for(;;){
	if (lk->locked == 0){
	lk->locked = 1;
		break;
	}
	}
}
```

上面这段代码反应了`acquire` 的基本思想，当我们调用`acquire` 时， 进程会不断去check当前的`locked` 是否为0，也就是当前的锁是否被释放。如果没有被释放，那么这个进程就会陷入无限循环，仿佛就在等待一样。一旦这个`lockde` 等于0，这个进程马上把`locked` 赋值为1并且结束循环，开始执行`acquire` 后面的代码，此时如果其他进程也调用了`acquire`,  那么它就会等待当前进程释放锁。

但是我们会发现这个`acquire` 函数会有一个bug, 由于进程是并发执行的，我们可以想象一个场景： 此时锁是释放的，一个进程执行了第3行代码，在其还没有执行第4行代码时，另外一个进程跑去执行了第3行代码，此时在两个进程的视角来看，它们都以为自己拿到了这个锁，这就导致了问题。

因此，我们不允许一个进程在执行第三行代码和第四行代码之间发生中断，也就是说这两步操作必须连贯执行。

解决方法：

```c
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```

什么是`__sync_lock_test_and_set(&lk->locked, 1)`?

这个函数的作用是交换`lk->locked` 的值 和 1,  返回`lk->locked` 在未交换之前的值。

如果当前`lk->locked` 的值为1，我们就不断往`lk->locked` 写入1，这并不影响其值。一旦拥有这个锁的进程释放了它，当前进程会马上结束循环，并拿到这个锁。

最重要的是，函数`__sync_lock_test_and_set(&lk->locked, 1)` 是atomic的，也就是说它的执行过程不会被中断。因此如果有多个进程想要这个锁，有且仅有一个进程会把1写入`lk->locked`, 然后从这一时刻开始，在所有其他进程的视角里面，这个锁已经被占据了。

综上，我们很好地解决了问题。

`__sync_synchronize()` 是在告诉编译器不要把一些指令提前执行了，有些时候编译器为了优化性能，其会调整指令的执行顺序，这些都是在程序员不知情的情况下执行地，但是我们在没有拿到锁之前是坚决不能动地，所以这个函数很有必要。

# release

```c
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```

`__sync_lock_release` 把0 写入`lk->locked`, 这也是个原语，这个过程不会被中断。

# Lock and Interrupts

有些时候，进程和`interrupt handler` 会请求同一个锁。例如：在系统调用`sys_sleep` 中，它会请求`tickslock`这个锁，如果此时发生`timer interrupt` ,  对应的`handler` 为`clockintr` ， 其也会请求`tickslock` 这个锁，但这个锁被`sys_sleep` 持有，那么`clockintr` 会一直卡在`acquire` ，并且如果`clockintr` 不返回, `sys_sleep` 也无法执行下去，也就无法释放这个锁，此时便发生了死锁。

因此，我们可以得到一个结论：如果一个`interruct handler` 会需要某个锁，那么当一个CPU拿到这个锁后，它必须禁止中断的发生。

在这方面，`xv6` 更加的保守，当CPU获得任何一个锁时，它总是会禁止中断的发生。

那么此时我们不得不考虑嵌套`acquire` 的情况了，每`acquire` 一下，我们禁止中断的次数需要加1，每`release` 的时候，我们减1，知道所有的锁都被释放，我们才允许中断的发生。

# push_off

```c
void
push_off(void)
{
  int old = intr_get();

  intr_off();
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}
```

首先，每个CPU有一个字段`noff`, 代表`acquire` 嵌套的层数，`intena` 表示中断在`push_off`之前是否被禁止，也就是旧状态。 

`intr_off` 是关闭中断的意思，如果`noff = 0`, 也就是说当前遇到第一个`acquire`,  那么我们先保存旧状态，然后`noff++`.

# pop_off

```c
void
pop_off(void)
{
  struct cpu *c = mycpu();
  if(intr_get())
    panic("pop_off - interruptible");
  if(c->noff < 1)
    panic("pop_off");
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}
```



