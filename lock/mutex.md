## 前言

mutex是内核中的互斥锁实现，本文对内核中的mutex机制进行了学习，在此记录一下。

## mutex结构体和定义

```c
struct mutex {
    atomic_long_t  owner;       //mutex持有的task
    spinlock_t  wait_lock;      //wait-lock的spinlock
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; // Spinner MCS lock，优化加锁过程
#endif
    struct list_head wait_list;
#ifdef CONFIG_DEBUG_MUTEXES
    void   *magic;              //debug相关
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map; //debug相关
#endif
};
```

mutex是内核的互斥锁，mutex具有以下几个语义：

1. 同一时刻只有一个task能够持有该锁
2. 只有锁的owner可以unlock
3. 多次unlock是不允许的
4. 递归加锁是不允许的
5. mutex对象必须使用APT来初始化
6. mutex不可以通过memset和copy来初始化
7. 当task持有mutex锁时无法exit
8. mutex所在的内存区域不能被释放
9. 持有的锁对象不能被重新初始化
10. mutex不能在硬中断或者软中断上下文中使用，比如tasklet和timer

## mutex的初始化

mutex的初始化是一个宏定义mutex_init，调用了__mutex_init函数

```c
#define mutex_init(mutex)
    --> __mutex_init((mutex), #mutex, &__key)
```

在__mutex_init中初始化lock->owner为0，也就是标记当前mutex是一个unlock状态。
初始化访问waitlist的spinlock以及wait_list。

```c
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
    atomic_long_set(&lock->owner, 0);
    spin_lock_init(&lock->wait_lock);
    INIT_LIST_HEAD(&lock->wait_list);
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    osq_lock_init(&lock->osq);
#endif

    debug_mutex_init(lock, name, key);
}
 ```

## mutex lock过程

mutex_lock请求对mutex进行加锁，如果当前锁不可用task会陷入sleep直到获取成功。
mutex具有以下几个限制：

1. mutex必须被持有锁的task释放
2. 递归的进行lock是不允许的
3. 持有mutex的还未unlock的task是无法exit
4. mutex所在的内核内存是无法释放的
5. mutex第一次加锁前需要被初始化
6. memset对mutex进行初始化是非法的

mutex_lock首先通过might_sleep()对该函数进行注释，表示该函数可能sleep。然后mutex_lock首先尝试使用__mutex_trylock_fast进行快速加锁，

```c
void __sched mutex_lock(struct mutex *lock)
{
    might_sleep();
    if (!__mutex_trylock_fast(lock))
        __mutex_lock_slowpath(lock);
}
```

__mutex_trylock_fast通过原子的CAS操作对mutex进行加锁，如果当前的lock->owner是zero，表示当前mutex未加锁，则将lock->owner设置为curr，加锁成功。否则返回false表示加锁失败。curr是通过current宏获取当前task的task_struct指针的地址。

```c
static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    unsigned long zero = 0UL;
    if (atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr))
        return true;
    return false;
}
```

如果在__mutex_trylock_fast中加锁失败则会进行__mutex_lock_slowpath。

```c
static noinline void __sched
__mutex_lock_slowpath(struct mutex *lock)
{
    __mutex_lock(lock, TASK_UNINTERRUPTIBLE, 0, NULL, _RET_IP_);
}

static int __sched
__mutex_lock(struct mutex *lock, long state, unsigned int subclass,
        struct lockdep_map *nest_lock, unsigned long ip)
{
    return __mutex_lock_common(lock, state, subclass, nest_lock, ip, NULL, false);
}
```

最终__mutex_lock_slowpath会进入__mutex_lock_common，除了lock和state是TASK_UNINTERRUPTIBLE外，剩下的参数都是0、NULL、false。

