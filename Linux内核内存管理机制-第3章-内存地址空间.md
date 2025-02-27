# Linux内核内存管理机制

## 3. 内存地址空间与TLB管理

### 3.1 内核空间与用户空间

Linux系统将虚拟地址空间分为内核空间和用户空间两部分，这种分离提供了安全性和稳定性保障。

#### 3.1.1 地址空间划分

在64位系统上，典型的地址空间划分如下：

```
0000000000000000 - 00007fffffffffff (=47位) 用户空间
ffff800000000000 - ffffffffffffffff (=47位) 内核空间
```

这种划分使得：
- 每个进程拥有独立的用户空间
- 所有进程共享同一个内核空间
- 用户程序无法直接访问内核空间

#### 3.1.2 内核空间布局

内核空间的典型布局（ARM64架构）：

```
ffffc90000000000 - ffffc9ffffffffff : 模块映射区域
ffffc10000000000 - ffffc1ffffffffff : vmalloc区域
ffff800000000000 - ffff80ffffffffff : 直接映射区域
```

1. **直接映射区域**：物理内存的线性映射，虚拟地址到物理地址的转换简单快速
2. **vmalloc区域**：用于分配虚拟连续但物理不连续的内存
3. **模块映射区域**：用于加载内核模块

#### 3.1.3 用户空间布局

用户空间的典型布局：

```
地址从低到高：
[0] 保留区域（不可访问）
[低地址] 代码段（.text）
[...] 数据段（.data, .bss）
[...] 堆（向上增长）
[...] 内存映射区域（mmap区域）
[...] 栈（向下增长）
[高地址] 参数和环境变量
```

用户空间布局由内核控制，通过`/proc/sys/vm/mmap_min_addr`等参数可以调整某些区域的边界。

### 3.2 虚拟内存区域（VMA）

#### 3.2.1 VMA基本概念

虚拟内存区域（Virtual Memory Area, VMA）是进程地址空间的基本管理单元，表示一段连续的虚拟地址范围。

VMA的核心数据结构：

```c
struct vm_area_struct {
    /* VMA的起始和结束地址 */
    unsigned long vm_start;
    unsigned long vm_end;

    /* 指向所属mm_struct的指针 */
    struct mm_struct *vm_mm;

    /* 访问权限标志 */
    pgprot_t vm_page_prot;
    unsigned long vm_flags;

    /* 红黑树节点 */
    struct rb_node vm_rb;

    /* 链表节点 */
    struct list_head vm_next;

    /* 文件映射相关 */
    struct file *vm_file;
    unsigned long vm_pgoff;

    /* VMA操作函数 */
    const struct vm_operations_struct *vm_ops;

    /* 私有数据 */
    void *vm_private_data;
};
```

### 3.3 TLB管理与优化

#### 3.3.1 TLB基本概念

TLB（Translation Lookaside Buffer）是一个硬件缓存，用于加速虚拟地址到物理地址的转换过程。它缓存了最近使用的页表项，减少了访问内存中页表的次数。

#### 3.3.2 TLB优化策略

1. **TLB预取技术**
   - 基于空间局部性的预取：预取相邻页表项
   - 基于访问模式的预测：分析内存访问模式预取TLB
   - 软件引导预取：编译器插入TLB预取指令
   - 硬件实现：ARM v8.2 TTLBOS (Translation Table Level Based Prefetch) 机制
   - 性能影响：可减少15-25%的TLB miss率

2. **TLB能耗优化**
   - 选择性TLB失效：只刷新必要的TLB条目
   - 自适应TLB大小：根据工作负载动态调整TLB大小
   - 智能TLB电源管理：低功耗状态下部分TLB逻辑断电
   - ASID分配优化：减少全局TLB刷新频率

3. **TLB监控与分析**
   ```bash
   # 监控TLB miss事件
   perf stat -e dTLB-load-misses,dTLB-store-misses ./your_application

   # 查看当前TLB状态
   cat /proc/pagetypeinfo
   cat /proc/vmstat | grep tlb
   ```

### 3.4 内存屏障与一致性

#### 3.4.1 内存屏障基础

内存屏障是确保多处理器系统中内存操作顺序的低级同步原语。它们解决了以下问题：

- 编译器重排序：编译器优化可能改变指令顺序
- CPU重排序：处理器可能乱序执行内存访问
- 缓存一致性：多核系统中的缓存同步延迟

#### 3.4.2 Linux内存屏障API

Linux内核提供了一组内存屏障原语：

```c
/* 通用内存屏障 */
void mb(void);      /* 完全内存屏障 */
void rmb(void);     /* 读内存屏障 */
void wmb(void);     /* 写内存屏障 */

/* 带获取/释放语义的屏障 */
void smp_mb(void);  /* SMP完全内存屏障 */
void smp_rmb(void); /* SMP读内存屏障 */
void smp_wmb(void); /* SMP写内存屏障 */

/* 编译器屏障 */
void barrier(void); /* 防止编译器重排序 */
```

#### 3.4.3 内存屏障使用场景

内存屏障在以下场景中至关重要：

1. **锁实现**：
   ```c
   /* 自旋锁实现示例 */
   void spin_lock(spinlock_t *lock)
   {
       /* 尝试获取锁 */
       while (test_and_set_bit_lock(0, &lock->slock)) {
           /* 等待 */
       }

       /* 获取锁后的内存屏障 */
       smp_mb__after_spinlock();
   }
   ```

2. **无锁数据结构**：
   ```c
   /* 无锁队列示例 */
   void queue_push(struct queue *q, struct node *n)
   {
       n->next = NULL;
       struct node *prev = xchg(&q->tail, n);

       /* 确保节点完全初始化后再链接 */
       smp_wmb();

       prev->next = n;
   }
   ```

1. **批量TLB Flush**
   - 收集多个TLB失效请求，一次性处理
   - 减少TLB刷新频率
   - 可显著提升性能

2. **延迟TLB Flush**
   - 推迟TLB flush到必要时刻
   - 使用generation计数跟踪延迟状态
   - 适用于频繁修改但访问不频繁的页表项

