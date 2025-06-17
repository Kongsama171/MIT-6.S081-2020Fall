# Lab9: file system

**花费时间：** 2h

---

## Large files (难度：moderate)

**耗时：** 2h

> 修改`bmap()`，以便除了直接块和一级间接块之外，它还实现二级间接块。你只需要有11个直接块，而不是12个，为你的新的二级间接块腾出空间；不允许更改磁盘inode的大小。`ip->addrs[]`的前11个元素应该是直接块；第12个应该是一个一级间接块（与当前的一样）；13号应该是你的新二级间接块。

### 提示与思路

> 1.  确保您理解`bmap()`。写出`ip->addrs[]`、间接块、二级间接块和它所指向的一级间接块以及数据块之间的关系图。确保您理解为什么添加二级间接块会将最大文件大小增加256*256个块（实际上要-1，因为您必须将直接块的数量减少一个）。

* **`ip->addrs[]`** ：inode的第n个块的地址。比如块0的地址就是`ip->addrs[0]`
* **二级间接块和它所指向的一级间接块以及数据块** ：其实类似于多级页表。其中一个块的大小是1024字节。`uint*`指针大小是四个字节。因此一个块可以被分为256个`uint*`指针，每个指针指向另一个块。二级间接块包含256个指向一级间接块地址的指针，一级间接块包含256个指向数据块的指针，从而实现最大文件大小的增加。
* `bmap`逻辑
  * **直接块** 直接返回直接块地址。假如未分配则分配
  * **一级间接块** 假如一级间接块未分配先分配。然后获取一级间接块的缓冲区，查看缓冲区中的数据（`a[bn]`）也就是块中的数据。`a[n]`指的就是一级间接块包含的256个数据块地址中，第n块的地址。假如数据块未分配先分配，**同时将信息写入log中**。
  * **二级间接块** 整体思路与一级间接块类似。只是需要多迭代一次
  * **踩过的坑** 忘记在二级间接块创建一级间接块的时候写入日志，没有记录相关的修改。

> 2.  如果更改NDIRECT的定义，则可能必须更改file.h文件中struct inode中addrs[]的声明。

*  原本的13个块被分为12个直接块和1个一级间接块。因此原本的`NDIRECT`为12，`MAXFILE`为(NDIRECT + NINDIRECT)。现在我们使用11个块作为直接块，1个一级间接块，1个二级间接块，因此修改`NDIRECT`。`MAXFILE`假如不修改会导致程序报错，因此也需要修改

> 3.  确保`itrunc`释放文件的所有块，包括二级间接块。

* `itrunc`逻辑。类似于多级页表
  * **直接块** 直接`bfree`，并将地址置为0.
  * **一级间接块** 首先对于一级间接块指向的数据块进行全部`bfree`。然后再`bfree`一级间接块本身
  * **二级间接块** 整体思路与一级间接块类似。只是需要多迭代一次
  * **踩过的坑** 把`bfree`函数移到了`if`判断逻辑外面。这样会导致`bfree`为零的块地址。

**代码修改：**

*   **`kernel/fs.h`**:
    ```c
    #define NDIRECT 11
    #define NINDIRECT (BSIZE / sizeof(uint))
    #define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)

    // On-disk inode structure
    struct dinode {
      short type;           // File type
      short major;          // Major device number (T_DEVICE only)
      short minor;          // Minor device number (T_DEVICE only)
      short nlink;          // Number of links to inode in file system
      uint size;            // Size of file (bytes)
      uint addrs[NDIRECT+2];   // Data block addresses
    };
    ```
