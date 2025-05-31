# Lab5: xv6 lazy page allocation

**花费时间：** 5h

---

## Part 1: Eliminate allocation from `sbrk()` (难度：easy)

> 实验要求：删除`sbrk(n)`系统调用中的页面分配代码（位于`sysproc.c`中的函数`sys_sbrk()`）。`sbrk(n)`系统调用将进程的内存大小增加n个字节，然后返回新分配区域的开始部分（即旧的大小）。新的`sbrk(n)`应该只将进程的大小（`myproc()->sz`）增加n，然后返回旧的大小。它不应该分配内存——因此您应该删除对`growproc()`的调用（但是您仍然需要增加进程的大小！）。

此部分的实现参考了课堂演示。核心改动在于 `sys_sbrk()` 函数：当参数 `n > 0` 时，仅增加进程逻辑大小 `p->sz`，而不实际分配物理页面。

## Part 2: Lazy allocation (难度：moderate)

> 实验要求：修改`trap.c`中的代码以响应来自用户空间的页面错误，方法是新分配一个物理页面并映射到发生错误的地址，然后返回到用户空间，让进程继续执行。您应该在生成“`usertrap(): …`”消息的`printf`调用之前添加代码。你可以修改任何其他xv6内核代码，以使`echo hi`正常工作。

此部分的实现同样基于课堂演示的思路。主要是在 `usertrap` 函数中捕获由用户空间访问引起的页错误（`scause 13` 代表Load page fault, `scause 15` 代表Store/AMO page fault），并在此时按需分配物理页面、建立映射，然后返回用户空间继续执行。

## Part 3: Lazytests and Usertests (难度：moderate)

**耗时：** 5h

### 实现思路分析

> 实验要求：处理`sbrk()`参数为负的情况。
*   当 `sbrk()` 参数为正时，页错误机制会在首次访问相关页面时处理物理内存的分配。
*   当 `sbrk()` 参数为负时，进程的逻辑大小 `p->sz` 会减小。此时，原先大于新的 `p->sz` 但小于旧的 `p->sz` 的虚拟地址空间可能仍然保留着对物理页面的映射。若不及时解除这些映射，在进程退出时，`uvmfree` 可能无法回收这些物理页面，导致内存泄漏。因此，需要借鉴原版 `growproc()` 中处理 `n<0` 的逻辑，即调用 `uvmdealloc` 来释放这部分缩小的虚拟地址空间对应的物理资源和页表项。

> 实验要求：如果某个进程在高于`sbrk()`分配的任何虚拟内存地址上出现页错误，则终止该进程。

在页错误处理程序中，需要检查发生错误的虚拟地址 `va`。如果 `va` 大于当前进程的逻辑大小 `p->sz`，则认为是非法访问，应终止该进程。

> 实验要求：在`fork()`中正确处理父到子内存拷贝。

为支持惰性分配，在 `uvmcopy` 函数（`fork()` 时调用）中，如果父进程的某个页面是惰性分配且尚未实际映射到物理内存（即其PTE无效或PTE路径不存在），则子进程在创建时不应为该页面分配物理内存。实现上，当 `walk()` 在父进程页表中查找某虚拟地址对应的PTE失败（返回0）或PTE无效（`(*pte & PTE_V) == 0`）时，`uvmcopy` 应跳过对该页面的处理，直接 `continue` 到下一个页面。
示意代码：
```c
if((*pte & PTE_V) == 0)
  // panic("uvmcopy: page not present"); 
  continue; 
```

> 实验要求：处理这种情形：进程从`sbrk()`向系统调用（如`read`或`write`）传递有效地址，但尚未分配该地址的内存。

*   当系统调用（如 `read` 或 `write`）通过 `argaddr` (或类似函数) 获取用户传入的缓冲区地址时，该地址可能指向一个通过 `sbrk()` 扩展但尚未实际分配物理页的惰性区域。
*   在系统调用执行路径中，内核代码（如 `copyin`/`copyout`）通过 `walkaddr` 检查这类地址时，如果 `walkaddr` 仅仅检查页表而不触发CPU级别的内存访问，则不会发生页错误。
*   因此，修改 `walkaddr` 函数：当它发现一个地址在进程的有效虚拟地址范围 `p->sz` 内，但对应的PTE不存在或无效时，`walkaddr` 需要主动尝试分配物理页面并建立映射。这样，后续的 `copyin`/`copyout` 才能成功访问。
*   **注意点**：在 `walkaddr` 中实现按需分配时，若 `kalloc()` 返回的物理地址变量名与页错误处理程序中使用的变量名（如 `pa`）相同，可能会引发 `test copyinstr3` 等测试的失败。应使用不同的变量名（如 `mypa`）。

