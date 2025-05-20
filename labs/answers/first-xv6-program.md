# xv6 启动及第一个用户进程执行流程详解

本文结合 xv6 启动过程的官方描述和实际 GDB 调试记录，详细解析 xv6 内核如何启动，创建第一个用户进程，并最终运行起 shell。

## 1\. 内核启动：从 `_entry` 到 `main`

xv6 的生命周期始于机器上电。

根据描述：

> 当RISC-V计算机上电时，它会初始化自己并运行一个存储在只读内存中的引导加载程序。引导加载程序将xv6内核加载到内存中。然后，在机器模式下，中央处理器从\_entry (kernel/entry.S:6)开始运行xv6。Xv6启动时页式硬件（paging hardware）处于禁用模式：也就是说虚拟地址将直接映射到物理地址。
> 加载程序将xv6内核加载到物理地址为0x80000000的内存中。它将内核放在0x80000000而不是0x0的原因是地址范围0x0:0x80000000包含I/O设备。

内核执行的第一个指令位于 `_entry`。此时 CPU 处于**机器模式 (M-mode)**，虚拟地址转换**禁用**，地址被解释为**物理地址**。内核代码加载到物理地址 `0x80000000`。地址`0x80000000`是一个被QEMU认可的地址。也就是说如果你想使用QEMU，那么第一个指令地址必须是它。所以，我们会让内核加载器从那个位置开始加载内核。这个地址在`kernel.ld`里面被设定。

`_entry` 的汇编代码首先进行非常基础的设置，包括设置初始栈区。

根据描述：

> \_entry的指令设置了一个栈区，这样xv6就可以运行C代码。Xv6在start. c (kernel/start.c:11)文件中为初始栈stack0声明了空间。由于RISC-V上的栈是向下扩展的，所以\_entry的代码将栈顶地址stack0+4096加载到栈顶指针寄存器sp中。现在内核有了栈区，\_entry便调用C代码start(kernel/start.c:21)。

设置好栈 `stack0` 后，`_entry` 跳转到 C 函数 `start`。此时仍在 M-mode。

`start` 函数负责完成剩余的机器模式下的初始化工作，并准备切换到管理模式。

根据描述：

> 函数start执行一些仅在机器模式下允许的配置，然后切换到管理模式。RISC-V提供指令mret以进入管理模式，该指令最常用于将管理模式切换到机器模式的调用中返回。而start并非从这样的调用返回，而是执行以下操作：它在寄存器mstatus中将先前的运行模式改为管理模式，它通过将main函数的地址写入寄存器mepc将返回地址设为main，它通过向页表寄存器satp写入0来在管理模式下禁用虚拟地址转换，并将所有的中断和异常委托给管理模式。
> 在进入管理模式之前，start还要执行另一项任务：对时钟芯片进行编程以产生计时器中断。清理完这些“家务”后，start通过调用mret“返回”到管理模式。这将导致程序计数器（PC）的值更改为main(kernel/main.c:11)函数地址。

`start` 函数的关键操作包括：

  * 配置 `mstatus` 寄存器，设置 M-mode 先前的模式为 S-mode。
  * 将 `main` 函数的**物理地址**写入 `mepc` 寄存器。
  * 将 `satp` 寄存器清零（`csrw satp, 0`），**禁用**管理模式下的虚拟地址转换。
  * 配置中断/异常委托和时钟芯片。
  * 执行 `mret` 指令。`mret` 指令会让 CPU 从 M-mode 切换到 S-mode，并将 PC 设置为 `mepc` 中的值。

因此，`mret` 执行后，CPU 进入管理模式，PC 跳转到 `main` 函数的**物理地址**，程序从 `main` 开始执行。

我们在 GDB 中连接到 QEMU 后，最初看到的 PC 位置通常是 `0x1000` 或 `0x8000xxxx` 附近，这是内核的物理地址空间中的某个位置，取决于 GDB 连接时的具体暂停点。

```gdb
(gdb) target remote localhost:26000
Remote debugging using localhost:26000
0x0000000000001000 in ?? ()
```

这里的 `0x1000` 表示 GDB 连接时程序暂停在物理地址 `0x1000`，`in ?? ()` 说明 GDB 对这个地址没有符号信息。

