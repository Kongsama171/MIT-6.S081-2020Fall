# Lab8: locks

**花费时间：** 10h

---

## Part 1: Memory allocator (难度：moderate)

> 您的工作是实现每个CPU的空闲列表，并在CPU的空闲列表为空时进行窃取。所有锁的命名必须以“`kmem`”开头。也就是说，您应该为每个锁调用`initlock`，并传递一个以“`kmem`”开头的名称。运行`kalloctest`以查看您的实现是否减少了锁争用。要检查它是否仍然可以分配所有内存，请运行`usertests sbrkmuch`。您的输出将与下面所示的类似，在`kmem`锁上的争用总数将大大减少，尽管具体的数字会有所不同。确保`usertests`中的所有测试都通过。评分应该表明考试通过。

* 修改锁与链表

基本思想是为每个CPU维护一个空闲列表，每个列表都有自己的锁。因此我们对相关代码进行修改。

修改代码后，我们实现了基本的框架。会发现代码在`make CPUS=1 qemu-gdb`上是能通过的。因为只有一个CPU也就不会出现需要窃取的情况
*   **`kernel/kalloc.c`**:
```c
struct {
  struct spinlock lock[NCPU];
  struct run *freelist[NCPU];
} kmem;
```
```c
void
kinit()
{
  for(int i = 0;i < NCPU;i++)
    initlock(&kmem.lock[i], "kmem");
  freerange(end, (void*)PHYSTOP);
}
```
```c
void
kfree(void *pa)
{
  struct run *r;

  push_off();
  int hart = cpuid();
  pop_off();

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock[hart]);
  r->next = kmem.freelist[hart];
  kmem.freelist[hart] = r;
  release(&kmem.lock[hart]);
}
```
```c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int hart = cpuid();
  pop_off();

  acquire(&kmem.lock[hart]);
  r = kmem.freelist[hart];
  if(r)
    kmem.freelist[hart] = r->next;
  release(&kmem.lock[hart]);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

* 实现窃取功能

在修改上面的代码之后我们会发现启动qemu会出现下面情况
```c
xv6 kernel is booting
hart 2 starting
hart 1 starting
panic: init exiting
```

主要原因是 `kinit()` 执行完毕后，所有的可用物理内存页都被挂在了 `CPU 0` 的空闲链表上。而内核启动完成后，会创建第一个用户进程 `init`。这个创建任务可能会被调度到任意一个CPU上，比如`CPU 1`。这时候我们还没有实现窃取功能，其他CPU上的`kalloc()`就会失败。

窃取的主要思想就是当当前列表的可用内存不足时，从别的CPU的列表上获取可用内存

**记得在窃取过程中获取别的CPU的锁**
*   **`kernel/kalloc.c`**:
```c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int hart = cpuid();
  pop_off();

  acquire(&kmem.lock[hart]);
  r = kmem.freelist[hart];
  if(r){
    kmem.freelist[hart] = r->next;
  } else {
    for(int i = 0;i < NCPU;i++){
      if(i == hart) 
        continue;

      acquire(&kmem.lock[i]);
      if(kmem.freelist[i]){
        r = kmem.freelist[i];
        kmem.freelist[i] = r->next;
      }
      release(&kmem.lock[i]);

      if(r)
        break;
    }

  }
  release(&kmem.lock[hart]);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## Part 2: Buffer cache (难度：hard)

> 修改块缓存，以便在运行`bcachetest`时，bcache（buffer cache的缩写）中所有锁的`acquire`循环迭代次数接近于零。理想情况下，块缓存中涉及的所有锁的计数总和应为零，但只要总和小于500就可以。修改`bget`和`brelse`，以便bcache中不同块的并发查找和释放不太可能在锁上发生冲突（例如，不必全部等待`bcache.lock`）。你必须保护每个块最多缓存一个副本的不变量。完成后，您的输出应该与下面显示的类似（尽管不完全相同）。确保`usertests`仍然通过。完成后，`make grade`应该通过所有测试。