```c
static __always_inline int __sched
__mutex_lock_common(struct mutex *lock, long state, unsigned int subclass,
        struct lockdep_map *nest_lock, unsigned long ip,
        struct ww_acquire_ctx *ww_ctx, const bool use_ww_ctx)
{
    struct mutex_waiter waiter;
    bool first = false;
    struct ww_mutex *ww;
    int ret;
    ...
    // 禁止抢占
    preempt_disable();
    mutex_acquire_nest(&lock->dep_map, subclass, 0, nest_lock, ip);
    
    // 尝试获取lock
    if (__mutex_trylock(lock) ||
        mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx, NULL)) {
        /* got the lock, yay! */
    
        preempt_enable();
        return 0;
    }
    
    // 获取lock->wait_lock
    spin_lock(&lock->wait_lock);
    
    // 再获取了wait_lock后 过了一段时间 再次尝试获取lock
    if (__mutex_trylock(lock)) {
        goto skip_wait;
    }
    
    // 将waiter加入到lock的wait_list上
    if (!use_ww_ctx) {
        /*add waiting tasks to the end of the waitqueue (FIFO):*/
        __mutex_add_waiter(lock, &waiter, &lock->wait_list);
    } else {
        ...
    }
    //设置waiter的task为current
    waiter.task = current;
    //设置task状态为TASK_UNINTERRUPTIBLE
    set_current_state(state);
    
    for (;;) {
    /*
    * Once we hold wait_lock, we're serialized against
    * mutex_unlock() handing the lock off to us, do a trylock
    * before testing the error conditions to make sure we pick up
    * the handoff.
    */  
        // 再次尝试获取lock
        if (__mutex_trylock(lock))
            goto acquired;
    
        /*
        
    * Check for signals and kill conditions while holding
    * wait_lock. This ensures the lock cancellation is ordered
    * against mutex_unlock() and wake-ups do not go missing.
            */
        // 检查信号和kill条件
        if (unlikely(signal_pending_state(state, current))) {
            ret = -EINTR;
            goto err;
        }
        //释放wait_list锁
        spin_unlock(&lock->wait_lock);
        // 将cpu的抢占计数减1 然后调度别的task运行
        schedule_preempt_disabled();
        
            /*
        
    * ww_mutex needs to always recheck its position since its waiter
    * list is not FIFO ordered.
            */
        // 检查当前 waiter是否是lock_list的首位
        if ((use_ww_ctx && ww_ctx) || !first) {
            first = __mutex_waiter_is_first(lock, &waiter);
            // 如果是first则标记flag为MUTEX_FLAG_HANDOFF，表示需要将mutex转移给wait-list的第一个task
            if (first)
                __mutex_set_flag(lock, MUTEX_FLAG_HANDOFF);
        }
        //设置task状态为TASK_UNINTERRUPTIBLE
        set_current_state(state);
        /*
        * Here we order against unlock; we must either see it change
                *state back to RUNNING and fall through the next schedule(),
        * or we must see its unlock and acquire.
        */
        if (__mutex_trylock(lock) ||
        (first && mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx, &waiter)))
            break;
    
        spin_lock(&lock->wait_lock);
    }
        ...
}

```

对lock过程进行总结：

1. 首先尝试进行加锁
2. 如果加锁失败，则禁用抢占，获取mutex的wait-list的锁
3. 由于加入wait-list也要加锁，加入wait-list以后可能mutex已经空闲了，可以再次尝试加锁
4. 依然加锁失败，则将自己加入wait-list，设置状态为TASK_UNINTERRUPTIBLE后进入循环
5. 进入循环后
   1. 尝试加锁
   2. 失败后，释放wait-list的锁 调度别的task运行
   3. 被唤醒后 检查当前task是否是wait-list的首位，如果是设置lock的flag为MUTEX_FLAG_HANDOFF
   4. 设置task状态为TASK_UNINTERRUPTIBLE
   5. 尝试获取锁
   6. 如果失败，对lock->wait_lock加锁，回到步骤1

为了避免无意义的等待，lock过程中有多次尝试加锁，其中最关键的尝试加锁的操作包括两部分。

