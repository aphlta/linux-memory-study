# Linux内核内存管理机制

## 5. 页面置换算法

### 5.1 页面回收基础

#### 5.1.1 回收的必要性

内存回收是操作系统内存管理的关键组成部分，它解决了以下问题：

- 物理内存是有限的资源
- 系统需要为新的分配请求腾出空间
- 提高内存利用率，避免浪费

Linux内核通过页面回收机制，在内存压力下释放不活跃的页面，以满足新的内存请求。

#### 5.1.2 可回收页面类型

Linux内核中主要有三类可回收页面：

1. **文件页面**：映射磁盘文件的页面
   - 包括可执行文件、库文件、数据文件等
   - 如果页面未修改（干净），可以直接丢弃
   - 如果页面已修改（脏），需要先写回磁盘

2. **匿名页面**：不与文件关联的页面
   - 包括堆、栈、数据段等
   - 必须写入交换空间才能回收
   - 回收成本通常高于文件页面

3. **页缓存**：用于加速文件I/O的缓存
   - 读缓存：加速文件读取
   - 写缓存：延迟写入，合并写操作
   - 通常是回收的首选目标

#### 5.1.3 回收策略

Linux采用多层次的回收策略：

1. **后台回收**：由kswapd守护进程执行
   - 在内存压力较低时主动回收
   - 保持系统有足够的空闲内存
   - 对性能影响较小

2. **直接回收**：由分配路径触发
   - 在内存分配失败时执行
   - 可能导致应用程序暂停
   - 对性能影响较大

3. **紧急回收**：在极端内存压力下执行
   - 更激进的回收策略
   - 可能触发OOM killer终止进程
   - 是最后的防线

### 5.2 LRU算法实现

#### 5.2.1 LRU基本原理

LRU（Least Recently Used，最近最少使用）是Linux内核采用的主要页面置换算法。其基本思想是：

- 最近使用过的页面可能在不久的将来再次使用
- 长时间未使用的页面可能在不久的将来也不会使用
- 优先回收长时间未使用的页面

#### 5.2.2 多级LRU链表

Linux实现了多级LRU链表来跟踪页面活跃度：

```c
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];
    struct zone_reclaim_stat reclaim_stat;
    /* 其他字段... */
};

enum lru_list {
    LRU_INACTIVE_ANON = 0, /* 不活跃匿名页面 */
    LRU_ACTIVE_ANON = 1,   /* 活跃匿名页面 */
    LRU_INACTIVE_FILE = 2, /* 不活跃文件页面 */
    LRU_ACTIVE_FILE = 3,   /* 活跃文件页面 */
    LRU_UNEVICTABLE = 4,   /* 不可回收页面 */
    NR_LRU_LISTS = 5,      /* 链表总数 */
};
```

这种设计有以下优点：
- 区分匿名页面和文件页面，可以针对不同类型采用不同策略
- 区分活跃和不活跃页面，提高回收效率
- 识别不可回收页面，避免无效的回收尝试

#### 5.2.3 页面活跃度跟踪

Linux通过以下机制跟踪页面活跃度：

1. **页表访问位**：
   ```c
   /* 检查页面是否被访问 */
   static bool page_is_accessed(struct page *page)
   {
       return pte_young(ptep_get(page_table, page_address));
   }
   ```

2. **页面老化**：
   ```c
   /* 页面老化过程 */
   static void page_age(struct page *page)
   {
       /* 清除访问位 */
       pte_t ptent = ptep_get_and_clear_young(page_table, page_address);

       /* 如果页面最近被访问，重置年龄 */
       if (pte_young(ptent))
           page->age = PAGE_MAX_AGE;
       else
           page->age--;
   }
   ```

3. **活跃/不活跃迁移**：
   ```c
   /* 将页面从不活跃链表提升到活跃链表 */
   static void activate_page(struct page *page)
   {
       /* 从不活跃链表移除 */
       list_del(&page->lru);

       /* 添加到活跃链表 */
       list_add(&page->lru, &lruvec->lists[active_lru]);

       /* 更新统计信息 */
       __count_vm_event(PGACTIVATE);
   }

   /* 将页面从活跃链表降级到不活跃链表 */
   static void deactivate_page(struct page *page)
   {
       /* 从活跃链表移除 */
       list_del(&page->lru);

       /* 添加到不活跃链表 */
       list_add(&page->lru, &lruvec->lists[inactive_lru]);
   }
   ```

