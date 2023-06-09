# 前言

参考《Linux内核源码情景分析》对缺页异常的处理过程，但是书中的kernel version版本较老，因此本文基于`kernel version 4.19.20`源码，对x86架构下的缺页异常，参考old version的内核源码剖析，再次进行了阅读。

# 缺页异常的产生原因

缺页异常就在通过虚拟地址去访问物理内存的过程中出现失败时抛出的异常，访问的过程包括几个阶段，首先是通过虚拟地址访问页表转换为物理地址，第二个阶段是通过物理地址访问物理内存，而错误的原因有以下几种：
1. 第一个阶段出现错误，通过MMU将虚拟地址转换为物理地址时发现不存在映射关系
2. 第二个阶段出错，通过物理地址访问内存，发现page不在内存当中
3. 第二个阶段出错，通过物理地址访问内存，发现权限错误

# 缺页异常的处理过程

x86架构下的缺页异常处理过程入口在`/arch/x86/mm/fault.c`的`do_page_fault`函数。

首先`do_page_fault`通过读取`cr2`寄存器（`cr2`中存放address，`cr3`中存放页表地址）获得访问的虚拟地址，调用`__do_page_fault`开始缺页处理。

```c
dotraplinkage void notrace
do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
    unsigned long address = read_cr2(); /* Get the faulting address */

    __do_page_fault(regs, error_code, address);
}
```

这里会输入`error code`，`error code`包括五位，`bit 0`指示是没有建立映射还是权限错误，`bit1 `指示是读操作还是写操作，`bit2 `表示发起访问的是用户模式还是内核模式，`bit3 `表示访问了保留位。

```c
/*
 * Page fault error code bits:
 *
 *   bit 0 ==	 0: no page found	1: protection fault
 *   bit 1 ==	 0: read access		1: write access
 *   bit 2 ==	 0: kernel-mode access	1: user-mode access
 *   bit 3 ==				1: use of reserved bit detected
 *   bit 4 ==				1: fault was an instruction fetch
 *   bit 5 ==				1: protection keys block access
 */
enum x86_pf_error_code {
    X86_PF_PROT		=		1 << 0,
    X86_PF_WRITE	=		1 << 1,
    X86_PF_USER		=		1 << 2,
    X86_PF_RSVD		=		1 << 3,
    X86_PF_INSTR	=		1 << 4,
    X86_PF_PK		=		1 << 5,
};
```

## 内核地址的异常处理

在`__do_page_fault`中会用到这些`error_code`进行逻辑判断。首先是进行判断`address`是否属于内核空间，如果是内核空间检查是正常访问，只是没有建立映射或者是不在内存中导致的异常，如果是对`vmalloc area`进行缺页处理，否则检查是否是TLB陈旧导致的，最后说明访问了不该访问的内存，此时调用`bad_area_nosemaphore`进行处理，其中`nosemaphore`表示不需要获取`mm_struct`的信号量，因为处理过程不会涉及到`mm_struct`。

```c
static noinline void
__do_page_fault(struct pt_regs *regs, unsigned long error_code,
        unsigned long address)
{
    struct vm_area_struct *vma;
    struct task_struct *tsk;
    struct mm_struct *mm;
    vm_fault_t fault, major = 0;
    unsigned int flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;
    u32 pkey;

    tsk = current;
    mm = tsk->mm;
    // 预取一个独占的cache line，避免出现cache一致性问题 
    prefetchw(&mm->mmap_sem);

    if (unlikely(kmmio_fault(regs, address)))
        return;

    //  如果fault的地址在处于内核地址空间（高地址部分）
    if (unlikely(fault_in_kernel_space(address))) {
        // 如果不是访问了保留位bit、在用户模式访问内核地址、权限问题等错误，说明了在内核中正常访问内存，
        // 但是内存没有建立映射或者不在内存中，则进行vmalloc area的缺页处理
        if (!(error_code & (X86_PF_RSVD | X86_PF_USER | X86_PF_PROT))) {
            if (vmalloc_fault(address) >= 0)
                return;
        }

        // 可能由于TLB的权限陈旧和page table entry的权限不一致 出现的伪错误
        /* Can handle a stale RO->RW TLB: */
        if (spurious_fault(error_code, address))
            return;
        
        /*
         * Don't take the mm semaphore here. If we fixup a prefetch
         * fault we could otherwise deadlock:
         */
        // 在内核中访问了bad_area 进行处理
        bad_area_nosemaphore(regs, error_code, address, NULL);

        return;
    }
    ...
}
```