3. **范围TLB Flush**
   - 只刷新特定地址范围的TLB条目
   - 避免全局TLB flush带来的性能损失
   - 特别适用于大内存系统

#### 3.3.3 ARM64架构下的TLB Shootdown机制

TLB Shootdown是多处理器系统中维护TLB一致性的关键机制。当一个CPU修改了页表项时，需要通知其他CPU使其TLB中的相应表项失效。在ARM64架构中，这个过程得到了硬件的特殊支持。

1. **触发场景**
   - 页表项修改
   - 进程地址空间修改
   - 内存映射变更
   - 进程地址空间释放
   - 大页拆分或合并
   - mprotect系统调用改变内存权限

2. **基本流程**
   ```
   CPU 0                     CPU 1,2,3...
   修改页表项   ----IPI----> 接收TLB失效请求
      |                          |
   等待确认    <---ACK-----  执行TLB失效
      |                          |
   继续执行    <------------  继续执行
   ```

#### 3.3.2 ARM64硬件支持

1. **TLBI指令族**
   ```c
   // ARM64提供了一系列TLB失效指令
   TLBI ALLE1                 // 使所有EL1级别的TLB表项失效
   TLBI VAE1, Xt             // 使指定虚拟地址的EL1表项失效
   TLBI ASIDE1, Xt           // 使指定ASID的表项失效
   ```

2. **硬件广播机制**
   - ARM64处理器支持硬件级别的TLB广播
   - 通过系统总线自动传播TLB失效请求
   - 支持细粒度的地址范围失效

#### 3.3.3 Linux内核实现

1. **MMU Notifier机制**
   ```c
    /*
     * 处理MMU通知器范围失效的起始阶段
     * @subscriptions: MMU通知器订阅列表
     * @range: 需要失效的地址范围信息
     * 
     * 该函数遍历所有注册的MMU通知器，调用它们的invalidate_range_start回调函数
     * 主要用于通知各个子系统（如设备驱动、虚拟化层）页表即将发生变化
     */
    static int mn_hlist_invalidate_range_start(
        struct mmu_notifier_subscriptions *subscriptions,
        struct mmu_notifier_range *range)
    {
        struct mmu_notifier *subscription;
        int ret = 0;
        int id;

        /* 获取SRCU读锁，保护遍历过程中的并发安全 */
        id = srcu_read_lock(&srcu);
        /* 使用RCU安全地遍历所有注册的通知器 */
        hlist_for_each_entry_rcu(subscription, &subscriptions->list, hlist,
                    srcu_read_lock_held(&srcu)) {
            const struct mmu_notifier_ops *ops = subscription->ops;

            if (ops->invalidate_range_start) {
                int _ret;

                /* 处理非阻塞模式的特殊情况 */
                if (!mmu_notifier_range_blockable(range))
                    non_block_start();
                /* 调用通知器的回调函数 */
                _ret = ops->invalidate_range_start(subscription, range);
                if (!mmu_notifier_range_blockable(range))
                    non_block_end();
                if (_ret) {
                    /* 处理回调函数执行失败的情况 */
                    pr_info("%pS callback failed with %d in %sblockable context.\n",
                        ops->invalidate_range_start, _ret,
                        !mmu_notifier_range_blockable(range) ?
                            "non-" :
                            "");
                    /* 在阻塞上下文中，只允许-EAGAIN错误 */
                    WARN_ON(mmu_notifier_range_blockable(range) ||
                        _ret != -EAGAIN);
                    /*
                     * 对于EAGAIN错误，我们仍然会调用所有的通知器
                     * 因为通知器无法知道其start方法是否失败
                     * 所以执行EAGAIN的start不能同时执行end
                     */
                    WARN_ON(ops->invalidate_range_end);
                    ret = _ret;
                }
            }
        }

        /* 处理失败情况下的清理工作 */
        if (ret) {
            /*
             * 到达这里必须是非阻塞模式
             * 如果有多个通知器且其中一个或多个start失败
             * 对于那些start成功的通知器，需要调用它们的end方法
             */
            hlist_for_each_entry_rcu(subscription, &subscriptions->list,
                        hlist, srcu_read_lock_held(&srcu)) {
                if (!subscription->ops->invalidate_range_end)
                    continue;

                subscription->ops->invalidate_range_end(subscription,
                                    range);
            }
        }
        /* 释放SRCU读锁 */
        srcu_read_unlock(&srcu, id);

        return ret;
    }

    /*
     * MMU通知器范围失效的主入口函数
     * @range: 需要失效的地址范围信息
     *
     * 该函数首先处理区间树通知器，然后处理哈希链表中的通知器
     */
    int __mmu_notifier_invalidate_range_start(struct mmu_notifier_range *range)
    {
        struct mmu_notifier_subscriptions *subscriptions =
            range->mm->notifier_subscriptions;
        int ret;

        /* 首先处理区间树通知器 */
        if (subscriptions->has_itree) {
            ret = mn_itree_invalidate(subscriptions, range);
            if (ret)
                return ret;
        }
        /* 然后处理哈希链表中的通知器 */
        if (!hlist_empty(&subscriptions->list))
            return mn_hlist_invalidate_range_start(subscriptions, range);
        return 0;
    }

    /*
     * 处理MMU通知器范围失效的结束阶段
     * @subscriptions: MMU通知器订阅列表
     * @range: 需要失效的地址范围信息
     *
     * 该函数遍历所有注册的MMU通知器，调用它们的invalidate_range_end回调函数
     */
    static void
    mn_hlist_invalidate_end(struct mmu_notifier_subscriptions *subscriptions,
                struct mmu_notifier_range *range)
    {
        struct mmu_notifier *subscription;
        int id;

        /* 获取SRCU读锁 */
        id = srcu_read_lock(&srcu);
        /* 遍历所有注册的通知器 */
        hlist_for_each_entry_rcu(subscription, &subscriptions->list, hlist,
                    srcu_read_lock_held(&srcu)) {
            if (subscription->ops->invalidate_range_end) {
                /* 处理非阻塞模式 */
                if (!mmu_notifier_range_blockable(range))
                    non_block_start();
                /* 调用通知器的end回调函数 */
                subscription->ops->invalidate_range_end(subscription,
                                    range);
                if (!mmu_notifier_range_blockable(range))
                    non_block_end();
            }
        }
        /* 释放SRCU读锁 */
        srcu_read_unlock(&srcu, id);
    }

    /*
     * MMU通知器范围失效结束的主入口函数
     * @range: 需要失效的地址范围信息
     *
     * 该函数完成整个范围失效过程，包括区间树和哈希链表中的通知器
     */
    void __mmu_notifier_invalidate_range_end(struct mmu_notifier_range *range)
    {
        struct mmu_notifier_subscriptions *subscriptions =
            range->mm->notifier_subscriptions;

        /* 获取锁以确保与start操作的同步 */
        lock_map_acquire(&__mmu_notifier_invalidate_range_start_map);
        /* 处理区间树通知器的结束阶段 */
        if (subscriptions->has_itree)
            mn_itree_inv_end(subscriptions);

        /* 处理哈希链表中通知器的结束阶段 */
        if (!hlist_empty(&subscriptions->list))
            mn_hlist_invalidate_end(subscriptions, range);
        /* 释放锁 */
        lock_map_release(&__mmu_notifier_invalidate_range_start_map);
    }

    /*
     * 使二级TLB失效的架构特定函数
     * @mm: 进程的内存描述符
     * @start: 需要失效的起始地址
     * @end: 需要失效的结束地址
     *
     * 该函数用于处理特定架构上的二级TLB失效操作
     * 主要用于虚拟化环境中Guest OS的TLB管理
     */
    void __mmu_notifier_arch_invalidate_secondary_tlbs(struct mm_struct *mm,
                        unsigned long start, unsigned long end)
    {
        struct mmu_notifier *subscription;
        int id;

        /* 获取SRCU读锁 */
        id = srcu_read_lock(&srcu);
        /* 遍历所有注册的通知器 */
        hlist_for_each_entry_rcu(subscription,
                    &mm->notifier_subscriptions->list, hlist,
                    srcu_read_lock_held(&srcu)) {
            /* 调用架构特定的二级TLB失效函数 */
            if (subscription->ops->arch_invalidate_secondary_tlbs)
                subscription->ops->arch_invalidate_secondary_tlbs(
                    subscription, mm,
                    start, end);
        }
        /* 释放SRCU读锁 */
        srcu_read_unlock(&srcu, id);
    }
   ```

