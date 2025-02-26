# Linux内核内存管理机制

## 4. 内存分配算法

### 4.1 伙伴系统（Buddy System）

伙伴系统是Linux内核中管理物理页面的核心算法，它能高效地分配和释放连续的物理内存块。

#### 4.1.1 基本原理

伙伴系统将物理内存划分为2^n个页面大小的块，并按照大小分组管理：

- 每个分配阶（order）管理特定大小的内存块
- 阶数从0到MAX_ORDER-1（通常为10或11）
- 第n阶管理2^n个连续页面的块

当需要分配内存时：
1. 查找满足请求大小的最小阶
2. 如果该阶没有空闲块，向上查找更大的阶
3. 找到更大的块后，将其分割成两个"伙伴"
4. 一个伙伴用于分配，另一个放入较低阶的空闲列表

当释放内存时：
1. 检查其"伙伴"是否也是空闲的
2. 如果是，合并两个伙伴形成更大的块
3. 继续检查合并后的块的伙伴，直到无法继续合并

#### 4.1.2 数据结构

伙伴系统的核心数据结构：

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long nr_free;
};

struct zone {
    /* 每个阶的空闲区域 */
    struct free_area free_area[MAX_ORDER];

    /* 其他字段... */
};
```

每个迁移类型都有自己的空闲列表，以减少内存碎片：

```c
enum migratetype {
    MIGRATE_UNMOVABLE,   /* 不可移动页面 */
    MIGRATE_MOVABLE,     /* 可移动页面 */
    MIGRATE_RECLAIMABLE, /* 可回收页面 */
    MIGRATE_PCPTYPES,    /* 每CPU页面类型的边界 */
    MIGRATE_HIGHATOMIC,  /* 高阶原子分配 */
    MIGRATE_TYPES        /* 迁移类型总数 */
};

### 4.2 每CPU页面分配器（PCP）

#### 4.2.1 基本原理

每CPU页面分配器（Per-CPU Page Allocator）是一个优化机制，用于提高单页面分配的性能：

- 每个CPU维护一个本地页面缓存
- 避免了频繁访问全局页面列表
- 减少了自旋锁竞争
- 提高了缓存命中率

#### 4.2.2 数据结构

```c
struct per_cpu_pages {
    int count;      /* 当前页面数量 */
    int high;       /* 高水位线 */
    int batch;      /* 批量操作大小 */
    struct list_head lists[MIGRATE_PCPTYPES]; /* 页面列表 */
};

struct per_cpu_pageset {
    struct per_cpu_pages pcp;    /* 热页面列表 */
    /* 其他统计信息... */
};
```

#### 4.2.3 工作流程

1. **分配过程**：
   ```c
   static struct page *buffered_rmqueue(struct zone *zone, int order,
                                       gfp_t gfp_flags)
   {
       struct per_cpu_pages *pcp;
       struct page *page;
       
       /* 对于单页请求，优先从PCP获取 */
       if (likely(order == 0)) {
           pcp = &this_cpu_ptr(zone->pageset)->pcp;
           if (pcp->count) {
               page = list_first_entry(&pcp->lists[migratetype],
                                      struct page, lru);
               list_del(&page->lru);
               pcp->count--;
               return page;
           }
       }
       
       /* PCP为空或请求多页，回退到伙伴系统 */
       return __rmqueue(zone, order, migratetype);
   }
   ```

2. **补充机制**：
   - 当PCP中的页面数量低于阈值时
   - 从伙伴系统批量获取页面
   - 减少访问全局分配器的频率

3. **释放过程**：
   ```c
   void free_hot_cold_page(struct page *page, bool cold)
   {
       struct zone *zone = page_zone(page);
       struct per_cpu_pages *pcp;
       
       if (likely(!cold)) {
           pcp = &this_cpu_ptr(zone->pageset)->pcp;
           if (pcp->count < pcp->high) {
               list_add(&page->lru, &pcp->lists[migratetype]);
               pcp->count++;
               return;
           }
       }
       
       /* PCP已满或冷页面，直接返回伙伴系统 */
       __free_one_page(page, pfn, zone, 0, migratetype);
   }
   ```