*   **`kernel/fs.c`**:
    ```c
    static uint
    bmap(struct inode *ip, uint bn)
    {
      uint addr, *a;
      struct buf *bp, *bp2;

      if(bn < NDIRECT){
        if((addr = ip->addrs[bn]) == 0)
          ip->addrs[bn] = addr = balloc(ip->dev);
        return addr;
      }
      bn -= NDIRECT;

      if(bn < NINDIRECT){
        // Load indirect block, allocating if necessary.
        if((addr = ip->addrs[NDIRECT]) == 0)
          ip->addrs[NDIRECT] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        if((addr = a[bn]) == 0){
          a[bn] = addr = balloc(ip->dev);
          log_write(bp);
        }
        brelse(bp);
        return addr;
      }
      bn -= NINDIRECT;

      if(bn < NINDIRECT * NINDIRECT){
        // Load second indirect block, allocating if necessary.
        if((addr = ip->addrs[NDIRECT + 1]) == 0)
          ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        if((addr = a[bn / NINDIRECT]) == 0){
          a[bn / NINDIRECT] = addr = balloc(ip->dev);
          log_write(bp);
        }
        bp2 = bread(ip->dev, addr);
        a = (uint*)bp2->data;
        if((addr = a[bn % NINDIRECT]) == 0){
          a[bn % NINDIRECT] = addr = balloc(ip->dev);
          log_write(bp2);
        }
        brelse(bp2);
        brelse(bp);
        return addr;
      }

      panic("bmap: out of range");
    }
    ```
    ```c
    void
    itrunc(struct inode *ip)
    {
      int i, j, k;
      struct buf *bp, *bp2;
      uint *a, *a2;

      for(i = 0; i < NDIRECT; i++){
        if(ip->addrs[i]){
          bfree(ip->dev, ip->addrs[i]);
          ip->addrs[i] = 0;
        }
      }

      if(ip->addrs[NDIRECT]){
        bp = bread(ip->dev, ip->addrs[NDIRECT]);
        a = (uint*)bp->data;
        for(j = 0; j < NINDIRECT; j++){
          if(a[j])
            bfree(ip->dev, a[j]);
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT]);
        ip->addrs[NDIRECT] = 0;
      }

      if(ip->addrs[NDIRECT + 1]){
        bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
        a = (uint*)bp->data;
        for(j = 0; j < NINDIRECT; j++){
          if(a[j]){
            bp2 = bread(ip->dev, a[j]);
            a2 = (uint*)bp2->data;
            for(k = 0; k < NINDIRECT; k++){
              if(a2[k])
                bfree(ip->dev, a2[k]);
            }
            brelse(bp2);
            bfree(ip->dev, a[j]);
          }
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT + 1]);
        ip->addrs[NDIRECT + 1] = 0;
      }

      ip->size = 0;
      iupdate(ip);
    }
    ```



---

## Symbolic links (难度：moderate)

**耗时：** 5.5h

> Your Job：您将实现`symlink(char *target, char *path)`系统调用，该调用在引用由`target`命名的文件的路径处创建一个新的符号链接。有关更多信息，请参阅`symlink`手册页（注：执行`man symlink`）。要进行测试，请将`symlinktest`添加到*Makefile*并运行它。

### 什么是软链接/符号链接 (Symbolic Link)

符号链接（symlink）是 Unix-like 文件系统中的一种特殊文件类型，其作用是作为对另一个文件或目录的间接引用。与包含用户数据的常规文件不同，符号链接的数据内容仅仅是它所指向的目标的路径字符串。

符号链接的核心功能在于内核如何处理它。在路径解析过程中，当遇到一个符号链接时，对于大多数系统调用（如 `open`, `stat`），内核会自动进行**解引用（dereference）**。这意味着内核会读取链接中存储的路径，并无缝地将操作重定向到该目标路径上。这个过程对应用程序是透明的，使得链接在大多数情况下表现得如同目标文件本身。

这种间接性使符号链接与硬链接（hard link）有本质区别。硬链接是同一 inode 的多个目录项，而符号链接是拥有独立 inode 的文件。这一结构差异赋予了符号链接极大的灵活性：它们可以跨越不同的文件系统，并且可以指向目录，这两点都是硬链接无法做到的。

由于链接和目标是解耦的，当需要操作链接文件本身而非其目标时，需要使用特定的系统调用，如 `readlink`（读取链接内容）和 `lstat`（获取链接自身的元数据）。同样，如果目标文件被删除，链接不会被移除，而是会变成一个指向不存在路径的“悬空链接”（dangling link）。

### 总结对比表

| 特性 | 硬链接 (Hard Link) | 软链接/符号链接 (Symbolic Link) |
| :--- | :--- | :--- |
| **本质** | 同一个文件的多个名字 | 一个独立的文件，内容是另一个文件的路径 |
| **Inode** | 与原文件共享**同一个 Inode** | 有**自己的独立 Inode** |
| **删除原文件** | 链接**依然有效**，数据不受影响（只要链接数>0） | 链接会**失效**（变成悬空链接） |
| **删除链接** | 原文件不受影响，链接数减一 | 原文件**不受影响** |
| **跨文件系统** | **不可以** | **可以** |
| **链接到目录** | **不可以** | **可以** |
| **`ls -li` 查看**| Inode 号码**相同**，链接数增加 | Inode 号码**不同**，文件类型为 `l`，会显示 `->` 指向 |
| **占用空间**| 几乎不占空间（只增加一个目录条目）| 占用一个新 Inode 和少量数据块（用于存储路径） |