2. **IPI处理流程**
   ```c
   /*
    * 在多处理器系统中刷新其他CPU的TLB
    * @cpumask: 需要执行TLB刷新的CPU掩码
    * @mm: 需要刷新TLB的进程内存描述符
    * @start: 需要刷新的虚拟地址范围起始地址
    * @end: 需要刷新的虚拟地址范围结束地址
    *
    * 该函数通过处理器间中断(IPI)实现跨CPU的TLB同步
    * 这是保持多CPU系统TLB一致性的关键机制
    */
   static void native_flush_tlb_others(const struct cpumask *cpumask,
           struct mm_struct *mm, unsigned long start, unsigned long end)
   {
       /* 准备TLB刷新信息 */
       struct tlb_flush_info info;
       info.mm = mm;      /* 要刷新的地址空间 */
       info.start = start;  /* 刷新范围的起始地址 */
       info.end = end;      /* 刷新范围的结束地址 */
       
       /* 通过IPI触发其他CPU执行TLB刷新
        * 最后的参数1表示等待所有CPU完成刷新操作
        */
       smp_call_function_many(cpumask, flush_tlb_func, &info, 1);
   }
   ```

3. **延迟TLB Flush优化**
   ```c
   /*
    * TLB批处理结构体，用于优化TLB刷新操作
    * 通过将多个TLB刷新请求合并处理，减少IPI中断次数
    * 这对于大量小粒度的TLB刷新操作特别有效
    */
   struct tlb_batch {
       struct mm_struct *mm;    /* 地址空间描述符 */
       unsigned long start;     /* 批处理范围的最小地址 */
       unsigned long end;       /* 批处理范围的最大地址 */
       unsigned int nr;        /* 当前批次中累积的页面数量 */
       bool flush_needed;      /* 标记是否需要执行实际的刷新操作 */
   };

   // 批量处理TLB flush请求
   static inline void tlb_batch_add_page(struct tlb_batch *batch,
                                      unsigned long addr)
   {
       batch->start = min(batch->start, addr);
       batch->end = max(batch->end, addr + PAGE_SIZE);
       if (++batch->nr > TLB_BATCH_MAX_PAGES)
           tlb_batch_flush(batch);
   }
   ```

#### 3.3.4 性能优化策略

1. **批量处理**
   - 收集多个TLB失效请求一次性处理
   - 减少IPI中断次数
   - 优化多页面失效场景

2. **范围失效**
   - 使用地址范围而不是单个页面
   - 利用ARM64的TLBI VAE1指令
   - 减少TLB操作次数

3. **ASID优化**
   - 合理分配和复用ASID
   - 避免不必要的TLB刷新
   - 利用ARM64的16位ASID支持

4. **性能监控**
   ```c
   // 使用PMU监控TLB相关事件
   perf_event_attr attr = {
       .type = PERF_TYPE_RAW,
       .config = ARM64_PMU_TLB_FLUSH,
       .disabled = 1,
       .exclude_kernel = 1,
   };
   ```

#### 3.2.2 VMA的类型和标志

VMA可以分为几种主要类型：

1. **匿名映射**：不与文件关联的内存区域，如堆
   ```c
   vma = vm_area_alloc(mm);
   vma->vm_start = addr;
   vma->vm_end = addr + len;
   vma->vm_flags = VM_READ | VM_WRITE | VM_EXEC;
   ```

2. **文件映射**：与文件关联的内存区域，如共享库
   ```c
   vma = vm_area_alloc(mm);
   vma->vm_file = file;
   vma->vm_pgoff = pgoff;
   ```