```

#### 4.1.3 分配算法

页面分配的核心函数是`__alloc_pages_nodemask`：

```c
struct page *__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
                                   int preferred_nid, nodemask_t *nodemask)
{
    struct page *page;
    unsigned int alloc_flags = ALLOC_WMARK_LOW;
    gfp_t alloc_gfp;

    /* 准备分配标志 */
    alloc_gfp = gfp_mask & GFP_RECLAIM_MASK;

    /* 首次尝试分配 */
    page = get_page_from_freelist(alloc_gfp, order, alloc_flags,
                                 preferred_nid, nodemask);
    if (page)
        goto out;

    /* 如果允许回收，尝试回收内存后再分配 */
    if (alloc_gfp & __GFP_DIRECT_RECLAIM) {
        page = __alloc_pages_direct_reclaim(alloc_gfp, order,
                                          alloc_flags, nodemask,
                                          preferred_nid, &did_some_progress);
        if (page)
            goto out;
    }

    /* 如果允许紧急分配，尝试紧急分配 */
    if (alloc_gfp & __GFP_KSWAPD_RECLAIM) {
        page = __alloc_pages_kswapd_reclaim(alloc_gfp, order,
                                          alloc_flags, nodemask,
                                          preferred_nid, &did_some_progress);
        if (page)
            goto out;
    }

    /* 最后尝试 */
    page = __alloc_pages_high_priority(alloc_gfp, order, preferred_nid, nodemask);

out:
    return page;
}
```

实际分配过程在`get_page_from_freelist`函数中：

```c
static struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order,
                                         int alloc_flags, int preferred_nid,
                                         nodemask_t *nodemask)
{
    struct page *page;
    struct zone *zone;
    struct zoneref *z;

    /* 遍历所有可用区域 */
    for_each_zone_zonelist_nodemask(zone, z, zonelist, high_zoneidx, nodemask) {
        /* 检查区域是否有足够的空闲页面 */
        if (!zone_watermark_ok(zone, order, mark, classzone_idx, alloc_flags))
            continue;

        /* 尝试从区域分配页面 */
        page = buffered_rmqueue(zone, order, gfp_mask);
        if (page)
            return page;
    }

    return NULL;
}
```

#### 4.1.4 释放算法

页面释放的核心函数是`__free_pages`和`free_one_page`：

```c
void __free_pages(struct page *page, unsigned int order)
{
    if (put_page_testzero(page)) {
        /* 如果引用计数为零，释放页面 */
        free_the_page(page, order);
    }
}

static void free_one_page(struct zone *zone, struct page *page, unsigned long pfn,
                         unsigned int order, int migratetype)
{
    unsigned long combined_pfn;
    unsigned long uninitialized_var(buddy_pfn);
    struct page *buddy;
    unsigned int max_order = MAX_ORDER - 1;

    /* 尝试合并伙伴 */
    while (order < max_order) {
        /* 计算伙伴的页帧号 */
        buddy_pfn = __find_buddy_pfn(pfn, order);
        buddy = page + (buddy_pfn - pfn);

        /* 检查伙伴是否空闲且可合并 */
        if (!page_is_buddy(page, buddy, order))
            break;

        /* 从空闲列表中移除伙伴 */
        list_del(&buddy->lru);
        zone->free_area[order].nr_free--;

        /* 合并后继续向上合并 */
        combined_pfn = buddy_pfn & pfn;
        page = page + (combined_pfn - pfn);
        pfn = combined_pfn;
        order++;
    }

    /* 将页面添加到适当的空闲列表 */
    __free_one_page(page, pfn, zone, order, migratetype);
}
```

#### 4.1.5 伙伴系统的优缺点

**优点**：
- 高效的内存分配和释放
- 快速的伙伴查找（通过位操作）
- 良好的内存利用率（对于2的幂次大小的请求）

**缺点**：
- 内部碎片（分配大于实际需要的内存）
- 外部碎片（无法满足大块内存请求）
- 不适合小对象分配

### 4.2 SLAB分配器

SLAB分配器建立在伙伴系统之上，专门用于高效分配和释放小对象。

#### 4.2.1 基本原理

SLAB分配器的核心思想是：
- 为常见大小的对象预先分配内存池
- 重用已释放的对象，避免频繁的内存分配和释放
- 利用CPU缓存特性提高性能

SLAB分配器将对象组织为三个层次：
1. **缓存（Cache）**：特定类型对象的集合
2. **SLAB**：从伙伴系统分配的连续页面
3. **对象（Object）**：实际分配给用户的内存块

#### 4.2.2 数据结构

SLAB分配器的核心数据结构：

```c
/* 缓存描述符 */
struct kmem_cache {
    /* 每CPU数据 */
    struct array_cache __percpu *cpu_cache;

