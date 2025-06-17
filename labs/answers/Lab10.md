# Lab10: mmap

**花费时间：** 8h

---

## mmap (难度：hard)

> 您应该实现足够的`mmap`和`munmap`功能，以使`mmaptest`测试程序正常工作。如果`mmaptest`不会用到某个`mmap`的特性，则不需要实现该特性。

### 提示与思路

### 第一步

> 1.  首先，向`UPROGS`添加`_mmaptest`，以及`mmap`和`munmap`系统调用，以便让**user/mmaptest.c**进行编译。现在，只需从`mmap`和`munmap`返回错误。我们在**kernel/fcntl.h**中为您定义了`PROT_READ`等。运行`mmaptest`，它将在第一次`mmap`调用时失败。

* 修改相关的系统调用

> 2.  惰性地填写页表，以响应页错误。也就是说，`mmap`不应该分配物理内存或读取文件。相反，在`usertrap`中（或由`usertrap`调用）的页面错误处理代码中执行此操作，就像在**lazy page allocation**实验中一样。惰性分配的原因是确保大文件的`mmap`是快速的，并且比物理内存大的文件的`mmap`是可能的。

*  与**lazy page allocation**实验类似，我们在`mmap`中只增加用户进程地址空间的大小，而不进行实际的分配。只是对`p->sz`进行增加

> 3.  跟踪`mmap`为每个进程映射的内容。定义与第15课中描述的VMA（虚拟内存区域）对应的结构体，记录`mmap`创建的虚拟内存范围的地址、长度、权限、文件等。由于xv6内核中没有内存分配器，因此可以声明一个固定大小的VMA数组，并根据需要从该数组进行分配。大小为16应该就足够了。

*  `vma`用于记录`mmap`的相关信息，包括地址、长度、权限、文件
*  除此之外，我们还需要一个flag来记录当前`vma`是否被使用。这里命名为used
*  `vma`的数组存储在进程中，意思是每个进程最多可以管理16个`mmap`打开的文件

> 4.  实现`mmap`：在进程的地址空间中找到一个未使用的区域来映射文件，并将VMA添加到进程的映射区域表中。VMA应该包含指向映射文件对应`struct file`的指针；`mmap`应该增加文件的引用计数，以便在文件关闭时结构体不会消失（提示：请参阅`filedup`）。

* 在`mmap`中做以下事情
  * 映射文件
  * 将VMA添加到数组中
  * 保存相关信息
  * 增加引用计数

**代码修改：**

*   **`kernel/proc.h`**:
    ```c
    #define NVMA 16

    struct vma {
      int used;
      uint64 addr;
      int length;
      int prot;
      int flags;
      int fd;
      int offset;
      struct file* file;
    };

    struct proc {
      // other code
      struct vma vma[NVMA];
    };
    ```
*   **`kernel/sysfile.c`**:
    ```c
    uint64
    sys_mmap(void)
    {
      // void* addr = 0;
      // int offset = 0;
      int length, prot, flags, fd;
      struct proc* p = myproc();
      struct vma* v = 0;
      struct file* f;
      uint64 sz, pa = 0;

      if(argint(1, &length) || argint(2, &prot) || argint(3, &flags) || argfd(4, &fd, &f))
        return -1;
      if(length < 0){
        printf("length < 0");
        return -1;
      }
      
      // lazy allocation 
      // 或许要用trapframe下面的空间作为对象
      sz = p->sz;
      p->sz += length;
      for(int i = 0; i < NVMA; i++){
        if(p->vma[i].used == 0){
          v = &(p->vma[i]);
          break;
        }
      }
      if(v == 0)
        panic("mmap: no free vma");

      if(mappages(p->pagetable, sz, length, pa, prot) != 0)
        panic("mmap: mappages");
      
      filedup(f);
      v->addr = sz;
      v->fd = fd;
      v->prot = prot;
      v->length = length;
      v->flags = flags;
      v->offset = 0;
      v->file = f;
      v->used = 1;

      return sz;
    }
    ```

**测试输出：**
> 运行`mmaptest`：第一次`mmap`应该成功，但是第一次访问被`mmap`的内存将导致页面错误并终止`mmaptest`。

实际输出：
```shell
$ mmaptest
mmap_test starting
test mmap f
usertrap(): unexpected scause 0x000000000000000d pid=3
            sepc=0x0000000000000076 stval=0x0000000000004000
```

