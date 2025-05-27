# Lab3: page tables

**花费时间：** 20h

---

## Print a page table (难度：easy) 3h

> 定义一个名为 `vmprint()` 的函数。它应当接收一个 `pagetable_t` 作为参数，并以下面描述的格式打印该页表。在 `exec.c` 中的 `return argc` 之前插入 `if(p->pid==1) vmprint(p->pagetable)`，以打印第一个进程的页表。如果你通过了 `pte printout` 测试的 `make grade`，你将获得此作业的满分。

不难的实验，主要是为了后面的实验做准备吧。主要是根据 `freewalk()` 函数改的：
1.  `freewalk()` 函数是一个利用递归对一个页表的所有内容进行清理的函数。
2.  借鉴 `freewalk()` 函数和 `walk()` 函数。其中 `freewalk()` 函数给了遍历所有条目的思路，`walk()` 函数给了遍历三个页表的思路。注意设置递归退出条件就行。代码如下：

```c
// kernel/vm.c
void vmprint(pagetable_t pagetable) {
  printf("vmprint called\n");
  printf("page table %p\n", pagetable);
  int level = 2;
  vmprint_r(pagetable, level);
}

void vmprint_r(pagetable_t pagetable, int level) {
  if (level < 0)
    return;

  // there are 2^9 = 512 PTEs in a page table.
  for (int i = 0; i < 512; i++) {
    // 获取页表条目
    pte_t pte = pagetable[i];
    // valid标志位为1
    //
    if ((pte & PTE_V)) {
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      if (level == 2) {
        printf("..");
      } else if (level == 1) {
        printf(".. ..");
      } else if (level == 0) {
        printf(".. .. ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      vmprint_r((pagetable_t)child, level - 1);
    }
  }
}
```

> 根据文本中的图3-4解释 `vmprint` 的输出。page 0包含什么？page 2中是什么？在用户模式下运行时，进程是否可以读取/写入page 1映射的内存？

