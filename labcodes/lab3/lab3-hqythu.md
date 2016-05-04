# 操作系统实验Lab 3
## 实验报告

计34 何钦尧 2012010548

## 练习1
### 实现过程
在do_pgfault函数当中

```
	if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
        cprintf("get_pte in do_pgfault failed\n");
        return ret;
    }
```
这里用来从addr获取pte(page table entry)。参数1表示如果不存在该页表（PT）的话，
就创建一个（详见pmm.c中的get_pte函数的实现）。

之后

```
	if (*ptep == 0) {
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
            return ret;
        }
    }
```
如果pte的值为0（这表示无效，即没有对应的物理页面），则分配一个物理页面。
如果不是0，由于这里已经进入了do_pgfault处理流程，那么产生缺页错误的原因只可能是
页面不在内存中而在磁盘上，我们需要从磁盘中调入相应的页面。

```
	else {
        if(swap_init_ok) {
            struct Page *page=NULL;
            if ((ret = swap_in(mm, addr, &page)) != 0) {
                cprintf("swap_in in do_pgfault failed\n");
                return ret;
            }    
            page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
            page->pra_vaddr = addr;
        }
```
swap_in函数定义在swap.c中，将根据mm和addr来获取pte（同样使用get_pte函数），然后
根据pte中的内容去磁盘中调入该页面，存放在新分配的一个物理Page中（传入的page参数就更新为
这个新分配的物理页面）。

然后page_insert会建立该虚拟地址到物理页面的映射（修改页表）。最后标记该page为swappable。

### 页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处

#### 页目录项
|位|说明|
|---|---|
|31-8|页表页帧号|
|7-3|预留位|
|2|PTE_U, User can access|
|1|PTE_W, Writeable|
|0|PTE_P, Present|

#### 页表项

|位|说明|
|---|---|
|31-8|物理页帧号|
|7-3|预留位|
|2|PTE_U, User can access|
|1|PTE_W, Writeable|
|0|PTE_P, Present|

#### 作用
当需要进行页面在内存和磁盘的换入换出时，可以将PTE的物理页帧号用来索引磁盘中储存的页。

### 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
缺页服务例程运行在内核态，此时发生了异常，于是按照内核态的异常处理流程，
硬件需要将EFLAGS，CS，EIP，error-code压栈（此时是内核栈），并跳转到异常处理入口地址处执行。


## 练习2
### 实现过程
主要补充实现了_fifo_map_swappable和_fifo_swap_out_victim两个函数。

_fifo_map_swappable需要将最近访问过的页面添加到链表的头部，即

```
	list_add(head, entry);
```
这样插入到head的后面，就成为链表的第一个。

_fifo_swap_out_victim中需要选出一个作为换出，这里取出的是链表的尾部，使用head->prev来获取（由于这里是双向链表）

```
	list_entry_t *le = head->prev;
    struct Page *p = le2page(le, pra_page_link);
    list_del(le);
    assert(p != NULL);
    *ptr_page = p;
```
这里面le就是要换出的那个Page。然后从链表中将le删除掉。

### 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？
可行。

对swap_manager接口的实现：

init:
建立循环链表。

map_swappable:
读取TLB，将访问位更新，即被访问过的页的访问位++，清零TLB上的该标记。
根据时钟算法，顺着顺便链表搜索和修改相应的标志位，直到找到合适的页，返回该页。

swap_out_victim:
将新的页加入链表，访问位0++