在 `main` 函数中，内核会初始化许多核心子系统（内存分配、进程表、文件系统等）。在这个阶段，**S-mode 的虚拟内存仍然是禁用的** (`satp=0`)，地址被当作**物理地址**处理。`main` 函数的一个重要任务就是**设置内核自己的页表**，并将内核代码和数据映射到 S-mode 的虚拟地址空间，然后将内核页表的地址写入 `satp`，**启用 S-mode 的虚拟地址转换**。

## 2\. 第一个用户进程的诞生与初次进入用户态 (initcode)

在 `main` 完成核心初始化并启用 S-mode 虚拟内存后，它会创建第一个用户进程。

根据描述：

> 在main(kernel/main.c:11)初始化几个设备和子系统后，便通过调用userinit (kernel/proc.c:212)创建第一个进程，第一个进程执行一个用RISC-V程序集写的小型程序：initcode. S (user/initcode.S:1)，它通过调用exec系统调用重新进入内核。

`userinit` 函数会分配一个进程结构体 (`struct proc`)，设置其必要字段。它还会为这个新进程创建**第一个用户页表**，这是这个进程用户地址空间的蓝图。然后，`userinit` 会将 `user/initcode.S` 中编译好的机器码加载到这个用户进程**虚拟地址空间**的**地址 0** 处。它还会设置好进程的初始寄存器状态（包括设置 `sepc` 寄存器为 `0`，作为这个进程用户态的入口地址），并将进程状态设置为可运行。

当内核从某个陷阱处理（如启动时的首次调度）返回到用户态时，它会检查是否有可运行的用户进程，并准备切换过去。`userinit` 创建的进程就是第一个可运行的用户进程。返回用户态的流程会经过 `usertrapret` 函数和 trampoline 汇编代码。

此时，内核准备将执行权交给第一个用户进程，其入口点是虚拟地址 `0`。我们在调试时可以在这里设置断点来观察。

根据你的 GDB 调试记录：

```gdb
(gdb) b *0x0    
Breakpoint 1 at 0x0
(gdb) c
Continuing.

Breakpoint 1, 0x0000000000000000 in ?? ()
=> 0x0000000000000000:  17 05 00 00     auipc   a0,0x0
```

你在虚拟地址 `0` 处设置断点 (`b *0x0`)。当内核完成第一个进程的设置并通过 `usertrapret` → trampoline → `sret` 返回到用户态时，CPU 的 PC 被设置为 `0`，命中断点并暂停。这时 CPU 切换到了用户模式，并且 MMU 正在使用第一个用户进程的页表进行虚拟地址转换。你看到的地址 `0x0` 是这个进程**用户地址空间中的虚拟地址**。

## 3\. `initcode` 的核心功能：执行 `exec("init")`

一旦进入用户态，CPU就开始执行加载在虚拟地址 `0` 处的 `initcode.S` 代码。

根据描述：

> 它通过调用exec系统调用重新进入内核。正如我们在第1章中看到的，exec用一个新程序（本例中为 /init）替换当前进程的内存和寄存器。

`initcode.S` 是一段非常短的汇编代码，它的唯一目的就是执行 `exec("/init", argv)` 系统调用，其中 `argv` 包含 `"/init"` 和一个 `NULL`。

根据你的 GDB `si` 调试记录：

```gdb
=> 0x0000000000000000:  17 05 00 00     auipc   a0,0x0
(gdb) si
0x0000000000000004 in ?? ()
=> 0x0000000000000004:  13 05 05 02     addi    a0,a0,32
(gdb) si
0x0000000000000008 in ?? ()
=> 0x0000000000000008:  97 05 00 00     auipc   a1,0x0
(gdb) si
0x000000000000000c in ?? ()
=> 0x000000000000000c:  93 85 05 02     addi    a1,a1,32
(gdb) si
0x0000000000000010 in ?? ()
=> 0x0000000000000010:  9d 48   li      a7,7
(gdb) si
0x0000000000000012 in ?? ()
=> 0x0000000000000012:  73 00 00 00     ecall
```

这些汇编指令就是在准备系统调用 `exec` 的参数：将路径字符串 `"/init"` 和参数数组的地址放入寄存器，并将系统调用号 7（`SYS_EXEC`）放入 `a7` 寄存器。最后的 `ecall` 指令触发一个**系统调用陷阱**，再次进入内核态。