### 3.3 TLB优化策略详解

#### 3.3.1 批量TLB刷新机制

批量TLB刷新是一种重要的性能优化技术，其核心思想是收集多个TLB失效请求，然后一次性处理。

1. **实现原理**
   ```c
   struct mmu_gather {
       struct mm_struct *mm;
       unsigned int fullmm;
       unsigned int need_flush_all;
       struct mmu_table_batch *batch;
       unsigned int local_count;
       unsigned int pages_count;
   };
   ```

2. **性能优势**
   - 减少TLB刷新次数，每次刷新有开销
   - 提高缓存利用率
   - 减少处理器流水线停顿
   - 实测可减少50%以上TLB flush开销

3. **典型应用场景**
   - 大量页表项同时修改
   - 进程地址空间批量修改
   - 内存热迁移

#### 3.3.2 延迟TLB刷新

延迟TLB刷新通过推迟TLB刷新操作到必要时刻，减少不必要的刷新。

1. **实现机制**
   ```c
   void tlb_flush_mmu(struct mmu_gather *tlb)
   {
       if (!tlb->need_flush_all) {
           // 只在必要时执行刷新
           tlb_flush_mmu_tlbonly(tlb);
       }
       tlb_batch_pages_flush(tlb);
   }
   ```

2. **延迟策略**
   - 使用generation计数跟踪延迟状态
   - 维护TLB条目的生命周期
   - 智能判断刷新时机

3. **适用场景**
   - 频繁修改但访问不频繁的页表项
   - 短时间内多次修改同一区域
   - 系统负载较重时

#### 3.3.3 范围TLB刷新

范围TLB刷新通过只刷新特定地址范围的TLB条目来优化性能。

1. **实现方式**
   ```c
   void flush_tlb_mm_range(struct mm_struct *mm,
                          unsigned long start,
                          unsigned long end,
                          unsigned long vmflag)
   {
       // 根据范围大小选择刷新策略
       if ((end - start) <= tlb_fast_mode_size) {
           // 使用范围刷新
           flush_tlb_range(mm, start, end);
       } else {
           // 完全刷新
           flush_tlb_mm(mm);
       }
   }
   ```

2. **ARM64特定实现**
   - 使用TLBI VALE/VALE1指令
   - 支持细粒度的TLB条目失效
   - 硬件加速范围检查

3. **优化效果**
   - 避免全局TLB flush的开销
   - 保留其他地址范围的TLB映射
   - 特别适合大内存系统

#### 3.3.4 ASID优化

ASID（地址空间标识符）优化是避免进程切换时全局TLB刷新的关键技术。

1. **ASID分配策略**
   ```c
   void switch_mm_irqs_off(struct mm_struct *prev,
                          struct mm_struct *next,
                          struct task_struct *tsk)
   {
       if (prev != next) {
           // 使用ASID区分不同进程
           unsigned long flags;
           unsigned int cpu = smp_processor_id();
           
           if (next->context.asid == 0) {
               // 分配新的ASID
               flags = next->context.asid = 
                       get_new_asid(next, cpu);
           }
           // 切换ASID而不是刷新TLB
           cpu_switch_mm(next, cpu);
       }
   }
   ```

2. **ASID管理机制**
   - 每个进程分配唯一ASID
   - ASID复用策略
   - 延迟分配机制

3. **ARM64架构支持**
   - 通过TTBR0_EL1/TTBR1_EL1的ASID字段指定
   - 硬件级别的地址空间隔离
   - 16位ASID支持（最多65536个ASID）

4. **性能提升**
   - 显著减少进程切换开销
   - 提高TLB利用率
   - 改善上下文切换性能

#### 3.3.5 TLB优化的最佳实践

1. **系统级优化**
   - 合理使用大页（Huge Page）
   - 优化内存分配策略
   - 调整进程调度策略

2. **应用级优化**
   - 优化内存访问模式
   - 合理使用内存映射
   - 控制工作集大小

3. **监控与调优**
   - 使用perf工具监控TLB miss
   - 分析TLB相关性能计数器
   - 根据实际负载调整参数

4. **注意事项**
   - 平衡TLB优化与其他系统开销
   - 考虑NUMA架构的影响
   - 关注安全性影响

#### 3.2.1 VMA基本概念

虚拟内存区域（Virtual Memory Area, VMA）是进程地址空间的基本管理单元，表示一段连续的虚拟地址范围。

VMA的核心数据结构：

```c
struct vm_area_struct {
    /* VMA的起始和结束地址 */
    unsigned long vm_start;
    unsigned long vm_end;

    /* 指向所属mm_struct的指针 */
    struct mm_struct *vm_mm;

    /* 访问权限标志 */
    pgprot_t vm_page_prot;
    unsigned long vm_flags;

    /* 红黑树节点 */
    struct rb_node vm_rb;

    /* 链表节点 */
    struct list_head vm_next;

    /* 文件映射相关 */
    struct file *vm_file;
    unsigned long vm_pgoff;

    /* VMA操作函数 */
    const struct vm_operations_struct *vm_ops;

    /* 私有数据 */
    void *vm_private_data;
};
```

### 3.3 TLB管理与优化

#### 3.3.1 TLB基本概念

TLB（Translation Lookaside Buffer）是一个硬件缓存，用于加速虚拟地址到物理地址的转换过程。它缓存了最近使用的页表项，减少了访问内存中页表的次数。

#### 3.3.2 TLB优化策略

1. **TLB预取技术**
   - 基于空间局部性的预取：预取相邻页表项
   - 基于访问模式的预测：分析内存访问模式预取TLB
   - 软件引导预取：编译器插入TLB预取指令
   - 硬件实现：ARM v8.2 TTLBOS (Translation Table Level Based Prefetch) 机制
   - 性能影响：可减少15-25%的TLB miss率

2. **TLB能耗优化**
   - 选择性TLB失效：只刷新必要的TLB条目
   - 自适应TLB大小：根据工作负载动态调整TLB大小
   - 智能TLB电源管理：低功耗状态下部分TLB逻辑断电
   - ASID分配优化：减少全局TLB刷新频率