`bad_area_nosemaphore的`处理，如果当前是用户模式，并且访问的地址是内核地址则设置task的`cr2`为`address`，`error code`设置上`X86_PF_PROT`，表示有权限问题，`trap_nr`设置为`X86_TRAP_PF`，表示page fault，将信号发送给task后就处理完毕了。此时task应该就寄了。
```c
static void
__bad_area_nosemaphore(struct pt_regs *regs, unsigned long error_code,
               unsigned long address, u32 *pkey, int si_code)
{
    struct task_struct *tsk = current;

    /* User mode accesses just cause a SIGSEGV */
    if (error_code & X86_PF_USER) {
            ...
        if (address >= TASK_SIZE_MAX)
            error_code |= X86_PF_PROT;

        if (likely(show_unhandled_signals))
            show_signal_msg(regs, error_code, address, tsk);

        tsk->thread.cr2		= address;
        tsk->thread.error_code	= error_code;
        tsk->thread.trap_nr	= X86_TRAP_PF;

        force_sig_info_fault(SIGSEGV, si_code, address, tsk, pkey, 0);

        return;
    }

    if (is_f00f_bug(regs, address))
        return;

    no_context(regs, error_code, address, SIGSEGV, si_code);
}
```

## 用户地址空间的异常处理
处理流程的核心代码如下：
```c
/*
 * This routine handles page faults.  It determines the address,
 * and the problem, and then passes it off to one of the appropriate
 * routines.
 */
static noinline void
__do_page_fault(struct pt_regs *regs, unsigned long error_code,
        unsigned long address)
{
    struct vm_area_struct *vma;
    struct task_struct *tsk;
    struct mm_struct *mm;
    vm_fault_t fault, major = 0;
    unsigned int flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;
    u32 pkey;

    tsk = current;
    mm = tsk->mm;

    ...

    /*
     * It's safe to allow irq's after cr2 has been saved and the
     * vmalloc fault has been handled.
     *
     * User-mode registers count as a user access even for any
     * potential system fault or CPU buglet:
     */
    if (user_mode(regs)) {
        local_irq_enable();
        error_code |= X86_PF_USER;
        flags |= FAULT_FLAG_USER;
    } else {
        if (regs->flags & X86_EFLAGS_IF)
            local_irq_enable();
    }

    if (error_code & X86_PF_WRITE)
        flags |= FAULT_FLAG_WRITE;
    if (error_code & X86_PF_INSTR)
        flags |= FAULT_FLAG_INSTRUCTION;

    // 从mm_struct中查找address属于的vma
    vma = find_vma(mm, address);

    // 如果不存在 说明访问了bad_area
    if (unlikely(!vma)) {
        bad_area(regs, error_code, address);
        return;
    }
    // 如果找到了vma address 处于vma中则转到good_area处理
    if (likely(vma->vm_start <= address))
        goto good_area;
    

    if (unlikely(!(vma->vm_flags & VM_GROWSDOWN))) {
        bad_area(regs, error_code, address);
        return;
    }

    if (error_code & X86_PF_USER) {
        /*
         * Accessing the stack below %sp is always a bug.
         * The large cushion allows instructions like enter
         * and pusha to work. ("enter $65535, $31" pushes
         * 32 pointers and then decrements %sp by 65535.)
         */
        if (unlikely(address + 65536 + 32 * sizeof(unsigned long) < regs->sp)) {
            bad_area(regs, error_code, address);
            return;
        }
    }
    if (unlikely(expand_stack(vma, address))) {
        bad_area(regs, error_code, address);
        return;
    }

    /*
     * Ok, we have a good vm_area for this memory access, so
     * we can handle it..
     */
good_area:
    if (unlikely(access_error(error_code, vma))) {
        bad_area_access_error(regs, error_code, address, vma);
        return;
    }

    /*
     * If for any reason at all we couldn't handle the fault,
     * make sure we exit gracefully rather than endlessly redo
     * the fault.  Since we never set FAULT_FLAG_RETRY_NOWAIT, if
     * we get VM_FAULT_RETRY back, the mmap_sem has been unlocked.
     *
     * Note that handle_userfault() may also release and reacquire mmap_sem
     * (and not return with VM_FAULT_RETRY), when returning to userland to
     * repeat the page fault later with a VM_FAULT_NOPAGE retval
     * (potentially after handling any pending signal during the return to
     * userland). The return to userland is identified whenever
     * FAULT_FLAG_USER|FAULT_FLAG_KILLABLE are both set in flags.
     * Thus we have to be careful about not touching vma after handling the
     * fault, so we read the pkey beforehand.
     */
    pkey = vma_pkey(vma);
    fault = handle_mm_fault(vma, address, flags);
    major |= fault & VM_FAULT_MAJOR;

    /*
     * If we need to retry the mmap_sem has already been released,
     * and if there is a fatal signal pending there is no guarantee
     * that we made any progress. Handle this case first.
     */
    if (unlikely(fault & VM_FAULT_RETRY)) {
        /* Retry at most once */
        if (flags & FAULT_FLAG_ALLOW_RETRY) {
            flags &= ~FAULT_FLAG_ALLOW_RETRY;
            flags |= FAULT_FLAG_TRIED;
            if (!fatal_signal_pending(tsk))
                goto retry;
        }

        /* User mode? Just return to handle the fatal exception */
        if (flags & FAULT_FLAG_USER)
            return;

        /* Not returning to user mode? Handle exceptions or die: */
        no_context(regs, error_code, address, SIGBUS, BUS_ADRERR);
        return;
    }

    up_read(&mm->mmap_sem);
    if (unlikely(fault & VM_FAULT_ERROR)) {
        mm_fault_error(regs, error_code, address, &pkey, fault);
        return;
    }
}
```