查看`mmaptest.asm`，76行指令在`_v1()`函数中，是读取指令，且`scause`为13，是读页面时的page fault代码。

### 第二步

> 1.  添加代码以导致在`mmap`的区域中产生页面错误，从而分配一页物理内存，将4096字节的相关文件读入该页面，并将其映射到用户地址空间。使用`readi`读取文件，它接受一个偏移量参数，在该偏移处读取文件（但必须lock/unlock传递给`readi`的索引结点）。不要忘记在页面上正确设置权限。运行`mmaptest`；它应该到达第一个`munmap`

* 在trap.c中添加相关的代码。其中由于第一个mmaptest的_v1()函数只测试读功能，因此将检测`r_scause()`设置为13。**一开始设置为了15，没有分清读页面和写页面的`r_scause()`值**


**bug：**

* 第一次运行结果
  ```shell
  $ mmaptest
  mmap_test starting
  test mmap f
  panic: trap: readi
  ```
  * **错误原因** readi每次读取的返回值不一定每次都是PGSIZE。
  * 原来的写法
    ```c
    ilock(v->file->ip);
    if(readi(v->file->ip, 0, (uint64)pa, v->offset, PGSIZE) != PGSIZE)
      panic("trap: readi");
    v->offset += PGSIZE;
    iunlock(v->file->ip);
    ```
  * 改正后
    ```c
    ilock(v->file->ip);
    if((r = readi(v->file->ip, 0, (uint64)pa, v->offset, PGSIZE)) <= 0)
      panic("trap: readi");
    v->offset += r;
    iunlock(v->file->ip);
    ```
* 第二次运行结果
  ```c
  $ mmaptest
  mmap_test starting
  test mmap f
  mismatch at 6144, wanted zero, got 0x5
  mmaptest: mmap_test failed: v1 mismatch (2), pid=3
  ```
  * **错误原因** kalloc()会将内存置为5.所以读到垃圾数据。用memset置0。
* 第三个错误。不要在sys_mmap中mappages。
* 第四个错误。忘记设置PTE_U权限
* 第五个错误。readi的offset设置错误。开始传入的是v->offset，但是不一定满足随机读写要求

**代码修改：**

*   **`kernel/trap.c`**:
    ```c
    } else if(r_scause() == 13){
      uint64 va = r_stval();
      uint64 pa;
      struct vma* v;
      struct proc* p = myproc();
      uint pte_flags = 0;
      int r;

      for(int i = 0;i < NVMA; i++){
        v = &(p->vma[i]);
        if(va >= v->addr && va <= v->addr + v->length)
          break;
      }
      if(v->used == 0)
        panic("trap: vma is not using");

      if((pa = (uint64)kalloc()) == 0)
        panic("trap: kalloc");
      memset((void*)pa, 0, PGSIZE);

      ilock(v->file->ip);
      if((r = readi(v->file->ip, 0, (uint64)pa, PGROUNDDOWN(va)  - v->addr, PGSIZE)) <= 0)
        panic("trap: readi");
      printf("readi: %d\n", r);
      iunlock(v->file->ip);

      if (v->prot & PROT_READ)
          pte_flags |= PTE_R;
      if (v->prot & PROT_WRITE)
          pte_flags |= PTE_W;
      if (v->prot & PROT_EXEC)
          pte_flags |= PTE_X;
      if(mappages(p->pagetable, va, PGSIZE, pa, pte_flags | PTE_U) != 0)
        panic("mmap: mappages");
    ```
*   **`kernel/sysfile.c`**:
    ```c
    uint64
    sys_mmap(void)
    {
      // void* addr = 0;
      // int offset = 0;
      int length, prot, flags, fd;
      struct proc* p = myproc();
      struct vma* v = 0;
      struct file* f;
      uint64 sz;

      if(argint(1, &length) || argint(2, &prot) || argint(3, &flags) || argfd(4, &fd, &f))
        return -1;
      if(length < 0){
        printf("length < 0");
        return -1;
      }
      
      // lazy allocation 
      // 或许要用trapframe下面的空间作为对象
      sz = p->sz;
      p->sz += length;
      for(int i = 0; i < NVMA; i++){
        if(p->vma[i].used == 0){
          v = &(p->vma[i]);
          break;
        }
      }
      if(v == 0)
        panic("mmap: no free vma");
      
      filedup(f);
      v->addr = sz;
      v->fd = fd;
      v->prot = prot;
      v->length = length;
      v->flags = flags;
      v->offset = 0;
      v->file = f;
      v->used = 1;

      return sz;
    }
    ```