3. **TLB监控与分析**
   ```bash
   # 监控TLB miss事件
   perf stat -e dTLB-load-misses,dTLB-store-misses ./your_application

   # 查看当前TLB状态
   cat /proc/pagetypeinfo
   cat /proc/vmstat | grep tlb
   ```

### 3.4 内存屏障与一致性

#### 3.4.1 内存屏障基础

内存屏障是确保多处理器系统中内存操作顺序的低级同步原语。它们解决了以下问题：

- 编译器重排序：编译器优化可能改变指令顺序
- CPU重排序：处理器可能乱序执行内存访问
- 缓存一致性：多核系统中的缓存同步延迟

#### 3.4.2 Linux内存屏障API

Linux内核提供了一组内存屏障原语：

```c
/* 通用内存屏障 */
void mb(void);      /* 完全内存屏障 */
void rmb(void);     /* 读内存屏障 */
void wmb(void);     /* 写内存屏障 */

/* 带获取/释放语义的屏障 */
void smp_mb(void);  /* SMP完全内存屏障 */
void smp_rmb(void); /* SMP读内存屏障 */
void smp_wmb(void); /* SMP写内存屏障 */

/* 编译器屏障 */
void barrier(void); /* 防止编译器重排序 */
```

#### 3.4.3 内存屏障使用场景

内存屏障在以下场景中至关重要：

1. **锁实现**：
   ```c
   /* 自旋锁实现示例 */
   void spin_lock(spinlock_t *lock)
   {
       /* 尝试获取锁 */
       while (test_and_set_bit_lock(0, &lock->slock)) {
           /* 等待 */
       }

       /* 获取锁后的内存屏障 */
       smp_mb__after_spinlock();
   }
   ```

2. **无锁数据结构**：
   ```c
   /* 无锁队列示例 */
   void queue_push(struct queue *q, struct node *n)
   {
       n->next = NULL;
       struct node *prev = xchg(&q->tail, n);

       /* 确保节点完全初始化后再链接 */
       smp_wmb();

       prev->next = n;
   }
   ```

1. **批量TLB Flush**
   - 收集多个TLB失效请求，一次性处理
   - 减少TLB刷新频率
   - 可显著提升性能

2. **延迟TLB Flush**
   - 推迟TLB flush到必要时刻
   - 使用generation计数跟踪延迟状态
   - 适用于频繁修改但访问不频繁的页表项

3. **范围TLB Flush**
   - 只刷新特定地址范围的TLB条目
   - 避免全局TLB flush带来的性能损失
   - 特别适用于大内存系统

#### 3.3.3 ARM64架构下的TLB Shootdown机制

TLB Shootdown是多处理器系统中维护TLB一致性的关键机制。当一个CPU修改了页表项时，需要通知其他CPU使其TLB中的相应表项失效。在ARM64架构中，这个过程得到了硬件的特殊支持。

1. **触发场景**
   - 页表项修改
   - 进程地址空间修改
   - 内存映射变更
   - 进程地址空间释放
   - 大页拆分或合并
   - mprotect系统调用改变内存权限

2. **基本流程**
   ```
   CPU 0                     CPU 1,2,3...
   修改页表项   ----IPI----> 接收TLB失效请求
      |                          |
   等待确认    <---ACK-----  执行TLB失效
      |                          |
   继续执行    <------------  继续执行
   ```

#### 3.3.2 ARM64硬件支持

1. **TLBI指令族**
   ```c
   // ARM64提供了一系列TLB失效指令
   TLBI ALLE1                 // 使所有EL1级别的TLB表项失效
   TLBI VAE1, Xt             // 使指定虚拟地址的EL1表项失效
   TLBI ASIDE1, Xt           // 使指定ASID的表项失效
   ```

2. **硬件广播机制**
   - ARM64处理器支持硬件级别的TLB广播
   - 通过系统总线自动传播TLB失效请求
   - 支持细粒度的地址范围失效

#### 3.3.3 Linux内核实现

1. **MMU Notifier机制**
   ```c
   /*
    * MMU Notifier机制的核心函数，用于通知所有注册的监听器页表即将发生变化
    * @mm: 进程的内存描述符，包含了进程的地址空间信息
    * @start: 需要失效的虚拟地址范围的起始地址
    * @end: 需要失效的虚拟地址范围的结束地址
    *
    * 该函数主要用于以下场景：
    * 1. 虚拟化环境中，当Guest OS修改页表时，通知Hypervisor
    * 2. 设备驱动程序需要维护自己的地址转换表时
    * 3. 其他需要跟踪页表变化的子系统
    */
   static void __mmu_notifier_invalidate_range_start(struct mm_struct *mm,
           unsigned long start, unsigned long end)
   {
       struct mmu_notifier *mn;

       /* 使用RCU机制遍历所有注册的notifier
        * 这里使用RCU是为了在遍历列表时允许并发的注册新的notifier
        */
       hlist_for_each_entry_rcu(mn, &mm->mmu_notifier_mm->list, hlist)
           if (mn->ops->invalidate_range_start)
               /* 调用每个notifier注册的回调函数 */
               mn->ops->invalidate_range_start(mn, mm, start, end);
   }
   ```

2. **IPI处理流程**
   ```c
   /*
    * 在多处理器系统中刷新其他CPU的TLB
    * @cpumask: 需要执行TLB刷新的CPU掩码
    * @mm: 需要刷新TLB的进程内存描述符
    * @start: 需要刷新的虚拟地址范围起始地址
    * @end: 需要刷新的虚拟地址范围结束地址
    *
    * 该函数通过处理器间中断(IPI)实现跨CPU的TLB同步
    * 这是保持多CPU系统TLB一致性的关键机制
    */
   static void native_flush_tlb_others(const struct cpumask *cpumask,
           struct mm_struct *mm, unsigned long start, unsigned long end)
   {
       /* 准备TLB刷新信息 */
       struct tlb_flush_info info;
       info.mm = mm;      /* 要刷新的地址空间 */
       info.start = start;  /* 刷新范围的起始地址 */
       info.end = end;      /* 刷新范围的结束地址 */
       
       /* 通过IPI触发其他CPU执行TLB刷新
        * 最后的参数1表示等待所有CPU完成刷新操作
        */
       smp_call_function_many(cpumask, flush_tlb_func, &info, 1);
   }
   ```