#### 5.2.4 LRU改进：双链表LRU

为了解决传统LRU算法的缺点，Linux实现了双链表LRU：

```
活跃链表 (Active List)
    ^
    |  提升
    |
不活跃链表 (Inactive List)
```

工作流程：
1. 新页面首先添加到不活跃链表
2. 如果不活跃链表中的页面被访问，提升到活跃链表
3. 当活跃链表过长时，将尾部页面降级到不活跃链表
4. 回收操作主要针对不活跃链表

这种设计有以下优点：
- 防止一次性扫描大量数据导致活跃页面被错误回收
- 提供了页面"第二次机会"的机制
- 减少了"抖动"（频繁换入换出同一页面）

### 5.3 页面回收实现

#### 5.3.1 kswapd守护进程

kswapd是Linux内核中负责后台内存回收的守护进程：

```c
/* kswapd主循环 */
static int kswapd(void *p)
{
    struct task_struct *tsk = current;
    struct zone *zone = p;

    /* 设置进程名称和优先级 */
    tsk->flags |= PF_KSWAPD;
    set_freezable();

    /* 主循环 */
    for ( ; ; ) {
        /* 检查是否需要回收 */
        if (zone_watermark_ok(zone, order, mark, classzone_idx, alloc_flags))
            goto sleep;

        /* 执行回收 */
        shrink_zone(zone, NULL);

        /* 检查是否达到目标 */
        if (zone_watermark_ok(zone, order, high_wmark, classzone_idx, alloc_flags))
            goto sleep;

    sleep:
        /* 等待唤醒或超时 */
        wait_event_freezable_timeout(kswapd_wait,
                                   kswapd_should_wake(zone),
                                   HZ/10);
    }

    return 0;
}
```

kswapd的主要特点：
- 在后台持续监控内存水位
- 当内存低于特定水位时被唤醒
- 回收内存直到达到高水位
- 使用轻量级回收策略，减少对系统性能的影响

#### 5.3.2 直接回收路径

当内存分配失败且kswapd无法及时回收足够内存时，触发直接回收：

```c
/* 直接回收路径 */
static int __perform_reclaim(gfp_t gfp_mask, unsigned int order)
{
    struct reclaim_state reclaim_state;
    int progress;

    /* 准备回收状态 */
    current->reclaim_state = &reclaim_state;

    /* 执行回收 */
    progress = try_to_free_pages(zonelist, order, gfp_mask);

    /* 清理回收状态 */
    current->reclaim_state = NULL;

    return progress;
}
```

直接回收的特点：
- 在分配路径上同步执行
- 会导致分配进程暂停
- 使用更激进的回收策略
- 对系统性能影响较大

#### 5.3.3 页面回收核心函数

页面回收的核心函数是`shrink_zone`和`shrink_lruvec`：

```c
/* 区域回收函数 */
static unsigned long shrink_zone(struct zone *zone, struct scan_control *sc)
{
    unsigned long nr_reclaimed = 0;
    struct lruvec *lruvec;

    /* 获取区域的LRU向量 */
    lruvec = zone_lruvec(zone);

    /* 扫描并回收页面 */
    nr_reclaimed = shrink_lruvec(lruvec, sc);

    /* 更新统计信息 */
    zone_reclaim_stat_update(zone, nr_reclaimed);

    return nr_reclaimed;
}

/* LRU向量回收函数 */
static unsigned long shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
    unsigned long nr_reclaimed = 0;
    unsigned long nr_to_scan;

    /* 确定扫描页面数量 */
    nr_to_scan = calculate_nr_to_scan(lruvec, sc);

    /* 扫描不活跃文件页面 */
    nr_reclaimed += shrink_inactive_list(nr_to_scan, lruvec, sc, LRU_INACTIVE_FILE);

    /* 扫描活跃文件页面 */
    if (sc->nr_scanned < sc->nr_to_scan)
        nr_reclaimed += shrink_active_list(sc->nr_to_scan - sc->nr_scanned, lruvec, sc, LRU_ACTIVE_FILE);

    /* 如果文件页面回收不足，扫描匿名页面 */
    if (sc->nr_scanned < sc->nr_to_scan && sc->priority < DEF_PRIORITY / 2) {
        /* 扫描不活跃匿名页面 */
        nr_reclaimed += shrink_inactive_list(sc->nr_to_scan - sc->nr_scanned, lruvec, sc, LRU_INACTIVE_ANON);

        /* 扫描活跃匿名页面 */
        if (sc->nr_scanned < sc->nr_to_scan)
            nr_reclaimed += shrink_active_list(sc->nr_to_scan - sc->nr_scanned, lruvec, sc, LRU_ACTIVE_ANON);
    }

    return nr_reclaimed;
}
```

