#  Lab2: system calls

花费时间：7h

## System call tracing(难度：moderate)

最初遇到的问题是 `make qemu` 报错，原因是 **trace** 系统调用未定义。解决方法是在相关文件中定义所有必要的追踪信息。完成这些定义后，运行 `trace 32 grep hello README` 就不再报错了。

本节的核心工作主要有两部分：

1.  在 `kernel/sysproc.c` 中添加 `sys_trace()` 函数：此函数旨在将参数保存到 **proc** 结构体（参见 `kernel/proc.h`）中的新变量里。
2.  修改 `kernel/syscall.c` 中的 `syscall()` 函数：此修改是为了实现追踪输出的打印。

这两点花了我大部分时间，大约一个小时。我最初编写的 `sys_trace` 函数如下：

```c
// kernel/sysproc.c
//将参数保存到proc结构体里的一个新变量来实现新的系统调用（需要实现）
//在syscall.c中返回值会放在p->trapframe->a0
uint64 sys_trace(void)
{
    //读取给的第一个系统参数，就是 掩码mask，存放在n中
    int mask;
    if (argint(0, &mask) < 0) {
        printf("sys_trace read mask failed\n");
        return -1;
    }

    return mask;
}
```

我当时不知道如何将掩码保存到 `proc` 结构体中。查阅 AI 后，我发现需要使用 `myproc` 函数。通过这个函数，我能够在 `sys_trace` 函数中将掩码保存到 `proc` 结构体里。

修改后的 `kernel/syscall.c` 实现了追踪逻辑：

```c
// kernel/syscall.c
static char* sys_call_name[] = {"fork",/* ... 补全数组 ... */};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    /*-------------core code-------------*/
    if (((p->trace_mask >> num) & 1) == 1) {
        printf("%d : syscall %s -> %d\n", p->pid, sys_call_name[num-1], p->trapframe->a0);
    }
    /*-------------core code-------------*/
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
至此，第一个小实验完成。

提交时发现了两个小 bug：一个是输出格式多了个空格导致判错，另一个是 `fork()` 功能未完全实现。`fork()` 问题的解决办法是修改 `fork()` 函数，将父进程的掩码传递给子进程。

-----

## Sysinfo(难度：moderate)

与 `trace` 系统调用类似，实现 `sysinfo` 也需要在各个文件中添加其信息才能编译通过。

下一步是实现在 `kernel/kalloc.c` 中的 `get_free_mem()` 和 `kernel/proc.c` 中的 `get_proc_num()`，然后是 `sys_sysinfo` 函数。

1.  **`get_proc_num()`**: 这个函数是参考 `proc.c` 中的 `allocproc()` 函数实现的。方法是遍历所有进程，并计算状态不是 `UNUSED` 的进程数量。

    ```c
    // kernal/proc.c
    uint64 get_proc_num(void)
    {
        // printf("get_proc_num called\n");
        int          cnt = 0;
        struct proc* p;

        for (p = proc; p < &proc[NPROC]; p++) {
            acquire(&p->lock);
            if (p->state != UNUSED) 
                cnt++;
            release(&p->lock);
        }
        return cnt;
    }
    ```

2.  **`get_free_mem()`**: 这个函数借鉴了 `kalloc.c` 文件中的 `kalloc()` 函数。xv6 中物理内存的分配是通过链表数据结构进行的，其中 `freelist` 指向下一个空闲页内存（大小为 4096 字节）。因此，任务是遍历这个链表并计数，以获取空闲内存量。

    ```c
    struct run {
      struct run *next;
    };

    struct {
      struct spinlock lock;
      struct run *freelist;
    } kmem;
    ```

    在实现过程中，有两点需要特别注意：

      * **必须在循环外部获取锁**，否则在循环过程中可能有其他程序修改链表状态，导致竞态条件。
      * 对 `r` 的条件判断出现了问题，导致在 `while(r)` 中出现无限循环。

    我最初编写的 **错误代码** `get_free_mem()`：


    ```c
    // 错误代码
    // kernal/proc.c
    uint64 get_free_mem(void)
    {
        printf("get_free_mem called\n");
        int         cnt = 0;
        struct run* r = kmem.freelist;
        while (r) {
            acquire(&kmem.lock);
            if (r->next) {
                r = r->next;
                cnt++;
            }
            release(&kmem.lock);
        }
        return cnt * PGSIZE;
    }
    ```

    **正确代码** `get_free_mem()`：

    ```c
    // 正确代码
    // kernel/kalloc.c
    uint64 get_free_mem(void)
    {
        // printf("get_free_mem called\n");
        int         cnt = 0;
        struct run* r = kmem.freelist;

        acquire(&kmem.lock); // 在循环外获取锁
        while (r) {
            r = r->next;
            cnt++;
        }
        release(&kmem.lock); // 在循环外释放锁
        return cnt * PGSIZE;
    }
    ```



3. `sys_sysinfo` 函数
   
    修改完这些问题后，在提交测试过程中又发现了两个错误：一个是进程计数不匹配（`sysinfotest: FAIL nproc is 16032 instead of 16033`），另一个是空闲内存值不正确（`FAIL: there is no free mem, but sysinfo.freemem=-24656`）。

    进程数最多为 64，为什么会显示 16032 个进程呢？我猜测大概是在返回 `info` 时没有设置正确的数字。经过排查，我发现在使用 `copyout` 将 `sysinfo` 结构体传回用户空间时，我传递了一个 `sysinfo` 结构体的指针（难怪 `clang` 静态分析一直报错）。传回指针后，使用 `sizeof(*info)` 只会有 8 字节，导致没有将所有数据传回。这说明我的 C 语言基础还需要提高啊。

    **正确代码** `sys_sysinfo()`：

    ```c
    // kernel/sysproc.c
    uint64 sys_sysinfo(void)
    {
        printf("sys_sysinfo called\n");

        uint64 info_addr;

        if (argaddr(0, &info_addr) < 0) {
            printf("sysinfo read user argv failed\n");
            return -1;
        }

        struct sysinfo info; // 创建一个局部结构体
        info.freemem = get_free_mem();
        info.nproc = get_proc_num();
        struct proc* p = myproc();

        // 将局部结构体的内容完整拷贝到用户空间
        if (copyout(p->pagetable, info_addr, (char*)&info, sizeof(info)) < 0)
            return -1;
        return 0;
    }
    ```