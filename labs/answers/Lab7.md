# Lab7: Multithreading

**花费时间：** 5h

---

## Part 1: Uthread: switching between threads (难度：moderate)

> 您的工作是提出一个创建线程和保存/恢复寄存器以在线程之间切换的计划，并实现该计划。完成后，make grade应该表明您的解决方案通过了uthread测试。

> 1. 一个目标是确保当`thread_schedule()`第一次运行给定线程时，该线程在自己的栈上执行传递给`thread_create()`的函数。

根据上课所讲，类似`allocproc`函数，在第一次创建进程的时候给`context.ra`分配想要第一次运行返回的地址，就是相应的函数地址，给`context.sp`分配指向栈的地址。
```c
void 
thread_create(void (*func)())
{
  
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)t->stack;
}
```

> 2. 另一个目标是确保thread_switch保存被切换线程的寄存器，恢复切换到线程的寄存器，并返回到后一个线程指令中最后停止的点。

参考xv6中的`swtch`函数
```c
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

> 3. 您必须决定保存/恢复寄存器的位置；修改`struct thread`以保存寄存器是一个很好的计划。

```c
struct context {
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

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct     context context;
};
```

> 4. 您需要在`thread_schedule`中添加对`thread_switch`的调用；您可以将需要的任何参数传递给`thread_switch`，但目的是将线程从t切换到`next_thread`。

```c
/* YOUR CODE HERE
  * Invoke thread_switch to switch from t to next_thread:
  * thread_switch(??, ??);
  */
thread_switch((uint64)&t->context, (uint64)&next_thread->context);
```

### bug
运行输出
```shell
$ uthread
thread_a started
thread_b started
thread_c started
thread_c 0
thread_c 1
thread_c 2
···
thread_c: exit after 100
thread_schedule: no runnable threads
```

通过debug发现在`thread_b`函数中运行会导致`thread_a`，也就是`all_thread[1]`的state会突然变成一个很大的数。是由于栈溢出了覆盖了不属于自己的部分。更改代码后测试成功。

```c
t->context.ra = (uint64)func;
t->context.sp = (uint64)t->stack + STACK_SIZE;
```

## Part 2: Using threads (难度：moderate)

> 为什么两个线程都丢失了键，而不是一个线程？确定可能导致键丢失的具有2个线程的事件序列。在**answers-thread.txt**中提交您的序列和简短解释。

**解释**
```c
static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}
```

**可能导致键丢失的事件序列（以两个线程T1和T2为例，假设它们要插入到同一个桶 `i`）：**

假设初始时 `table[i]` 为 `NULL`（或者指向某个旧的链表头 `OldHead`）。

1.  **线程T1** 调用 `put(key1, value1)`：
    *   计算出桶索引 `i`。
    *   `put` 函数中，它准备调用 `insert(key1, value1, &table[i], table[i])`。
    *   在进入 `insert` 之前，假设操作系统发生上下文切换。或者，T1进入`insert`。
    *   T1 (在 `insert` 中): `struct entry *n_t1 = table[i];` (此时 `n_t1` 是 `OldHead`)
    *   T1 (在 `insert` 中): `e1 = malloc(...)`; `e1->key = key1; e1->value = value1;`
    *   T1 (在 `insert` 中): `e1->next = n_t1;` (即 `e1->next = OldHead`)
    *   **上下文切换到线程T2** (在T1执行 `*p = e1` 即 `table[i] = e1` 之前)

2.  **线程T2** 调用 `put(key2, value2)`：
    *   计算出相同的桶索引 `i`。
    *   `put` 函数中，它准备调用 `insert(key2, value2, &table[i], table[i])`。
    *   T2 (在 `insert` 中): `struct entry *n_t2 = table[i];` (此时 `table[i]` 仍然是 `OldHead`，因为T1还没有更新它)
    *   T2 (在 `insert` 中): `e2 = malloc(...)`; `e2->key = key2; e2->value = value2;`
    *   T2 (在 `insert` 中): `e2->next = n_t2;` (即 `e2->next = OldHead`)
    *   T2 (在 `insert` 中): `*(&table[i]) = e2;` (即 `table[i] = e2`)。现在 `table[i]` 指向 `e2`，并且 `e2->next` 指向 `OldHead`。

3.  **上下文切换回线程T1**：
    *   T1 (在 `insert` 中): 继续执行 `*(&table[i]) = e1;` (即 `table[i] = e1`)。

**结果：**
此时，`table[i]` 指向 `e1`，并且 `e1->next` 指向 `OldHead`。
线程T2插入的 `e2` 节点丢失了！它不再是哈希桶链表的一部分，因为 `table[i]` 被T1的赋值覆盖了，而 `e1->next` 指向的是 `OldHead`，跳过了 `e2`。

**为什么两个线程都报告丢失了键？**

`get_thread` 函数会检查 *所有* `NKEYS` 是否存在。
每个 `put_thread` 负责插入一部分 `keys`。
如果由于上述竞态条件，某个由线程0（或线程1）插入的键丢失了，那么当线程0的 `get_thread` 和线程1的 `get_thread` 遍历所有 `NKEYS` 时，它们都会发现这个键的丢失。

所以，"1: 16579 keys missing" 和 "0: 16579 keys missing" 意味着总共有16579个本应存在于哈希表中的键找不到了。这两个 `get_thread` 线程独立地检查了整个哈希表，并得出了相同的缺失键总数。这并不是说线程0丢失了它自己插入的16579个键，线程1也丢失了它自己插入的16579个键，而是总共丢失了16579个键，这个结果被两个检查线程同时观察到。

> **TIP**: 为了避免这种事件序列，请在**notxv6/ph.c**中的`put`和`get`中插入`lock`和`unlock`语句，以便在两个线程中丢失的键数始终为0。相关的pthread调用包括：
> 
> `pthread_mutex_t lock; // declare a lock`
> `pthread_mutex_init(&lock, NULL); // initialize the lock`
> `pthread_mutex_lock(&lock); // acquire lock`
> `pthread_mutex_unlock(&lock); // release lock`
> 当`make grade`说您的代码通过`ph_safe`测试时，您就完成了，该测试需要两个线程的键缺失数为0。在此时，`ph_fast`测试失败是正常的。