### address合法性检查

#### 查找vma

访问task的`mm_struct`，从`mm_struct`中查找address所在的vma，查找返回第一个vm_end大于address的vma。
首先会从mm的cache中查找vma，缓存查找失败则从红黑树中查找。

```c
/* Look up the first VMA which satisfies  addr < vm_end,  NULL if none. */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    struct rb_node *rb_node;
    struct vm_area_struct *vma;

    /* Check the cache first. */
    vma = vmacache_find(mm, addr);
    if (likely(vma))
        return vma;

    rb_node = mm->mm_rb.rb_node;

    while (rb_node) {
        struct vm_area_struct *tmp;

        tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);

        if (tmp->vm_end > addr) {
            vma = tmp;
            if (tmp->vm_start <= addr)
                break;
            rb_node = rb_node->rb_left;
        } else
            rb_node = rb_node->rb_right;
    }

    if (vma)
        vmacache_update(addr, vma);
    return vma;
}
```

#### 地址所处位置的判断

vma管理着所有的非空洞地址区域，堆地址空间通过brk系统调用从低地址往高地址增加，stack空间则是从高地址往低地址增加，因此如果没有查找到vma，说明所有的vma的vm_end都小于address，也就是访问了内核地址空间。

第二种情况，如果找到了一个vma，并且address处于vm_start和vm_end之间说明访问了有效的地址空间。

第三种情况，如果找到了一个vma，但是address也小于vm_start（不满足第二种情况时），此时访问的地址位于该vma的下方，但是又不在任何一个vma中，也就是访问的是一个空洞。
但是空洞存在两种，第一种是通过mmap和unmmap后在堆上或者是指定地址区域出现的空洞，另一种是处于堆栈之间还未使用的空间。此时根据vma（地址上方的vma）的`vm_flags`进行判断，如果标记了`VM_GROWSDOWN`，说明地址空间是向下增长的也就是处于堆栈之间，否则说明访问的第一种空洞。


```
            -->												<--
|code|data|heap---------------|---------hole---------|---------------stack|----kernel space----|
          |--hole--|----|-hole|
```
```c
    // 如果不存在 说明访问了bad_area
    if (unlikely(!vma)) {
        bad_area(regs, error_code, address);
        return;
    }
    // 如果找到了vma address 处于vma中则转到good_area处理
    if (likely(vma->vm_start <= address))
        goto good_area;
    

    if (unlikely(!(vma->vm_flags & VM_GROWSDOWN))) {
        bad_area(regs, error_code, address);
        return;
    }
```