3. **延迟TLB Flush优化**
   ```c
   /*
    * TLB批处理结构体，用于优化TLB刷新操作
    * 通过将多个TLB刷新请求合并处理，减少IPI中断次数
    * 这对于大量小粒度的TLB刷新操作特别有效
    */
   struct tlb_batch {
       struct mm_struct *mm;    /* 地址空间描述符 */
       unsigned long start;     /* 批处理范围的最小地址 */
       unsigned long end;       /* 批处理范围的最大地址 */
       unsigned int nr;        /* 当前批次中累积的页面数量 */
       bool flush_needed;      /* 标记是否需要执行实际的刷新操作 */
   };

   // 批量处理TLB flush请求
   static inline void tlb_batch_add_page(struct tlb_batch *batch,
                                      unsigned long addr)
   {
       batch->start = min(batch->start, addr);
       batch->end = max(batch->end, addr + PAGE_SIZE);
       if (++batch->nr > TLB_BATCH_MAX_PAGES)
           tlb_batch_flush(batch);
   }
   ```

#### 3.3.4 性能优化策略

1. **批量处理**
   - 收集多个TLB失效请求一次性处理
   - 减少IPI中断次数
   - 优化多页面失效场景

2. **范围失效**
   - 使用地址范围而不是单个页面
   - 利用ARM64的TLBI VAE1指令
   - 减少TLB操作次数

3. **ASID优化**
   - 合理分配和复用ASID
   - 避免不必要的TLB刷新
   - 利用ARM64的16位ASID支持

4. **性能监控**
   ```c
   // 使用PMU监控TLB相关事件
   perf_event_attr attr = {
       .type = PERF_TYPE_RAW,
       .config = ARM64_PMU_TLB_FLUSH,
       .disabled = 1,
       .exclude_kernel = 1,
   };
   ```

#### 3.2.2 VMA的类型和标志

VMA可以分为几种主要类型：

1. **匿名映射**：不与文件关联的内存区域，如堆
   ```c
   vma = vm_area_alloc(mm);
   vma->vm_start = addr;
   vma->vm_end = addr + len;
   vma->vm_flags = VM_READ | VM_WRITE | VM_EXEC;
   vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
   vma->vm_ops = NULL;
   vma->vm_file = NULL;
   ```

2. **文件映射**：与文件关联的内存区域，如代码段、mmap映射
   ```c
   vma = vm_area_alloc(mm);
   vma->vm_start = addr;
   vma->vm_end = addr + len;
   vma->vm_flags = VM_READ | VM_SHARED;
   vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
   vma->vm_ops = &file_vm_ops;
   vma->vm_file = get_file(file);
   vma->vm_pgoff = pgoff;
   ```

常用的VMA标志：

| 标志 | 描述 |
|------|------|
| VM_READ | 可读 |
| VM_WRITE | 可写 |
| VM_EXEC | 可执行 |
| VM_SHARED | 共享映射 |
| VM_MAYREAD | 可设置为可读 |
| VM_MAYWRITE | 可设置为可写 |
| VM_MAYEXEC | 可设置为可执行 |
| VM_GROWSDOWN | 向下增长（如栈） |
| VM_GROWSUP | 向上增长（如堆） |
| VM_DONTEXPAND | 禁止扩展 |
| VM_DONTCOPY | fork时不复制 |
| VM_LOCKED | 锁定在内存中，不交换 |

#### 3.2.3 VMA的管理

Linux使用两种数据结构来管理VMA：

1. **链表**：按地址顺序链接所有VMA
   ```c
   struct list_head vma_list;
   ```

2. **红黑树**：用于快速查找特定地址所在的VMA
   ```c
   struct rb_root mm_rb;
   ```

查找VMA的主要函数：

```c
/* 在红黑树中查找包含addr的VMA */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    struct rb_node *rb_node;
    struct vm_area_struct *vma;

    /* 首先检查缓存的VMA */
    vma = mm->cached_vma;
    if (vma && vma->vm_end > addr && vma->vm_start <= addr)
        return vma;

    /* 在红黑树中查找 */
    rb_node = mm->mm_rb.rb_node;
    while (rb_node) {
        struct vm_area_struct *tmp;
        tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);

        if (tmp->vm_end > addr) {
            if (tmp->vm_start <= addr) {
                mm->cached_vma = tmp;
                return tmp;
            }
            rb_node = rb_node->rb_left;
        } else
            rb_node = rb_node->rb_right;
    }
    return NULL;
}
```

#### 3.2.4 VMA操作

VMA支持多种操作，通过`vm_operations_struct`结构定义：

```c
struct vm_operations_struct {
    /* 打开VMA时调用 */
    void (*open)(struct vm_area_struct *);

    /* 关闭VMA时调用 */
    void (*close)(struct vm_area_struct *);

    /* 处理缺页异常 */
    vm_fault_t (*fault)(struct vm_fault *);

    /* 处理大页缺页异常 */
    vm_fault_t (*huge_fault)(struct vm_fault *, enum page_entry_size);

    /* 映射页面到用户空间 */
    int (*map_pages)(struct vm_fault *, pgoff_t, pgoff_t);

    /* 访问页面 */
    int (*access)(struct vm_area_struct *, unsigned long, void *, int, int);

    /* 其他操作... */
};
```

这些操作函数使得不同类型的内存区域可以有不同的行为，例如文件映射和设备映射。

### 3.3 进程地址空间

#### 3.3.1 mm_struct结构

每个进程的地址空间由`mm_struct`结构表示：