## 4\. `ecall` 陷阱与内核处理系统调用

`ecall` 指令会强制 CPU 从用户模式切换回管理模式，并将 PC 设置为内核预设的陷阱入口地址。

根据描述和你的 GDB 记录：

> 您的目标是使用gdb来观察最开始的“内核空间到用户空间”的转换。...然后进入我们想要看的系统调用sys\_exec中。

`ecall` 后，CPU 执行流程跳转到内核的 trap 向量处（通常在 trampoline 页的高地址）。你的 GDB 记录显示 PC 跳到了 `0x3ffffff004` 等地址，开始执行 trap 处理汇编代码，然后进入 C 函数 `usertrap`。

```gdb
=> 0x0000000000000012:  73 00 00 00     ecall
(gdb) si
0x0000003ffffff004 in ?? () // 跳到 trampoline/trap 入口
=> 0x0000003ffffff004:  23 34 15 02     sd      ra,40(a0)
(gdb) si
0x0000003ffffff008 in ?? ()
// ... (其他保存寄存器的汇编指令) ...
usertrap () at kernel/trap.c:41 // 进入 C 函数 usertrap
41      {
```

`usertrap` 函数处理所有来自用户态的陷阱（系统调用、中断、异常）。它检查陷阱的原因（通过 `scause` 寄存器），对于系统调用（`scause == 8`），它会验证系统调用号（从 `a7` 寄存器获取）并调用 `syscall()`。

```gdb
(gdb) 56        if(r_scause() == 8){ // 检查是否是系统调用陷阱
// ...
(gdb) 70          syscall(); // 调用 syscall 总入口
(gdb) s // 进入 syscall 函数
syscall () at kernel/syscall.c:138
138       struct proc *p = myproc();
(gdb) n
140       num = p->tf->a7; // 获取系统调用号，此时 num == 7
// ... (根据 num 查找并调用对应的 sys_ 函数) ...
(gdb) s // 进入 sys_exec 函数
sys_exec () at kernel/sysfile.c:419 // 进入 sys_exec
```

`syscall` 函数根据 `a7` 寄存器的值 (`num` = 7)，从系统调用表中找到对应的函数 `sys_exec` 并调用它。

## 5\. 执行 `SYS_EXEC` 系统调用 (`sys_exec`)

`sys_exec` 函数负责将一个新的程序加载到当前进程中执行。

根据描述和你的 GDB 记录：

> 正如我们在第1章中看到的，exec用一个新程序（本例中为 /init）替换当前进程的内存和寄存器。
> ... 系统调用detected，编号为7，查看kerne/syscall.h可知，编号为7的系统调用是SYS\_EXEC。... OK，exec的东西我们已经可以知道了，它要将init这个程序“装入”到内核中。这个程序对应的C代码在user/init.c下，对应的ELF文件为user/\_init。
> ... 在执行完csrw satp, t1后，我们终于能在exec上打下断点了！

在 `sys_exec` 函数内部：

  * 内核从用户空间的参数地址读取要执行的文件路径（`/init`）和参数列表。
  * 它会验证文件是否存在和格式（ELF）。
  * 它会为当前进程构建**新的用户地址空间**（新的页表）。
  * 它会将新程序的代码和数据从文件系统加载到这个新的地址空间中。
  * 它会设置进程的初始寄存器状态，尤其是将新程序的入口地址（`/init` 的入口，通常也是 0）写入 `sepc` 寄存器。
  * 它会关闭旧的用户程序文件描述符。

你的 GDB 记录中的 `p path` 命令确认了要执行的程序路径是 `"/init"`：

```gdb
442       int ret = exec(path, argv);
(gdb) p path
$1 = "/init\000\000\000 ... // 输出显示路径字符串以 "/init" 开头
```

你还遇到了在执行 `csrw satp, t1` 之后才能在 `exec` 上成功设置断点的问题。这可能与 GDB 在内核态理解地址空间有关。在内核设置好自己的 S-mode 页表并写入 `satp` 之前，GDB 可能无法正确解析内核虚拟地址，导致符号断点或地址断点设置失败。`csrw satp, t1` 是将内核页表地址写入 `satp` 的指令，执行后 S-mode 虚拟地址转换启用，内核的代码和数据就可以通过其虚拟地址正常访问了，此时 GDB 也能正确解析符号和地址了。

