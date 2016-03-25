# 操作系统实验Lab 2
## 实验报告

计34 何钦尧 2012010548

## 练习1
### 实现过程

主要是实现了三个函数
#### default\_init\_memmap
需要循环n个块并设置Page的各种flag，插入free_list
```
    SetPageProperty(p);
    p->property = 0;
    set_page_ref(p, 0);
    list_add_before(&free_list, &(p->page_link));
```
在循环完之后，需要设置内存块的头`base->property = n;`，表示其拥有连续n个页面。
以及`nr_free += n;`，表示累计总的free pages。

#### default\_alloc\_pages
使用first fit算法，从头开始循环找到第一个满足的块即可。
```
    while((le=list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if(p->property >= n){
            // find suitable pages and do something
        }
    }
```
对于分配的块，需要设置flag，以及从free_list中删去
```
    for (i = 0; i < n; i++) {
        len = list_next(le);
        struct Page *pp = le2page(le, page_link);
        SetPageReserved(pp);
        ClearPageProperty(pp);
        list_del(le);
        le = len;
    }
```
最后是更新空闲页数
```
    if (p->property > n) {
        (le2page(le, page_link))->property = p->property - n;
    }
    ClearPageProperty(p);
    SetPageReserved(p);
    nr_free -= n;
```

#### default\_free\_pages
释放页面，需要将释放的部分按顺序插入free_list，同时检查释放的部分中有没有可以合并的。

首先找到插入位置
```
    while ((le=list_next(le)) != &free_list) {
        p = le2page(le, page_link);
        if (p > base) {
            break;
        }
    }
```
然后是插入链表，并且设置flag
```
    for (p = base; p < base + n; p++) {
        list_add_before(le, &(p->page_link));
    }
    base->flags = 0;
    set_page_ref(base, 0);
    ClearPageProperty(base);
    SetPageProperty(base);
    base->property = n;
```
最后要尝试合并
```
    p = le2page(le, page_link) ;
    if (base+n == p){
        base->property += p->property;
        p->property = 0;
    }
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if (le != &free_list && p == base-1) {
        while (le != &free_list) {
            if (p->property) {
                p->property += base->property;
                base->property = 0;
                break;
            }
            le = list_prev(le);
            p = le2page(le,page_link);
        }
    }
```

### 设计实现的first fit算法是否有进一步的改进空间
目前的first fit算法实现，是把所有的页面都加入了链表，这样可能导致链表过大。

可以的改进思路有，在链表中把页面按连续的块管理，每个块含有连续的若干个页面。
这样可以减少链表的大小，以及提高查找的速度。

## 练习2
### 实现过程
需要实现一个函数`get_pte(pde_t *pgdir, uintptr_t la, bool create)`

首先获取页目录项，如果页目录项有效，即可直接获得页表项的地址。否则需要创建页表并填写页目录项。
```
    pde_t *pde_ptr = &pgdir[PDX(la)];
    if (!(*pde_ptr & PTE_P)) {
        // create pte
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pde_ptr)))[PTX(la)];
```
创建新的页表，首先需要新分配一个页
```
    alloc_page()
```
使得页目录项指向该物理页。然后是填写页目录项
```
    set_page_ref(page, 1);
    uintptr_t pa = page2pa(page);
    memset(KADDR(pa), 0, PGSIZE);
    *pde_ptr = pa | PTE_U | PTE_W | PTE_P;
```

### 页目录项和页表中每个组成部分的含义和以及对ucore而言的潜在用处
#### 页目录项

|位|说明|
|---|---|
|31-8|页表页帧号|
|7-3|预留位|
|2|PTE_U, User can access|
|1|PTE_W, Writeable|
|0|PTE_P, Present|

### 页表项

|位|说明|
|---|---|
|31-8|物理页帧号|
|7-3|预留位|
|2|PTE_U, User can access|
|1|PTE_W, Writeable|
|0|PTE_P, Present|

### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

硬件需要触发异常，保存现场信息，然后进入异常处理。

## 练习3
### 实现过程
首先检查pte的present位
```
    if (*ptep & PTE_P) {
        // some work
    }
```
对于合法的pte，清空页表项，释放物理页，并设置TLB无效
```
    if (page_ref_dec(page) == 0) {
        free_page(page);
    }
    *ptep = 0;
    tlb_invalidate(pgdir, la);
```

### 数据结构Page的全局变量的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

物理地址的高24位为pages数组的索引。

页目录项和页表项对虚拟地址转换出来的物理地址就用户page数组的访问。

### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事

需要使得物理页面的分配使用和虚拟地址相同的地址。需要修改alloc_page()函数的实现。
