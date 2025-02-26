
# Linux内核内存管理机制

## 2. 内存初始化与基本分配

### 2.1 ARM64架构启动阶段内存管理

ARM64架构在启动阶段的内存管理有其特殊性，主要包括以下几个关键步骤：

#### 2.1.1 早期初始化

1. **CPU初始化**
   ```c
   void __init arm64_init_mmu(void)
   {
       // 配置MAIR_EL1内存属性
       write_sysreg(MAIR_VALUE, mair_el1);
       
       // 配置TCR_EL1转换控制
       write_sysreg(TCR_VALUE, tcr_el1);
       
       // 设置TTBR0_EL1和TTBR1_EL1
       write_sysreg(TTBR0_VALUE, ttbr0_el1);
       write_sysreg(TTBR1_VALUE, ttbr1_el1);
   }
   ```

2. **设备树解析**
   ```c
   void __init early_init_dt_scan_memory(void)
   {
       // 扫描memory节点
       of_scan_flat_dt(early_init_dt_scan_memory_arch, NULL);
       
       // 解析内存布局信息
       memblock_allow_resize();
       memblock_dump_all();
   }
   ```

#### 2.1.2 页表初始化

1. **初始页表设置**
   ```c
   void __init paging_init(void)
   {
       // 设置初始页表
       map_kernel();
       map_mem();
       
       // 初始化内存管理区域
       zone_sizes_init();
   }
   ```

2. **内存属性配置**
   - 设备内存：nGnRnE、nGnRE、GRE等
   - 正常内存：缓存策略、共享属性
   - 访问权限：EL0/EL1权限控制

#### 2.1.3 EL级别内存管理

1. **EL0（用户空间）**
   - 使用TTBR0_EL1页表
   - 受EL1权限控制
   - 仅能访问用户空间地址

2. **EL1（内核空间）**
   - 使用TTBR1_EL1页表
   - 可访问内核空间地址
   - 控制用户空间访问权限

3. **EL2（虚拟化）**
   - 配置第二阶段转换
   - 管理虚拟机内存隔离
   - 控制VM内存访问权限

### 2.2 物理内存分配

#### 2.2.1 页面分配器

页面分配器是Linux内核中最基础的内存分配机制，它直接管理物理内存页面。页面分配器的核心API包括：

```c
// 分配2^order个连续页面
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);

// 分配单个页面的简化版本
struct page *alloc_page(gfp_t gfp_mask);

// 获取页面对应的虚拟地址
void *page_address(struct page *page);

// 释放页面
void __free_pages(struct page *page, unsigned int order);
void free_page(unsigned long addr);
```

页面分配器使用伙伴系统算法来管理物理内存，这使得它能够高效地分配和释放连续的物理内存块。

#### 2.2.2 内存区域（Zone）

Linux将物理内存划分为几个区域（Zone），以适应不同硬件的限制和需求：

1. **ZONE_DMA**：适用于传统DMA设备，物理地址通常低于16MB
2. **ZONE_DMA32**：适用于32位DMA设备，物理地址低于4GB
3. **ZONE_NORMAL**：常规内存区域，可以被内核直接映射
4. **ZONE_HIGHMEM**：高端内存，在32位系统上不能被内核直接映射
5. **ZONE_MOVABLE**：可移动页面区域，用于内存热插拔和内存碎片整理

每个区域都有自己的页面分配器和空闲页面列表。分配内存时，可以指定首选区域和回退策略。

#### 2.1.3 GFP标志

内存分配函数接受GFP（Get Free Page）标志，用于指定分配策略和约束：

| 标志 | 描述 |
|------|------|
| GFP_KERNEL | 标准内核分配，可能会休眠 |
| GFP_ATOMIC | 不会休眠的分配，用于中断上下文 |
| GFP_USER | 用户空间分配 |
| GFP_HIGHUSER | 来自ZONE_HIGHMEM的用户空间分配 |
| GFP_DMA | 从ZONE_DMA分配 |
| GFP_NOWAIT | 不等待内存可用，可能失败 |
| GFP_NOFS | 不执行文件系统操作 |
| __GFP_ZERO | 分配后将内存清零 |