## 6\. 从内核返回用户空间 (`usertrapret`, Trampoline, `sret`)

`sys_exec` 函数完成它的任务（准备好执行 `/init` 程序）后，会返回到 `syscall`，然后返回到 `usertrap`。 `usertrap` 在做一些清理工作后，会调用 `usertrapret` 来准备最终从内核返回用户态。

根据描述和你的 GDB 记录：

> 从sys\_exec中跳出，回到了trap.c的usertrap()函数中，下一步就会从用户trap里返回用户态：
> ...
> 继续调试，终于我们看到了系统调用总入口，按下s进入系统调用总入口syscall，然后进入我们想要看的系统调用sys\_exec中。
> ...
> 最后我们来看一下init.c的代码：
> ...
> （在执行完sys\_exec，回到usertrap后）
> 86        usertrapret();
> (gdb) s // 进入 usertrapret 函数
> usertrapret () at kernel/trap.c:95 // 进入 usertrapret
> // ... usertrapret 内部配置寄存器和 trapframe ...
> 130       ((void (\*)(uint64,uint64))fn)(TRAPFRAME, satp); // 调用 trampoline
> (gdb) si // 单指令步过函数指针调用
> 0x0000003ffffff090 in ?? () // 进入 trampoline 汇编
> \=\> 0x0000003ffffff090:  73 90 05 18     csrw    satp,a1 // 切换到用户页表
> ... (其他 trampoline 汇编指令) ...
> // 最终执行 sret 指令

`usertrapret` 函数（在 `kernel/trap.c` 中）负责将当前进程从内核态返回用户态前所需的所有状态（寄存器值、`sstatus`、`sepc` 等，这些信息保存在 trapframe 中）准备好。它会将用户页表地址放入 `satp`（使 MMU 使用用户页表进行翻译），将用户态的下一条指令地址（对于刚 `exec` 的 `/init`，这个地址是 0）放入 `sepc`，设置 `sstatus` 的 SPP 位为 0（表示将返回用户模式），并将 CPU 的栈切换回内核栈。

最后，`usertrapret` 通过调用 trampoline 汇编代码中的一个入口点（通常通过函数指针或 `jr` 跳转实现）完成返回的用户态前的收尾工作。

根据你的 GDB 记录，你通过 `si` 步进观察了这段 trampoline 汇编代码。这段代码负责从 trapframe 中恢复用户模式所需的寄存器值，并将用户页表地址 (`a1`) 写入 `satp` (`csrw satp, a1`)，刷新 TLB (`sfence.vma`)，恢复栈指针等等，并将 `sepc` 的值放入 `sret` 指令隐式使用的来源。

最终，trampoline 汇编的最后一条指令是 `sret`。

`sret` 指令是 RISC-V 中从 S-mode 返回 U-mode 的指令。它会将 CPU 模式切换到用户模式，并将 PC 的值设置为 `sepc` 寄存器中的地址。此时 `sepc` 中的值是 `/init` 程序的入口地址 `0`。

## 7\. 执行 `/init` 用户程序

根据描述和前面的流程：

> 一旦内核完成exec，它就返回/init进程中的用户空间。
> ...
> init(user/init.c:15)将创建一个新的控制台设备文件，然后以文件描述符0、1和2打开它。然后它在控制台上启动一个shell。系统就这样启动了。

CPU 现在执行的是 `/init` 程序在**虚拟地址 0** 处的代码。`/init` 是用 C 语言编写的。

根据你提供的 `init.c` 代码：

```c
// init: The initial user-level program
// ... includes and argv declaration ...
int main(void)
{
  int pid, wpid;

  // 打开标准输入输出错误对应的终端 (console)
  if(open("console", O_RDWR) < 0){
    mknod("console", 1, 1);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){ // 无限循环
    printf("init: starting sh\n");
    pid = fork(); // 创建一个子进程
    if(pid < 0){
      printf("init: fork failed\n");
      exit(1);
    }
    if(pid == 0){ // 子进程执行这里
      exec("sh", argv); // 加载并执行 shell 程序 /bin/sh
      printf("init: exec sh failed\n", ); // 如果 exec 失败
      exit(1);
    }
    while((wpid=wait(0)) >= 0 && wpid != pid){
      // 父进程等待其子进程 (shell) 结束
    }
  }
}
```