### 提示与思路

> 1.  系统调用相关

修改就行

> 2.  实现`symlink(target, path)`系统调用，以在`path`处创建一个新的指向`target`的符号链接。请注意，系统调用的成功不需要`target`已经存在。您需要选择存储符号链接目标路径的位置，例如在inode的数据块中。`symlink`应返回一个表示成功（0）或失败（-1）的整数，类似于`link`和`unlink`。

* 结合软链接/符号链接的特性，我们需要在`path`地址创建一个文件，然后在这个文件内部放入`target`的地址内容，不管这个地址内容是否正确
* `sys_symlink`逻辑
  * 获取`path`的`inode`。假如不存在的话就创建一个
  * 把`target path`写入这个软链接`inode`的数据`[0, MAXPATH]`位置内

> 3.  修改`open`系统调用以处理路径指向符号链接的情况。如果文件不存在，则打开必须失败。当进程向`open`传递`O_NOFOLLOW`标志时，`open`应打开符号链接（而不是跟随符号链接）

* 在对omode判断的时候，添加对`O_NOFOLLOW`标志的判断。同时由于对于普通文件我们需要使其跟随符号链接而不是打开符号链接，因此我们需要重新写一个`follownamei`函数来跟随而不是用`namei`来打开文件。

> 4.  如果链接文件也是符号链接，则必须递归地跟随它，直到到达非链接文件为止。如果链接形成循环，则必须返回错误代码。你可以通过以下方式估算存在循环：通过在链接深度达到某个阈值（例如10）时返回错误代码。

* 对于follownamei，主要是需要通过递归，对于`ip->type`类型是`T_SYMLINK`的`inode`进行递归处理
* 与`sys_symlink`相反，我们需要用readi读取里面的内容，然后递归处理直到超出深度或者找到类型不为`T_SYMLINK`的文件
* **踩过的坑** 结束的时候没有`iunlockput(ip);`

**代码修改：**

*   **`kernel/sysfile.c`**:
    ```c
    uint64
    sys_symlink(void){
      char name[DIRSIZ], target[MAXPATH], path[MAXPATH];
      struct inode *dp, *ip;

      if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
        return -1;

      begin_op();

      if((dp = nameiparent(path, name)) == 0){
        end_op();
        return -1;
      }
      ilock(dp);

      if((ip = ialloc(dp->dev, T_SYMLINK)) == 0)
        panic("sys_symlink: ialloc");

      ilock(ip);
      uint off = 0;
      if(writei(ip, 0, (uint64)&target, off, MAXPATH) != MAXPATH)
        panic("sys_symlink: writei");
      ip->nlink = 1;
      iupdate(ip);
      iunlock(ip);

      if(dirlink(dp, name, ip->inum) < 0){
        iunlockput(dp);
        ilock(ip);
        ip->nlink--;
        iupdate(ip);
        iunlockput(ip);
        end_op();
        return -1;
      }

      iunlockput(dp);
      iput(ip);

      end_op();
      
      return 0;
    }
    ```
    ```c
    // sys_open
    if(omode & O_CREATE){
      ip = create(path, T_FILE, 0, 0);
      if(ip == 0){
        end_op();
        return -1;
      }
    } else if(omode & O_NOFOLLOW){
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type == T_DIR && omode != O_RDONLY){
        iunlockput(ip);
        end_op();
        return -1;
      }
    } else {
      if((ip = follownamei(path, 0)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type == T_DIR && omode != O_RDONLY){
        iunlockput(ip);
        end_op();
        return -1;
      }
    }
    ```
*   **`kernel/fs.c`**:
    ```c
    struct inode*
    follownamei(char *path, int depth)
    {
      struct inode *ip;
      char name[DIRSIZ], newpath[MAXPATH];

      if(depth > 10)
        return 0;
      if((ip = namex(path, 0, name)) == 0)
        return 0;
      if(ip->type != T_SYMLINK)
        return ip;

      ilock(ip);
      uint off = 0;
      if(readi(ip, 0, (uint64)&newpath, off, MAXPATH) != MAXPATH)
        panic("follownamei: readi");
      iunlockput(ip);// 修改的第一个问题

      return follownamei(newpath, depth + 1);
    }
    ```





---

## 附录：`man symlink`

`man symlink` 通常指的是 `symlink(2)` 这个系统调用，它是创建符号链接的编程接口。

