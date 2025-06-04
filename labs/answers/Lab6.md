# Lab6: Copy-on-Write Fork for xv6

**花费时间：** 8h

---

## Implement copy-on-write (难度：hard)

### cowtest

> 1. 修改`uvmcopy()`将父进程的物理页映射到子进程，而不是分配新页。在子进程和父进程的PTE中清除`PTE_W`标志。

按照上课讲的写就行。注意**PTE_W**位和**PTE_COW**位的设置与擦除。

要清除一个特定的标志位 `FLAG`，应该使用 `variable &= ~FLAG;`
要设置一个特定的标志位 `FLAG`，应该使用 `variable |= FLAG;`

*   **`kernel/vm.c`**:
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
        pagerefcnt[pa / PGSIZE]++;
        *pte &= ~PTE_W;
        *pte |= PTE_COW;
        flags = PTE_FLAGS(*pte);
        if(mappages(new, i, PGSIZE, pa, flags) != 0){
          uvmunmap(new, 0, i / PGSIZE, 0);
          *pte |= PTE_W;
          return -1;
        }
      }
      return 0;
    }
    ```

> 2. 修改`usertrap()`以识别页面错误。当COW页面出现页面错误时，使用`kalloc()`分配一个新页面，并将旧页面复制到新页面，然后将新页面添加到PTE中并设置`PTE_W`。
>
>     如果出现COW页面错误并且没有可用内存，则应终止进程。

注意**PTE_W**位和**PTE_COW**位的设置与擦除。同时还有引用计数的设置。

*   **`kernel/trap.c`**:
    ```c
      } else if((which_dev = devintr()) != 0){
        // ok
      } else if(r_scause() == 13 || r_scause() == 15){
        uint64 va = r_stval();
        pte_t* pte;
        uint64 pa_from, pa_to;
        uint   flags;

        if ((pte = walk(p->pagetable, va, 0)) == 0)
          p->killed = 1;
        if ((*pte & PTE_COW) != 0) {
          if ((*pte & PTE_V) == 0)
              p->killed = 1;
          pa_from = PTE2PA(*pte);
          flags = PTE_FLAGS(*pte);
          flags &= ~PTE_COW;

          if ((pa_to = (uint64)kalloc()) == 0) {
              p->killed = 1;
          }
          memmove((char*)pa_to, (char*)pa_from, PGSIZE);

          *pte = PA2PTE(pa_to) | flags | PTE_W;

          pagerefcnt[pa_from / PGSIZE]--;
          kfree((void*)pa_from);
        }
      } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
        p->killed = 1;
      }
    ```

> 3. 确保每个物理页在最后一个PTE对它的引用撤销时被释放——而不是在此之前。这样做的一个好方法是为每个物理页保留引用该页面的用户页表数的"引用计数"。
>
>     当`kalloc()`分配页时，将页的引用计数设置为1。当`fork`导致子进程共享页面时，增加页的引用计数；每当任何进程从其页表中删除页面时，减少页的引用计数。
>
>     `kfree()`只应在引用计数为零时将页面放回空闲列表。可以将这些计数保存在一个固定大小的整型数组中。
>
>     你必须制定一个如何索引数组以及如何选择数组大小的方案。例如，您可以用页的物理地址除以4096对数组进行索引，并为数组提供等同于`kalloc.c`中`kinit()`在空闲列表中放置的所有页面的最高物理地址的元素数。

xv6内核的可用物理内存地址是从`memlayout.h`中的这两行代码展示的：
```c
#define KERNBASE 0x80000000L
#define PHYSTOP (KERNBASE + 128*1024*1024)
```
因此我们创建一个数组，用于记录所有物理页的引用计数。相关的设置与加减在题目中说明。

*   **`kernel/kalloc.c`**:
    ```c
    uint pagerefcnt[PHYSTOP / PGSIZE];
    ```
    ```c
    void
    kfree(void *pa)
    {
      ···
      if(pagerefcnt[(uint64)pa / PGSIZE] != 0)
        return;
      // Fill with junk to catch dangling refs.
      memset(pa, 1, PGSIZE);
      ···
    }
    ```
    ```c
    void *
    kalloc(void)
    {
      struct run *r;

      acquire(&kmem.lock);
      r = kmem.freelist;
      if(r)
        kmem.freelist = r->next;
      release(&kmem.lock);
      if(r){
        memset((char*)r, 5, PGSIZE); // fill with junk
        pagerefcnt[(uint64)r / PGSIZE] = 1;
      }
      return (void*)r;
    }
    ```

> 4. 修改`copyout()`在遇到COW页面时使用与页面错误相同的方案。

**注意：** 不要忘记复制内容到新创建的page。

*   **`kernel/vm.c`** (相关函数修改，`copyout`):
    ```c
    int
    copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
    {
      uint64 n, va0, pa0;
      pte_t* pte;
      uint64 pa_from;
      uint flags;

      while(len > 0){
        va0 = PGROUNDDOWN(dstva);
        if((pte = walk(pagetable, va0, 0)) == 0)
          return -1;
        if((*pte & PTE_COW) != 0 ){
          if ((*pte & PTE_V) == 0)
            return -1;
          if ((*pte & PTE_U) == 0)
            return -1;
          if((pa0 = (uint64)kalloc()) == 0)
            return -1;
          pa_from = PTE2PA(*pte);
          flags = PTE_FLAGS(*pte);
          flags &= ~PTE_COW;
          memmove((char*)pa0, (char*)pa_from, PGSIZE);
          *pte = PA2PTE(pa0) | flags | PTE_W;

          pagerefcnt[pa_from / PGSIZE]--;
          kfree((void*)pa_from);
        } else {
          pa0 = walkaddr(pagetable, va0);
          if(pa0 == 0)
            return -1;
        }
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

### usertest

上面的代码能够跑通`cowtest`，但是在跑`usertest`会出错。

#### 1. `copyout`错误
##### 报错
```c
panic("walk");
```
##### 解决方法
在`copyout`的`walk`前加上：
```c
if(va0 > MAXVA)
  return -1;
```

#### 2. `execout`和`bagarg`错误
##### 报错
```shell
test badarg: scause 0x000000000000000f
sepc=0x0000000080000dca stval=0x0000000000000000
```
##### 解决方法
查看`kernel.asm`发现是`memmove`出了问题。排查了很多遍，最后发现是在`trap.c`函数的`usertrap`函数中的处理逻辑中出现了问题。

**对应逻辑**：如果出现COW页面错误并且没有可用内存，则应终止进程。
*   原逻辑：发现没有足够的页面，将`p->killed = 1`，但是没有立即退出。
*   实际逻辑：将判断逻辑1封装为函数，在COW页面错误中没有可用内存，应该立即返回错误参数（不进行后面的`memmove`），然后将`p->killed = 1`。

#### 3. 内存泄漏问题
##### 报错
```shell
test badarg: 显示没有足够内存
FAILED -- lost some free pages 30190 (out of 30193)
```
##### 解决方法
实际上是内存泄漏。由于我们不在`kfree`函数中进行引用计数的减少，因此`freewalk`中不会对页表本身的引用计数进行减少，导致页表本身的内存不会被释放。

**重要提示**：由于一开始我想的是，`kfree`函数只用于引用计数的判断，用专门的一个函数来进行引用计数的增减。但是后面的绝大部分bug都是这个想法引起的。因为在整个xv6中有非常多的地方都使用了`kfree`函数，假如这么做，需要在每个`kfree`函数之前都将相关的引用计数减去1。因此将减去引用计数的逻辑放在`kfree`函数中（记得加锁），好多bug就都解决了。

#### 4. 关于引用计数的竞态问题
为了解决引用计数的竞态问题，需要创建一个带锁的结构体，用于每次更改`pagerefcnt`的时候进行加锁。
```c
struct ref_stru{
  uint pagerefcnt[PHYSTOP / PGSIZE];
  struct spinlock lock;
} ref;
```
**重要提示**：这里使用自旋锁是考虑到这种情况：进程P1和P2共用内存M，M引用计数为2，此时CPU1要执行`fork`产生P1的子进程，CPU2要终止P2，那么假设两个CPU同时读取引用计数为2，执行完成后CPU1中保存的引用计数为3，CPU2保存的计数为1，那么后赋值的语句会覆盖掉先赋值的语句，从而产生错误。

## 最终代码整合

*   **`kernel/kalloc.c`**
    ```c
    struct ref_stru{
      uint pagerefcnt[PHYSTOP / PGSIZE];
      struct spinlock lock;
    } ref;
    ```
    ```c
    void
    kinit()
    {
      initlock(&kmem.lock, "kmem");
      /*----------new code----------*/
      initlock(&ref.lock, "ref");
      /*----------new code----------*/
      freerange(end, (void*)PHYSTOP);
    }
    ```
    ```c
    void
    freerange(void *pa_start, void *pa_end)
    {
      char *p;
      p = (char*)PGROUNDUP((uint64)pa_start);
      /*----------new code----------*/
      for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE){
        acquire(&ref.lock);
        ref.pagerefcnt[(uint64)p / PGSIZE] = 1;
        release(&ref.lock);
        kfree(p);
      }
      /*----------new code----------*/
    }
    ```
    ```c
    void
    kfree(void *pa)
    {
      struct run *r;

      if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");

      /*----------new code----------*/
      acquire(&ref.lock);
      ref.pagerefcnt[(uint64)pa / PGSIZE]--;
      int cnt = ref.pagerefcnt[(uint64)pa / PGSIZE];
      release(&ref.lock);
      if(cnt > 0)
        return;
      /*----------new code----------*/
      // Fill with junk to catch dangling refs.
      memset(pa, 1, PGSIZE);

      r = (struct run*)pa;

      acquire(&kmem.lock);
      r->next = kmem.freelist;
      kmem.freelist = r;
      release(&kmem.lock);
    }
    ```
    ```c
    void *
    kalloc(void)
    {
      struct run *r;

      acquire(&kmem.lock);
      r = kmem.freelist;
      if(r)
        kmem.freelist = r->next;
      release(&kmem.lock);

      /*----------new code----------*/
      if(r){
        memset((char*)r, 5, PGSIZE); // fill with junk
        acquire(&ref.lock);
        ref.pagerefcnt[(uint64)r / PGSIZE] = 1;
        release(&ref.lock);
      }
      /*----------new code----------*/
      return (void*)r;
    }
    ```
    ```c
    void kaddrefcnt(void* pa){
      if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
        panic("kaddrefcnt");
      acquire(&ref.lock);
      ref.pagerefcnt[(uint64)pa / PGSIZE]++;
      release(&ref.lock);
    }
    ```

*   **`kernel/defs.h`**
    ```c
    // kalloc.c
    void            kaddrefcnt(void*);
    ```

*   **`kernel/riscv.h`**
    ```c
    #define PTE_COW (1L << 8)
    ```

*   **`kernel/trap.c`**
    ```c
      } else if(r_scause() == 15){
        if(cowalloc(p->pagetable, r_stval()) != 0)
          p->killed = 1;
    ```

*   **`kernel/vm.c`**
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
        if(*pte & PTE_W)
           *pte = (*pte & ~PTE_W) | PTE_COW;
        flags = PTE_FLAGS(*pte);
        if(mappages(new, i, PGSIZE, pa, flags) != 0){
          uvmunmap(new, 0, i / PGSIZE, 0);
          *pte |= PTE_W;
          *pte &= ~PTE_COW;
          return -1;
        }
        kaddrefcnt((void *)pa);
      }
      return 0;
    }
    ```
    ```c
    int
    copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
    {
      uint64 n, va0, pa0;
      pte_t *pte;

      while(len > 0){
        va0 = PGROUNDDOWN(dstva);
        if(va0 >= MAXVA)
          return -1;

        pte = walk(pagetable, va0, 0);
        if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
          return -1;
        if((*pte & PTE_COW) != 0){
          if(cowalloc(pagetable, va0) != 0)
            return -1;
        }

        pa0 = PTE2PA(*pte);
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
    ```c
    int cowalloc(pagetable_t pagetable, uint64 va){
      pte_t* pte;
      uint64 pa_from, pa_to;
      uint   flags;

      if ((pte = walk(pagetable, va, 0)) == 0)
        panic("cowalloc: pte not exists");
      if ((*pte & PTE_COW) == 0)
        return 0;
      if ((*pte & PTE_V) == 0)
        panic("cowalloc: pte permission err");
      pa_from = PTE2PA(*pte);
      *pte = (*pte & ~PTE_COW) | PTE_W;
      flags = PTE_FLAGS(*pte);

      if ((pa_to = (uint64)kalloc()) == 0) {
          *pte |= PTE_COW;
          *pte &= ~PTE_W;
          return -1;
      }
      memmove((char*)pa_to, (char*)pa_from, PGSIZE);

      *pte = PA2PTE(pa_to) | flags | PTE_W;
      kfree((void*)pa_from);
      return 0;
    }
    ```