```c
struct mm_struct {
    /* VMA管理 */
    struct vm_area_struct *mmap;        /* 链表头 */
    struct rb_root mm_rb;               /* 红黑树根 */
    struct vm_area_struct *cached_vma;  /* 缓存的VMA */

    /* 统计信息 */
    unsigned long total_vm;    /* 总页数 */
    unsigned long locked_vm;   /* 锁定页数 */
    unsigned long pinned_vm;   /* 固定页数 */
    unsigned long data_vm;     /* 数据段页数 */
    unsigned long exec_vm;     /* 可执行段页数 */
    unsigned long stack_vm;    /* 栈页数 */

    /* 内存段边界 */
    unsigned long start_code, end_code;
    unsigned long start_data, end_data;
    unsigned long start_brk, brk;
    unsigned long start_stack;
    unsigned long arg_start, arg_end;
    unsigned long env_start, env_end;

    /* 页表 */
    pgd_t *pgd;

    /* 引用计数 */
    atomic_t mm_users;    /* 用户数 */
    atomic_t mm_count;    /* 引用计数 */

    /* 锁 */
    spinlock_t page_table_lock;
    struct rw_semaphore mmap_sem;

    /* 其他字段... */
};
```

#### 3.3.2 进程地址空间的创建和销毁

进程地址空间的生命周期：

1. **创建**：在`fork`系统调用中通过`copy_mm`函数创建
   ```c
   static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
   {
       struct mm_struct *mm, *oldmm;
       int retval;

       oldmm = current->mm;

       /* 如果是线程，共享mm */
       if (clone_flags & CLONE_VM) {
           mmget(oldmm);
           tsk->mm = oldmm;
           return 0;
       }

       /* 否则创建新的mm */
       mm = dup_mm(tsk);
       if (!mm)
           return -ENOMEM;

       tsk->mm = mm;
       return 0;
   }
   ```

2. **复制**：通过`dup_mm`函数复制父进程的地址空间
   ```c
   static struct mm_struct *dup_mm(struct task_struct *tsk)
   {
       struct mm_struct *mm, *oldmm;

       oldmm = current->mm;

       /* 分配新的mm_struct */
       mm = allocate_mm();
       if (!mm)
           return NULL;

       /* 复制mm_struct内容 */
       memcpy(mm, oldmm, sizeof(*mm));

       /* 初始化各种计数器和锁 */
       mm->mm_users = 1;
       atomic_set(&mm->mm_count, 1);
       init_rwsem(&mm->mmap_sem);
       spin_lock_init(&mm->page_table_lock);

       /* 复制页表 */
       if (init_new_context(tsk, mm) < 0)
           goto fail_nocontext;

       /* 复制VMA */
       retval = dup_mmap(mm, oldmm);
       if (retval)
           goto fail_dup_mmap;

       return mm;
   }
   ```

3. **销毁**：在进程退出时通过`exit_mm`函数释放
   ```c
   void exit_mm(struct task_struct *tsk)
   {
       struct mm_struct *mm = tsk->mm;

       if (!mm)
           return;

       /* 清除mm指针 */
       tsk->mm = NULL;

       /* 如果是线程组最后一个线程 */
       if (atomic_read(&mm->mm_users) == 1) {
           /* 处理挂起的IPI */
           flush_tlb_mm(mm);
       }

       mmput(mm);
   }
   ```

#### 3.3.3 内存映射

内存映射是将文件或设备映射到进程地址空间的机制，通过`mmap`系统调用实现：

```c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, off)
{
    /* 参数验证 */
    if (offset_in_page(off))
        return -EINVAL;

    /* 执行映射 */
    return do_mmap(addr, len, prot, flags, fd, off);
}
```

`do_mmap`函数的主要步骤：

1. 查找合适的地址空间区域
2. 创建新的VMA
3. 设置VMA属性和操作函数
4. 插入到mm_struct的链表和红黑树中

```c
unsigned long do_mmap(unsigned long addr, unsigned long len,
                     unsigned long prot, unsigned long flags,
                     unsigned long fd, unsigned long offset)
{
    struct file *file = NULL;
    unsigned long retval;

    /* 获取文件对象 */
    if (!(flags & MAP_ANONYMOUS)) {
        file = fget(fd);
        if (!file)
            return -EBADF;
    }

    /* 查找地址空间 */
    addr = get_unmapped_area(file, addr, len, offset, flags);
    if (IS_ERR_VALUE(addr)) {
        retval = addr;
        goto out;
    }

    /* 创建VMA */
    retval = mmap_region(file, addr, len, flags, prot, offset);

out:
    if (file)
        fput(file);
    return retval;
}
```

### 3.4 内存描述符和页表管理

#### 3.4.1 内存描述符

内存描述符（`mm_struct`）是进程地址空间的核心数据结构，它包含了进程的所有内存管理信息：

```c
struct mm_struct {
    /* 页表根目录 */
    pgd_t *pgd;

    /* 地址空间限制 */
    unsigned long task_size;
    unsigned long highest_vm_end;

    /* 内存使用统计 */
    unsigned long hiwater_rss;  /* 历史最高RSS */
    unsigned long hiwater_vm;   /* 历史最高VM */

    /* 内存策略 */
    struct mempolicy *policy;

    /* 缺页统计 */
    unsigned long faultstamp;
    unsigned long min_flt, maj_flt;

    /* 其他字段... */
};
```

#### 3.4.2 页表管理

Linux使用多级页表结构管理虚拟地址到物理地址的映射：

1. **页表分配**：在进程创建时分配页表
   ```c
   static int mm_init(struct mm_struct *mm, struct task_struct *p)
   {
       /* 分配PGD */
       mm->pgd = pgd_alloc(mm);
       if (!mm->pgd)
           return -ENOMEM;

       /* 初始化其他字段 */
       return 0;
   }
   ```

2. **页表项设置**：在缺页处理时设置页表项
   ```c
   static int do_anonymous_page(struct vm_fault *vmf)
   {
       struct vm_area_struct *vma = vmf->vma;
       struct page *page;
       pte_t entry;

       /* 分配新页面 */
       page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
       if (!page)
           return VM_FAULT_OOM;

       /* 设置页表项 */
       entry = mk_pte(page, vma->vm_page_prot);
       if (vma->vm_flags & VM_WRITE)
           entry = pte_mkwrite(pte_mkdirty(entry));

       /* 更新页表 */
       set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);

       /* 更新统计信息 */
       update_mmu_cache(vma, vmf->address, vmf->pte);

       return VM_FAULT_MINOR;
   }
   ```