    /* 缓存特性 */
    unsigned int object_size;    /* 对象大小 */
    unsigned int size;           /* 对象大小（包括元数据） */
    unsigned int align;          /* 对齐要求 */

    /* SLAB管理 */
    unsigned int num;            /* 每个SLAB的对象数 */
    struct kmem_cache_node *node[MAX_NUMNODES];

    /* 构造函数 */
    void (*ctor)(void *obj);

    /* 缓存名称 */
    const char *name;

    /* 其他字段... */
};

/* 每CPU缓存 */
struct array_cache {
    unsigned int avail;          /* 可用对象数 */
    unsigned int limit;          /* 上限 */
    unsigned int batchcount;     /* 批量操作大小 */
    unsigned int touched;        /* 最近使用标记 */
    void *entry[];               /* 对象指针数组 */
};

/* SLAB描述符 */
struct slab {
    struct list_head list;       /* 链表节点 */
    unsigned long colouroff;     /* 颜色偏移 */
    void *s_mem;                 /* 对象区域起始地址 */
    unsigned int inuse;          /* 已分配对象数 */
    kmem_bufctl_t free;          /* 第一个空闲对象索引 */
};
```

#### 4.2.3 分配和释放过程

对象分配的基本流程：

```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
    void *ret = NULL;
    struct array_cache *ac;

    /* 尝试从每CPU缓存分配 */
    ac = cpu_cache_get(cachep);
    if (likely(ac->avail)) {
        ac->touched = 1;
        ret = ac->entry[--ac->avail];
        goto out;
    }

    /* 每CPU缓存为空，从共享缓存或新SLAB获取对象 */
    ret = cache_alloc_refill(cachep, flags);

out:
    return ret;
}
```

对象释放的基本流程：

```c
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
{
    struct array_cache *ac;

    /* 获取每CPU缓存 */
    ac = cpu_cache_get(cachep);

    /* 如果每CPU缓存已满，将一批对象释放到共享缓存 */
    if (unlikely(ac->avail == ac->limit)) {
        cache_flusharray(cachep, ac);
    }

    /* 将对象添加到每CPU缓存 */
    ac->entry[ac->avail++] = objp;
}
```

#### 4.2.4 SLAB着色

为了提高缓存利用率，SLAB分配器使用"着色"技术：

```c
/* 计算SLAB着色 */
static void cache_estimate(unsigned long gfporder, size_t size,
                         size_t align, int flags, size_t *left_over,
                         unsigned int *num)
{
    int i;
    size_t wastage = PAGE_SIZE << gfporder;
    size_t extra = 0;
    size_t base = 0;

    /* 计算基本对象布局 */
    if (flags & CFLGS_OFF_SLAB) {
        base = sizeof(struct slab);
        extra = sizeof(kmem_bufctl_t);
    } else {
        base = sizeof(struct slab) + sizeof(kmem_bufctl_t);
        extra = 0;
    }

    /* 计算每个SLAB可容纳的对象数 */
    *num = 0;
    while (wastage >= size + extra) {
        wastage -= size + extra;
        (*num)++;
    }

    /* 计算剩余空间用于着色 */
    *left_over = wastage;
}
```

着色通过在SLAB开始处添加不同大小的偏移，使不同SLAB中的对象映射到不同的缓存行，减少缓存冲突。

### 4.3 SLUB分配器

SLUB是SLAB的改进版本，旨在简化设计并提高性能。

#### 4.3.1 与SLAB的区别

SLUB相比SLAB的主要改进：
- 移除了SLAB描述符，减少元数据开销
- 简化了每CPU缓存机制
- 改进了多核扩展性
- 减少了锁竞争
- 提供了更好的调试支持

#### 4.3.2 数据结构

SLUB的核心数据结构：

```c
struct kmem_cache {
    /* SLUB特有字段 */
    struct kmem_cache_cpu __percpu *cpu_slab;
    unsigned long flags;
    unsigned long min_partial;
    int size;                    /* 对象大小 */
    int object_size;             /* 原始对象大小 */
    int offset;                  /* 空闲指针偏移 */
    int cpu_partial;             /* 每CPU部分SLAB数 */
    struct kmem_cache_order_objects oo;

    /* 节点特定数据 */
    struct kmem_cache_node *node[MAX_NUMNODES];

    /* 其他字段... */
};

/* 每CPU SLUB数据 */
struct kmem_cache_cpu {
    void **freelist;             /* 空闲对象链表 */
    struct page *page;           /* 当前SLUB页面 */
    int node;                    /* 页面所在节点 */
    unsigned int partial;        /* 部分使用的SLUB数 */
};