> 实验要求：如果在页面错误处理程序中执行`kalloc()`失败，则终止当前进程。

页错误处理程序中，在调用 `kalloc()` 后，需要检查其返回值。如果 `kalloc()` 返回 `0`（表示物理内存不足），则应将当前进程标记为 `p->killed = 1`，使其在后续的检查点被终止。
示意代码：
```c
if (pa == 0) { 
    p->killed = 1;
}
```

> 实验要求：处理用户栈下面的无效页面上发生的错误。

在页错误处理程序中，除了检查访问地址 `va` 是否超过 `p->sz`，还需检查 `va` 是否低于当前用户栈帧指针 `p->trapframe->sp` 所在的页的基地址 `PGROUNDDOWN(p->trapframe->sp)`。如果低于此地址，也视为非法访问，应终止进程。

### 代码实现

*   **`kernel/trap.c`** (页错误处理相关逻辑):
```c
  } else if((which_dev = devintr()) != 0){
    // ok
  }
  else if (r_scause() == 13 || r_scause() == 15) {
      uint64 va = r_stval();
      if (va > p->sz || va < PGROUNDDOWN(p->trapframe->sp)) {
          printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
          printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
          p->killed = 1;
          exit(-1);
      }
      uint64 pa = (uint64)kalloc();
      if (pa == 0) {
          p->killed = 1;
      }
      memset((void*)pa, 0, PGSIZE);
      va = PGROUNDDOWN(va);
      if (mappages(p->pagetable, va, PGSIZE, pa, PTE_W | PTE_R | PTE_U) < 0) {
          kfree((void*)pa);
          p->killed = 1;
      }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);
```

*   **`kernel/sysproc.c`** (`sys_sbrk` 实现):
```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(n > 0){
    myproc()->sz += n;
  } else if(n < 0){
    uint sz = myproc()->sz;
    sz = uvmdealloc(myproc()->pagetable, sz, sz + n);
    myproc()->sz = sz;
  }
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

*   **`kernel/vm.c`** (相关函数修改):
    *   确保 `vm.c` 包含必要的头文件，如 `proc.h` 以使用 `myproc()`。
    ```c
    #include "spinlock.h"
    #include "proc.h"
    ```
    *   修改 `walkaddr` 以支持惰性分配的按需映射：
    ```c
    uint64
    walkaddr(pagetable_t pagetable, uint64 va)
    {
      pte_t *pte;
      uint64 pa;
      struct proc *p = myproc();

      if(va >= MAXVA)
        return 0;

      pte = walk(pagetable, va, 0);
      // if(pte == 0)
      //   return 0;
      // if((*pte & PTE_V) == 0)
      //   return 0;
      if(pte == 0 || (*pte & PTE_V) == 0){
          if (va >= p->sz || va < PGROUNDDOWN(p->trapframe->sp))
              return 0;
          uint64 mypa = (uint64)kalloc();
          if (mypa == 0) 
              return 0;
          memset((void*)mypa, 0, PGSIZE);
          if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, mypa, PTE_W | PTE_R | PTE_U) < 0) {
              kfree((void*)mypa);
              return 0;
          }
      }
      if((*pte & PTE_U) == 0)
        return 0;
      pa = PTE2PA(*pte);
      return pa;
    }
    ```
    *   修改 `uvmunmap` 以正确处理可能未映射的惰性页面：
    ```c
    void
    uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
    {
      // other code
      for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
        if((pte = walk(pagetable, a, 0)) == 0)
          // panic("uvmunmap: walk");
          continue;
        if((*pte & PTE_V) == 0)
          continue;
        // other code
      }
    }
    ```
    *   修改 `freewalk` 以适应惰性分配场景：
    ```c
    void
    freewalk(pagetable_t pagetable)
    {
      // other code
        } else if(pte & PTE_V){
          // panic("freewalk: leaf");
          continue;
        }
    }
    ```
    *   修改 `uvmcopy` 以正确处理父进程中的惰性页面：
    ```c
    int
    uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
    {
      // other code
        if((pte = walk(old, i, 0)) == 0)
          // panic("uvmcopy: pte should exist");
          continue;
        if((*pte & PTE_V) == 0)
          // panic("uvmcopy: page not present");
          continue;
      // other code
    }
    ```