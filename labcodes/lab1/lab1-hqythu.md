# 操作系统实验Lab 1
## 实验报告

计34 何钦尧 2012010548

## 练习1
生成ucore.img的代码为
```
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```
这个依赖kernel和bootblock

## 练习2

## 练习3
在bootasm.S中
```
seta20.1:				# 循环等待8042键盘控制器
    inb $0x64, %al
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al		# 发送写8042输出端口的指令
    outb %al, $0x64

seta20.2:				# 循环等待8042键盘控制器
    inb $0x64, %al
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al		# 打开A20
    outb %al, $0x60
```
通过一条指令直接加载进来已经在内存中的GDT
```
	lgdt gdtdesc
```
将CR0寄存器的PE位置1就进入了保护模式
```
	movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```
然后设置各段寄存器
```
	movw $PROT_MODE_DSEG, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %fs
    movw %ax, %gs
    movw %ax, %ss
```
之后直接进入bootmain主方法

## 练习4
readseg和readsect函数实现了读取硬盘扇区到一个内存地址。

在bootmain中首先读取8个扇区，获取ELF头部
```
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
```
然后通过头部中的magic number来判断是否是合法的ELF
```
	if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
```
之后按照ELF头部中的描述表加载ELF到内存中的指定位置，并跳转到入口处
```
	for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

## 练习5
ebp指向的地址中中存放的是caller执行时的ebp，由此回溯就可以找出整个调用栈的栈帧，直到ebp=0（即栈的底端）

ebp+4中存放的是caller的eip（返回地址），ebp+8等依次是调用参数。

最后一行应该是第一个进入调用栈的函数，应为bootmain。
bootloader设置的栈起始地址是0x7c00，于是进入bootmain后压栈，ebp应为0x7bf8。

## 练习6
中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移， 两者联合便是中断处理程序的入口地址。

2、3问的实现见代码

## 扩展练习
在idt_init中，将用户态调用SWITCH_TOK中断的权限打开。
```
	SETGATE(idt[T_SWITCH_TOK], 1, KERNEL_CS, __vectors[T_SWITCH_TOK], 3);
```

在trap_dispatch中，将iret时会从堆栈弹出的段寄存器进行修改
	对TO User
```
	    tf->tf_cs = USER_CS;
	    tf->tf_ds = USER_DS;
	    tf->tf_es = USER_DS;
	    tf->tf_ss = USER_DS;
```
	对TO Kernel

```
	    tf->tf_cs = KERNEL_CS;
	    tf->tf_ds = KERNEL_DS;
	    tf->tf_es = KERNEL_DS;
```

在lab1_switch_to_user中，调用T_SWITCH_TOU中断。
注意从中断返回时，会多pop两位，并用这两位的值更新ss,sp，损坏堆栈。
所以要先把栈压两位，并在从中断返回后修复esp。
```
	asm volatile (
	    "sub $0x8, %%esp \n"
	    "int %0 \n"
	    "movl %%ebp, %%esp"
	    : 
	    : "i"(T_SWITCH_TOU)
	);
```

在lab1_switch_to_kernel中，调用T_SWITCH_TOK中断。
注意从中断返回时，esp仍在TSS指示的堆栈中。所以要在从中断返回后修复esp。
```
	asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n"
	    : 
	    : "i"(T_SWITCH_TOK)
	);
```

但这样不能正常输出文本。根据提示，在trap_dispatch中转User态时，将调用io所需权限降低。
```
	tf->tf_eflags |= 0x3000;
```