/* 节点特定数据 */
struct kmem_cache_node {
    spinlock_t list_lock;        /* 保护部分列表 */
    unsigned long nr_partial;    /* 部分SLUB数量 */
    struct list_head partial;    /* 部分SLUB列表 */
};
```

#### 4.3.3 分配和释放过程

SLUB的分配流程：

```c
void *kmem_cache_alloc(struct kmem_cache *s, gfp_t gfpflags)
{
    void *ret = slab_alloc(s, gfpflags, _RET_IP_);
    return ret;
}

static __always_inline void *slab_alloc(struct kmem_cache *s,
                                      gfp_t gfpflags, unsigned long addr)
{
    void *ret;
    struct kmem_cache_cpu *c;

    /* 快速路径：从当前CPU的freelist分配 */
    c = get_cpu_slab(s);
    if (likely(c->freelist)) {
        ret = c->freelist;
        c->freelist = get_freepointer(s, ret);
        return ret;
    }

    /* 慢速路径：获取新的SLUB页面 */
    ret = __slab_alloc(s, gfpflags, addr);
    return ret;
}
```

SLUB的释放流程：

```c
void kmem_cache_free(struct kmem_cache *s, void *x)
{
    slab_free(s, virt_to_page(x), x, _RET_IP_);
}

static __always_inline void slab_free(struct kmem_cache *s, struct page *page,
                                    void *x, unsigned long addr)
{
    void **object = (void *)x;
    struct kmem_cache_cpu *c;

    /* 快速路径：释放到当前CPU的freelist */
    c = get_cpu_slab(s);
    if (likely(page == c->page)) {
        set_freepointer(s, object, c->freelist);
        c->freelist = object;
        return;
    }

    /* 慢速路径：处理跨CPU释放或部分SLUB */
    __slab_free(s, page, x, addr);
}
```

#### 4.3.4 SLUB调试功能

SLUB提供了强大的调试功能：

```c
/* 启用SLUB调试 */
static int __init setup_slub_debug(char *str)
{
    slub_debug = DEBUG_DEFAULT_FLAGS;
    if (*str++ != '=' || !*str)
        return 1;

    /* 解析调试选项 */
    for (; *str; str++) {
        switch (*str) {
        case 'F': /* 空闲跟踪 */
            slub_debug |= SLAB_DEBUG_FREE;
            break;
        case 'Z': /* 红区检查 */
            slub_debug |= SLAB_RED_ZONE;
            break;
        case 'P': /* 毒化 */
            slub_debug |= SLAB_POISON;
            break;
        case 'U': /* 用户跟踪 */
            slub_debug |= SLAB_STORE_USER;
            break;
        /* 其他选项... */
        }
    }

    return 1;
}
```

调试功能包括：
- 红区检测（越界访问）
- 毒化（使用特殊模式填充内存）
- 对象跟踪（记录分配/释放位置）
- 完整性检查

### 4.4 内存池（mempool）

内存池是一种保证关键分配成功的机制，特别适用于内存压力大的情况。

#### 4.4.1 基本原理

内存池的核心思想是：
- 预先分配一定数量的对象作为预留
- 正常情况下直接使用底层分配器
- 内存紧张时使用预留池中的对象
- 当对象被释放时，尽可能将其返回预留池

#### 4.4.2 数据结构

内存池的核心数据结构：

```c
struct mempool_s {
    spinlock_t lock;
    int min_nr;                  /* 最小对象数 */
    int curr_nr;                 /* 当前预留数 */
    void **elements;             /* 预留对象数组 */

