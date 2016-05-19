# 操作系统实验Lab 5
## 实验报告

计34 何钦尧 2012010548

## 练习0
### 原有lab代码的改动
#### trap.c
在idt_init中，需要添加

```
SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
```
表示T\_SYSCALL这个trap的特权级应当是DPL\_USER，可以在代码中使用int指令来触发
该trap，实现系统调用的功能。

#### proc.c
由于```proc_struct```中添加了一些新的字段
（wait\_state，cptr，yptr，optr），因此需要对其进行初始化：

```
    proc->wait_state = 0;
    proc->cptr = proc->optr = proc->yptr = NULL;
```
以及在```do_fork```的时候，还需要额外设置进程之间关系的链接：

```
    proc->yptr = NULL;
    if ((proc->optr = proc->parent->cptr) != NULL) {
        proc->optr->yptr = proc;
    }
    proc->parent->cptr = proc;
```

## 练习1
### 实现过程
此处是需要设置新进程的trapframe中的内容，以便该进程在从内核态（中断或者系统调用）中返回后，能够正确的在用户态执行。

为了在trap返回时能够从内核态进入用户态，代码段寄存器（CS）必须要设置为
USER\_CS，USER\_CS中对应设置的DPL为3（表示用户态），这样iret时，
判断栈中将要pop出的CS的
特权级和当前特权级，如果当前特权级小于CS的特权级，说明要发生特权级的转换，这时候
CPU会再多pop两个（即SS和ESP）出来。

然后就是将DS，ES，SS等都设置为USER\_DS，将esp设置为USTACKTOP（即用户
地址控件的栈顶），EIP设置为加载的镜像的入口地址。

```
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
```

### 描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的
加载应用程序使用SYS_exec系统调用，执行该系统调用的就是创建的新进程，因此这里
需要描述的实际上是进程进入系统调用之后是如何返回，并且执行加载后的应用程序的。

加载程序会执行```do_execve```函数，在```load_icode```中会设置
当前进程的trapframe，如上面所示。

在do\_execve函数退出后，顺次返回到```syscall```，```trap_dispatch```，
```trap```函数，然后退回到trapentry.S中的```__alltraps```。

```
.globl __alltraps
__alltraps:
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal

    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es

    pushl %esp

    call trap

    popl %esp

.globl __trapret
__trapret:
    popal

    popl %gs
    popl %fs
    popl %es
    popl %ds

    addl $0x8, %esp
    iret
```
从```call trap```返回后，当前的栈中也就是我们刚才设置好的trapframe，
这里一路pop恢复各种寄存器的值，然后执行```iret```指令。

由intel Software Developer’s Manual手册可以知道，iret执行的时候，会
依次pop EIP，CS，EFLAGS。然后判断CS中的特权级标识。由于我们之前设置的
CS为USER\_CS，那么CPU会意识到需要从内核态返回用户态。同时由于trapframe中的
EIP被设置为了加载的程序的entry地址，因此iret返回之后，自然就在用户态开始
运行被加载的程序。

## 练习2
### 实现过程
这里需要实现的部分是两个page之间的数据拷贝，一个是原进程的页，一个是
为新进程分配的页。实现代码如下：

```
    void * kva_src = page2kva(page);
    void * kva_dst = page2kva(npage);
    memcpy(kva_dst, kva_src, PGSIZE);
    ret = page_insert(to, npage, start, perm);
```
由于当前运行在内核态中，而系统也处于保护模式，因此实际上不能对物理地址直接进行操作，而是需要把物理地址转换为内核的虚拟地址。物理地址到内核虚拟地址的转换参见
pmm.h中定义的```KADDR```宏

```
#define KADDR(pa) ({                                                    \
            uintptr_t __m_pa = (pa);                                    \
            size_t __m_ppn = PPN(__m_pa);                               \
            if (__m_ppn >= npage) {                                     \
                panic("KADDR called with invalid pa %08lx", __m_pa);    \
            }                                                           \
            (void *) (__m_pa + KERNBASE);                               \
        })
```
可以发现这个转换就是将物理地址加上KERNBASE（是0xC0000000）。

然后在内核中就可以通过这个得到的内核虚拟地址进行内存的操作，这里直接使用
```memcpy```函数完成内存块的复制。

然后将这个新分配的page加入到新进程的mm中。

### 简要说明如何设计实现“Copy on Write 机制”
“Copy on Write”机制需要一开始建立内存的共享映射，在执行```do_folk```的时候
不新分配物理page并拷贝原来的每一个页面，而是拷贝原来进程的整个页表，并把所有
page的权限都设置为只读。

这样新进程就共享了原进程的物理内存。而新的进程如果要对某个页进行写操作，就会
产生一个page fault，这样我们可以在```do_pgfault```中检测到这个问题，并
分配新的物理页面，重新建立页表映射，进行数据拷贝，然后返回之后新进程就可以
在新分配的物理页面进行写入，而不影响原进程。

为了判断是正常的对只读页面进行写操作（这种情况应当要处理错误）还是在
“Copy on Write”中的情况，需要在页表项中添加新的标志位。在目前的ucore设计中，
PTE中尚还有可以扩展表示的位。


## 练习3
### 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的
```fork```调用会创建一个新的进程。原进程在执行系统调用之后直接返回，仍然
保持运行状态。新建的进程会变为就绪状态，加入到就绪队列当中，在之后的进程调度中，
可以被变为运行状态执行。

```exec```调用由处在运行状态的进程调用，而且调用完毕后依然返回原进程执行，
依然保持运行状态。

```wait```wait会寻找该进程的子进程中处于```PROC_ZOMBIE```状态的，如果
找到，那么释放该子进程的资源，返回```do_wait```调用立即返回，原父进程依然保持
运行状态。如果没有找到，会将该进程设置为```PROC_SLEEPING```，然后进行
```schedule()```。于是该进程进入阻塞状态。

```exit```执行```do_exit```后当前进程的state被设置为```PROC_ZOMBIE```，
然后查找其父进程是否处于wait等待子进程结束，并将父进程wakeup。此时进程控制块仍然没有被释放。需要等到父进程执行```do_wait```找到该处于僵尸状态
的子进程之后，才会释放进程控制块。

### 请给出ucore中一个用户态进程的执行状态生命周期图（包括执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）
ucore中进程状态变换主要依靠上面几个系统调用。

进程通过fork创建之后，处于就绪态。

wait可能会使进程进入阻塞态（也可能不会）。

exit使得进程变为僵尸状态。