**测试输出：**
> 运行`mmaptest`；它应该到达第一个`munmap`

实际输出：
```shell
$ mmaptest
mmap_test starting
test mmap f
readi: 4096
readi: 2048
mmaptest: mmap_test failed: munmap (1), pid=3
```

### 第三步

> 1.  实现`munmap`：找到地址范围的VMA并取消映射指定页面（提示：使用`uvmunmap`）。如果`munmap`删除了先前`mmap`的所有页面，它应该减少相应`struct file`的引用计数。如果未映射的页面已被修改，并且文件已映射到`MAP_SHARED`，请将页面写回该文件。查看`filewrite`以获得灵感。
>
> 2. 理想情况下，您的实现将只写回程序实际修改的`MAP_SHARED`页面。RISC-V PTE中的脏位（`D`）表示是否已写入页面。但是，`mmaptest`不检查非脏页是否没有回写；因此，您可以不用看`D`位就写回页面。

* `munmap`实现的是对`mmap`映射的区域进行取消。对于已经分配了的物理页面，通过`uvmunmap`来进行取消映射和释放物理页面。但是对于没有进行实际映射的页面，我们什么都不需要做。因此对于这方面需要进行相关的判断

**bug：**

* 第一次运行结果
  ```shell
  test mmap two files
  readi: 5
  panic: trap: readi
  ```
  * **错误原因** 之前写的时候没有对于`MAP_SHARED`和`MAP_PRIVATE`进行相关的判断。假如设置为`MAP_SHARED`，是不能够用可写的标志位来`mmap`只读的文件，需要直接返回错误
  * 添加判断
    ```c
    if((flags & MAP_SHARED) != 0){
      if((f->writable == 0) && ((prot & PROT_WRITE) != 0))
        return -1;
    }
    ```
* 第二次运行结果
  * **错误原因** 对于查找vma的逻辑。由于vma的区间是左闭右开。因此在判断的时候左边可以为>=，右边得是<。
  * 原来的写法
    ```c
    for(int i = 0;i < NVMA; i++){
      v = &(p->vma[i]);
      if(addr >= v->addr && addr <= v->addr + v->length)
        break;
    }
    ```
  * 改正后
    ```c
    for(int i = 0;i < NVMA; i++){
      v = &(p->vma[i]);
      if(addr >= v->addr && addr < v->addr + v->length)
        break;
    }
    ```
**代码修改：**

*   **`kernel/sysfile.c`**:
    ```c
    uint64
    sys_mmap(void)
    {
      // void* addr = 0;
      // int offset = 0;
      int length, prot, flags, fd;
      struct proc* p = myproc();
      struct vma* v = 0;
      struct file* f;
      uint64 sz;

      if(argint(1, &length) || argint(2, &prot) || argint(3, &flags) || argfd(4, &fd, &f))
        return -1;
      if(length < 0){
        printf("length < 0");
        return -1;
      }

      if((flags & MAP_SHARED) != 0){
        if((f->writable == 0) && ((prot & PROT_WRITE) != 0))
          return -1;
      }
      
      sz = p->sz;
      p->sz += length;
      for(int i = 0; i < NVMA; i++){
        if(p->vma[i].used == 0){
          v = &(p->vma[i]);
          break;
        }
      }
      if(v == 0)
        panic("mmap: no free vma");
      if(v->used != 0)
        panic("mmap: vma used\n");
      
      filedup(f);
      v->addr = sz;
      v->fd = fd;
      v->prot = prot;
      v->length = length;
      v->flags = flags;
      v->offset = 0;
      v->file = f;
      v->used = 1;

      return sz;
    }
    ```
    ```c
    uint64
    sys_munmap(void)
    {
      uint64 addr;
      int length;
      struct proc* p = myproc();
      struct vma* v = 0;

      if(argaddr(0, &addr) || argint(1, &length))
        return -1;
      
      if(length < 0)
        return -1;
      for(int i = 0;i < NVMA; i++){
        v = &(p->vma[i]);
        if(addr >= v->addr && addr < v->addr + v->length && v->used == 1)
          break;
      }
      if (v == 0) {
        printf("munmap: unvalid addr\n");
        return -1;
      }

      if(length % PGSIZE != 0)
        panic("munmap: length not aligned\n");
      // write back
      if(v->flags == MAP_SHARED){
        filewrite(v->file, addr, length);
      }

      if(walkaddr(p->pagetable, PGROUNDDOWN(addr)) != 0)
        uvmunmap(p->pagetable, addr, length / PGSIZE, 1);
      v->addr += length;
      v->length -= length;
      if(v->length == 0){
        fileclose(v->file);
        v->addr = 0;
        v->fd = 0;
        v->prot = 0;
        v->flags = 0;
        v->offset = 0;
        v->file = 0;
        v->used = 0;
      }
      
      return 0;
    }
    ```