回收过程的主要步骤：
1. 首先尝试回收不活跃文件页面（成本最低）
2. 然后尝试回收活跃文件页面
3. 如果仍需要更多内存，回收不活跃匿名页面
4. 最后尝试回收活跃匿名页面（成本最高）

#### 5.3.4 页面回收优先级

Linux使用优先级机制控制回收的激进程度：

```c
/* 回收控制结构 */
struct scan_control {
    /* 回收优先级 */
    int priority;

    /* 扫描和回收计数 */
    unsigned long nr_scanned;
    unsigned long nr_reclaimed;

    /* 回收标志 */
    unsigned int may_writepage:1;
    unsigned int may_unmap:1;
    unsigned int may_swap:1;

    /* 其他字段... */
};
```

优先级值从0到DEF_PRIORITY（通常为12）：
- 较低的值表示更高的回收优先级（更激进）
- 较高的值表示更低的回收优先级（更保守）

随着回收的进行，优先级会逐渐提高（值减小），回收策略变得更加激进：
```c
/* 调整回收优先级 */
static void update_reclaim_priority(struct scan_control *sc, int priority_adj)
{
    sc->priority -= priority_adj;
    if (sc->priority < 0)
        sc->priority = 0;
}
```

### 5.4 页面回收策略

#### 5.4.1 文件页面回收

文件页面回收是最简单和最高效的回收类型：

```c
/* 回收文件页面 */
static int pageout(struct page *page, struct address_space *mapping)
{
    /* 如果页面干净，直接释放 */
    if (!PageDirty(page)) {
        __delete_from_page_cache(page);
        return 1;
    }

    /* 如果页面脏，尝试写回 */
    if (mapping->a_ops->writepage) {
        /* 写回页面 */
        write_one_page(page, mapping);
        return 1;
    }

    return 0;
}
```

文件页面回收的特点：
- 干净页面可以直接丢弃
- 脏页面需要先写回磁盘
- 回收成本低于匿名页面
- 可以通过预读和缓存快速恢复

#### 5.4.2 匿名页面回收

匿名页面回收需要使用交换空间：

```c
/* 回收匿名页面 */
static int swap_out_page(struct page *page)
{
    swp_entry_t entry;

    /* 分配交换空间槽位 */
    entry = get_swap_page();
    if (!entry.val)
        return 0;

    /* 将页面写入交换空间 */
    if (swap_writepage(page, entry) != 0) {
        /* 写入失败，释放交换槽位 */
        free_swap_entry(entry);
        return 0;
    }

    /* 更新页表，将页面标记为已交换 */
    rmap_walk(page, set_swap_entry, &entry);

    /* 释放页面 */
    __delete_from_lru_cache(page);

    return 1;
}
```

匿名页面回收的特点：
- 需要可用的交换空间
- 回收成本高于文件页面
- 恢复（换入）成本也较高
- 如果没有交换空间，匿名页面不可回收

#### 5.4.3 页面回收调优参数

Linux提供了多个参数来调整页面回收行为：

1. **vm.swappiness**：控制匿名页面相对于文件页面的回收倾向
   ```bash
   # 查看当前值
   cat /proc/sys/vm/swappiness

   # 设置新值（0-100）
   echo 60 > /proc/sys/vm/swappiness
   ```
   - 较高的值增加对匿名页面的回收
   - 较低的值优先回收文件页面
   - 默认值通常为60

2. **vm.vfs_cache_pressure**：控制目录项和inode缓存的回收压力
   ```bash
   # 查看当前值
   cat /proc/sys/vm/vfs_cache_pressure

   # 设置新值
   echo 100 > /proc/sys/vm/vfs_cache_pressure
   ```
   - 较高的值增加对缓存的回收压力
   - 较低的值保留更多缓存
   - 默认值通常为100

3. **vm.min_free_kbytes**：控制保留的最小空闲内存
   ```bash
   # 查看当前值
   cat /proc/sys/vm/min_free_kbytes

   # 设置新值
   echo 65536 > /proc/sys/vm/min_free_kbytes
   ```
   - 较高的值提前启动回收
   - 较低的值允许更多内存被使用
   - 默认值根据系统内存大小自动计算

