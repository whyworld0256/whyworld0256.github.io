# Lab 8

## task - 1 Memory allocator

### 问题背景

首先我们知道，在`XV6		` 中，所有空闲的物理内存都以页为单位组织成一个链表，每当我们申请内存时，就从这个链表的头部拿一页出来，每当有物理内存被释放时，我们就把这一页插入到物理内存当中去。为了避免竞争，这个链表是被一个锁保护起来的，也就是说，当我们尝试去修改这个链表时，我们必须先拿到这个锁，因此尽管是多核`CPU`,  多个进程可以并行进行，但是某一时刻仅有一个进程可以修改这个链表。

现在我们可以重新设计这个策略。考虑为每个`CPU` 分配一部分物理内存，它们还是形成一个链表，并配有一个锁。那么在不同`CPU`上分配内存和释放内存的行为可以并行执行，因为它们不会受到同一个锁的限制，这样会大大提高机器的性能。但是，如果某个`CPU` 的空闲内存耗尽了，但总的空闲内存还很充足呢？我们可以考虑从别的`CPU` 的空闲内存那里去偷一些过来为自己所用，此时我们就不得不面临锁带来的限制了。

### 解决方案

- 首先，我们先定义一个全局数组，每个位置是一个链表和配备的锁，数组长度为`CPU` 的个数。

```c
// kernel/kalloc.c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
```

- 修改`kinit` 函数，使其可以为每个锁配备一个名字，最后释放所有的可用物理地址。

```c
// kernel/kalloc.c
void
kinit()
{
  char lockname[8];
  for(int i = 0;i < NCPU; i++) {
    snprintf(lockname, sizeof(lockname), "kmem_%d", i);
    initlock(&kmem[i].lock, lockname);
  }
  freerange(end, (void*)PHYSTOP);
}
```

- 修改`kfree`,  使得当前释放的物理页被插入到了当前`CPU` 的空闲物理页链表当中。

官方提醒我们要加上`push_off`,  `pop_off`