其中 `page 0`、`page 1`、`page 2` 指的是 `vmprint` 输出中的：
```shell
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

*   **`page 0` 包含什么？**
    *   由于虚拟地址低，根据图3-4，应该是 `/init` 的 `text`，也就是指令段，存放程序的指令。所以 `page 0` 包含 `/init` 程序的**代码 (指令)**，也就是 `/init` 程序的入口点 (`main` 函数的开头)。
*   **`page 2` 中是什么？**
    *   由于虚拟地址低，根据图3-4，一般来说用于存放 `/init` 程序的 `data` 段，也就是数据段。`page 2` 包含 `/init` 程序的**全局/静态数据**。
*   **在用户模式下运行时，进程是否可以读取/写入 `page 1` 映射的内存？**
    *   我们来看 `page 1` 的页表项 `pte 0x0000000021fda00f`。
    *   其最低位的标志是 `0f` (二进制 `00001111`)。
        *   `V` (Valid，有效) 位是 1。
        *   `R` (Readable，可读) 位是 1。
        *   `W` (Writable，可写) 位是 1。
        *   `X` (Executable，可执行) 位是 1。
        *   **`U` (User-accessible，用户可访问) 位是 0。**
    *   **答案：** 由于 `page 1` 的页表项中，`U` (用户可访问) 标志位是 **0**，这意味着这个页面是**内核独占的**。在用户模式下运行的进程**不可以**读取或写入 `page 1` 映射的内存。尝试访问它会导致页错误 (Page Fault)。这可能是一个 xv6 特定的设计，比如在 `/init` 程序的地址空间中，某些页面（例如，也许是某些内核辅助数据页或者未被用户模式访问的保留页）被映射但没有赋予用户模式访问权限。

```shell
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000  // 对应虚拟页 0 (VA 0x0)
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000  // 对应虚拟页 1 (VA 0x1000)
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000  // 对应虚拟页 2 (VA 0x2000)
```

---

## A kernel page table per process (难度：hard) （15h）

> 你的第一项工作是修改内核来让每一个进程在内核中执行时使用它自己的内核页表的副本。修改 `struct proc` 来为每一个进程维护一个内核页表，修改调度程序使得切换进程时也切换内核页表。对于这个步骤，每个进程的内核页表都应当与现有的的全局内核页表完全一致。如果你的 `usertests` 程序正确运行了，那么你就通过了这个实验。

写了一天，才把整体内容写的差不多。其中复习了好几遍内核栈，还有相关概念。

**遇到的 Bug 及解决过程：**

1.  **第一个 bug** 出现在了 `proc_kvminit()` 函数中。不能使用 `kvmmap` 函数。因为 `kvmmap` 函数会直接在 `kernel_pagetable` 中设置页表，但是我们需要在进程自己的内核页表中实现。因此 `proc_kvminit()` 内部需要自己单独实现。

2.  **第二个 bug** 出现在了 `procinit()` 迁移后的内容中。一开始写的代码如下。发现 `mappages()` 函数内部会报错。锁定在了 `walk` 内部实现的 `if(*pte & PTE_V)` 上。
    ```c
      char *pa = kalloc();
      if (pa == 0)
        panic("kalloc");
      uint64 va = KSTACK(0);
      if (mappages(p->kn_pagetable, va, PGSIZE, (uint64)pa, PTE_R | PTE_W) != 0)
        panic("allocproc");
      p->kstack = va;
    ```
    报错信息：
    ```shell
    scause 0x000000000000000d
    sepc=0x0000000080001072 stval=0x00000000000007f8
    panic: kerneltrap
    ```
    反复 debug 发现不是出错在了上面那个部分。反而是出错在了 `kn_pagetable` 的分配上。照抄 `kvminit` 函数，然而 `kvminit` 函数里面的 `kernel_pagetable` 是全局定义的。我按照这么写，虽然在函数内部给 `p->kn_pagetable` 分配了内存，然而出去之后，在 `allocproc` 函数里面，`p->kn_pagetable` 的地址依然是 `0x0`。导致后面出现了内存访问错误。

3.  **第三个 bug** 出现在 `scheduler()`，在输出 "hart 1 starting" 和 "hart 2 starting" 后，会卡一会弹出一大串 `qemu-system-riscv64: clint: invalid write:`。假如注释掉添加的代码会卡住。
    ```c
      w_satp(MAKE_SATP(p->kn_pagetable));
      sfence_vma();
    ```
    通过排查，发现是在将 `procinit` 的内容移植到 `allocproc` 的时候，自作主张的将 `uint64 va = KSTACK((int)(p - proc))` 中改动 `(int)(p - proc)` 改为了 `1`。我一开始是觉得既然每个进程有自己的内核页表，那可以把内核栈都放在最高的位置上。
    但是事实上，将所有进程的内核栈虚拟地址（`p->kstack`）设置为同一个值（如 `KSTACK(1)`）是一个不好的实践，也违背了 xv6 中 `KSTACK(pnum)` 的设计意图。虽然每个进程的页表可以将这个相同的 VA 映射到不同的 PA，从而在物理上隔离数据，但这：
    *   失去了虚拟地址的隔离性：使得不同进程的内核栈在虚拟地址空间中重叠。这对于调试和系统的某些内部逻辑可能是不利的，也可能与内核对虚拟地址布局的某些隐式期望冲突。
    *   可能与某些内核机制冲突：如果内核的某些部分依赖于 `p->kstack` 的虚拟地址的唯一性来做判断或查找，就会出错。
    将代码改回来之后 bug 消失。

4.  **第四个 bug** 是启动后出现 `panic("kvmpa")`。然后一直不知道什么原因一直查，最后才发现 `kvmpa` 本身就是一个需要我们改的函数。因为他是使用 `kernel_pagetable` 去解读虚拟地址的，当然会出错。

5.  **第五个 bug** 是 `freeproc` 中对于内存的释放。一开始照着 `trapframe` 用的 `kfree` 释放，但是 `kfree` 用的是物理地址。我们需要用虚拟地址。用 `uvmunmap` 就好了。

**`usertests` 测试结果：**

不 OK 的报错有如下：
```shell
test reparent2: panic: etext
test twochildren: panic: kalloc
test unlinkread: panic: etext
test sbrkmuch: sbrkmuch: sbrk test failed to grow big address space; enough phys mem?
FAILED
```

单独测试结果如下。有两个测试会少释放 146 个 pages。
```shell
$ usertests sbrkmuch
usertests starting
test sbrkmuch: OK
FAILED -- lost some free pages 32085 (out of 32231)

$ usertests unlinkread
usertests starting
test unlinkread: OK
FAILED -- lost some free pages 31793 (out of 31939)

$ usertests twochildren
usertests starting
test twochildren: 2.twochildren: fork failed
FAILED
SOME TESTS FAILED