1. 第一部分是__mutex_trylock_or_owner，函数中检查lock状态，如果lock的owner是当前task并且flag设置了MUTEX_FLAG_PICKUP，则尝试去拿起锁。
2. 如果不满足条件，进入第二部分mutex_optimistic_spin，需要等待锁的释放，但是不是直接陷入睡眠，如果当前的mutex的owner正在cpu上运行则很可能会快速释放锁，此时进行自旋等待。

其中第一部分__mutex_trylock_or_owner的实现细节如下：

```c
/*
* Actual trylock that will work on any unlocked state.
*/
static inline bool __mutex_trylock(struct mutex*lock)
{
    return !__mutex_trylock_or_owner(lock);
}

/*
* Trylock variant that retuns the owning task on failure.
*/
static inline struct task_struct*__mutex_trylock_or_owner(struct mutex *lock)
{
    unsigned long owner, curr = (unsigned long)current;
    // 读取出当前的锁的持有者
    owner = atomic_long_read(&lock->owner);
    for (;;) { /*must loop, can race against a flag*/
        // 获取owner的flag
        unsigned long old, flags = __owner_flags(owner);
        // 消除指针的flag位 获取原始的task地址
        unsigned long task = owner & ~MUTEX_FLAGS;
        // 如果存在owner
        if (task) {
            // 如果owner不是本task 则获取失败
            if (likely(task != curr))
                break;
            // 如果当前flags不包含MUTEX_FLAG_PICKUP，也就是锁当前状态不是在等待别的task拿起获取，则获取失败
            if (likely(!(flags & MUTEX_FLAG_PICKUP)))
                break;
            // 说明获取锁成功 则消除flags的MUTEX_FLAG_PICKUP位
            flags &= ~MUTEX_FLAG_PICKUP;
        } else {
    # ifdef CONFIG_DEBUG_MUTEXES
            DEBUG_LOCKS_WARN_ON(flags & MUTEX_FLAG_PICKUP);
    # endif
        }

        /*
        * We set the HANDOFF bit, we must make sure it doesn't live
        * past the point where we acquire it. This would be possible
        * if we (accidentally) set the bit on an unlocked mutex.
        */
        // 消除flags的MUTEX_FLAG_HANDOFF位
        flags &= ~MUTEX_FLAG_HANDOFF;

        // 当前锁是可获取的，尝试修改lock的owner为task 并设置标志位为flags
        old = atomic_long_cmpxchg_acquire(&lock->owner, owner, curr | flags);
        // 修改成功 成功获得锁
        if (old == owner)
            return NULL;

        owner = old;
    }
    // 获取锁失败 当前锁有owner或者owner为当前task 但是不是等待pickup状态 返回当前owner
    return __owner_task(owner);
}
```

其中涉及到了lock的flag，关于这些flag的说明如下，由于地址是对齐的，所以在低位中可以存放flag标志。

1. bit-0表示队列非空
2. bit-1表示解锁需要将锁交给wait-list的第一位task
3. bit-2表示锁的转移已经完成，等待task拿起

```c
/*

* @owner: contains: 'struct task_struct *' to the current lock owner,
* NULL means not owned. Since task_struct pointers are aligned at
* at least L1_CACHE_BYTES, we have low bits to store extra state.
*
* Bit0 indicates a non-empty waiter list; unlock must issue a wakeup.
* Bit1 indicates unlock needs to hand the lock to the top-waiter
* Bit2 indicates handoff has been done and we're waiting for pickup.
 */
# define MUTEX_FLAG_WAITERS 0x01
# define MUTEX_FLAG_HANDOFF 0x02
# define MUTEX_FLAG_PICKUP 0x04
# define MUTEX_FLAGS  0x07
```

第二部分mutex_optimistic_spin函数会在每次唤醒时调用，如果当前task是wait-list的首位task，但是__mutex_trylock_or_owner获取锁失败，则尝试自旋等待一会。

mutex_optimistic_spin的实现细节如下，lock->osq是一个类似于MCS的自旋锁，在每次尝试加锁前会进行一些检查，确认当前有必要继续去spin在当前owner上。
如果出现以下场景则退出spin：