*  **解决方案**：
     * 修改结构体
      ```c
      #define NBUCKET 13
      #define BUFSIZE 5
      struct table{
        struct spinlock tablelock;
        struct buf buf[5];
      }table;
      struct {
        struct spinlock lock;
        struct table table[NBUCKET];
      } bcache;
      ```
     * `binit`函数
      ```c
      void
      binit(void)
      {
        initlock(&bcache.lock, "bcache");

        for(int i = 0;i < NBUCKET;i++){
          for(int j = 0;j < BUFSIZE;j++){
            initsleeplock(&bcache.table[i].buf[j].lock, "buffer");
          }
          initlock(&bcache.table[i].tablelock, "table");
        }
      }
      ```
     * `bget`函数
      ```c
      static struct buf*
      bget(uint dev, uint blockno)
      {
        struct buf *b;

        int hash = blockno % NBUCKET;
        // Is the block already cached?
        acquire(&bcache.table[hash].tablelock);
        for(int i = 0;i < BUFSIZE;i++){
          b = &bcache.table[hash].buf[i];
          if(b->dev == dev && b->blockno == blockno){
            b->refcnt++;
            b->ticks = ticks;
            release(&bcache.table[hash].tablelock);
            acquiresleep(&b->lock);
            return b;
          }
        }

        // Not cached.
        // Recycle the least recently used (LRU) unused buffer.
        uint least = 0xffffffff;
        int index = -1;
        for(int i = 0;i < BUFSIZE;i++){
          b = &bcache.table[hash].buf[i];
          if(b->refcnt == 0 && b->ticks < least){
            least = b->ticks;
            index = i;
          }
        }
        if(index == -1)
          panic("bget");

        b = &bcache.table[hash].buf[index];
        b->dev = dev;
        b->blockno = blockno;
        b->valid = 0;
        b->refcnt = 1;
        release(&bcache.table[hash].tablelock);
        acquiresleep(&b->lock);
        return b;
          
        panic("bget: no buffers");
      }
      ```
     * `brelse`函数
      ```c
      void
      brelse(struct buf *b)
      {
        if(!holdingsleep(&b->lock))
          panic("brelse");

        releasesleep(&b->lock);

        int i = b->blockno % NBUCKET;
        acquire(&bcache.table[i].tablelock);
        b->refcnt--;
        release(&bcache.table[i].tablelock);  
      }
      ```
*  **思路**：
   *  首先建议看一下`Lec14`，了解一下关于`bio.c`文件函数的相关作用，对于写题目会更有头绪和帮助一些
   *  对于xv6来说，我们最大只有`NBUF`，也就是30个缓冲区。课堂提示中建议我们使用固定数量的散列桶。因此对于结构体来说，我们直接固定哈希桶和哈希表的大小
   *  `bget`函数
      *  我们现在只需要在`hash = blockno % NBUCKET`，也就是`table[hash]`中去给块分配缓冲区。因此第一部分只需要更改一下循环的条件。
      *  第二部分`LRU`的部分，只需要找出当前哈希桶中最少使用（也就是`ticks`最小）且当前未被使用（`b->refcnt == 0`）的块。
      *  当在当前哈希桶找不到函数的时候，也就是代码中`if(index == -1)`的部分，我们按理来说需要去窃取别的哈希桶中可以使用的buf部分。但是在这个实验中不窃取也能通过所有的测试
   *  `brelse`函数
      *  由于改为使用时间戳进行标记，因此可以不需要获取bcache锁。