### 第四步

> 1.  修改`exit`将进程的已映射区域取消映射，就像调用了`munmap`一样。运行`mmaptest`；`mmap_test`应该通过，但可能不会通过`fork_test`。
> 
> 2.  修改`fork`以确保子对象具有与父对象相同的映射区域。不要忘记增加VMA的`struct file`的引用计数。在子进程的页面错误处理程序中，可以分配新的物理页面，而不是与父级共享页面。后者会更酷，但需要更多的实施工作。运行`mmaptest`；它应该通过`mmap_test`和`fork_test`。

* exit需要取消相关的映射。我们将munmap的代码放进去稍加修改即可。
* fork需要让子进程与父进程的VMA相同。同时页表已经分配了的内容也需要复制过去。

**bug：**
* 第一个问题
  * 直接调用xv6原本的uvmcopy会出现`panic("uvmcopy: page not present")`。原因是原来的xv6中，进程页表与`p->sz`一一对应，不可能sz以下的地址有找不到物理内存的情况。但是与懒分配类似，mmap会破坏这种情况
  * 类似的对于`uvmunmap`我们也要进行相关的修改
  * **修改代码**
  ```c
  int
  uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
  {
      if((*pte & PTE_V) == 0)
        // panic("uvmcopy: page not present");
        continue;
  }
  ```
  ```c
  void
  uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
  {
      if((*pte & PTE_V) == 0)
        // panic("uvmunmap: not mapped");
        continue;
  }
  ```
* 第二个问题
  * 经测试在以下三个测试中会出现`kernmem: panic: trap: no aviliable vma`。这panic是当在trap中，找遍了进程所有的vma也没找到合适的vma会出现的panic。=
  * 查看这三个测试
    * kernmem: 访问内核地址 KERNBASE。 *a
    * stacktest: 访问栈底下受保护的“警戒页”(guard page)。 *sp (sp 已经减去 PGSIZE)
    * sbrkfail: 在子进程中分配巨量内存，直到物理内存耗尽。这会导致 uvmalloc 失败，在 sbrk 返回 -1 之后，子进程继续访问这些未成功分配的内存，触发 page fault。
  * 它们都在一个 `fork()` 出来的子进程中，去访问一个注定会失败的地址。这个失败在标准 xv6 中，会直接导致内核杀死子进程（通过 `access fault` 或者 `page fault` 后的 `p->killed=1` 逻辑），父进程 `wait` 捕获到 `-1` 的状态，测试通过。
    ```c
    {stacktest, "stacktest"},
    {kernmem, "kernmem"},
    {sbrkfail, "sbrkfail"},
    ```
  * **修改代码** 添加trap处理中访问错误地址的问题。
    ```c
    if(v->used == 0){
      printf("trap: no avaliable vma\n");
      p->killed = 1;
    } else {
      ···
    }
    ```

## 最终代码修改
系统调用相关的代码不具体列出
*   **`kernel/proc.c`**:
    ```c
    //fork()
    for(int i = 0; i < NVMA; i++){
      if(p->vma[i].used == 1){
        np->vma[i].used = 1;
        np->vma[i].addr = p->vma[i].addr;
        np->vma[i].fd = p->vma[i].fd;
        np->vma[i].file = p->vma[i].file;
        np->vma[i].flags = p->vma[i].flags;
        np->vma[i].length = p->vma[i].length;
        np->vma[i].prot = p->vma[i].prot;
        np->vma[i].offset = p->vma[i].offset;
        filedup(np->vma[i].file);
      }
    }
    ```
    ```c
    // exit()
    struct vma* v;
    for(int i = 0;i < NVMA; i++){
      v = &(p->vma[i]);
      if(v->used != 1)
        continue;
      if(v->flags == MAP_SHARED)
        filewrite(v->file, v->addr, v->length);
      if(walkaddr(p->pagetable, PGROUNDDOWN(v->addr)) != 0)
        uvmunmap(p->pagetable, v->addr, v->length / PGSIZE, 1);
      fileclose(v->file);
      v->addr = 0;
      v->fd = 0;
      v->prot = 0;
      v->flags = 0;
      v->offset = 0;
      v->file = 0;
      v->used = 0;
    }
    ```