这些标志可以组合使用，例如：`GFP_KERNEL | __GFP_ZERO`表示标准内核分配并将内存清零。

#### 2.1.4 内存分配器概述

Linux内核提供了多种内存分配器，每种都有其特定用途：

1. **每CPU页面分配器（PCP）**
   - 为每个CPU维护本地页面缓存
   - 减少锁竞争，提高单页分配性能
   - 适用于频繁的单页面分配场景

2. **kmalloc**
   - 分配物理上连续的内存
   - 基于SLAB/SLUB分配器实现
   - 适用于小块内存分配

3. **vmalloc**
   - 分配虚拟连续但物理不连续的内存
   - 适用于大块内存分配
   - 性能较kmalloc慢

4. **kmem_cache**
   - 针对固定大小对象的专用缓存
   - 减少内存碎片
   - 适用于频繁分配/释放相同大小对象

5. **mempool**
   - 预留内存池机制
   - 保证关键操作的内存分配
   - 适用于高可靠性场景

这些分配器的具体实现细节将在第4章详细介绍。

### 2.3 用户空间内存分配

#### 2.3.1 brk和mmap系统调用

用户空间内存分配最终依赖于两个系统调用：

1. **brk**：调整进程数据段的末尾
   ```c
   // 系统调用实现
   SYSCALL_DEFINE1(brk, unsigned long, brk)
   {
       unsigned long retval;
       unsigned long newbrk, oldbrk;
       struct mm_struct *mm = current->mm;
   
       // 获取当前brk值
       oldbrk = mm->brk;
       newbrk = PAGE_ALIGN(brk);
   
       // 检查是否有效
       if (brk < mm->end_data)
           goto out;
   
       // 检查是否超出限制
       if (check_data_rlimit(...))
           goto out;
   
       // 实际调整brk
       if (do_brk_flags(...) < 0)
           goto out;
   
       mm->brk = brk;
       retval = brk;
       // ...
   }
   ```

2. **mmap**：创建新的内存映射
   ```c
   // 系统调用实现
   SYSCALL_DEFINE6(mmap, ...)
   {
       // 参数处理
       // ...
   
       // 执行实际映射
       return do_mmap(file, addr, len, prot, flags, pgoff);
   }
   ```

这些系统调用是C库中`malloc`、`free`等函数的基础。

#### 2.3.2 页面错误处理

用户空间内存分配通常采用延迟分配策略，只有当程序实际访问内存时才分配物理页面：

```c
// 页面错误处理函数
static int __do_page_fault(struct mm_struct *mm, ...)
{
    // 检查错误类型
    // ...
   
    // 处理缺页异常
    if (!(flags & FAULT_FLAG_INSTRUCTION) &&
        handle_mm_fault(mm, vma, address, flags) <= 0)
        goto bad_area;
   
    // ...
}
```

这种按需分页机制提高了内存利用率，允许进程拥有比实际物理内存更大的地址空间。

#### 2.3.3 内存过量使用

Linux允许内存过量使用（overcommit），即系统可以分配超过物理内存和交换空间总和的虚拟内存：

```c
// 检查是否允许分配内存
static int __vm_enough_memory(struct mm_struct *mm, long pages, int cap_sys_admin)
{
    long free, allowed;
   
    // 根据overcommit策略计算允许的分配量
    vm_acct_memory(pages);
   
    // 如果禁用overcommit，执行严格检查
    if (sysctl_overcommit_memory == OVERCOMMIT_NEVER) {
        // 计算可用内存
        free = global_zone_page_state(NR_FREE_PAGES);
        free += get_nr_swap_pages();
        // ...
   
        // 检查是否超出限制
        allowed = (totalram_pages - hugetlb_total_pages()) *
                  sysctl_overcommit_ratio / 100;
        // ...
   
        if (free <= 0 || free < allowed)
            return -ENOMEM;
    }
   
    return 0;
}
```