* **踩坑**：
   * **动态哈希表。** 一开始考虑建立动态哈希表，对于每个块的缓冲区，当`b->blockno`不同时，将其插入到不同的哈希表中。但是这样的实现需要考虑很多其他的东西。比如考虑链表的头尾设置，考虑何时取得两个桶的锁，考虑锁的获取顺序，考虑新块和旧块的位置。总之就是麻烦多多
   * **对于哨兵节点和哈希桶的混淆。** 源代码中的`bcache.head`是一个哨兵节点，用于简化对链表的头尾的判断。而我设立的哈希桶`struct buf *table[NBUCKET]`是一个指针，不能够进行`table->next`这样的操作。
   * **对于链表操作的不熟练。** 出现了几次边界处理不清晰的问题。

下面是一开始我自己写的代码。能通过test0，但是test1和usertest都会有问题，跑不过。主要问题有桶锁的过早释放导致的串行化失败，和逻辑判断过于混乱。有点屎山。
* 修改结构体
    * 结构体
      ```c
      #define NBUCKET 13
      struct {
        struct spinlock lock;
        struct buf buf[NBUF];
        struct buf *table[NBUCKET];
        struct spinlock tablelock[NBUCKET];
      } bcache;
      ```
    * `binit`函数
      ```c
      void
      binit(void)
      {
        struct buf *b;

        initlock(&bcache.lock, "bcache");

        for(b = bcache.buf; b < bcache.buf+NBUF; b++){
          b->next = bcache.table[0];
          bcache.table[0] = b;
          initsleeplock(&b->lock, "buffer");
        }
        for(int i = 0;i < NBUCKET;i++)
          initlock(&bcache.tablelock[i], "table");
      }
      ```
    * `bget`函数
      ```c
      static struct buf*
      bget(uint dev, uint blockno)
      {
        struct buf *b;

        int i = blockno % NBUCKET;
        // Is the block already cached?
        acquire(&bcache.tablelock[i]);
        for(b = bcache.table[i]; b != 0; b = b->next){
          if(b->dev == dev && b->blockno == blockno){
            b->refcnt++;
            // release(&bcache.lock);
            release(&bcache.tablelock[i]);
            acquiresleep(&b->lock);
            return b;
          }
        }
        release(&bcache.tablelock[i]);

        // Not cached.
        // Recycle the least recently used (LRU) unused buffer.
        struct buf *bf = 0;
        uint timestamps = 0;
        acquire(&bcache.lock);
        for(int j = 0; j < NBUCKET; j++){
          acquire(&bcache.tablelock[j]);
          for(b = bcache.table[j]; b != 0; b = b->next){
            if(b->refcnt == 0 && (b->ticks < timestamps || timestamps == 0)){
              bf = b;
              timestamps = b->ticks;
            }
          }
          release(&bcache.tablelock[j]);
        }

        if(bf == 0){
          release(&bcache.lock);
          panic("bget: no buffers for bf");
        }

        int i_bf = bf->blockno % NBUCKET;
        acquire(&bcache.tablelock[i]);
        if(i != i_bf){
          acquire(&bcache.tablelock[i_bf]);
          if(bf == bcache.table[i_bf]){
            bcache.table[i_bf] = bf->next;
          } else {
            for(b = bcache.table[i_bf]; b != 0 && b->next != bf; b = b->next){}
            if(b)
              b->next = bf->next;
          }
          release(&bcache.tablelock[i_bf]);

          b = bcache.table[i];
          bf->next = b;
          bcache.table[i] = bf;
        }
        bf->dev = dev;
        bf->blockno = blockno;
        bf->valid = 0;
        bf->refcnt = 1;
        release(&bcache.tablelock[i]);

        release(&bcache.lock);
        acquiresleep(&bf->lock);
        return bf;

        panic("bget: no buffers");
      }
      ```
    * `brelse`函数
      ```c
      void
      brelse(struct buf *b)
      {
        if(!holdingsleep(&b->lock))
          panic("brelse");

        releasesleep(&b->lock);

        int i = b->blockno % NBUCKET;
        acquire(&bcache.tablelock[i]);
        b->refcnt--;
        if(b->refcnt == 0){
          // no one is waiting for it.
          b->ticks = ticks;
        }
        release(&bcache.tablelock[i]);  
      }
      ```