---

### **名称 (NAME)**

`symlink`, `symlinkat` - 为一个文件创建一个新的名字（符号链接）

### **概要 (SYNOPSIS)**

```c
#include <unistd.h>

int symlink(const char *target, const char *linkpath);

#include <fcntl.h>           /* Definition of AT_* constants */
#include <unistd.h>

int symlinkat(const char *target, int newdirfd, const char *linkpath);
```

### **描述 (DESCRIPTION)**

`symlink()` 会创建一个名为 `linkpath` 的符号链接（symbolic link，也叫软链接, soft link）。

这个新创建的链接 `linkpath` 会包含字符串 `target` 作为其内容。`target` 路径可以是绝对路径，也可以是相对路径。

符号链接是对 `target` 的一个**间接指针**；当对该链接进行大多数操作时（例如 `open()`, `stat()` 等），内核会自动**解引用（dereference）**它，即操作会应用到 `target` 指向的文件上，而不是链接本身。

与硬链接（hard link）不同：
1.  在调用 `symlink()` 时，`target` **不需要存在**。你可以创建一个指向不存在的文件的“悬空链接”（dangling link）。
2.  `target` 可以是一个目录。
3.  `linkpath` 和 `target` 可以位于不同的文件系统中。

链接本身的权限位是没有意义的。对链接的访问权限由其所在目录的权限决定，而通过链接访问目标文件时的权限检查，则是在目标文件 `target` 上进行的。

如果你需要操作链接文件本身而不是它指向的目标（例如，读取链接指向的路径字符串，或者删除链接文件本身），你应该使用特殊的系统调用，如 `readlink(2)`, `lstat(2)` 和 `unlink(2)`。

### **返回值 (RETURN VALUE)**

成功时，返回 `0`。
失败时，返回 `-1`，并设置 `errno` 来指示错误。

### **错误 (ERRORS)**

常见的 `errno` 值包括：

*   `EACCES`
    没有权限在包含 `linkpath` 的目录中进行写操作。
*   `EEXIST`
    `linkpath` 已经存在。
*   `ELOOP`
    解析 `linkpath` 时遇到了太多的符号链接。
*   `ENAMETOOLONG`
    `target` 或 `linkpath` 的路径名太长。
*   `ENOENT`
    `linkpath` 路径中的一个目录部分不存在或是一个悬空的符号链接。
*   `ENOSPC`
    设备上没有足够的空间来创建链接文件或目录条目。
*   `EROFS`
    `linkpath` 所在的文件系统是只读的。

### **与硬链接（Hard Link）的区别**

| 特性 | 符号链接 (Symbolic Link) | 硬链接 (Hard Link) |
| :--- | :--- | :--- |
| **本质** | 一个特殊的文件，内容是另一个文件的路径字符串。 | 同一个文件数据的多个目录项，指向同一个 inode。 |
| **跨文件系统** | **可以**。 | **不可以**，必须在同一个文件系统内。 |
| **指向目录** | **可以**。 | **不可以**（通常由系统限制，以防形成循环）。 |
| **源文件要求** | 源文件（target）**可以不存在**。 | 源文件**必须存在**。 |
| **删除源文件** | 链接会变成“悬空/失效链接”。 | 链接仍然有效，文件数据只有当所有链接都被删除后才会真正被删除。 |
| **`ls -l` 显示** | `l` 开头，显示 `link -> target`。 | 与普通文件无异，链接数增加。 |
| **占用空间** | 占用一个 inode 和少量数据块来存储路径字符串。| 只占用一个目录项，不占用新的 inode。|

### **示例代码 (EXAMPLE)**

下面的 C 程序演示了如何使用 `symlink()`。

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
    // 目标文件，我们将为它创建符号链接
    const char* target_file = "/etc/passwd";
    // 符号链接文件的名字
    const char* link_name = "passwd.sym";

    printf("Creating symbolic link '%s' pointing to '%s'\n", link_name, target_file);

    // 调用 symlink 创建链接
    if (symlink(target_file, link_name) == -1) {
        perror("symlink failed");
        exit(EXIT_FAILURE);
    }

    printf("Symbolic link created successfully.\n");

    // 你可以在 shell 中验证
    printf("Run 'ls -l %s' to see the link.\n", link_name);
    printf("Run 'readlink %s' to see where it points.\n", link_name);
    printf("Run 'cat %s' to see the content of the target file.\n", link_name);

    return 0;
}
```