3. **页表遍历**：查找特定虚拟地址的页表项
   ```c
   pte_t *find_pte(struct mm_struct *mm, unsigned long address)
   {
       pgd_t *pgd;
       p4d_t *p4d;
       pud_t *pud;
       pmd_t *pmd;

       /* 获取PGD项 */
       pgd = pgd_offset(mm, address);
       if (pgd_none(*pgd) || pgd_bad(*pgd))
           return NULL;

       /* 获取P4D项 */
       p4d = p4d_offset(pgd, address);
       if (p4d_none(*p4d) || p4d_bad(*p4d))
           return NULL;

       /* 获取PUD项 */
       pud = pud_offset(p4d, address);
       if (pud_none(*pud) || pud_bad(*pud))
           return NULL;

       /* 获取PMD项 */
       pmd = pmd_offset(pud, address);
       if (pmd_none(*pmd) || pmd_bad(*pmd))
           return NULL;

       /* 获取PTE项 */
       return pte_offset_map(pmd, address);
   }
   ```

#### 3.4.3 缺页处理

缺页处理是按需分配物理内存的核心机制：

```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;

    /* 检查PTE是否存在 */
    if (!vmf->pte) {
        /* 处理不存在的PTE */
        if (vma->vm_ops && vma->vm_ops->fault)
            return vma->vm_ops->fault(vmf);

        /* 匿名页面 */
        if (vma->vm_flags & VM_SHARED)
            return do_shared_fault(vmf);
        else
            return do_anonymous_page(vmf);
    }

    /* PTE存在但页面不在内存中 */
    entry = *vmf->pte;
    if (!pte_present(entry)) {
        /* 页面在交换空间 */
        if (pte_none(entry))
            return do_no_page(vmf);
        else if (pte_file(entry))
            return do_file_page(vmf);
        else
            return do_swap_page(vmf);
    }

    /* 写保护页面 */
    if (vmf->flags & FAULT_FLAG_WRITE && !pte_write(entry))
        return do_wp_page(vmf);

    /* 其他情况 */
    return VM_FAULT_NOPAGE;
}
```

缺页处理的主要类型：

1. **匿名页面缺页**：分配新的物理页面
2. **文件映射缺页**：从文件读取数据到页面
3. **写时复制缺页**：复制共享页面
4. **交换页面缺页**：从交换空间读回页面

### 3.5 NUMA内存策略

#### 3.5.1 NUMA基本概念

NUMA（Non-Uniform Memory Access，非统一内存访问）是一种内存架构，其中内存访问时间取决于内存相对于处理器的位置。

NUMA系统的特点：
- 系统划分为多个节点（Node）
- 每个节点包含处理器和本地内存
- 访问本地内存比访问远程内存快

#### 3.5.2 NUMA内存策略

Linux支持多种NUMA内存策略：

1. **默认策略**：优先在本地节点分配
   ```c
   /* 设置默认策略 */
   set_mempolicy(MPOL_DEFAULT, NULL, 0);
   ```

2. **绑定策略**：仅在指定节点分配
   ```c
   /* 设置绑定策略 */
   unsigned long nodemask = (1 << node_id);
   set_mempolicy(MPOL_BIND, &nodemask, NUMA_NUM_NODES);
   ```

3. **优先策略**：优先在指定节点分配，失败时尝试其他节点
   ```c
   /* 设置优先策略 */
   unsigned long nodemask = (1 << preferred_node);
   set_mempolicy(MPOL_PREFERRED, &nodemask, NUMA_NUM_NODES);
   ```

4. **交错策略**：在多个节点上轮流分配
   ```c
   /* 设置交错策略 */
   unsigned long nodemask = node_mask;
   set_mempolicy(MPOL_INTERLEAVE, &nodemask, NUMA_NUM_NODES);
   ```

#### 3.5.3 NUMA内存分配

NUMA感知的内存分配流程：

```c
/* 分配页面时考虑NUMA策略 */
struct page *alloc_pages_vma(gfp_t gfp, int order,
                            struct vm_area_struct *vma,
                            unsigned long addr, int node)
{
    struct mempolicy *pol;

    /* 获取内存策略 */
    pol = get_vma_policy(vma, addr);

    /* 根据策略选择节点 */
    if (pol->mode == MPOL_INTERLEAVE) {
        /* 交错策略 */
        node = interleave_nodes(pol, vma, addr);
    } else if (pol->mode == MPOL_PREFERRED) {
        /* 优先策略 */
        node = pol->v.preferred_node;
    } else if (pol->mode == MPOL_BIND) {
        /* 绑定策略 */
        node = find_first_bit(pol->v.nodes, MAX_NUMNODES);
    }

    /* 在选定节点分配 */
    return __alloc_pages_node(node, gfp, order);
}
```

#### 3.5.4 NUMA优化技术

1. **自动NUMA平衡**：内核自动迁移页面和线程
   ```bash
   # 启用自动NUMA平衡
   echo 1 > /proc/sys/kernel/numa_balancing
   ```

2. **NUMA统计信息**：监控NUMA性能
   ```bash
   # 查看NUMA统计
   numastat

   # 查看详细NUMA内存信息
   cat /proc/vmstat | grep numa
   ```

3. **NUMA内存迁移**：手动迁移页面
   ```c
   /* 将页面迁移到指定节点 */
   int migrate_pages(struct list_head *from,
                    struct list_head *to,
                    struct list_head *failed,
                    int node_to)
   ```

4. **NUMA拓扑感知调度**：调度器考虑NUMA拓扑
   ```c
   /* 调度时考虑NUMA距离 */
   int sched_numa_find_closest(struct task_struct *p,
                              int cpu,
                              int *dist)
   ```

通过这些NUMA优化技术，Linux能够有效管理大型NUMA系统的内存，提高内存访问性能。