#### bad_area

如果在用户模式访问了内核地址空间或者访问了不处于堆栈之间的空洞空间，则使用bad_area进行处理，调用`__bad_area_nosemaphore`，给进行发送一个SIGSEGV信号。

```c
static void
__bad_area(struct pt_regs *regs, unsigned long error_code,
       unsigned long address,  struct vm_area_struct *vma, int si_code)
{
    struct mm_struct *mm = current->mm;
    u32 pkey;

    if (vma)
        pkey = vma_pkey(vma);

    /*
     * Something tried to access memory that isn't in our memory map..
     * Fix it, but check if it's kernel or user first..
     */
    up_read(&mm->mmap_sem);

    __bad_area_nosemaphore(regs, error_code, address,
                   (vma) ? &pkey : NULL, si_code);
}
```

### 权限检查

当地址合法以后，进入`good_area`处理流程，`good area`是`__do_page_fault`的一个goto lable。第一步是进行权限检查。

```c
if (unlikely(access_error(error_code, vma))) {
        bad_area_access_error(regs, error_code, address, vma);
        return;
    }
```

权限检查中针对protection key、读写权限进行对比，如果vma的权限不满足则返回non-zero。

```c
static inline int
access_error(unsigned long error_code, struct vm_area_struct *vma)
{
    /* This is only called for the current mm, so: */
    bool foreign = false;

    /*
     * Read or write was blocked by protection keys.  This is
     * always an unconditional error and can never result in
     * a follow-up action to resolve the fault, like a COW.
     */
    if (error_code & X86_PF_PK)
        return 1;

    /*
     * Make sure to check the VMA so that we do not perform
     * faults just to hit a X86_PF_PK as soon as we fill in a
     * page.
     */
    if (!arch_vma_access_permitted(vma, (error_code & X86_PF_WRITE),
                       (error_code & X86_PF_INSTR), foreign))
        return 1;

    if (error_code & X86_PF_WRITE) {
        /* write, present and write, not present: */
        if (unlikely(!(vma->vm_flags & VM_WRITE)))
            return 1;
        return 0;
    }

    /* read, present: */
    if (unlikely(error_code & X86_PF_PROT))
        return 1;

    /* read, not present: */
    if (unlikely(!(vma->vm_flags & (VM_READ | VM_EXEC | VM_WRITE))))
        return 1;

    return 0;
}
```

针对鉴权失败，老规矩发送`SIGSEGV`，但是根据情况不同设置不同的`si_code`，也就是细化信号码。可能为proctection key导致的S`EGV_PKUERR`，或者是读写执行权限导致的`SEGV_ACCERR`。

```c
static noinline void
bad_area_access_error(struct pt_regs *regs, unsigned long error_code,
              unsigned long address, struct vm_area_struct *vma)
{
    /*
     * This OSPKE check is not strictly necessary at runtime.
     * But, doing it this way allows compiler optimizations
     * if pkeys are compiled out.
     */
    if (bad_area_access_from_pkeys(error_code, vma))
        __bad_area(regs, error_code, address, vma, SEGV_PKUERR);
    else
        __bad_area(regs, error_code, address, vma, SEGV_ACCERR);
}

```

### 处理page fault

此时发现地址、权限都没有问题，可以考虑处理异常，建立映射、将page从文件交换区swap进内存或者是分配内存等操作。

按照vm是否使用的hugetlb分为`hugetlb_fault`和`__handle_mm_fault`两种处理操作。

```c
fault = handle_mm_fault(vma, address, flags);

/*
 * By the time we get here, we already hold the mm semaphore
 *
 * The mmap_sem may have been released depending on flags and our
 * return value.  See filemap_fault() and __lock_page_or_retry().
 */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
        unsigned int flags)
{
    vm_fault_t ret;

    if (unlikely(is_vm_hugetlb_page(vma)))
        ret = hugetlb_fault(vma->vm_mm, vma, address, flags);
    else
        ret = __handle_mm_fault(vma, address, flags);

    /*
        * The task may have entered a memcg OOM situation but
        * if the allocation error was handled gracefully (no
        * VM_FAULT_OOM), there is no need to kill anything.
        * Just clean up the OOM state peacefully.
        */
    if (task_in_memcg_oom(current) && !(ret & VM_FAULT_OOM))
        mem_cgroup_oom_synchronize(false);
    

    return ret;
}
```