4. **vm.dirty_ratio**和**vm.dirty_background_ratio**：控制脏页面写回
   ```bash
   # 查看当前值
   cat /proc/sys/vm/dirty_ratio
   cat /proc/sys/vm/dirty_background_ratio

   # 设置新值
   echo 20 > /proc/sys/vm/dirty_ratio
   echo 10 > /proc/sys/vm/dirty_background_ratio
   ```
   - dirty_ratio：触发同步写回的阈值
   - dirty_background_ratio：触发后台写回的阈值

### 5.5 交换空间管理

#### 5.5.1 交换空间基本概念

交换空间是用于存储从内存中换出的页面的磁盘区域：

- 可以是专用的交换分区
- 也可以是文件系统上的交换文件
- 支持多个交换空间，具有优先级

交换空间的作用：
- 扩展可用内存
- 支持匿名页面的回收
- 存储不活跃的内存内容

#### 5.5.2 交换空间数据结构

Linux使用以下数据结构管理交换空间：

```c
/* 交换区描述符 */
struct swap_info_struct {
    unsigned int flags;           /* 交换区标志 */
    int prio;                     /* 交换区优先级 */
    struct file *swap_file;       /* 交换文件 */
    struct block_device *bdev;    /* 块设备 */

    unsigned int max;             /* 最大交换槽位数 */
    unsigned int inuse_pages;     /* 已使用页面数 */

    unsigned long *swap_map;      /* 使用计数映射 */
    unsigned long *cluster_info;  /* 簇信息 */

    /* 其他字段... */
};

/* 交换项 */
typedef struct {
    unsigned long val;
} swp_entry_t;
```

交换项格式：
```
+------------------------+------------+
| 交换类型ID (5-8位)     | 交换偏移量  |
+------------------------+------------+
```

#### 5.5.3 交换空间操作

1. **激活交换空间**：
   ```c
   /* 激活交换空间 */
   int swap_shmem_alloc(swp_entry_t *entry)
   {
       /* 分配交换槽位 */
       return get_swap_page(entry);
   }
   ```

2. **页面换出**：
   ```c
   /* 将页面写入交换空间 */
   int swap_writepage(struct page *page, swp_entry_t entry)
   {
       struct swap_info_struct *sis = swap_info[swp_type(entry)];
       pgoff_t offset = swp_offset(entry);

       /* 设置页面PG_writeback标志 */
       set_page_writeback(page);

       /* 写入交换空间 */
       submit_bio(WRITE, bio);

       return 0;
   }
   ```

3. **页面换入**：
   ```c
   /* 从交换空间读取页面 */
   int swap_readpage(struct page *page, swp_entry_t entry)
   {
       struct swap_info_struct *sis = swap_info[swp_type(entry)];
       pgoff_t offset = swp_offset(entry);

       /* 从交换空间读取 */
       submit_bio(READ, bio);

       return 0;
   }
   ```

4. **处理缺页异常**：
   ```c
   /* 处理交换页面缺页异常 */
   static int do_swap_page(struct vm_fault *vmf)
   {
       struct page *page;
       swp_entry_t entry;

       /* 从页表项获取交换项 */
       entry = pte_to_swp_entry(vmf->orig_pte);

       /* 从交换空间读取页面 */
       page = lookup_swap_cache(entry);
       if (!page) {
           /* 页面不在交换缓存中，需要从磁盘读取 */
           page = alloc_page(GFP_HIGHUSER_MOVABLE);
           if (!page)
               return VM_FAULT_OOM;

           /* 读取页面内容 */
           swap_readpage(page, entry);
       }

       /* 将页面映射到用户空间 */
       vmf->pte = mk_pte(page, vmf->vma->vm_page_prot);

       return VM_FAULT_MAJOR;
   }
   ```

#### 5.5.4 交换缓存

交换缓存是一个重要的优化，用于减少不必要的I/O：

```c
/* 查找交换缓存 */
struct page *lookup_swap_cache(swp_entry_t entry)
{
    /* 在页面缓存中查找交换项 */
    return find_get_page(swap_address_space(entry), swp_offset(entry));
}

/* 添加到交换缓存 */
int add_to_swap_cache(struct page *page, swp_entry_t entry)
{
    /* 将页面添加到交换缓存 */
    return add_to_page_cache(page, swap_address_space(entry),
                           swp_offset(entry), GFP_ATOMIC);
}
```

