# 操作系统实验Lab 6
## 实验报告

计34 何钦尧 2012010548

## 练习0
### 原有lab代码的改动
#### proc.c
```proc_struct```中添加了本次实验需要的进程调度相关的几个字段：

```
    int time_slice;
    skew_heap_entry_t lab6_run_pool;
    uint32_t lab6_stride;
    uint32_t lab6_priority;
```

```run_pool```是一个斜堆（skew_heap）。在```alloc_proc```中需要对这几个新的字段添加
初始化：

```
    proc->time_slice = 0;
    proc->lab6_run_pool.left = NULL;
    proc->lab6_run_pool.right = NULL;
    proc->lab6_run_pool.parent = NULL;
    proc->lab6_stride = 0;
    proc->lab6_priority = 0;
```

#### trap.c
在每个时钟中断的时候需要给调度器通知，以更新各种相关的数据。在```trap_dispatch```函数中的
```case IRQ_OFFSET + IRQ_TIMER:```分支中添加下面一句：

```
    sched_class_proc_tick(current);
```

需要注意的是，在```sched.c```中该函数被声明为static，不能直接被外部调用。
这里为了达到这个实现的目的，将该函数的static修饰去掉，并且在```sched.h```中添加
该函数的声明。


## 练习1
### Round Robin 调度算法, ucore调度过程
在实现Round Robin算法的```default_sched.c```中，有如下：

```
struct sched_class default_sched_class = {
    .name = "RR_scheduler",
    .init = RR_init,
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};
```
在```default_sched.h```中定义了

```
extern struct sched_class default_sched_class;
```
于是```sched.c```中就可以使用```sched_class```来调用RR算法的功能。

```sched_class```通用的接口就是，入队列，出队列，获取队首，时钟事件等几个操作。
这里实现的RR算法维护了一个就绪进程的队列。每次进行调度的时候，把当前进程（如果还是处于就绪）
加入到队列的末尾，然后取出该队列头部的进程（同时从队列中删除），选择其调度执行。如果
队列中没有可调度的进程，就选取idleproc运行。

```wakeup_proc```函数中，也直接将要唤醒的那个进程加入就绪队列的末尾。

### 设计实现多级反馈队列调度算法
多级反馈队列算法需要若干个不同优先级的队列。为了记录进程的优先级，也需要在```proc_struct```
中添加优先级的变量。

enqueue的时候，需要根据进程的优先级判断应该加入哪个队列。dequeue的时候根据proc中保存的rq
信息找到对应的队列进行删除。

为了实现多种不同的时间片大小，应该在入队的时候给不同优先级的进程设置不同大小的time_slice。
在调度切换的时候需要判断是因为进程运行结束（或者是因为阻塞IO）而切换还是因为被抢占式打断。
这可以通过对```proc->time_slice```是否等于0的判断来实现。如果等于0，说明是被打断，需要
降低进程的优先级，并加入对应优先级的队列。

dequeue的时候，从优先级高的队列开始查找，如果为空就往下查找优先级低一级的进程。

## 练习2
### 实现过程
将```default_sched_stride_c```覆盖```default_sched.c```后，需要编写的就是几个函数

```
    stride_init();
    stride_enqueue();
    stride_dequeue();
    stride_pick_next();
    stride_proc_tick();
```

#### stride\_init
需要对几个变量初始化

```
    list_init(&(rq->run_list));
    rq->lab6_run_pool = NULL;
    rq->proc_num = 0;
```
将```run_list```设置为空列表，```run_pool```为NULL，表示一个空的斜堆。

#### stride\_enqueue
需要把当前的进程添加到斜堆当中，使用```skew_heap```提供的工具函数```skew_heap_insert```。

```
    rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, 
        &(proc->lab6_run_pool), proc_stride_comp_f);
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num++;
```
斜堆中使用的比较函数为```proc_stride_comp_f```，这里使得每次从堆中取出的一定是stride
最小的进程。

#### stride\_dequeue
在堆中删除该进程

```
    rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool,
        &(proc->lab6_run_pool), proc_stride_comp_f);
    rq->proc_num--;
```

#### stride\_pick\_next
取当前stride最小的进程（也就是堆顶元素，```lab6_run_pool```）。然后
根据进程的优先级增加这个进程的stride。

```
    if (rq->lab6_run_pool == NULL) {
        return NULL;
    }
    struct proc_struct* p = le2proc(rq->lab6_run_pool, lab6_run_pool);
    if (p->lab6_priority == 0) {
        p->lab6_stride += BIG_STRIDE;
    } else {
        p->lab6_stride += BIG_STRIDE / p->lab6_priority;
    }
    return p;
```

#### stride\_proc\_tick
减少该进程还可以运行的time_slice，到0了之后设置需要调度的标志。

```
    if (proc->time_slice > 0) {
        proc->time_slice--;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
```

