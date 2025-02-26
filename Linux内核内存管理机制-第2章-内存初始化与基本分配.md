
## 2. 内存分配与释放

### 2.1 物理内存分配

#### 2.1.1 页面分配器

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

#### 2.1.2 内存区域（Zone）

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

#### 2.1.4 每CPU页面分配器

为了减少锁竞争和提高性能，Linux实现了每CPU页面分配器（PCP）：

```c
// 每CPU页面分配器结构
struct per_cpu_pages {
    int count;          // 当前页面数量
    int high;           // 高水位线
    int batch;          // 批量操作大小
    struct list_head list; // 页面列表
};

// 从每CPU缓存分配页面
static struct page *buffered_rmqueue(struct zone *zone, int order,
                                    gfp_t gfp_flags)
{
    struct per_cpu_pages *pcp;
    struct page *page;

    // 对于order=0的请求，尝试从PCP获取
    if (order == 0) {
        pcp = &this_cpu_ptr(zone->pageset)->pcp;
        if (pcp->count) {
            page = list_first_entry(&pcp->list, struct page, lru);
            list_del(&page->lru);
            pcp->count--;
            return page;
        }
    }

    // 如果PCP没有页面或order>0，回退到伙伴系统
    // ...
}
```

PCP通过批量操作减少了对全局锁的争用，显著提高了单页分配的性能。

### 2.2 内核内存分配

#### 2.2.1 kmalloc

`kmalloc`是内核中最常用的内存分配函数，它分配物理上连续的内存：

```c
// 分配指定大小的内存
void *kmalloc(size_t size, gfp_t flags);

// 释放内存
void kfree(const void *ptr);
```

`kmalloc`内部使用slab/slub分配器，为常见大小的对象维护缓存。它保证分配的内存在物理上连续，这对于DMA操作和硬件交互很重要。

#### 2.2.2 vmalloc

当需要大块虚拟连续但物理上可以不连续的内存时，可以使用`vmalloc`：

```c
// 分配虚拟连续内存
void *vmalloc(unsigned long size);

// 释放vmalloc分配的内存
void vfree(const void *addr);
```

`vmalloc`通过页表将不连续的物理页面映射到连续的虚拟地址空间。它适用于大型缓冲区，但访问速度比`kmalloc`慢，因为可能导致TLB缺失。

#### 2.2.3 kmem_cache

对于频繁分配和释放相同大小对象的场景，可以使用`kmem_cache`：

```c
// 创建对象缓存
struct kmem_cache *kmem_cache_create(const char *name, size_t size,
                                    size_t align, unsigned long flags,
                                    void (*ctor)(void *));

// 从缓存分配对象
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags);

// 释放对象
void kmem_cache_free(struct kmem_cache *cachep, void *objp);

// 销毁缓存
void kmem_cache_destroy(struct kmem_cache *cachep);
```

`kmem_cache`减少了内存碎片和分配开销，特别适合内核数据结构如任务描述符、索引节点等。

#### 2.2.4 mempool

内存池提供了一种在内存压力大时仍能分配内存的机制：

```c
// 创建内存池
mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
                         mempool_free_t *free_fn, void *pool_data);

// 从内存池分配对象
void *mempool_alloc(mempool_t *pool, gfp_t gfp_mask);

// 释放对象回内存池
void mempool_free(void *element, mempool_t *pool);

// 销毁内存池
void mempool_destroy(mempool_t *pool);
```

内存池预先分配一定数量的对象，确保在系统内存紧张时仍有可用资源。它常用于文件系统和块设备驱动程序中的关键操作。

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