1. 出现need_resched标志：说明当前task即将被调度
2. !task->on_cpu：当前task已经不在cpu上执行
3. cpu处于preempted：cpu存在抢占

否则就尝试去获取mutex，直到成功或者没有必要继续spin则退出。

```c

#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
/*
 * Optimistic spinning.
 *
 * We try to spin for acquisition when we find that the lock owner
 * is currently running on a (different) CPU and while we don't
 * need to reschedule. The rationale is that if the lock owner is
 * running, it is likely to release the lock soon.
 *
 * The mutex spinners are queued up using MCS lock so that only one
 * spinner can compete for the mutex. However, if mutex spinning isn't
 * going to happen, there is no point in going through the lock/unlock
 * overhead.
 *
 * Returns true when the lock was taken, otherwise false, indicating
 * that we need to jump to the slowpath and sleep.
 *
 * The waiter flag is set to true if the spinner is a waiter in the wait
 * queue. The waiter-spinner will spin on the lock directly and concurrently
 * with the spinner at the head of the OSQ, if present, until the owner is
 * changed to itself.
 */
static __always_inline bool
mutex_optimistic_spin(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
        const bool use_ww_ctx, struct mutex_waiter *waiter)
{
    if (!waiter) {
        /*
        * The purpose of the mutex_can_spin_on_owner() function is
        * to eliminate the overhead of osq_lock() and osq_unlock()
        * in case spinning isn't possible. As a waiter-spinner
        * is not going to take OSQ lock anyway, there is no need
        * to call mutex_can_spin_on_owner().
        */
        // 检查是否需要获取osq-lock，如果cpu设置了need_resched或者当前cpu不在cpu上或者cpu已经被抢占则 不需要等待
        if (!mutex_can_spin_on_owner(lock))
            goto fail;

        /*
        * In order to avoid a stampede of mutex spinners trying to
        * acquire the mutex all at once, the spinners need to take a
        * MCS (queued) lock first before spinning on the owner field.
        */
        // 如果可以spin在owner上 则获取osq-lock
        if (!osq_lock(&lock->osq))
            goto fail;
    }

    for (;;) {
        struct task_struct *owner;

        /* Try to acquire the mutex... */
        // 尝试获得mutex
        owner = __mutex_trylock_or_owner(lock);
        if (!owner)
            break;

        /*
        * There's an owner, wait for it to either
        * release the lock or go to sleep.
        */
        // 再次检查是否需要spin
        if (!mutex_spin_on_owner(lock, owner, ww_ctx, waiter))
            goto fail_unlock;

        /*
        * The cpu_relax() call is a compiler barrier which forces
        * everything in this loop to be re-loaded. We don't need
        * memory barriers as we'll eventually observe the right
        * values at the cost of a few extra spins.
        */
        cpu_relax();
    }

    if (!waiter)
        osq_unlock(&lock->osq);

    return true;


    fail_unlock:
        if (!waiter)
            osq_unlock(&lock->osq);

    fail:
        /*
        * If we fell out of the spin path because of need_resched(),
        * reschedule now, before we try-lock the mutex. This avoids getting
        * scheduled out right after we obtained the mutex.
        */
        if (need_resched()) {
            /*
            * We _should_ have TASK_RUNNING here, but just in case
            * we do not, make it so, otherwise we might get stuck.
            */
            __set_current_state(TASK_RUNNING);
            schedule_preempt_disabled();
        }

    return false;
}
#else
static __always_inline bool
mutex_optimistic_spin(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
        const bool use_ww_ctx, struct mutex_waiter *waiter)
{
    return false;
}
#endif
```

## mutex unlock过程

mutex的unlock过程和mutex类似也是分为fast和slowpath两种。

```c
/**

* mutex_unlock - release the mutex
* @lock: the mutex to be released
*
* Unlock a mutex that has been locked by this task previously.
*
* This function must not be used in interrupt context. Unlocking
* of a not locked mutex is not allowed.
*
* This function is similar to (but not equivalent to) up().
 */
void __sched mutex_unlock(struct mutex*lock)
{
# ifndef CONFIG_DEBUG_LOCK_ALLOC
    if (__mutex_unlock_fast(lock))
        return;
# endif
    __mutex_unlock_slowpath(lock, _RET_IP_);
}
```