#### hugetlb的处理

看着比较复杂 等以后研究了hugetlb的实现机理再回来看。

```c

vm_fault_t hugetlb_fault(struct mm_struct *mm, struct vm_area_struct *vma,
            unsigned long address, unsigned int flags)
{
    pte_t *ptep, entry;
    spinlock_t *ptl;
    vm_fault_t ret;
    u32 hash;
    pgoff_t idx;
    struct page *page = NULL;
    struct page *pagecache_page = NULL;
    struct hstate *h = hstate_vma(vma);
    struct address_space *mapping;
    int need_wait_lock = 0;
    unsigned long haddr = address & huge_page_mask(h);

    ptep = huge_pte_offset(mm, haddr, huge_page_size(h));
    if (ptep) {
        entry = huge_ptep_get(ptep);
        if (unlikely(is_hugetlb_entry_migration(entry))) {
            migration_entry_wait_huge(vma, mm, ptep);
            return 0;
        } else if (unlikely(is_hugetlb_entry_hwpoisoned(entry)))
            return VM_FAULT_HWPOISON_LARGE |
                VM_FAULT_SET_HINDEX(hstate_index(h));
    } else {
        ptep = huge_pte_alloc(mm, haddr, huge_page_size(h));
        if (!ptep)
            return VM_FAULT_OOM;
    }

    mapping = vma->vm_file->f_mapping;
    idx = vma_hugecache_offset(h, vma, haddr);

    /*
     * Serialize hugepage allocation and instantiation, so that we don't
     * get spurious allocation failures if two CPUs race to instantiate
     * the same page in the page cache.
     */
    hash = hugetlb_fault_mutex_hash(h, mm, vma, mapping, idx, haddr);
    mutex_lock(&hugetlb_fault_mutex_table[hash]);

    entry = huge_ptep_get(ptep);
    if (huge_pte_none(entry)) {
        ret = hugetlb_no_page(mm, vma, mapping, idx, address, ptep, flags);
        goto out_mutex;
    }

    ret = 0;

    /*
     * entry could be a migration/hwpoison entry at this point, so this
     * check prevents the kernel from going below assuming that we have
     * a active hugepage in pagecache. This goto expects the 2nd page fault,
     * and is_hugetlb_entry_(migration|hwpoisoned) check will properly
     * handle it.
     */
    if (!pte_present(entry))
        goto out_mutex;

    /*
     * If we are going to COW the mapping later, we examine the pending
     * reservations for this page now. This will ensure that any
     * allocations necessary to record that reservation occur outside the
     * spinlock. For private mappings, we also lookup the pagecache
     * page now as it is used to determine if a reservation has been
     * consumed.
     */
    if ((flags & FAULT_FLAG_WRITE) && !huge_pte_write(entry)) {
        if (vma_needs_reservation(h, vma, haddr) < 0) {
            ret = VM_FAULT_OOM;
            goto out_mutex;
        }
        /* Just decrements count, does not deallocate */
        vma_end_reservation(h, vma, haddr);

        if (!(vma->vm_flags & VM_MAYSHARE))
            pagecache_page = hugetlbfs_pagecache_page(h,
                                vma, haddr);
    }

    ptl = huge_pte_lock(h, mm, ptep);

    /* Check for a racing update before calling hugetlb_cow */
    if (unlikely(!pte_same(entry, huge_ptep_get(ptep))))
        goto out_ptl;

    /*
     * hugetlb_cow() requires page locks of pte_page(entry) and
     * pagecache_page, so here we need take the former one
     * when page != pagecache_page or !pagecache_page.
     */
    page = pte_page(entry);
    if (page != pagecache_page)
        if (!trylock_page(page)) {
            need_wait_lock = 1;
            goto out_ptl;
        }

    get_page(page);

    if (flags & FAULT_FLAG_WRITE) {
        if (!huge_pte_write(entry)) {
            ret = hugetlb_cow(mm, vma, address, ptep,
                      pagecache_page, ptl);
            goto out_put_page;
        }
        entry = huge_pte_mkdirty(entry);
    }
    entry = pte_mkyoung(entry);
    if (huge_ptep_set_access_flags(vma, haddr, ptep, entry,
                        flags & FAULT_FLAG_WRITE))
        update_mmu_cache(vma, haddr, ptep);
out_put_page:
    if (page != pagecache_page)
        unlock_page(page);
    put_page(page);
out_ptl:
    spin_unlock(ptl);

    if (pagecache_page) {
        unlock_page(pagecache_page);
        put_page(pagecache_page);
    }
out_mutex:
    mutex_unlock(&hugetlb_fault_mutex_table[hash]);
    /*
     * Generally it's safe to hold refcount during waiting page lock. But
     * here we just wait to defer the next page fault to avoid busy loop and
     * the page is not used after unlocked before returning from the current
     * page fault. So we are safe from accessing freed page, even if we wait
     * here without taking refcount.
     */
    if (need_wait_lock)
        wait_on_page_locked(page);
    return ret;
}
```