交换缓存的优点：
- 避免多次读写相同的页面
- 支持写时复制优化
- 减少磁盘I/O操作

### 5.6 内存压缩（Memory Compaction）

#### 5.6.1 内存碎片问题

内存碎片是指物理内存中的空闲页面不连续，导致无法分配大块连续内存。主要有两种类型：

1. **外部碎片**：空闲内存块之间被已分配的内存块分隔
2. **内部碎片**：分配的内存块大于实际需要的大小

内存碎片的影响：
- 无法满足大页面请求
- 降低内存利用率
- 增加TLB压力

#### 5.6.2 内存压缩原理

内存压缩的基本思想是将已使用的页面移动到内存的一端，将空闲页面集中到另一端，从而创建连续的空闲内存块：

```
压缩前：
[使用][空闲][使用][使用][空闲][使用][空闲][空闲][使用]

压缩后：
[使用][使用][使用][使用][使用][空闲][空闲][空闲][空闲]
```

#### 5.6.3 内存压缩实现

Linux内核通过`compact_zone`函数实现内存压缩：

```c
/* 区域压缩函数 */
int compact_zone(struct zone *zone, struct compact_control *cc)
{
    unsigned long migrate_pfn, free_pfn;

    /* 初始化迁移和空闲扫描指针 */
    migrate_pfn = zone->zone_start_pfn;
    free_pfn = zone_end_pfn(zone);

    /* 主压缩循环 */
    for (;;) {
        /* 查找可移动页面 */
        migrate_pfn = find_next_movable_pfn(zone, migrate_pfn, free_pfn);
        if (migrate_pfn >= free_pfn)
            break;

        /* 查找空闲页面 */
        free_pfn = find_next_free_pfn(zone, free_pfn, migrate_pfn);
        if (free_pfn <= migrate_pfn)
            break;

        /* 迁移页面 */
        migrate_pages_from_pfn(zone, migrate_pfn, free_pfn);
    }

    return cc->nr_migrated;
}
```

内存压缩的主要步骤：
1. 从内存低端开始扫描，查找可移动的已使用页面
2. 从内存高端开始扫描，查找空闲页面
3. 将可移动页面迁移到空闲页面位置
4. 重复上述过程，直到无法继续迁移

#### 5.6.4 触发内存压缩的方式

内存压缩可以通过多种方式触发：

1. **直接触发**：通过sysfs接口手动触发
   ```bash
   # 触发所有区域的压缩
   echo 1 > /proc/sys/vm/compact_memory

   # 触发特定区域的压缩
   echo 1 > /sys/devices/system/node/node0/compact
   ```

2. **间接触发**：当高阶分配失败时自动触发
   ```c
   /* 分配失败时尝试压缩 */
   static struct page *__alloc_pages_direct_compact(gfp_t gfp_mask, unsigned int order,
                                                 int alloc_flags, nodemask_t *nodemask,
                                                 int *did_some_progress)
   {
       /* 执行内存压缩 */
       *did_some_progress = compact_zone_order(zone, order, gfp_mask);

       /* 压缩后再次尝试分配 */
       return get_page_from_freelist(gfp_mask, order, alloc_flags, nodemask);
   }
   ```

3. **kcompactd守护进程**：后台持续监控和压缩内存
   ```c
   /* kcompactd主循环 */
   static int kcompactd(void *p)
   {
       /* 设置进程名称和优先级 */
       set_freezable();

       /* 主循环 */
       while (!kthread_should_stop()) {
           /* 检查是否需要压缩 */
           if (should_compact_zone(zone))
               compact_zone(zone, NULL);

           /* 等待唤醒或超时 */
           wait_event_freezable(kcompactd_wait,
                              kcompactd_should_wake(zone));
       }

       return 0;
   }
   ```

### 5.7 OOM Killer

#### 5.7.1 OOM基本概念

OOM（Out Of Memory）是指系统内存耗尽，无法满足新的内存分配请求的情况。当系统遇到OOM情况时，Linux内核会激活OOM Killer来终止一个或多个进程，释放内存。

OOM Killer的目标是：
- 在内存耗尽时保持系统运行
- 选择"最佳"进程终止，最小化损失
- 快速释放足够的内存

#### 5.7.2 OOM评分机制

OOM Killer使用评分机制来选择要终止的进程：

```c
/* 计算进程的OOM评分 */
static unsigned long oom_badness(struct task_struct *p)
{
    unsigned long points;

    /* 基础分数：使用的内存页面数