`/init` 首先确保标准文件描述符 0, 1, 2 连接到控制台。然后进入一个无限循环：

  * 打印 `init: starting sh`。
  * `fork()` 创建一个子进程。
  * 父进程 (`pid > 0`) 进入内部的 `while(wait(0))` 循环，等待其子进程（即将运行 shell 的进程）退出。
  * 子进程 (`pid == 0`) 执行 `exec("sh", argv)`。

## 8\. 启动 Shell (`sh`)

子进程执行 `exec("sh", argv)` 会再次触发一个 `ecall` 陷阱，进入内核处理 `SYS_EXEC` 系统调用。

内核再次处理 `exec`，这次它会加载 `/bin/sh` 这个 shell 程序，替换掉当前这个子进程的程序镜像。内核设置好 `/bin/sh` 的用户地址空间、初始寄存器等，并设置 `sepc` 为 `/bin/sh` 的入口地址（通常也是 0）。

内核返回到用户空间。此时，CPU 就在执行 `/bin/sh` 程序的代码了，进程也切换到了运行 shell 的上下文。shell 程序会显示它的提示符 ` $  `，等待用户输入命令。

父进程 `/init` 会在它的子进程（shell）退出后，通过 `wait` 得到通知，然后回收子进程资源，并再次进入无限循环，重新 `fork` 和 `exec("sh")`，这样即使 shell 崩溃退出，`/init` 也能重新启动一个新的 shell。

## 总结 xv6 第一个用户程序总流程

结合你提供的文字描述和 GDB 调试记录，xv6 启动第一个用户程序并运行 shell 的总流程是：

1.  **内核启动到 `main`：** 物理地址执行 (`_entry` -\> `start`)，切换到 S-mode，禁用 S-mode VM (`satp=0`)，PC 跳转到 `main` 的物理地址。
2.  **内核初始化 VM 并创建进程：** `main` 设置内核页表，启用 S-mode VM (`satp` 非零)，调用 `userinit` 创建第一个进程，加载 `initcode.S` 到其**用户虚拟地址 0**。
3.  **首次进入用户态 (执行 `initcode`)：** 内核通过 `usertrapret` → trampoline → `sret`（设置 `sepc=0`，切换到用户页表 `satp`），进入用户模式，PC = 用户虚拟地址 `0`。你在 `b *0x0` 命中断点。
4.  **`initcode` 执行 `exec("/init")`：** 你单步 (`si`) 调试看到 `initcode` 设置参数并执行 `ecall` (系统调用 7)。
5.  **`ecall` 陷阱处理：** CPU 进入内核态，PC 跳到 trampoline/trap 入口。内核执行 `usertrap` → `syscall` → `sys_exec`。
6.  **`sys_exec("/init")`：** 内核加载 `/init` 程序到当前进程，替换 `initcode`。设置新用户页表，设置 `sepc=0` (对于 `/init`)。你在这里可以通过 `p path` 看到路径是 "/init"。
7.  **返回用户态 (执行 `/init`)：** 内核再次通过 `usertrapret` → trampoline → `sret`（这次 `satp` 切换到加载了 `/init` 的新页表，`sepc` 仍然是 0），回到用户模式，PC = 用户虚拟地址 `0`。现在执行的是 `/init` 的代码。
8.  **`/init` 执行：** `/init` 打开标准 I/O，`fork` 子进程，父进程 `wait` 循环，子进程 `exec("sh")`。
9.  **`exec("sh")` 陷阱处理：** 子进程通过 `ecall` 再次进入内核，处理 `SYS_EXEC` (加载 `/bin/sh`)。
10. **返回用户态 (执行 `sh`)：** 内核返回用户模式，PC 跳转到 `/bin/sh` 的入口地址。现在执行的是 shell 程序。
11. Shell 打印提示符 ` $  `，等待用户交互。

掌握这段流程，特别是用户态和内核态的往返（`ecall` 进，`sret` 出），以及地址和页表的切换，是理解 xv6 的基础。