# 操作系统实验Lab 8
## 实验报告

计34 何钦尧 2012010548

## 练习0
### 原有lab代码的改动
#### proc.c
需要在```alloc_proc```中添加对于```proc_struct```中的文件结构的初始化：

```
    proc->filesp = NULL;
```
然后需要在```do_fork```中拷贝原来进程的文件信息：

```
    copy_files(clone_flags, proc)
```
在这个函数里面，如果```clone_flags```中有```CLONE_FS```，则新进程直接复制原来进程的
```file_struct```。否则则新建一个```file_struct```。

## 练习1
### 实现过程
需要在```sfs_io_nolock```函数中实现对于文件块（Block）的读写操作。该函数传入的参数
有文件系统结构体```sfs```，有文件inode结构体```sin```，以及对应的读写缓冲区，起始位置
和长度等信息。

文件操作都是以块为单位来进行的。ucore中文件块的大小和内存页的大小一致（4kB）。在对一定
长度的文件区域进行读写的时候，一般来讲会将读写的区域分成三块：

1. 开始位置可能不从某个块的开头开始
2. 中间部分的完成的若干块
3. 结尾部分不以块的末尾作为结束

在这里的实现里面也同样的对这样的三个部分进行处理。每个部分都执行一些基本的操作。

首先是根据文件系统和文件inode的结构体查询到要进行操作的块在磁盘上的块号

```
    sfs_bmap_load_nolock(sfs, sin, blkno, &ino)
```
返回的ino就是我们需要的磁盘块号。之后进行的读写操作都基于这个号来进行。然后对于整块的读写
或者是块中某一个小部分的读写，分别使用两个不同的方法来进行：

```
    sfs_block_op(sfs, buf, ino, 1)
```

```
    sfs_buf_op(sfs, buf, size, ino, blkoff)
```
需要注意着两个是函数指针，在之前分别根据是读还是写指向了具体的操作函数。

于是总的实现就变成了如下：

```
    if ((blkoff = offset % SFS_BLKSIZE) != 0) {
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
            goto out;
        }
        alen += size;
        if (nblks == 0) {
            goto out;
        }
        buf += size;
        blkno++;
        nblks--;
    }

    size = SFS_BLKSIZE;
    while (nblks != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) {
            goto out;
        }
        alen += size;
        buf += size;
        blkno++;
        nblks--;
    }

    if ((size = endpos % SFS_BLKSIZE) != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
            goto out;
        }
        alen += size;
    }
```
三段分别对应的就是上面所说的三种情况。

### UNIX的PIPE机制的设计实现
管道机制使得两个进程的标准输入输出可以连接起来，使得一个进程的输出可以作为另一个进程的输入。

实现的时候需要建立一个虚拟的设备，在内存中放置一个缓冲区，在fork进程的时候将其stdin和stdout
两个设备文件正确的设置为指向该设备。其他具体的实现可以参考已有的stdin和stdout两个设备，
包括合适的设置缓冲，以及等待输入时候的阻塞等待。

## 练习2
### 实现过程
这里需要对原有的```load_icode```进行修改，原来的是从一个binary的地址的指针直接加载，
现在需要从文件里面读出二进制的内容。于是只需要对原有的从指针中读出数据改为从文件中读入数据即可。

如将

```
    struct elfhdr *elf = (struct elfhdr *)binary;
```
改为

```
    struct elfhdr __elf, *elf = &__elf;
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {
        goto bad_elf_cleanup_pgdir;
    }
```

```load_icode_read```是一个辅助的函数，从```fd```表示的文件里面从指定的offset读指定
大小的字节到缓冲区中。

此外需要将程序调用时的参数放入栈中，并正确的设置栈顶

```
    uint32_t argv_size=0, i;
    for (i = 0; i < argc; i ++) {
        argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }

    uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
    char** uargv=(char **)(stacktop  - argc * sizeof(char *));
    
    argv_size = 0;
    for (i = 0; i < argc; i ++) {
        uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
        argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }
    
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;
```

这里将```USTACKTOP```减少一定的值作为实际程序的栈顶，多出来的空间用于压入argc，和argv的参数。

### UNIX的硬链接和软链接机制设计实现
硬软链接都应该在文件系统层面上实现，即ucore里的Simple File System，并储存在硬盘中。

硬链接中不同的文件对应同一个磁盘inode。ucore中已经实现了硬链接。在

```
struct sfs_disk_entry {
    uint32_t ino;                                   /* inode number */
    char name[SFS_MAX_FNAME_LEN + 1];               /* file name */
};
```
这个结构体是每个目录包含的文件的entry项。硬链接的方式是不同的文件的entry使用相同的ino number，
即实际上对应磁盘上相同的inode。同时在磁盘inode的结构体中

```
struct sfs_disk_inode {
    uint32_t size;                  /* size of the file (in bytes) */
    uint16_t type;                  /* one of SYS_TYPE_* above */
    uint16_t nlinks;                /* # of hard links to this file */
    uint32_t blocks;                /* # of blocks */
    uint32_t direct[SFS_NDIRECT];   /* direct blocks */
    uint32_t indirect;              /* indirect blocks */
};
```
```nlinks```表示有多少个文件链接到了这个同一个inode。当删除文件是递减这个计数，当减为0是
将该文件inode本身删除。

软链接实际上就是一个快捷方式，一个文件指向了另一个文件名。可以在文件内容中规定一个格式，
用来存放对应的链接到的文件名（包含完整目录），同时disk_inode的类型中增加软链接的类型作为标识。

文件系统在访问到一个文件发现是软链接时，将所有的文件操作等重新定向到链接到的文件上去。