**定义**
```c
pthread_mutex_t lock;
```
**初始化**
```c
pthread_mutex_init(&lock, NULL);
```
**put加锁**
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
    pthread_mutex_lock(&lock);// <-----------------加锁
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock);// <---------------释放锁
  }
}
```
**get加锁**
```c
static void *
get_thread(void *xa)
{
  int n = (int) (long) xa; // thread number
  int missing = 0;

  for (int i = 0; i < NKEYS; i++) {
    pthread_mutex_lock(&lock);// <-----------------加锁
    struct entry *e = get(keys[i]);
    pthread_mutex_unlock(&lock);// <---------------释放锁
    if (e == 0) missing++;
  }
  printf("%d: %d keys missing\n", n, missing);
  return NULL;
}
```

这样之后就能够通过`ph_safe`测试。但是实际上在这里put加锁的行为依然有可能导致竞态事件的发生。
```shell
./ph 1
100000 puts, 5.350 seconds, 18693 puts/second
0: 0 keys missing
100000 gets, 5.276 seconds, 18953 gets/second

./ph 2
100000 puts, 2.564 seconds, 38999 puts/second
1: 0 keys missing
0: 0 keys missing
200000 gets, 12.338 seconds, 16210 gets/second
```
**修改后的put加锁**
```c
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&lock);// <-----------------加锁
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
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);// <---------------释放锁
}
```

> 修改代码，使某些`put`操作在保持正确性的同时并行运行。当`make grade`说你的代码通过了`ph_safe`和`ph_fast`测试时，你就完成了。`ph_fast`测试要求两个线程每秒产生的`put`数至少是一个线程的1.25倍。

**提示**
每个散列桶加一个锁怎么样？

**定义**
```c
pthread_mutex_t lock[NBUCKET];
```
**初始化**
```c
for(int i = 0;i < NBUCKET;i++)
  pthread_mutex_init(&lock[i], NULL);
```
**put加锁**
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
    pthread_mutex_lock(&lock[i]);// <-----------------加锁
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock[i]);// <---------------释放锁
  }
}
```
**get加锁**
```c
static void *
get_thread(void *xa)
{
  int n = (int) (long) xa; // thread number
  int missing = 0;

  for (int i = 0; i < NKEYS; i++) {
    pthread_mutex_lock(&lock[i % NBUCKET]);// <-----------------加锁
    struct entry *e = get(keys[i]);
    pthread_mutex_unlock(&lock[i % NBUCKET]);// <---------------释放锁
    if (e == 0) missing++;
  }
  printf("%d: %d keys missing\n", n, missing);
  return NULL;
}
```

这样之后就能够通过`ph_fast`测试。
```shell
./ph 1
100000 puts, 5.398 seconds, 18526 puts/second
0: 0 keys missing
100000 gets, 5.361 seconds, 18652 gets/second

./ph 2
100000 puts, 3.250 seconds, 30770 puts/second
1: 0 keys missing
0: 0 keys missing
200000 gets, 5.197 seconds, 38481 gets/second
```

可以发现速度相比于只有一个锁速度快了很多

## Part 3: Barrier (难度：moderate)

> 您的目标是实现期望的屏障行为。除了在`ph`作业中看到的lock原语外，还需要以下新的pthread原语；详情请看这里和这里。
> * `// 在cond上进入睡眠，释放锁mutex，在醒来时重新获取`
> * `pthread_cond_wait(&cond, &mutex);`
> * `// 唤醒睡在cond的所有线程`
> * `pthread_cond_broadcast(&cond);`

*  `while` 循环是用来防止线程被伪唤醒
* 快速的线程进入下一轮并修改 `bstate.nthread` 不会影响到还在等待前一轮结束的线程，因为这些等待的线程是根据 ~`bstate.round`~ 是否已经改变来决定是否退出的

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  int myround = bstate.round;
  bstate.nthread++;
  if(bstate.nthread == nthread){
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } else {
    while (myround == bstate.round) {
      pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```