#### normal size page的处理

struct vm_fault 这个结构体记录了最后要处理的page的page table的等信息，对于每一级页表，如果映射路径中存在的entry不在需要进行alloc，如果分配失败则抛出VM_FAULT_OOM。

```c
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
        unsigned long address, unsigned int flags)
{
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .flags = flags,
        .pgoff = linear_page_index(vma, address),
        .gfp_mask = __get_fault_gfp_mask(vma),
    };
    unsigned int dirty = flags & FAULT_FLAG_WRITE;
    struct mm_struct *mm = vma->vm_mm;
    pgd_t *pgd;
    p4d_t *p4d;
    vm_fault_t ret;

    pgd = pgd_offset(mm, address);
    p4d = p4d_alloc(mm, pgd, address);
    if (!p4d)
        return VM_FAULT_OOM;

    vmf.pud = pud_alloc(mm, p4d, address);
    if (!vmf.pud)
        return VM_FAULT_OOM;
        ...
    
    vmf.pmd = pmd_alloc(mm, vmf.pud, address);
    if (!vmf.pmd)
        return VM_FAULT_OOM;
        ...

    return handle_pte_fault(&vmf);
}
```

最后携带上vm_fault信息，去创建pte(Page Table Entry)。 `handle_pte_fault`的核心代码如如下，如果pte是null,没有页表项说明还没有建立过映射，此时根据vma是否是匿名的，去分配匿名页或者非匿名page。是否为匿名page取决于page是否基于文件，比如动态内存分配获取的就是匿名内存，基于文件的mmap得到的就是非匿名page。如果pte存在，但是当前pte不在内存中，将page从文件交换区换入。

```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;

    if (!vmf->pte) {
        if (vma_is_anonymous(vmf->vma))
            return do_anonymous_page(vmf);
        else
            return do_fault(vmf);
    }

    if (!pte_present(vmf->orig_pte))
        return do_swap_page(vmf);

    if (pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
        return do_numa_page(vmf);
}
```

### 总结一下

用户空间的地址，出现缺页异常的处理过程：
1. 地址检查
   1. 处于内核地址、或者处于非栈顶部和堆底部之间的内存，此时访问非法内存
   2. 处于某个vma之内，或者处于堆栈之间的地址，此时为good area
2. 权限检查
   1. X86会有protection keys的检查（对page的某种安全检查，protection keys会占几个bit，实现机制还没有研究过）
   2. 读写执行等权限检查
3. 缺页处理
   1. 大页的处理
   2. 普通页的处理
      1. 建立中间页表目录项
      2. 检查pte是否存在（不存在说明没有分配过page）
         1. pte不存在，分配page，按照vma的匿名性调用`do_anonymous_page`分配匿名page或者调用`do_fault`分配非匿名page
         2. pte存在，检查present bit，说明分配过page但是page不在内存中，调用`do_swap_page`将文件交换区的page交换到内存中
         3. 最后如果pte存在，并且在内存中，调用`do_numa_page`将其他node的page迁移到该cpu的node上


# reference

[1] 《Linux内核源代码情景分析》
[2] [source code-4.19.20](https://elixir.bootlin.com/linux/v4.19.20/source)