$ usertests reparent2
panic: TRAMPOLINE
QEMU: Terminated
```

将问题锁定在了释放内核页表的函数。最后发现是没有对页表本身进行释放。这里重新梳理了一下几个相关函数的真实作用，说明做到这里的时候其实还是对于 `pagetable` 和 `pte` 之间在代码中的关系不是特别清晰。

### 思路和代码

> 在 `struct proc` 中为进程的内核页表增加一个字段

和用户自己的进程页表一样，在结构体中创建一个进程的内核页表。
```c
// kernel/proc.h
struct proc {
  // ... (existing fields)
  // kernel pagetable
  pagetable_t kn_pagetable;
};
```

> 为一个新进程生成一个内核页表的合理方案是实现一个修改版的 `kvminit`，这个版本中应当创造一个新的页表而不是修改 `kernel_pagetable`。你将会考虑在 `allocproc` 中调用这个函数

1.  查看 `kvminit()` 函数。该函数用于创建全局的 `kernel_pagetable`。
    1.  首先用 `kalloc()` 为 `kernel_pagetable` 的 L2 级页表（最高级）分配物理内存空间，并为其初始化。
    2.  使用 `kvmmap()` 对内核代码、数据、IO 设备等区域进行映射，内核页表为恒等映射。其中 `kvmmap()` 调用了 `mappages()` 函数，`mappages()` 会根据需要，在全局的 `kernel_pagetable` 的 L2 页表下，自动分配并填充 L1 或 L0 级别的页表页，最终建立起从虚拟地址到这些共享物理内存的映射。这些 L1、L0 页表页也是通过 `kalloc()` 动态分配的。
2.  为了实现为一个新进程生成一个内核页表，我们需要参考 `kvminit()` 函数。由于 `kvmmap()` 专门用于对全局的 `kernel_pagetable` 分配映射，我们创建一个自己的 `uvmmap()` 函数。
    ```c
    // kernel/vm.c
    // add a mapping to the user page table.
    // does not flush TLB or enable paging.
    void uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
    {
        if (mappages(pagetable, va, sz, pa, perm) != 0)
            panic("uvmmap");
    }

    // kernel/defs.h
    void uvmmap(pagetable_t, uint64, uint64, uint64, int);
    ```

3.  参考 `kvminit()` 函数，写 `proc_knpagetable()` 函数。 *(注：您在allocproc中实际调用的是 `proc_kvminit`, 这里是您之前写的 `proc_knpagetable`)*
    ```c
    // kernel/vm.c
    pagetable_t proc_knpagetable(struct proc* p)
    {
        pagetable_t kn_pagetable;

        // An empty page table.
        kn_pagetable = uvmcreate();
        if (kn_pagetable == 0)
            return 0;

        // uart registers
        uvmmap(kn_pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

        // virtio mmio disk interface
        uvmmap(kn_pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

        // CLINT
        uvmmap(kn_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

        // PLIC
        uvmmap(kn_pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

        // map kernel text executable and read-only.
        uvmmap(kn_pagetable, KERNBASE, KERNBASE, (uint64)etext - KERNBASE, PTE_R | PTE_X);

        // map kernel data and the physical RAM we'll make use of.
        uvmmap(kn_pagetable, (uint64)etext, (uint64)etext, PHYSTOP - (uint64)etext, PTE_R | PTE_W);

        // map the trampoline for trap entry/exit to
        // the highest virtual address in the kernel.
        uvmmap(kn_pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

        return kn_pagetable;
    }
    // kernel/vm.c
    ```

4.  在 `allocproc` 中为进程初始化的时候添加初始化进程页表的代码。
    ```c
    // kernel/proc.c
    static struct proc* allocproc(void)
    {
      struct proc *p;
      // ... (existing code from user, assuming ellipses cover loop and lock acquisition)
      // return 0; // This was likely part of the ... or commented out

    found:
      p->pid = allocpid();

      // ... (existing code from user)

      // An empty user page table.
      p->pagetable = proc_pagetable(p);
      if(p->pagetable == 0){
        freeproc(p);
        release(&p->lock);
        return 0;
      }

      // An empty user kernel pagetable
      p->kn_pagetable = proc_kvminit(p); // Actual name used in user's bug description
      if(p->kn_pagetable == 0){
        freeproc(p);
        release(&p->lock);
        return 0;
      }

      // ... (existing code from user)

      return p;
    }
    ```

> 确保每一个进程的内核页表都关于该进程的内核栈有一个映射。在未修改的XV6中，所有的内核栈都在 `procinit` 中设置。你将要把这个功能部分或全部的迁移到 `allocproc` 中。

1.  **内核栈：** 当一个进程从用户模式进入内核模式执行时，内核代码所使用的栈空间。
    1.  当发生以下情况时，CPU会从用户模式切换到内核模式，并开始执行内核代码：
        *   系统调用 (`System Call`)
        *   硬件中断 (`Interrupt`)
        *   异常 (`Exception`)
    2.  内核栈用途：
        *   保存从用户态切换到内核态时的CPU状态（一部分可能保存在 `trapframe` 中，`trapframe` 本身可能就位于内核栈的顶部）。
        *   内核函数调用时的返回地址、参数、局部变量等。例如，当处理一个系统调用时，内核可能会调用多个内部函数，这些函数调用的信息就都存储在当前进程的内核栈上。
        *   **每个进程都有内核栈**。每个进程在陷入内核时，都需要自己独立的内核栈。这样，即使有多个进程同时处于内核态（比如一个在等待I/O，另一个在处理中断），它们各自的内核函数调用和状态也不会相互干扰。
2.  首先看下 `procinit()` 原来的代码。未修改的 xv6 中，`procinit` 遍历每一个进程，在物理内存上为每一个进程的内核栈分配空间，然后在全局 `kernel_pagetable` 上为每个进程在内核的虚拟地址空间中分配一个唯一的、不重叠的区域作为其内核栈，并且可能带有保护页（guard page）以防止栈溢出。
    ```c
    // kernel/proc.c
    // initialize the proc table at boot time.
    void
    procinit(void)
    {
      struct proc *p;
      
      initlock(&pid_lock, "nextpid");
      for(p = proc; p < &proc[NPROC]; p++) {
          initlock(&p->lock, "proc");

          // Allocate a page for the process's kernel stack.
          // Map it high in memory, followed by an invalid
          // guard page.
          char *pa = kalloc();
          if(pa == 0)
            panic("kalloc");
          uint64 va = KSTACK((int) (p - proc));
          kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
          p->kstack = va;
      }
      kvminithart();
    }
    ```
3.  我们现在为每个进程创建了自己的内核页表，因此不需要在全局的内核页表中为每个进程创建内核栈，而只需要在他们自己的内核页表中为他们自己创建一个内核栈。
    1.  将原来 `procinit` 中的创建内核栈代码注释掉。
        ```c
        // kernel/proc.c
        // initialize the proc table at boot time.
        void
        procinit(void)
        {
          struct proc *p;
          
          initlock(&pid_lock, "nextpid");
          for(p = proc; p < &proc[NPROC]; p++) {
              initlock(&p->lock, "proc");

              // // Allocate a page for the process's kernel stack.
              // // Map it high in memory, followed by an invalid
              // // guard page.
              // char *pa = kalloc();
              // if(pa == 0)
              //   panic("kalloc");
              // uint64 va = KSTACK((int) (p - proc));
              // kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
              // p->kstack = va;
          }
          kvminithart();
        }
        ```
    2.  在上面写的创建进程的内核页表下面添加移植的代码。
        ```c
        // kernel/proc.c

          // An empty user kernel pagetable
          p->kn_pagetable = proc_kvminit(p);
          if(p->kn_pagetable == 0){
            freeproc(p);
            release(&p->lock);
            return 0;
          }

          // 新增
          char *pa = kalloc();
          if (pa == 0)
            panic("kalloc");
          uint64 va = KSTACK((int)(p - proc));
          uvmmap // This line is incomplete in the original text.
          p->kstack = va;
        ```

> 修改 `scheduler()` 来加载进程的内核页表到核心的 `satp` 寄存器(参阅 `kvminithart` 来获取启发)。不要忘记在调用完 `w_satp()` 后调用 `sfence_vma()`。
> 没有进程运行时 `scheduler()` 应当使用 `kernel_pagetable`。

1.  在初始化完第一个进程之后，CPU 会进入 `scheduler` 循环执行每一个需要执行的进程。首先 CPU 会查找需要运行的进程 (`RUNNABLE`)，然后把目前的进程设置为这个进程。
    ```c
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
    ```
2.  `swtch` 函数是用汇编写的函数，具体可见 `swtch.S`。`swtch` 函数的作用是上下文切换 (`context switch`)，保存当前寄存器到 `old` 指向的内存区域，然后从 `new` 指向的内存区域加载寄存器。将 CPU 目前的上下文（寄存器数据，`context`）保存起来，然后加载进程的上下文，将控制权转交给进程，并通过 `ret` 指令跳转到新上下文的执行点。其中 `ret` 返回的地址保存在 `p->context->ra` 中，在 `allocproc` 中被设置为 `forkret()` 函数。
3.  在未修改的 xv6 当中，在内核态运行的时候，`satp` 寄存器使用的是全局的内核页表。当 `swtch(&c->context, &p->context)` 执行完毕，控制权会从进程 `p` 返回到 `scheduler` 函数（即进程 `p` 主动调用 `yield()` 或因时钟中断等被切换出去）。
4.  由于我们为每个函数建立了自己的内核页表。我们希望在运行进程 `p` 之前切换到 `p->kn_pagetable`，确保了进程 `p` 在其自己的内核地址空间上下文中执行（特别是对于它自己的内核栈映射）。当进程 `p` 执行完毕（例如，它调用了 `yield()`，最终导致 `swtch` 返回到 `scheduler` 的上下文 `c->context`），立即将 `satp` 切换回全局的 `kernel_pagetable`。

有了以上过程，我们写出代码如下：
```c
// kernel/proc.c
w_satp(MAKE_SATP(p->kn_pagetable));
sfence_vma();

swtch(&c->context, &p->context);

w_satp(MAKE_SATP(kernel_pagetable));
sfence_vma();
```

> 在 `freeproc` 中释放一个进程的内核页表。
> 你需要一种方法来释放页表，而不必释放叶子物理内存页面。

1.  在前面的过程中，我们不仅创建了内核页表，同时也在其上面做了一些与物理内存的相关映射。这意味着我们不仅有 L2、L1、L0 页表，同时 L0 级页表的 `PTE` 还与物理内存上的一些内容相关联。
2.  用户自己的进程页表：
    *   先对 `TRAMPOLINE` 和 `TRAPFRAME` 这两个所有页表都会共同映射的区域，我们用 `uvmunmap()` 函数去除掉进程页表与他们物理内存的关联关系。这一过程只会将 L0 页表中相关的 `PTE` 置为0，而不会删除叶子物理内存。
    *   然后我们通过 `uvmfree()` 函数释放进程页表。
        *   先用 `uvmunmap()` 函数去除掉进程页表与他们物理内存的关联关系，同时释放相关的物理内存。这一过程只会将 L0 页表中相关的 `PTE` 置为0，并且删除叶子物理内存。
        *   调用 `freewalk()` 函数，通过递归的方式，从 L2 级页表向下查找到 L0 页表，假如 `PTE` 为0，则释放掉页表本身，然后返回。由于我们前面已经将 `PTE` 置为0，所以假如这里还有页表与物理内存有关联，则会报错。然后通过递归一层层删掉三级所有页表。
    *   最后将页表地址置为0。
3.  用户内核页表的初始化：
    *   用户内核页表中，分为多个部分。
        *   一方面：对于内核页表中的进程栈，我们在物理内存上分配了相应的位置。在释放的时候需要释放相关的物理内存。
        *   一方面：对于内核页表中内核代码、数据、IO 设备等区域，我们没有在物理内存上分配相应位置，而只是建立了映射。在释放的时候只需要释放相应的映射。
        *   一方面：需要释放内核页表本身的骨架。

在 `freeproc()` 函数中加入下面的代码。同时创建两个新的函数。`proc_kfreepagetable()` 用于释放内核页表及其映射，`knfreewalk()` 参考 `freewalk()` 函数，释放页表本身及其骨架。
```c
// kernel/proc.c 
// freeproc()
if(p->kstack)
  uvmunmap(p->kn_pagetable, p->kstack, 1, 1);
p->kstack = 0;

if(p->kn_pagetable)
  proc_kfreepagetable(p);
p->kn_pagetable = 0;

// kernel/proc.c 
void proc_kfreepagetable(struct proc *p)
{
    uvmunmap(p->kn_pagetable, UART0, 1, 0);
    uvmunmap(p->kn_pagetable, VIRTIO0, 1, 0);
    uvmunmap(p->kn_pagetable, PLIC, 0x400000 / PGSIZE, 0);
    uvmunmap(p->kn_pagetable, CLINT, 0x10000 / PGSIZE, 0);
    uvmunmap(p->kn_pagetable, KERNBASE, ((uint64)etext - KERNBASE) / PGSIZE, 0);
    uvmunmap(p->kn_pagetable, (uint64)etext, (PHYSTOP - (uint64)etext) / PGSIZE, 0);
    uvmunmap(p->kn_pagetable, TRAMPOLINE, 1, 0);
    knfreewalk(p->kn_pagetable);
}

// kernel/proc.c 
// Recursively free proc kernel page-table pages.
void
knfreewalk(pagetable_t knpagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = knpagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      knfreewalk((pagetable_t)child);
      knpagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("knfreewalk: leaf");
    }
  }
  kfree((void*)knpagetable);
}
```

---

## Simplify `copyin`/`copyinstr` (难度：hard)

> 将定义在 ***kernel/vm.c*** 中的 `copyin` 的主题内容替换为对 `copyin_new` 的调用（在 ***kernel/vmcopyin.c*** 中定义）；对 `copyinstr` 和 `copyinstr_new` 执行相同的操作。为每个进程的内核页表添加用户地址映射，以便 `copyin_new` 和 `copyinstr_new` 工作。如果 `usertests` 正确运行并且所有 `make grade` 测试都通过，那么你就完成了此项作业。

写完代码跑了第一遍就开始报错。又是痛苦 debug 的一天。

debug 了一个晚上发现是思路错了。一开始我的想法是写函数来将找到用户页表对应的物理内存，然后在内核页表上让相应的 `VA` 和 `PA` 进行重新映射。然而实际上操作更简单，只需要为用 `walk` 为虚拟地址分配 `PTE`，然后将用户页表的 `PTE` 复制过去就行了。

### 思路和代码

> 先用对 `copyin_new` 的调用替换 `copyin()`，确保正常工作后再去修改 `copyinstr`。

```c
// kernel/vm.c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

> 在内核更改进程的用户映射的每一处，都以相同的方式更改进程的内核页表。包括 `fork()`, `exec()`, 和 `sbrk()`。
> 别忘了上面提到的 `PLIC` 限制。

在三个地方每个地方都将用户进程的映射复制到内核页表当中。
*   `growproc` 中，调用了 `uvmalloc` 函数，`sz` 会变成新的大小，因此在 `u2kvmcopy` 函数中要用 `sz-n` 来代表旧的大小。
*   `fork` 中，不能用把 `np->pagetable` 换成 `p->pagetable` 是因为可能子进程还没退出，父进程就退出了。这时候假如还读取父进程的用户列表会导致读取到不应该读取的内存，导致出现错误。

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
```c
// kernel/proc.c
void
userinit(void)
{
  // ... (existing code)
  p->sz = PGSIZE;

  u2kvmcopy(p->pagetable, p->kn_pagetable, 0, p->sz);
  // ... (existing code)
}

int
growproc(int n)
{
  // ... (existing code, assumes sz and p are defined)
    // PLIC limit
    if (PGROUNDUP(sz + n) >= PLIC){ // Assuming sz is current size before growth
      return -1;
    }
    // if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) { // Original had this logic, assuming sz is updated by uvmalloc
    //   return -1;
    // }
    // u2kvmcopy(p->pagetable, p->kn_pagetable, sz - n, sz); // Assuming sz is new size, and sz-n is old_sz
  // ... (existing code)
  // The code for growproc was not fully provided in the context of where u2kvmcopy is called.
  // This is a placeholder based on the description. The user's original placement:
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) { // Assuming sz here is old_sz, and uvmalloc returns new_sz
      return -1;
    }
    u2kvmcopy(p->pagetable, p->kn_pagetable, sz - n, sz); // If sz is new_sz, then old_sz was p->sz before uvmalloc
}                                                       // If the user text means sz (old_sz) and newsz (returned by uvmalloc)
                                                        // then u2kvmcopy(p->pagetable, p->kn_pagetable, p->sz /*oldsz*/, new_sz_from_uvmalloc);
                                                        // The provided "sz-n" implies 'sz' is the *new* size.

int
fork(void)
{
  // ... (existing code, assumes np and p are defined)
  np->sz = p->sz;

  u2kvmcopy(np->pagetable, np->kn_pagetable, 0, np->sz);
  // ... (existing code)
}
```

> 不要忘记在 `userinit` 的内核页表中包含第一个进程的用户页表。

```c
// kernel/proc.c
void
userinit(void)
{
  // ... (existing code, assumes p is defined)
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  u2kvmcopy(p->pagetable, p->kn_pagetable, 0, p->sz);
  // ... (existing code)
}
```

> 用户地址的 `PTE` 在进程的内核页表中需要什么权限？(在内核模式下，无法访问设置了 `PTE_U` 的页面）

对于 `copyin` 来说，最重要的就是 `PTE_U` 为0，`PTE_R` 为1。