*   **`kernel/proc.h`**:
    ```c
    #define NVMA 16
    ```
    ```c
    struct proc {
      // other code
      struct vma vma[NVMA];
    };
    ```
    ```c
    struct vma {
      int used;
      uint64 addr;
      int length;
      int prot;
      int flags;
      int fd;
      int offset;
      struct file* file;
    };
    ```
*   **`kernel/sysfile.c`**:
    ```c
    uint64
    sys_mmap(void)
    {
      int length, prot, flags, fd;
      struct proc* p = myproc();
      struct vma* v = 0;
      struct file* f;
      uint64 sz;

      if(argint(1, &length) || argint(2, &prot) || argint(3, &flags) || argfd(4, &fd, &f))
        return -1;
      if(length < 0){
        printf("length < 0");
        return -1;
      }

      if((flags & MAP_SHARED) != 0){
        if((f->writable == 0) && ((prot & PROT_WRITE) != 0))
          return -1;
      }
      
      sz = p->sz;
      p->sz += length;
      for(int i = 0; i < NVMA; i++){
        if(p->vma[i].used == 0){
          v = &(p->vma[i]);
          break;
        }
      }
      if(v == 0)
        panic("mmap: no free vma");
      if(v->used != 0)
        panic("mmap: vma used\n");
      
      filedup(f);
      v->addr = sz;
      v->fd = fd;
      v->prot = prot;
      v->length = length;
      v->flags = flags;
      v->offset = 0;
      v->file = f;
      v->used = 1;

      return sz;
    }
    ```
    ```c
    uint64
    sys_munmap(void)
    {
      uint64 addr;
      int length;
      struct proc* p = myproc();
      struct vma* v = 0;

      if(argaddr(0, &addr) || argint(1, &length))
        return -1;
      
      if(length < 0)
        return -1;
      for(int i = 0;i < NVMA; i++){
        v = &(p->vma[i]);
        if(addr >= v->addr && addr < v->addr + v->length && v->used == 1)
          break;
      }
      if (v == 0) {
        printf("munmap: unvalid addr\n");
        return -1;
      }

      if(length % PGSIZE != 0)
        panic("munmap: length not aligned\n");
      if(v->flags == MAP_SHARED){
        filewrite(v->file, addr, length);
      }

      if(walkaddr(p->pagetable, PGROUNDDOWN(addr)) != 0)
        uvmunmap(p->pagetable, addr, length / PGSIZE, 1);
      v->addr += length;
      v->length -= length;
      if(v->length == 0){
        fileclose(v->file);
        v->addr = 0;
        v->fd = 0;
        v->prot = 0;
        v->flags = 0;
        v->offset = 0;
        v->file = 0;
        v->used = 0;
      }
      return 0;
    }
    ```
*   **`kernel/trap.c`**:
    ```c
    #include "sleeplock.h"
    #include "fs.h"
    #include "file.h"
    #include "fcntl.h"
    ```
    ```c
    } else if(r_scause() == 13){
      uint64 va = r_stval();
      uint64 pa;
      struct vma* v = 0;
      struct proc* p = myproc();
      uint pte_flags = 0;
      int r;

      for(int i = 0;i < NVMA; i++){
        v = &(p->vma[i]);
        if(va >= v->addr && va < v->addr + v->length && v->used == 1)
          break;
      }
      if(v->used == 0){
        printf("trap: no avaliable vma\n");
        p->killed = 1;
      } else {
        if((pa = (uint64)kalloc()) == 0)
          panic("trap: kalloc");
        memset((void*)pa, 0, PGSIZE);

        ilock(v->file->ip);
        if((r = readi(v->file->ip, 0, (uint64)pa, PGROUNDDOWN(va)  - v->addr, PGSIZE)) <= 0)
          panic("trap: readi");
        // printf("readi: %d\n", r);
        iunlock(v->file->ip);

        if (v->prot & PROT_READ)
            pte_flags |= PTE_R;
        if (v->prot & PROT_WRITE)
            pte_flags |= PTE_W;
        if (v->prot & PROT_EXEC)
            pte_flags |= PTE_X;
        if(mappages(p->pagetable, va, PGSIZE, pa, pte_flags | PTE_U) != 0)
          panic("mmap: mappages");
      }
    ```