过量使用策略通过`/proc/sys/vm/overcommit_memory`控制：
- 0：启发式策略
- 1：总是允许过量使用
- 2：严格限制过量使用

#### 2.1.4 ARM64内存初始化详细流程

1. **硬件初始化阶段**
   - MMU配置流程
     ```c
     void __init arm64_init_mmu_hw(void)
     {
         unsigned long mmfr0, mmfr1;
         
         // 检查硬件特性
         mmfr0 = read_cpuid(ID_AA64MMFR0_EL1);
         mmfr1 = read_cpuid(ID_AA64MMFR1_EL1);
         
         // 配置页大小支持
         if (mmfr0 & ID_AA64MMFR0_TGRAN4_SHIFT)
             pagetable_init_pmd(true);
             
         // 配置硬件表遍历
         if (mmfr1 & ID_AA64MMFR1_HPDS_SHIFT)
             cpu_set_hw_pan();
     }
     ```

2. **内存布局初始化**
   - 物理内存映射
     ```c
     void __init arm64_memblock_init(void)
     {
         const s64 linear_region_size = -(s64)PAGE_OFFSET;

         // 设置线性映射区域
         memblock_set_bottom_up(true);
         memblock_set_current_limit(linear_region_size);
         
         // 预留内核代码段
         memblock_reserve(__pa(_text), _end - _text);
         
         // 预留设备树
         memblock_reserve(__pa(initial_boot_params),
                         fdt_totalsize(initial_boot_params));
     }
     ```

3. **页表管理初始化**
   - 多级页表结构设置
     ```c
     void __init paging_init(void)
     {
         // 初始化页表池
         pgd_init_mm(&init_mm);
         
         // 创建内核页表
         kernel_map_init();
         
         // 设置固定映射
         fixmap_init();
         
         // 初始化TLB管理
         tlb_init_mm(&init_mm);
     }
     ```

4. **内存属性配置详解**
   - 内存类型定义
     ```c
     #define MT_DEVICE_nGnRnE    0   // 设备内存，无缓存
     #define MT_DEVICE_nGnRE     1   // 设备内存，允许重排序
     #define MT_DEVICE_GRE       2   // 设备内存，全局共享
     #define MT_NORMAL_NC        3   // 正常内存，无缓存
     #define MT_NORMAL           4   // 正常内存，支持缓存
     ```

   - MAIR寄存器配置
     ```c
     #define MAIR_VALUE          \
         ((0x00 << (8 * MT_DEVICE_nGnRnE))   | \
         (0x04 << (8 * MT_DEVICE_nGnRE))    | \
         (0x0c << (8 * MT_DEVICE_GRE))      | \
         (0x44 << (8 * MT_NORMAL_NC))       | \
         (0xff << (8 * MT_NORMAL)))
     ```

5. **内存管理数据结构初始化**
   ```c
   void __init mm_init(void)
   {
       // 初始化页帧数据结构
       page_ext_init();
       
       // 初始化内存域
       free_area_init();
       
       // 初始化slab分配器
       kmem_cache_init();
       
       // 初始化vmalloc
       vmalloc_init();
       
       // 初始化虚拟内存系统
       vm_init();
   }
   ```

6. **调试支持初始化**
   ```c
   void __init debug_memory_init(void)
   {
       // 初始化内存检测器
       kmemleak_init();
       
       // 初始化内存追踪
       debug_pagealloc_init();
       
       // 初始化内存统计
       memblock_analyze();
   }
   ```

### 2.4 伙伴系统与内存回收

#### 2.4.1 伙伴系统实现

1. **基本数据结构**
   ```c
   struct free_area {
       struct list_head free_list[MIGRATE_TYPES];
       unsigned long nr_free;
   };

   struct zone {
       struct free_area free_area[MAX_ORDER];
       // ...
   };
   ```

