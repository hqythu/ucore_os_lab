# 操作系统实验Lab 4
## 实验报告

计34 何钦尧 2012010548

## 练习1
### 实现过程
alloc_proc这个函数仅仅完成最基本的创建一个没有任何信息量的空白的proc_struct的任务。
实际上也就是将proc_struct的各个字段填零。

```
    proc->state = PROC_UNINIT;
    proc->pid = -1;
    proc->runs = 0;
    proc->kstack = 0;
    proc->need_resched = 0;
    proc->parent = NULL;
    proc->mm = NULL;
    memset(&(proc->context), 0, sizeof(struct context));
    proc->tf = NULL;
    proc->cr3 = boot_cr3;
    proc->flags = 0;
    memset(proc->name, 0, PROC_NAME_LEN);
```

### 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？
#### context
context的定义如下:

```
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
容易看出context这个struct的作用如它的名字所示，是用来在进程切换的时候保存进程的上下文
（寄存器），并在下一次恢复该进程的执行的时候恢复这些寄存器的值。
注意到这里没有eax寄存器，主要是由于eax一般都有别的用途，不需要保存。

#### trapframe
trapframe在trap.h中定义

```
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```
trapframe中储存的是各段寄存器，以及发生中断时被硬件压栈的若干寄存器
（err，eip，cs，eflags，esp，ss），表示当前中断产生的栈帧（我们知道，进程切换发生的时候
一定是因为发生了中断，导致陷入了内核态）。


## 练习2
### 实现过程
do_folk需要将一个进程的所有相关信息复制到另一个新建立的新进程中。

首先使用alloc_proc建立一个新的进程的结构体，并将其parent设置为当前的进程。

```
    if ((proc = alloc_proc()) == NULL) {
        goto fork_out;
    }
    proc->parent = current;
```
然后建立内核栈，从父进程中拷贝内存，拷贝线程（主要是拷贝各种trapframe和context寄存器的内容）。

```
    if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    copy_thread(proc, stack, tf);
```
之后将新建的进程分配pid，加入链表中，注意这里需要关闭中断，保证操作的原子性。

```
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    local_intr_restore(intr_flag);
```
最后将进程设置为RUNNABLE以便系统调度。
```
    wakeup_proc(proc);
```

### ucore是否做到给每个新fork的线程一个唯一的id？
分配pid由get_pid函数执行。

分析函数可以知道，设有一个MAX_PID，且一定大于MAX_PROCESS。

last_pid和next_safe为static变量，last_pid每次表示上一次分配的pid。
调用函数的时候先将pid自增，然后检查是否超过MAX_PID，超过则重新从1开始。
然后遍历所有的进程，比较有没有进程的pid和last_pid相同。如果没有就可以直接返回last_pid作为
本次分配的pid值。如果有相同的话，将last_pid加1，比较是否大于next_safe，然后继续和后面的进程比较。

此处last_pid和next_safe表示一个没有被占用的pid的上下区间。当next_safe或者last_pid溢出
MAX_PID之后，又重置为从0（next_safe从MAX_PID）开始，反复之前的查找过程。

由于MAX_PID总是大于MAX_PROCESS，因此这样的算法总是能够找到合适的pid，且绝对唯一。

## 练习3
### proc_run函数分析
核心的主要是以下一段函数

```
    local_intr_save(intr_flag);
    {
        current = proc;
        load_esp0(next->kstack + KSTACKSIZE);
        lcr3(next->cr3);
        switch_to(&(prev->context), &(next->context));
    }
    local_intr_restore(intr_flag);
```
首先需要关闭中断，以免在进程切换的过程中产生新的中断发生新的切换操作。这里需要保证执行的原子性。
这里加载esp0寄存器为新的内核栈地址，将cr3设置为新的页目录地址，然后就进入switch_to执行。

switch_to是一个汇编程序

```
switch_to:                      # switch_to(from, to)
    
    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)          # save esp::context of from
    movl %ebx, 8(%eax)          # save ebx::context of from
    movl %ecx, 12(%eax)         # save ecx::context of from
    movl %edx, 16(%eax)         # save edx::context of from
    movl %esi, 20(%eax)         # save esi::context of from
    movl %edi, 24(%eax)         # save edi::context of from
    movl %ebp, 28(%eax)         # save ebp::context of from
    
    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp         # restore ebp::context of to
    movl 24(%eax), %edi         # restore edi::context of to
    movl 20(%eax), %esi         # restore esi::context of to
    movl 16(%eax), %edx         # restore edx::context of to
    movl 12(%eax), %ecx         # restore ecx::context of to
    movl 8(%eax), %ebx          # restore ebx::context of to
    movl 4(%eax), %esp          # restore esp::context of to
    
    pushl 0(%eax)               # push eip
    ret
```
由x86的函数调用过程我们可以知道，当进入一个函数的时候，首先会从右往左依次压入各参数，
然后call指令会把函数的返回地址（EIP+4）压入栈中。于是进入switch_to之后，esp指向的是
压入的返回地址，esp+4为第一个参数（from），esp+8为第二个参数（to）。

然后把各个寄存器储存到from，即当前proc的context中。然后将to中的各个保存的寄存器的值恢复。

之后pushl 0(%eax)，eax指向的位置刚好存的是to的eip（eip是context中的第一个变量），
然后栈顶（esp）的位置就是要切换的进程的eip。

ret指令会pop出栈顶的值并设置为当前的eip，于是就完成了进程的切换。

### 在本实验的执行过程中，创建且运行了几个内核线程？
进入proc_init后，首先创建了一个idleproc，并设置为当前进程。

然后调用kernel_thread，里面调用do_fork，创建了一个新的进程，返回一个pid。在proc_init
将这个进程设置为initproc，并设置名字为“init”。

可以知道idleproc的pid为0，initproc的pid为1。可以通过运行的输出

```
this initproc, pid = 1, name = "init"
To U: "Hello world!!".
To U: "en.., Bye, Bye. :)"
```
验证这一点，表示我们运行了一个内核进程，执行的函数为init_main。

于是总共创建了两个内核线程。

### 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?
关闭中断，避免产生其他的陷入，产生另一个进程切换任务，这样会产生错误。这几句的执行需要保证原子性。