*   **`kernel/vm.c`**:
    ```c
    int
    uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
    {
        if((*pte & PTE_V) == 0)
          // panic("uvmcopy: page not present");
          continue;
    }
    ```
    ```c
    void
    uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
    {
        if((*pte & PTE_V) == 0)
          // panic("uvmunmap: not mapped");
          continue;
    }
    ```

















---

## 附录：`man 2 mmap`

`mmap` 是一个非常强大和重要的系统调用，它允许将文件或者设备的内容映射到进程的虚拟地址空间中。映射完成后，你就可以像访问普通内存数组一样，通过指针来读写文件内容，而无需使用 `read()` 和 `write()` 等系统调用，从而极大地提高了 I/O 效率。

---

### **名称 (NAME)**

`mmap`, `munmap` - 将文件或设备映射到内存，或解除映射

---

### **概要 (SYNOPSIS)**

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);

int munmap(void *addr, size_t length);
```

---

### **描述 (DESCRIPTION)**

`mmap()` 系统调用在调用进程的虚拟地址空间中创建一个新的映射。这个新映射的起始地址、长度、内存保护和标志等特性都由 `mmap()` 的参数指定。

核心思想是，`mmap()` 建立了一段虚拟内存地址和文件（或匿名内存）之间的关联。当程序访问这段内存时，操作系统内核会自动处理数据的换页（Paging）操作，即按需从文件中加载数据到物理内存，或者将修改过的数据写回文件。

`munmap()` 系统调用则用于删除指定地址范围内的映射。当不再需要这段映射时，调用 `munmap()` 来释放它是一个好习惯。当进程终止时，所有映射也会被自动解除。

---

### **参数详解 (PARAMETERS)**

#### `mmap()` 的参数：

*   `void *addr`
    *   **建议的**映射起始地址。通常情况下，我们将其设置为 `NULL`，表示让内核自己选择一个合适的、未被占用的地址来创建映射。这能让程序更具可移植性。
    *   如果确实需要指定地址（例如，使用 `MAP_FIXED` 标志），内核会尝试使用你提供的地址。

*   `size_t length`
    *   映射区域的长度（以字节为单位）。这个值必须大于 0。

*   `int prot` (Protection)
    *   描述映射区域的内存保护标志，即这块内存**可以**做什么操作。它是以下值的按位或（bitwise OR）组合：
        *   `PROT_EXEC`：页面可被执行（代码）。
        *   `PROT_READ`：页面可被读取。
        *   `PROT_WRITE`：页面可被写入。
        *   `PROT_NONE`：页面不可访问。

*   `int flags`
    *   这是一个非常重要的参数，用于指定映射的类型和行为。常用的标志有：
        *   `MAP_SHARED`：**共享映射**。对这块内存的修改对其他映射了同一个文件的进程是可见的。对于文件映射，修改的内容最终会被写回到磁盘上的文件。这是实现进程间通信（IPC）的一种高效方式。
        *   `MAP_PRIVATE`：**私有映射**。对这块内存的修改不会影响其他进程，也不会写回到原始文件。内核会使用“写时复制”（Copy-on-Write）技术。当进程试图写入时，内核会为该进程创建一个修改页面的私有副本。
        *   `MAP_ANONYMOUS` (或 `MAP_ANON`)：**匿名映射**。创建一个不与任何文件关联的映射，其内容被初始化为零。此时，参数 `fd` 应为 -1，`offset` 会被忽略。这常用于像 `malloc` 一样分配大块内存。
        *   `MAP_FIXED`：强制使用 `addr` 参数指定的地址。如果该地址范围已被占用，`mmap()` 会失败。这是一个危险的选项，应谨慎使用。

*   `int fd` (File Descriptor)
    *   要映射的文件的文件描述符。这个文件必须是以支持 `mmap` 的模式打开的（例如，读模式、读写模式）。
    *   如果使用了 `MAP_ANONYMOUS`，这个参数必须是 **-1**。

*   `off_t offset`
    *   文件映射的起始偏移量。它必须是系统页大小（page size）的整数倍。你可以通过 `sysconf(_SC_PAGE_SIZE)` 来获取页大小。

#### `munmap()` 的参数：

*   `void *addr`
    *   要解除映射的内存区域的起始地址，通常是之前 `mmap()` 返回的指针。
*   `size_t length`
    *   要解除映射的区域的长度。

---

### **返回值 (RETURN VALUE)**

*   `mmap()`
    *   **成功时**，返回一个指向映射区域的指针。
    *   **失败时**，返回 `MAP_FAILED`（其定义通常是 `(void *) -1`），并设置全局变量 `errno` 来指示错误原因。

*   `munmap()`
    *   **成功时**，返回 0。
    *   **失败时**，返回 -1，并设置 `errno`。

---

### **主要用途 (KEY USE CASES)**

1.  **高效的文件 I/O**：对于大文件的读写，通过 `mmap` 可以避免 `read`/`write` 的系统调用开销和内核/用户空间之间的数据拷贝，性能更高。
2.  **进程间通信 (IPC)**：两个或多个进程可以通过 `mmap` 一个相同的文件（使用 `MAP_SHARED` 标志），从而获得一块共享内存。一个进程对内存的修改会立刻对其他进程可见。
3.  **动态内存分配**：使用 `MAP_ANONYMOUS` 标志的 `mmap` 可以作为 `malloc` 的替代品，尤其适合分配非常大的内存块，因为它直接向内核申请内存页，绕过了 `malloc` 库的管理机制。

---

### **示例代码 (EXAMPLE)**

下面的例子演示了如何使用 `mmap` 将一个文件映射到内存，修改它，然后验证修改是否被写回了文件。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    const char *filepath = "mmap_test.txt";
    const char *text = "Hello, mmap world!";
    const char *new_text = "Hi"; // 要写入的新内容，长度为2

    // 1. 创建并写入一个测试文件
    int fd = open(filepath, O_RDWR | O_CREAT | O_TRUNC, (mode_t)0600);
    if (fd == -1) {
        perror("Error opening file for writing");
        exit(EXIT_FAILURE);
    }
    
    // 写入初始内容，确保文件有足够大小
    if (write(fd, text, strlen(text)) == -1) {
        perror("Error writing to file");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // 获取文件大小
    struct stat fileInfo;
    if (fstat(fd, &fileInfo) == -1) {
        perror("Error getting the file size");
        close(fd);
        exit(EXIT_FAILURE);
    }
    size_t file_size = fileInfo.st_size;

    // 2. 将文件映射到内存
    // addr=NULL: 内核选择地址, length=file_size, prot=读写, flags=共享映射, fd=文件描述符, offset=0
    char *map = mmap(NULL, file_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (map == MAP_FAILED) {
        perror("Error mmapping the file");
        close(fd);
        exit(EXIT_FAILURE);
    }

    printf("文件初始内容: %s\n", text);
    printf("映射到内存后的内容: %s\n", map);

    // 3. 像操作内存一样修改文件内容
    printf("通过指针修改映射内存的前%zu个字节为 '%s'...\n", strlen(new_text), new_text);
    // 将 "Hello" 的前两个字符替换为 "Hi"
    strncpy(map, new_text, strlen(new_text));

    // 4. 将修改同步回磁盘文件
    // msync() 可以强制同步，但通常在 munmap 时或内核决定时会自动同步
    if (msync(map, file_size, MS_SYNC) == -1) {
        perror("Could not sync the file to disk");
    }

    printf("修改后，映射内存的内容: %s\n", map);

    // 5. 解除映射
    if (munmap(map, file_size) == -1) {
        perror("Error un-mmapping the file");
        // 继续执行清理
    }

    // 6. 关闭文件描述符
    close(fd);

    // 7. 验证文件内容是否真的被修改
    fd = open(filepath, O_RDONLY);
    char buffer[100];
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer) - 1);
    buffer[bytes_read] = '\0';
    printf("重新读取文件，磁盘上的内容为: %s\n", buffer);
    close(fd);
    
    // 清理测试文件
    remove(filepath);

    return 0;
}
```

**编译和运行:**

```sh
gcc mmap_example.c -o mmap_example
./mmap_example
```

**预期输出:**

```
文件初始内容: Hello, mmap world!
映射到内存后的内容: Hello, mmap world!
通过指针修改映射内存的前2个字节为 'Hi'...
修改后，映射内存的内容: Hillo, mmap world!
重新读取文件，磁盘上的内容为: Hillo, mmap world!
```