2. **页面分配核心实现**
   ```c
   static inline struct page *
   __rmqueue_smallest(struct zone *zone, unsigned int order)
   {
       unsigned int current_order;
       struct free_area *area;
       struct page *page;

       // 从请求的order开始查找
       for (current_order = order; current_order < MAX_ORDER; ++current_order) {
           area = &(zone->free_area[current_order]);
           page = list_first_entry_or_null(&area->free_list[MIGRATE_UNMOVABLE],
                                          struct page, lru);
           if (!page)
               continue;
           
           // 从链表中移除
           list_del(&page->lru);
           // 减少空闲页计数
           area->nr_free--;
           // 拆分大块内存
           expand(zone, page, order, current_order);
           return page;
       }

       return NULL;
   }
   ```

3. **内存块拆分**
   ```c
   static inline void expand(struct zone *zone,
                            struct page *page,
                            int low,
                            int high)
   {
       unsigned long size = 1 << high;
       
       while (high > low) {
           high--;
           size >>= 1;
           
           // 将拆分的另一半添加到对应order的空闲链表
           list_add(&(page[size].lru),
                    &zone->free_area[high].free_list[MIGRATE_UNMOVABLE]);
           zone->free_area[high].nr_free++;
       }
   }
   ```

#### 2.4.2 内存回收机制

1. **页面回收触发条件**
   ```c
   static bool zone_watermark_fast(struct zone *zone,
                                  unsigned int order,
                                  unsigned long mark)
   {
       long free_pages = zone_page_state(zone, NR_FREE_PAGES);
       
       // 检查是否达到水位线
       if (free_pages <= mark) {
           // 触发回收
           wakeup_kswapd(zone);
           return false;
       }
       return true;
   }
   ```

2. **kswapd内核线程**
   ```c
   static int kswapd(void *p)
   {
       struct task_struct *tsk = current;
       
       while (!kthread_should_stop()) {
           // 检查是否需要回收
           if (!try_to_sleep_on_kswapd())
               // 执行页面回收
               do_page_reclaim();
       }
       return 0;
   }
   ```

3. **页面回收策略**
   ```c
   static unsigned long shrink_page_list(struct list_head *page_list,
                                        struct pglist_data *pgdat)
   {
       LIST_HEAD(ret_pages);
       LIST_HEAD(free_pages);
       
       while (!list_empty(page_list)) {
           struct page *page = lru_to_page(page_list);
           
           // 尝试回收页面
           if (page_has_private(page)) {
               if (!try_to_release_page(page))
                   continue;
           }
           
           // 处理匿名页
           if (PageAnon(page)) {
               if (PageSwapCache(page)) {
                   try_to_free_swap(page);
               } else {
                   try_to_swap_out(page);
               }
           }
           // 处理文件页
           else {
               if (page_mapped(page))
                   try_to_unmap(page);
               if (page_has_private(page))
                   try_to_release_page(page);
           }
           
           // 将可以释放的页面添加到free_pages链表
           list_move(&page->lru, &free_pages);
       }
       
       // 实际释放页面
       release_pages(&free_pages);
       return nr_reclaimed;
   }
   ```

4. **内存压缩**
   ```c
   static unsigned long compact_zone(struct zone *zone,
                                    struct compact_control *cc)
   {
       unsigned long nr_reclaimed;
       unsigned long nr_migrated = 0;
       
       // 迁移页面以减少碎片
       while ((nr_migrated = isolate_migratepages(zone, cc))) {
           // 将页面迁移到新位置
           nr_reclaimed = migrate_pages(&cc->migratepages,
                                      alloc_migrate_target,
                                      NULL,
                                      0,
                                      cc->mode,
                                      MR_COMPACTION);
           // 更新统计信息
           cc->nr_migrated += nr_reclaimed;
       }
       
       return cc->nr_migrated;
   }
   ```