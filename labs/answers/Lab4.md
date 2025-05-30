# Lab4: traps

**花费时间：** 7h

---

## RISC-V assembly (难度：easy)

**耗时：** 0.5h

> 理解一点RISC-V汇编是很重要的，你应该在6.004中接触过。xv6仓库中有一个文件user/call.c。执行make fs.img编译它，并在user/call.asm中生成可读的汇编版本。
>
> 阅读call.asm中函数g、f和main的代码。RISC-V的使用手册在参考页上。以下是您应该回答的一些问题（将答案存储在answers-traps.txt文件中）：

### 问题与答案

> 1.  哪些寄存器保存函数的参数？例如，在main对printf的调用中，哪个寄存器保存13？

**答：**
八个整数寄存器a0-a7和八个浮点寄存器fa0-fa7。
printf汇编代码如下。可见是a2保存13。
```S
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b050513          	addi	a0,a0,1968 # 7d8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	600080e7          	jalr	1536(ra) # 630 <printf>
```

> 2.  main的汇编代码中对函数f的调用在哪里？对g的调用在哪里(提示：编译器可能会将函数内联）

**答：**
在 `main` 函数的汇编代码中，为了计算 `printf` 的第二个参数 `f(8)+1`，编译器进行了优化。它在编译时计算出 `f(8)+1` 的结果是 `12`，因此直接使用指令 `26: li a1,12` 将 `12` 加载到 `a1` 寄存器。所以，在 `main` 中**没有**为这个表达式生成对 `f` 函数的实际调用指令。

函数 `g` 的调用在函数 `f` 的实现中被内联了。在 `f` 函数的汇编代码 `0xe <f>` 中，指令 `14: addiw a0,a0,3` 就是 `g` 函数 `return x+3;` 功能的内联实现。

> 3.  printf函数位于哪个地址？

**答：**
位于call.asm中的虚拟地址630。
```S
0000000000000630 <printf>:

void
printf(const char *fmt, ...){}
```

> 4.  在main中printf的jalr之后的寄存器ra中有什么值？

**答：**
`ra` 中有 `printf` 函数结束后应该返回的地址，也就是调用 `printf` 函数指令的下一条指令的地址。

> 5.  运行以下代码。
> ```c
> unsigned int i = 0x00646c72;
> printf("H%x Wo%s", 57616, &i);
> ```
> 程序的输出是什么？这是将字节映射到字符的ASCII码表。
>
> 输出取决于RISC-V小端存储的事实。如果RISC-V是大端存储，为了得到相同的输出，你会把i设置成什么？是否需要将57616更改为其他值？

**答：**
程序的输出是：**`HE110 World`**

**解释：**
1.  `57616` 的十六进制表示是 `0xE110`。所以 `printf` 中的 `%x` 会输出 `E110`。
2.  `unsigned int i = 0x00646c72;`。在小端RISC-V系统中，这个整数在内存中从低地址到高地址的字节顺序是：`72 6c 64 00`。
3.  `&i` 是 `i` 的地址。当 `%s` 接收 `&i` 时，它会将其解释为指向一个字符串的指针。
4.  根据ASCII码表：
    *   `0x72` 是 'r'
    *   `0x6c` 是 'l'
    *   `0x64` 是 'd'
    *   `0x00` 是空字符 (字符串结束符)。
5.  因此，`%s` 会打印字符串 "rld"。

**如果RISC-V是大端存储，为了得到相同的输出 "rld"：**
*   你需要将 `i` 设置为 **`0x726c6400`**。这样，在大端系统中，内存从低地址到高地址的字节顺序会是 `72 6c 64 00`，从而 `%s` 会打印 "rld"。
*   **不需要**将 `57616` 更改为其他值。`%x` 格式化的是整数的数值，与字节序无关。

> 6.  这里有一个小端和大端存储的描述和一个更异想天开的描述。
> 在下面的代码中，“y=”之后将打印什么(注：答案不是一个特定的值）？为什么会发生这种情况？
> ```c
> printf("x=%d y=%d", 3);
> ```

**答：**
"y=" 之后将打印一个**不确定的、通常是垃圾值的整数**。

**原因：**
1.  `printf` 函数根据格式字符串 (`"x=%d y=%d"`) 来确定它需要多少个参数以及这些参数的类型。
2.  这个格式字符串中有两个 `%d`，意味着 `printf` 期望接收两个整数参数（除了格式字符串本身之外）。
3.  在调用 `printf("x=%d y=%d", 3);` 时：
    *   格式字符串的地址会按约定传递给 `printf` (通常在 `a0` 寄存器)。
    *   第一个整数参数 `3` 会按约定传递给 `printf` (通常在 `a1` 寄存器)。
    *   **但是，没有为第二个 `%d` 提供对应的参数。**
4.  当 `printf` 处理到第二个 `%d` 时，它会尝试从预期的位置（通常是 `a2` 寄存器，如果是第三个参数的话）读取一个整数值。
5.  由于调用者没有在 `a2` 中为第二个 `%d` 放置一个有意义的值，`a2` 寄存器将包含**之前某个操作遗留下来的任何值**。这个值是不可预测的，因此 `printf` 会打印出一个不确定的、看起来像垃圾的整数。

这种行为在C语言中属于**未定义行为 (Undefined Behavior)**。编译器通常不会对此发出警告（尽管一些高级静态分析工具可能会），因为 `printf` 是一个可变参数函数，编译器在编译时无法严格检查参数数量和类型的匹配。

---

## Backtrace (难度：moderate)

**耗时：** 1h

**实现思路与代码：**

*   首先根据提示添加 `r_fp()` 函数，从而可以获取到当前的fp（frame pointer）。
*   再根据课堂笔记，仔细查看ra和prev fp的起始地址相对于当前fp的偏移量分别是-8和-16。因此只需要分别打印，和读取prev fp的地址，解引用获得上个stack frame的地址。
*   由于栈只有一个page，因此当fp处于栈底（从高往低生长），也就是 `PGROUNDDOWN(fp) == PGROUNDUP(fp)` 的时候（此处原文的条件，意指当fp为0或特定边界情况时，这两个宏可能返回相同的值，作为循环终止条件），就说明迭代到了终点，已经没有可读取的栈帧了。

```c
void backtrace() {
  uint64 fp;
  fp = r_fp();
  while (PGROUNDDOWN(fp) != PGROUNDUP(fp)) {
    printf("%p\n", *(uint64 *)(fp - 8));
    fp = *((uint64 *)(fp - 16));
  }
}
```

---

## Alarm (难度：hard)

**耗时：** 5h

> Your Job：在这个练习中你将向XV6添加一个特性，在进程使用CPU的时间内，XV6定期向进程发出警报。这对于那些希望限制CPU时间消耗的受计算限制的进程，或者对于那些计算的同时执行某些周期性操作的进程可能很有用。更普遍的来说，你将实现用户级中断/故障处理程序的一种初级形式。例如，你可以在应用程序中使用类似的一些东西处理页面故障。如果你的解决方案通过了`alarmtest`和`usertests`就是正确的。

### test0: 实现基础的 Alarm 和 Sigreturn

**步骤概述：**

> 1.  您需要修改Makefile以使alarmtest.c被编译为xv6用户程序。
> 2.  放入user/user.h的正确声明是...
> 3.  更新user/usys.pl（此文件生成user/usys.S）、kernel/syscall.h和kernel/syscall.c以允许alarmtest调用sigalarm和sigreturn系统调用。
> 4.  目前来说，你的sys_sigreturn系统调用返回应该是零。

**代码修改：**

参考lab2，添加相关定义即可。

*   **`user/usys.pl`**:
    ```perl
    # user/usys.pl
    entry("sigalarm");
    entry("sigreturn");
    ```
*   **`kernel/syscall.h`**:
    ```c
    // kernel/syscall.h
    #define SYS_sigalarm  22
    #define SYS_sigreturn 23
    ```
*   **`kernel/syscall.c`** (系统调用表和外部声明):
    ```c
    // kernel/syscall.c
    extern uint64 sys_sigalarm(void);
    extern uint64 sys_sigreturn(void);

    // static uint64 (*syscalls[])(void) = {
    // ...
    // [SYS_sigalarm]  sys_sigalarm,
    // [SYS_sigreturn] sys_sigreturn,
    // };
    ```
*   **`Makefile`** (UPROGS部分):
    ```Makefile
    # Makefile
    	$U/_alarmtest\
    ```

> 5.  你的sys_sigalarm()应该将报警间隔和指向处理程序函数的指针存储在struct proc的新字段中（位于kernel/proc.h）。
> 6.  你也需要在struct proc新增一个新字段。用于跟踪自上一次调用（或直到下一次调用）到进程的报警处理程序间经历了多少滴答；您可以在proc.c的allocproc()中初始化proc字段。
> 7.  仅当进程有未完成的计时器时才调用报警函数。请注意，用户报警函数的地址可能是0（例如，在user/alarmtest.asm中，periodic位于地址0）。

**代码修改 (proc结构体与初始化)：**

添加三个变量，分别记录 `sigalarm(n, fn)` 的 `n`，`fn`，还有记录经过时间来定时触发函数。并且在 `allocproc` 中对其初始化。

*   **`kernel/proc.h`**:
    ```c
    // kernel/proc.h
    struct proc {
      // ... other fields ...
      uint64  alarmfn;      // address to the alarmfn
      int     alarmint;
      int     alarmtimecnt;
    };
    ```
*   **`kernel/proc.c`** (`allocproc`函数内):
    ```c
    // kernel/proc.c
    // In allocproc()
    // ...
    // memset(&p->context, 0, sizeof(p->context));
    // p->context.ra = (uint64)forkret;
    // p->context.sp = p->kstack + PGSIZE;
    p->alarmint     = 0;
    p->alarmfn      = 0;
    p->alarmtimecnt = 0;
    // p->savereg will be initialized later or checked before use
    // ...
    ```

> 8.  如果产生了计时器中断，您只想操纵进程的报警滴答；你需要写类似下面的代码
> ```c
> if(which_dev == 2) ...
> ```
>
> 9.  您需要修改usertrap()，以便当进程的报警间隔期满时，用户进程执行处理程序函数。当RISC-V上的陷阱返回到用户空间时，什么决定了用户空间代码恢复执行的指令地址？

**解答 (关于指令地址)：**
根据上课的知识我们知道，在 `usertrap()` 的 `p->trapframe->epc` 中记录了用户空间代码恢复执行的指令地址。
在正常返回的情况下，`p->trapframe->epc` 需要作为 `ecall` 的下一条指令，因此需要+4。
在这个问题中，我们想要让当trap返回到用户空间后，执行 `sigalarm` 指定的 `fn`。因此我们需要将 `p->trapframe->epc` 的地址指定为那个函数的地址，也就是我们在 `proc` 结构体中记录的函数指针。

**代码修改 (`usertrap`函数内)：**

查看 `usertrap` 里面类似的函数可以看到，`which_dev` 是记录中断产生的原因，当 `which_dev` 为2代表是计时器产生的中断。因此我们需要在这个结构下面进行判断。
综合上面问题，我们在 `usertrap` 里面添加下面的代码：

```c
// kernel/trap.c
if (which_dev == 2) {
  if (p->alarmint != 0) {
    p->alarmtimecnt += 1;
    if (p->alarmtimecnt > 0 && (p->alarmtimecnt % p->alarmint) == 0) {
      p->alarmtimecnt = 0;
      p->trapframe->epc = p->alarmfn;
    }
  }
}
```
*运行后会发现有问题，继续往下做*

### test1: 保存和恢复寄存器，处理重入

> 1.  您的解决方案将要求您保存和恢复寄存器——您需要保存和恢复哪些寄存器才能正确恢复中断的代码？(提示：会有很多）
> 2.  当计时器关闭时，让usertrap在struct proc中保存足够的状态，以使sigreturn可以正确返回中断的用户代码。

**解答 (关于保存寄存器)：**
在运行原函数 `test` 的过程中，我们可能因为定时器中断调用了 `sigalarm(n,fn)` 的 `fn` 函数。那么按照上面写的逻辑，我们会去执行 `fn` 函数。但是在函数执行的过程当中会对寄存器进行改变，因此我们原本 `test` 函数所处的用户环境会因为 `fn` 的调用而被破坏。因此我们需要保存所有 `test` 函数被调用时刻的32个用户寄存器。

同时，我们想要在 `fn` 函数被调用之后回到 `test` 函数继续执行原来的代码。因此我们需要在 `fn` 函数结束调用 `sigreturn` 的时候，将 `p->trapframe->epc` 改为原 `test` 中的环境，而这个指令也需要一个空间来保存。

**代码修改：**

更改 `proc` 结构体并分配相应内存。**最后记得释放 `savereg` 这部分内存。**

*   **`kernel/trap.c`** (`usertrap`函数内，当警报触发时):
```c
// kernel/trap.c
  if (which_dev == 2) {
    if (p->alarmint != 0) {
      p->alarmtimecnt += 1;
      if (p->alarmtimecnt > 0 && (p->alarmtimecnt % p->alarmint) == 0) {
        p->savereg->sepc = p->trapframe->epc;

        p->savereg->ra=p->trapframe->ra;
        /*--对所有用户寄存器进行储存--*/
        p->savereg->t6=p->trapframe->t6;

        p->alarmtimecnt = 0;
        p->trapframe->epc = p->alarmfn;
      }
    }
  }
```

*   **`kernel/sysproc.c`** (`sys_sigreturn`函数):
```c
// kernel/sysproc.c
uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  p->trapframe->epc = p->savereg->sepc;

  p->trapframe->ra = p->savereg->ra;
  /*--对所有用户寄存器进行储存--*/
  p->trapframe->t6 = p->savereg->t6;
  return 0;
}
```

*   **`kernel/proc.h`** (定义 `savereg` 结构体):
```c
// kernel/proc.h
struct  savereg{
  // kernel reg
  uint64 sepc;
  // user reg
  uint64 ra;
  /*--对所有用户寄存器进行储存--*/
  uint64 t6;
};

struct proc{
  struct  savereg* savereg;
}
```

*   **`kernel/proc.c`** (`allocproc` 和 `freeproc`):
```c
// kernel/proc.c
// allocproc()
  if((p->savereg = (struct savereg *)kalloc()) == 0){
    release(&p->lock);
    kfree((void *)p->trapframe);
    return 0;
  }

// freeproc()
  if(p->savereg)
    kfree((void *)p->savereg);
  p->savereg = 0;
```

> 3.  防止对处理程序的重复调用——如果处理程序还没有返回，内核就不应该再次调用它。test2测试这个。

**关于重入问题的说明：**
这个没有特殊处理。好像是alarmtest里面的sigalarm(0,0)自动帮我处理了这个。最后所有的测试也都跑通了。