> The function `cpuid` returns the current core number, but     it's only safe to call it and use its result when    interrupts are turned off. You should use    `push_off()` and `pop_off()` to turn    interrupts off and on.    

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();  // 关中断
  int id = cpuid();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
  pop_off();  //开中断
}
```

- 修改`kalloc` 函数，我们先判断当前`CPU` 的空闲物理页链表是否为空，如果不为空就分配，如果为空，那么我们遍历所有的`CPU`, 如果谁的空闲物理页链表是空的，我们就分配。

```c
// kernel/kalloc.c
void *
kalloc(void)
{
  struct run *r;

  push_off();// 关中断
  int id = cpuid();
  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  else {
    int antid;  // another id
    // 遍历所有CPU的空闲列表
    for(antid = 0; antid < NCPU; ++antid) {
      if(antid == id)
        continue;
      acquire(&kmem[antid].lock);
      r = kmem[antid].freelist;
      if(r) {
        kmem[antid].freelist = r->next;
        release(&kmem[antid].lock);
        break;
      }
      release(&kmem[antid].lock);
    }
  }
  release(&kmem[id].lock);
  pop_off();  //开中断

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## task -2 Buffer cache

### 问题背景

首先在内存中有很多预先分配的`buf`块，每一块`buf` 可以缓存磁盘中的一个`block`.  所有的`buf` 组成了一个结构体`bcache`,  其用双链表的形式组织起了所有的`buf`.  用双链表的好处就在于我们可以很好地采用`LRU` 算法： 最近常用的`buf` 被组织到了链表的前端，最不常用的组织到了链表的尾端。如果某块`block` 已经被缓存，我们想在`bcache` 中找一块与之对应的`buf`,  我们从头往后找; 如果这块`block` 没被缓存，我们从后往前找，找到一块空闲的`buf`,  并用它来缓存此`block`.

串联好空闲的 buf 之后，开始正式的 I/O 流程。因为 xv6 是刚启动，所以 Buffer cache 没有从 disk 中拷贝过数据。此时来了进程 A ，带来了读 disk block0 的请求。进程 A 发出读请求后，就将自己切入至休眠状态了（主动让出 CPU ）。xv6 接收到读请求后，将 disk block0 中的数据拷贝到 Buffer cache 中，然后再通知正在休眠的进程 A （其需要的 I/O 条件已满足，可以继续运行）

上面提到的情形，是只有一个进程读 disk block0。现实中，肯定远远不止有一个进程，可能同时有一百个进程都想读 or 写 disk block0。我们知道，多个线程同时操纵一块内存时，需要通过锁机制来确保读写的有序进行，不然会出错。同样，在处理 Buffer cache 问题上也是利用这种手段.

xv6 帮我们实现好了一个简易的版本，Buffer cache 只是一块很大的内存空间，为了确保多进程之间有序的读写，xv6 为 Buffer cache 配备了一个 lock 。这已经可以实现有序操作了，但还是存在问题的，比如进程 A 只想修改 disk block0 的那一小块内容，但却要对整块 Buffer cache 上锁（假设 Buffer cache 可以装下100块 block ），这太得不偿失了

试想，若此时还有一个进程 B ，它仅仅是想读 disk block2 的数据。本来无需考虑除 disk block2 外的情况，结果却因为进程 A 对整个 Buffer cache 上锁了，被迫需要等待，等进程 A 放锁之后才能读取。这就是问题所在！专业术语，就是锁的颗粒度很大，需要优化

所以，这个子实验的目的很明确，就是解决上述的问题。针对进程 A 的请求可以只对 disk block0 在 Buffer cache 的映射区域上锁，而不是选择对一整块 Buffer cache 上锁，这样的话，也不会影响进程 B 的 disk block2 的请求

说的专业一点，就是将一块非常大的 Buffer cache 分而治之，划分成众多小区域，针对小区域进行上锁，而不是动不动就一整块上锁。这样可以明显提高并行效率，这种手段术语叫减小锁的颗粒度.

### 解决办法

我们更换`bcache`的组织方式，不采用双链表的方式，而是采用哈希表的方式。

为了最大可能地减少冲突率，我们一般取哈希函数中的模数为一个素数，比如为`13`.

那么`bcache` 就有`13`个哈希桶，对于磁盘中编号为`blocknu` 的块，它应该被属于`bcache[blocknu%13]`的`buf` 所缓存。

此时，我们会去`bcache[blocknu]` 上面挂载的所有`buf` 中找一个存在时间最久的那块，然后让其去缓存当前这块`block`. 

但是我们需要考虑当前`bcache[blocknu]` 上挂载的`buf`不够的情况， 那么就应该去其他桶里去借了。

- 定义新的组织结构

```c
#define NBUCKET 13
#define HASH(id) (id % NBUCKET)

struct hashbuf {
  struct buf head;       // 头节点
  struct spinlock lock;  // 锁
};

struct {
  struct buf buf[NBUF];
  struct hashbuf buckets[NBUCKET];  // 散列桶
} bcache;
```

- 初始化：初始化每个锁，把所有的`buf`都挂在到`bcache.buckets[0]` 上去。

```c
void
binit(void) {
  struct buf* b;
  char lockname[16];

  for(int i = 0; i < NBUCKET; ++i) {
    // 初始化散列桶的自旋锁
    snprintf(lockname, sizeof(lockname), "bcache_%d", i);
    initlock(&bcache.buckets[i].lock, lockname);

    // 初始化散列桶的头节点
    bcache.buckets[i].head.prev = &bcache.buckets[i].head;
    bcache.buckets[i].head.next = &bcache.buckets[i].head;
  }

  // Create linked list of buffers
  for(b = bcache.buf; b < bcache.buf + NBUF; b++) {
    // 利用头插法初始化缓冲区列表,全部放到散列桶0上
    b->next = bcache.buckets[0].head.next;
    b->prev = &bcache.buckets[0].head;
    initsleeplock(&b->lock, "buffer");
    bcache.buckets[0].head.next->prev = b;
    bcache.buckets[0].head.next = b;
  }
}
```

- 向`buf`结构体，添加新字段`timestamp` 记录当前`buf`的时间戳，时间戳越小，表示这块`buf`存在的时间越长，也就应该最先被替换。

```c
struct buf {
  ...
  ...
  uint timestamp;  // 时间戳
};
```

- 修改`brelse`, 本来我们需要获取整个`bcache`的锁，现在只需要对应哈希桶的锁，然后更改时间戳。

```c
void
brelse(struct buf* b) {
  if(!holdingsleep(&b->lock))
    panic("brelse");

  int bid = HASH(b->blockno);

  releasesleep(&b->lock);

  acquire(&bcache.buckets[bid].lock);
  b->refcnt--;

  // 更新时间戳
  // 由于LRU改为使用时间戳判定，不再需要头插法
  acquire(&tickslock);
  b->timestamp = ticks;
  release(&tickslock);

  release(&bcache.buckets[bid].lock);
}
```

- 修改`bget`

```c
static struct buf*
bget(uint dev, uint blockno) {
  struct buf* b;

  int bid = HASH(blockno);
  acquire(&bcache.buckets[bid].lock);

  // Is the block already cached?
  for(b = bcache.buckets[bid].head.next; b != &bcache.buckets[bid].head; b = b->next) {
    if(b->dev == dev && b->blockno == blockno) {
      b->refcnt++;

      // 记录使用时间戳
      acquire(&tickslock);
      b->timestamp = ticks;
      release(&tickslock);

      release(&bcache.buckets[bid].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  b = 0;
  struct buf* tmp;

  // Recycle the least recently used (LRU) unused buffer.
  // 从当前散列桶开始查找
  for(int i = bid, cycle = 0; cycle != NBUCKET; i = (i + 1) % NBUCKET) {
    ++cycle;
    // 如果遍历到当前散列桶，则不重新获取锁
    if(i != bid) {
      if(!holding(&bcache.buckets[i].lock))
        acquire(&bcache.buckets[i].lock);
      else
        continue;
    }

    for(tmp = bcache.buckets[i].head.next; tmp != &bcache.buckets[i].head; tmp = tmp->next)
      // 使用时间戳进行LRU算法，而不是根据结点在链表中的位置
      if(tmp->refcnt == 0 && (b == 0 || tmp->timestamp < b->timestamp))
        b = tmp;

    if(b) {
      // 如果是从其他散列桶窃取的，则将其以头插法插入到当前桶
      if(i != bid) {
        b->next->prev = b->prev;
        b->prev->next = b->next;
        release(&bcache.buckets[i].lock);

        b->next = bcache.buckets[bid].head.next;
        b->prev = &bcache.buckets[bid].head;
        bcache.buckets[bid].head.next->prev = b;
        bcache.buckets[bid].head.next = b;
      }

      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;

      acquire(&tickslock);
      b->timestamp = ticks;
      release(&tickslock);

      release(&bcache.buckets[bid].lock);
      acquiresleep(&b->lock);
      return b;
    } else {
      // 在当前散列桶中未找到，则直接释放锁
      if(i != bid)
        release(&bcache.buckets[i].lock);
    }
  }

  panic("bget: no buffers");
}
```

- 修改函数`bpin`, `bunpin` ,  修改锁的逻辑即可。

```c
void
bpin(struct buf* b) {
  int bid = HASH(b->blockno);
  acquire(&bcache.buckets[bid].lock);
  b->refcnt++;
  release(&bcache.buckets[bid].lock);
}

void
bunpin(struct buf* b) {
  int bid = HASH(b->blockno);
  acquire(&bcache.buckets[bid].lock);
  b->refcnt--;
  release(&bcache.buckets[bid].lock);
}
```







 

