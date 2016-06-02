# 操作系统实验Lab 7
## 实验报告

计34 何钦尧 2012010548

## 练习0
### 原有lab代码的改动
#### trap.c
在每个时钟中断的时候需要执行所设置的定时器，，在```trap_dispatch```函数中
```case IRQ_OFFSET + IRQ_TIMER:```分支中添加

```
    run_timer_list();
```
lab6中添加的

```
    sched_class_proc_tick(current);
```
由于已经被移植到timer list中，于是删去。

## 练习1
### 内核级信号量的设计描述
信号量的实现主要是```up```和```down```两个操作。在```sem.c```中实现的这两个函数，
分别调用```__up```和```__down```来实现具体的功能，传入```WT_KSEM```作为设置的wait_state
（表示等待的是内核信号量）。

以下分别对```__down```和```__up```两个函数进行分析。

#### __down
为了保证操作的原子性而不被打断，up和down操作都需要在进入函数之后关闭中断，然后在退出之前重新打开中断。

down操作需要先检查value的值是否大于零，如果是表示还有资源可用，就可以直接返回，
并将value值减一。如果不是大于零，就说明该信号量对应的资源已经分配完了，需要得到该资源则需要
进行阻塞的等待。

```
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
```
这里表示对该进程新建一个wait变量，然后将其加入该信号量的等待队列中。

然后进行```schedule```调度别的不阻塞的进程执行。等待该进程返回时（阻塞结束被调度运行），
再从信号量的等待队列里面将其删除。

#### __up
同样需要在进入和退出的时候关闭和打开中断。

执行up操作的时候需要从该信号量的等待队列里面拿出一个wakeup执行。如果没有正在等待的进程的话，
则增加信号量的value值。

```
    wait_t *wait;
    if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
        sem->value ++;
    }
    else {
        assert(wait->proc->wait_state == wait_state);
        wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
    }
```
注意到ucore中的信号量的实现和一般讲的不一样。通常的信号量的实现，value可以小于0的
（小于0多少就表示还有多少个进程在等待）。而ucore里面这里的实现是，value不可以小于0。
如果down的时候value已经为0，就不减少value的值，而是直接加入等待队列中。进而在up中也需要
进行修改，只有当等待队列为空的时候才增加value的值。


### 用户态线程信号量机制实现方案
用户态中不能关闭中断来保证若干条操作的原子性，这和在内核态中实现有一些不一样。

但是另一方面，用户态进程的切换的时机是用户态的运行库自己决定的。如果用户态线程库统一提供线程
以及同步互斥的功能的话，在信号量操作的内部完全可以通知线程库暂时关闭线程的切换。这样就保证了
原子性。其他的具体实现不需要变化。

或者可以采取另外的方案，设置一个位变量用来标识一个锁，然后通过CPU的原子指令，使用自旋锁那样
的方式来获取这个锁才能进入临界区进行信号量的相关操作。但是这会有忙等待的问题。

用户态和内核态实现的区别只是在于控制临界区访问的方法的不同，其他方面的主要原理都是一致的。

## 练习2
### 实现过程
这里基于已经有的信号量来实现条件变量，进而就实现了管程的功能。具体的实现是在
```monitor.c```中的```cond_signal```和```cond_wait```两个函数。

#### cond_wait
首先来看条件变量的结构体定义

```
typedef struct condvar{
    semaphore_t sem;
    int count;
    monitor_t * owner;
} condvar_t;
```
以及管程结构体的定义

```
typedef struct monitor{
    semaphore_t mutex;
    semaphore_t next;
    int next_count;
    condvar_t *cv;
} monitor_t;
```
在ucore中，一个条件变量总是属于一个管程的。管程本身的互斥锁由信号量实现。每个
条件变量总是初始化为0，这表示每次wait总是需要阻塞等待直到一个signal将其唤醒。

```condvar```中的```count```表示当前wait在该条件变量上的进程的数目。
```monitor```中的```next_count```表示在管程中因为signal而被阻塞的进程个数。

wait的具体实现为:

```
    cvp->count++;
    if (cvp->owner->next_count > 0) {
        up(&(cvp->owner->next));
    } else {
        up(&(cvp->owner->mutex));
    }
    down(&(cvp->sem));
    cvp->count--;
```
首先要将条件变量的计数加1，然后通过down条件变量的信号量来阻塞该进程。在从down中返回之后
表示已经从阻塞中恢复，再把计数减1。

中间的两句表示，如果当前在管程中有别的进程在等待，则将其唤醒执行。如果没有的话，释放管程
的互斥锁，使得新的进程可以进入管程中执行。这样子保证管程内部始终最多只有一个进程在执行，
而且在没有进程执行的时候，总是可以让新的进程进入。

#### cond_signal
先看signal的具体实现

```
    if(cvp->count>0) {
        cvp->owner->next_count++;
        up(&(cvp->sem));
        down(&(cvp->owner->next));
        cvp->owner->next_count--;
    }
```
if检查的是，如果没有进程在条件变量上等待，则什么都不做。如果有的话，要将其唤醒运行
```up(&(cvp->sem));```，并且要把当前进程阻塞到```owner->next```上，使得刚才唤醒的进程
能够马上调入运行，而不是等当前进程继续运行一段时间再被打断。这里同时修改```next_count```
的计数。

我们注意到一个进程在signal之后会立即阻塞并切换被signal唤醒的进程执行，于是这里实现的是
Hoare管程。

#### 使用管程解决哲学家就餐问题
初始化时设置管程中有N个条件变量（对应N个哲学家）。

需要实现的有```phi_take_forks_condvar```和```phi_put_forks_condvar```。

基于管程的实现，这两个函数都是在管程当中，在进入的时候需要获取管程访问权，退出的时候需要释放
管程访问。进入的时候获取管程的锁

```
    down(&(mtp->mutex));
```

退出的时候释放管程的锁，或者如果在管程中有进程处于阻塞等待，唤醒其执行而不释放管程锁。

```
    if(mtp->next_count>0)
        up(&(mtp->next));
    else
        up(&(mtp->mutex));
```

而在管程中，通过```phi_test_condvar```来获取两把叉子（即检测旁边的两个哲学家都没有在就餐），
如果没有成功获取的话，就会wait在自己对应的条件变量上。

在```phi_put_forks_condvar```中归还两把叉子并通知左右两边的两个哲学家检查是否可以开始
就餐，如果可以的话将其唤醒。

### 用户态线程条件变量实现方案
由于上面已经说明了用户态信号量的实现，可以直接利用用户态信号量来实现用户态的条件变量。
异同同上。