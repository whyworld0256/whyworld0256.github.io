# Lab7

# Uthread: switching between threads

在本任务中，我们要实现用户级线程的上下文切换机制，我们只需在`user/uthread.c` 中完善一些函数即可。

- 首先我们定义线程的上下文结构体

```c
// 用户线程的上下文结构体
struct tcontext {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

- 修改线程结构体，添加上下文字段

```c
struct thread {
  char            stack[STACK_SIZE];  /* the thread's stack */
  int             state;              /* FREE, RUNNING, RUNNABLE */
  struct tcontext context;            /* 用户进程上下文 */
};
```

- 模仿`kernel/swtch.S`, 修改`kernel/uthread_switch.S`

```c
.text

/*
* save the old thread's registers,
* restore the new thread's registers.
*/

.globl thread_switch
thread_switch:
    /* YOUR CODE HERE */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)
    ret    /* return to ra */
```

- 修改函数`thread_scheduler`,  完成线程切换的逻辑

```c
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&current_thread->context);
  } else
    next_thread = 0;
}
```

- 修改`thread_create`

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;                   // 设定函数返回地址
  t->context.sp = (uint64)t->stack + STACK_SIZE;  // 设定栈指针
}
```

# Using threads

为什么用多线程会增加缺失率？

假设现在有两个线程T1和T2，两个线程都走到put函数，且假设两个线程中key%NBUCKET相等，即要插入同一个散列桶中。两个线程同时调用insert(key, value, &table[i],  table[i])，insert是通过头插法实现的。如果先insert的线程还未返回另一个线程就开始insert，那么前面的数据会被覆盖。

解决办法很简单，调用`insert` 之前上锁即可。

定义锁：

```c
pthread_mutex_t lock[NBUCKET] = { PTHREAD_MUTEX_INITIALIZER }; // 每个散列桶一把锁
```

上锁

```c
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&lock[i]);
    // the new is new.
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock[i]);
  }
}
```

# Barrier

什么是`barrier`?

> In [parallel computing](https://en.wikipedia.org/wiki/Parallel_computing), a **barrier** is a type of [synchronization](https://en.wikipedia.org/wiki/Synchronization_(computer_science)) method. A barrier for a group of threads or processes in the source  code means any thread/process must stop at this point and cannot proceed until all other threads/processes reach this barrier.[[1\]](https://en.wikipedia.org/wiki/Barrier_(computer_science)#cite_note-:3-1)
>
> Many collective routines and directive-based parallel languages impose implicit barriers. For example, a parallel *do* loop in [Fortran](https://en.wikipedia.org/wiki/Fortran) with [OpenMP](https://en.wikipedia.org/wiki/OpenMP) will not be allowed to continue on any thread until the last iteration  is completed. This is in case the program relies on the result of the  loop immediately after its completion. In [message passing](https://en.wikipedia.org/wiki/Message_passing), any global communication (such as reduction or scatter) may imply a barrier.
>
> In [concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing), a barrier may be in a *raised* or *lowered state*. The term **latch** is sometimes used to refer to a barrier that starts in the raised state and cannot be re-raised once it is in the lowered state. The term **count-down latch** is sometimes used to refer to a latch that is automatically lowered  once a pre-determined number of threads/processes have arrived.

需要用到的函数

- `pthread_cond_wait(&cond, &mutex)`;  // go to sleep on cond, releasing lock mutex, acquiring upon wake up

- `pthread_cond_broadcast(&cond)`;     // wake up every thread sleeping on cond

```c
static void 
barrier()
{
  // 申请持有锁
  pthread_mutex_lock(&bstate.barrier_mutex);

  bstate.nthread++;
  if(bstate.nthread == nthread) {
    // 所有线程已到达
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } else {
    // 等待其他线程
    // 调用pthread_cond_wait时，mutex必须已经持有
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  // 释放锁
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

我们简单描述一下这个过程：假设我们有两个进程，假设线程`A` 是第一个执行`barrier` 的线程，此时它拿到锁`barried_mutex`, 并接着执行下去。此时`bstate.nthread!=nthread`,  那么进程`A` , 会沉睡，并且释放锁。此时进程`B`,  可能已经在请求锁了，或者还在来的路上，不管怎么样，当`B` 开始请求锁时，这个锁肯定有被`A`释放的时候，那么`B` 拿到这个锁。 但此时`bstate.nthread == nthread`,  那么唤醒睡觉的所有进程，大家又站在了同一起跑线上。