    void *pool_data;             /* 私有数据 */
    mempool_alloc_t *alloc;      /* 分配函数 */
    mempool_free_t *free;        /* 释放函数 */
    wait_queue_head_t wait;      /* 等待队列 */
};
typedef struct mempool_s mempool_t;
```

#### 4.4.3 分配和释放过程

内存池的分配函数：

```c
void *mempool_alloc(mempool_t *pool, gfp_t gfp_mask)
{
    void *element;

    /* 首先尝试使用底层分配器 */
    element = pool->alloc(gfp_mask, pool->pool_data);
    if (likely(element))
        return element;

    /* 如果失败，尝试从预留池分配 */
    spin_lock(&pool->lock);
    if (pool->curr_nr) {
        element = pool->elements[--pool->curr_nr];
        spin_unlock(&pool->lock);
        return element;
    }
    spin_unlock(&pool->lock);

    /* 如果允许等待，等待内存可用 */
    if (gfp_mask & __GFP_WAIT) {
        DEFINE_WAIT(wait);
        prepare_to_wait(&pool->wait, &wait, TASK_UNINTERRUPTIBLE);
        schedule_timeout(HZ/10);
        finish_wait(&pool->wait, &wait);

        /* 再次尝试分配 */
        return mempool_alloc(pool, gfp_mask);
    }

    return NULL;
}
```

释放对象到内存池的函数：

```c
void mempool_free(void *element, mempool_t *pool)
{
    /* 如果预留池未满，将对象添加到预留池 */
    spin_lock(&pool->lock);
    if (pool->curr_nr < pool->min_nr) {
        pool->elements[pool->curr_nr++] = element;
        spin_unlock(&pool->lock);
        return;
    }
    spin_unlock(&pool->lock);

    /* 预留池已满，直接释放对象 */
    pool->free(element, pool->pool_data);

    /* 唤醒等待的进程 */
    wake_up(&pool->wait);
}
```

#### 4.4.4 内存池的应用场景

内存池特别适用于以下场景：
- 文件系统操作（如日志提交）
- RAID重建
- 交换子系统
- 网络协议栈
- 其他关键路径上的内存分配

通过确保关键分配永远不会失败，内存池提高了系统的稳定性和可靠性。

### 4.5 每CPU分配器

为了减少锁竞争和提高性能，Linux实现了多种每CPU分配器。

#### 4.5.1 percpu_alloc

每CPU分配器允许每个CPU拥有独立的对象实例，避免了缓存行争用和锁竞争：

```c
/* 分配每CPU变量 */
void __percpu *__alloc_percpu(size_t size, size_t align)
{
    return pcpu_alloc(size, align, false);
}

/* 释放每CPU变量 */
void free_percpu(void __percpu *ptr)
{
    pcpu_free(ptr);
}

/* 访问每CPU变量 */
#define per_cpu(var, cpu) (*per_cpu_ptr(&(var), cpu))
#define this_cpu_ptr(var) per_cpu_ptr(var, smp_processor_id())
```

#### 4.5.2 应用场景

每CPU分配器特别适用于：
- 计数器（如统计信息）
- 小型缓冲区
- 状态变量
- 临时工作区域

通过避免跨CPU同步，每CPU分配器显著提高了多核系统的性能。

### 4.6 内存碎片化管理

内存碎片是影响系统性能的重要因素，Linux采用多种策略来管理碎片。

#### 4.6.1 内部碎片与外部碎片

- **内部碎片**：分配的内存大于实际需要的大小
- **外部碎片**：空闲内存块太小，无法满足大内存请求

#### 4.6.2 反碎片化策略

Linux采用以下策略减少碎片：

1. **迁移类型**：
   ```c
   /* 页面分配时考虑迁移类型 */
   static inline struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)
   {
       /* 根据GFP标志确定迁移类型 */
       int migratetype = gfp_migratetype(gfp_mask);
       return alloc_pages_node(numa_node_id(), gfp_mask, order, migratetype);
   }
   ```

2. **内存压缩**：
   ```c
   /* 压缩内存区域 */
   int compact_zone(struct zone *zone, struct compact_control *cc)
   {
       /* 将页面从区域的一端移动到另一端，创建连续空间 */
       /* ... */
       return 0;
   }
   ```

3. **内存回收**：
   ```c
   /* 回收页面以减少碎片 */
   static unsigned long shrink_zone(struct zone *zone, struct scan_control *sc)
   {
       /* 优先回收可移动页面 */
       /* ... */
       return nr_reclaimed;
   }
   ```

4. **页面迁移**：
   ```c
   /* 迁移页面到新位置 */
   int migrate_pages(struct list_head *from, new_page_t get_new_page,
                   free_page_t put_new_page, unsigned long private,
                   enum migrate_mode mode, int reason)
   {
       /* 将页面内容复制到新位置 */
       /* ... */
       return ret;
   }
   ```

#### 4.6.3 碎片监控

Linux提供了多种工具来监控内存碎片：

```bash
# 查看伙伴系统状态
$ cat /proc/buddyinfo
Node 0, zone   Normal   4096   3215   2145   1187    541    221     78     25      8      1      0

# 查看内存碎片指数
$ cat /proc/vmstat | grep fragmentation
pgalloc_normal 1234567
pgfree_normal 1234000
pgfrag_normal 123
```

通过这些机制，Linux能够有效管理内存碎片，提高内存利用率和系统性能。