在__mutex_unlock_fast中检查owner是否是当前task，并且当前没有任何标志位，表明没有task正在wait该mutex，如果是则将owner清零就可以退出，否则还需要处理wait-list，设置标志位，唤醒新的task去获取mutex。

```c
static__always_inline bool __mutex_unlock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;

    if (atomic_long_cmpxchg_release(&lock->owner, curr, 0UL) == curr)
        return true;

    return false;
}
```

在slowpath中，首先创建了一个唤醒队列。
检查当前MUTEX_FLAG_HANDOFF标志，如果有则不需要清零owner，直接去设置owner为新的task。否则将owner清零，成功清零后检查当前flags如果有MUTEX_FLAG_WAITERS，说明当前的wait-list中有等待的task。

然后检查wait-list，取出wait-list的第一个task加入到wake queue，如果当前owner标记了
MUTEX_FLAG_HANDOFF，则将该task标记为MUTEX_FLAG_WAITERS和MUTEX_FLAG_PICKUP后更新为新的owner，当该task被唤醒后就能够拿到mutex。

```c
/*
* Release the lock, slowpath:
 */
static noinline void __sched __mutex_unlock_slowpath(struct mutex*lock, unsigned long ip)
{
    struct task_struct *next = NULL;
    DEFINE_WAKE_Q(wake_q);
    unsigned long owner;

    /*
    * Release the lock before (potentially) taking the spinlock such that
    * other contenders can get on with things ASAP.
    *
    * Except when HANDOFF, in that case we must not clear the owner field,
    * but instead set it to the top waiter.
    */
    owner = atomic_long_read(&lock->owner);
    for (;;) {
        unsigned long old;
        // 检查 MUTEX_FLAG_HANDOFF，该标志位会被被尝试lock的task设置，如果存在说明有task在等待mutex
        if (owner & MUTEX_FLAG_HANDOFF)
            break;

        // 将mutex的owner清零但保留标志位
        old = atomic_long_cmpxchg_release(&lock->owner, owner,
                __owner_flags(owner));
        // 清除成功后 如果wait-list中有task 则将mutex转移
        if (old == owner) {
            if (owner & MUTEX_FLAG_WAITERS)
                break;

            return;
        }

        owner = old;
    }
    
    // 对wait-list加锁 取出list的第一个task 放入wake_up_q
    spin_lock(&lock->wait_lock);

    if (!list_empty(&lock->wait_list)) {
        /* get the first entry from the wait-list: */
        struct mutex_waiter *waiter =
        list_first_entry(&lock->wait_list,
            struct mutex_waiter, list);

        next = waiter->task;

        wake_q_add(&wake_q, next);
    }

    // 如果owner设置了MUTEX_FLAG_HANDOFF，则将标记task为MUTEX_FLAG_WAITERS和MUTEX_FLAG_PICKUP后，一起设置为owner
    if (owner & MUTEX_FLAG_HANDOFF)
        __mutex_handoff(lock, next);

    spin_unlock(&lock->wait_lock);

    // 唤醒task去获取锁
    wake_up_q(&wake_q);
}
```

## mutex 实现总结

在mutex的lock过程中有两个优化，第一个优化是当前mutex无人持有时可以快速获取，否则进入slowpath，第二个优化是如果锁的owner正在运行中则尝试自旋等待。

unlock同样，分为fast和slowpath两种情况，当没有task等待该mutex时清零后退出即可，如果有task等待该mutex则进入slowpath。

mutex的转移和owner的flags相关，这三个flag放置在每个task地址的低3bit中，MUTEX_FLAG_WAITERS、MUTEX_FLAG_HANDOFF、MUTEX_FLAG_PICKUP三个标识位控制加锁和释放过程的处理。
