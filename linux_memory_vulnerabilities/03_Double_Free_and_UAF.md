# Double Free 和 Use After Free 漏洞分析

## 1. 漏洞概述

Double Free和Use After Free (UAF)是两种常见的内存管理漏洞，它们都与内存释放后的不当操作有关。

### 1.1 Double Free
- **定义**: 对同一块内存重复执行释放操作
- **危害**: 可能导致堆损坏、任意代码执行
- **CVE示例**: CVE-2018-6555, CVE-2019-2215

### 1.2 Use After Free
- **定义**: 在内存释放后继续使用该内存
- **危害**: 可能导致信息泄露、代码执行
- **CVE示例**: CVE-2019-2181, CVE-2020-0022

## 2. 漏洞原理

### 2.1 Double Free示例
```c
// 典型的Double Free场景
void double_free_example(void)
{
    char *ptr = kmalloc(16, GFP_KERNEL);
    if (!ptr)
        return;

    // 第一次释放
    kfree(ptr);

    // 第二次释放同一指针
    kfree(ptr);  // Double Free!
}
```

### 2.2 Use After Free示例
```c
struct vulnerable_struct {
    void (*callback)(void);
    char data[64];
};

void uaf_example(void)
{
    struct vulnerable_struct *obj = kmalloc(sizeof(*obj), GFP_KERNEL);
    if (!obj)
        return;

    // 设置回调函数
    obj->callback = legitimate_function;

    // 释放对象
    kfree(obj);

    // 继续使用已释放的对象
    obj->callback();  // Use After Free!
}
```

## 3. 漏洞利用技术

### 3.1 Double Free利用
```c
// Double Free利用示例
void exploit_double_free(void)
{
    char *ptr1, *ptr2;

    // 1. 首次分配
    ptr1 = kmalloc(32, GFP_KERNEL);

    // 2. 首次释放
    kfree(ptr1);

    // 3. 第二次释放（Double Free）
    kfree(ptr1);

    // 4. 分配新内存，可能获得被破坏的块
    ptr2 = kmalloc(32, GFP_KERNEL);

    // 5. 此时ptr2可能指向被破坏的内存结构
}
```

### 3.2 UAF利用
```c
// UAF利用示例
struct control_struct {
    void (*fn)(void);
    char data[128];
};

void exploit_uaf(void)
{
    struct control_struct *ctrl;

    // 1. 分配对象
    ctrl = kmalloc(sizeof(*ctrl), GFP_KERNEL);

    // 2. 释放对象
    kfree(ctrl);

    // 3. 分配新对象到相同位置
    char *new_data = kmalloc(sizeof(*ctrl), GFP_KERNEL);
    memset(new_data, 'A', sizeof(*ctrl));

    // 4. 使用已释放的对象（现在包含攻击者控制的数据）
    ctrl->fn();  // 可能执行攻击者控制的代码
}
```

## 4. 防护措施

### 4.1 编译时防护

1. **KASAN (Kernel Address Sanitizer)**
```c
// 启用KASAN的编译选项
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

// KASAN检测示例
void *kasan_kmalloc(size_t size, gfp_t flags)
{
    void *ptr = kmalloc(size, flags);
    if (ptr)
        kasan_mark_initialized(ptr, size);
    return ptr;
}

void kasan_kfree(void *ptr)
{
    kasan_mark_freed(ptr);
    kfree(ptr);
}
```

2. **KMSAN (Kernel Memory Sanitizer)**
```c
// 启用KMSAN的编译选项
CONFIG_KMSAN=y

// KMSAN使用示例
void kmsan_check_memory_access(void *addr, size_t size)
{
    if (kmsan_is_uninitialized(addr, size))
        kmsan_report_error(addr, size);
}
```

### 4.2 运行时防护

1. **SLUB分配器防护**
```c
struct kmem_cache {
    unsigned long flags;
    struct kmem_cache_cpu __percpu *cpu_slab;
    struct kmem_cache_node *node[MAX_NUMNODES];

    // 对象追踪
    void (*ctor)(void *);
    unsigned long useroffset;    /* Usercopy region offset */
    unsigned long usersize;      /* Usercopy region size */

    // 调试信息
    const char *name;
    struct list_head list;
    int object_size;
    int size;

    // 检测机制
    atomic_t refcount;
    atomic_t active_objs;
    atomic_t total_objs;
};

// SLUB分配器防护实现
void *secure_slab_alloc(struct kmem_cache *cache, gfp_t flags)
{
    void *obj = slab_alloc(cache, flags);
    if (obj) {
        // 初始化内存
        memset(obj, 0, cache->object_size);
        // 记录分配信息
        track_alloc(obj, cache);
    }
    return obj;
}

void secure_slab_free(struct kmem_cache *cache, void *obj)
{
    // 检查是否重复释放
    if (is_already_freed(obj)) {
        report_double_free(obj);
        return;
    }

    // 清除内存内容
    memset(obj, POISON_PATTERN, cache->object_size);

    // 记录释放信息
    track_free(obj, cache);

    slab_free(cache, obj);
}
```

2. **指针追踪**
```c
struct ptr_track {
    void *ptr;
    unsigned long alloc_time;
    unsigned long free_time;
    const char *alloc_func;
    const char *free_func;
};

// 指针追踪实现
static DEFINE_HASHTABLE(ptr_track_table, 16);

void track_pointer_alloc(void *ptr, const char *func)
{
    struct ptr_track *track = kmalloc(sizeof(*track), GFP_ATOMIC);
    if (track) {
        track->ptr = ptr;
        track->alloc_time = jiffies;
        track->alloc_func = func;
        track->free_time = 0;
        hash_add(ptr_track_table, &track->node, (unsigned long)ptr);
    }
}

void track_pointer_free(void *ptr, const char *func)
{
    struct ptr_track *track;

    hash_for_each_possible(ptr_track_table, track, node, (unsigned long)ptr) {
        if (track->ptr == ptr) {
            if (track->free_time != 0) {
                report_double_free(ptr);
                return;
            }
            track->free_time = jiffies;
            track->free_func = func;
            return;
        }
    }
}
```

### 4.3 内存隔离技术

```c
// 页面隔离实现
struct isolated_page {
    struct page *page;
    unsigned long flags;
    void *shadow_copy;
};

struct isolated_page *isolate_page_for_security(struct page *page)
{
    struct isolated_page *iso_page = kmalloc(sizeof(*iso_page), GFP_KERNEL);
    if (!iso_page)
        return NULL;

    // 创建页面副本
    iso_page->shadow_copy = kmalloc(PAGE_SIZE, GFP_KERNEL);
    if (!iso_page->shadow_copy) {
        kfree(iso_page);
        return NULL;
    }

    // 复制页面内容
    memcpy(iso_page->shadow_copy, page_address(page), PAGE_SIZE);

    // 设置隔离标志
    iso_page->page = page;
    iso_page->flags = page->flags;
    SetPageIsolated(page);

    return iso_page;
}
```

## 5. 检测工具

### 5.1 静态分析工具
```c
// Sparse静态分析器配置
#ifdef __CHECKER__
#define __force __attribute__((force))
#define __acquires(x) __attribute__((context(x,0,1)))
#define __releases(x) __attribute__((context(x,1,0)))
#define __acquire(x) __context__(x,1)
#define __release(x) __context__(x,-1)
#endif

// 使用示例
void *__must_check check_pointer(void *ptr)
{
    if (IS_ERR(ptr))
        return NULL;
    return ptr;
}
```

### 5.2 动态分析工具
```c
// KMEMLEAK配置
CONFIG_DEBUG_KMEMLEAK=y

// KMEMLEAK使用示例
void check_memory_leaks(void)
{
    kmemleak_scan();  // 扫描内存泄漏

    // 手动标记对象
    void *ptr = kmalloc(size, GFP_KERNEL);
    kmemleak_alloc(ptr, size, 1, GFP_KERNEL);

    // 使用完毕后
    kmemleak_free(ptr);
    kfree(ptr);
}
```

## 6. 最佳实践

1. **安全编码规范**
```c
// 1. 使用安全的内存操作函数
void *secure_alloc(size_t size)
{
    void *ptr = kmalloc(size, GFP_KERNEL);
    if (ptr)
        memset(ptr, 0, size);
    return ptr;
}

// 2. 使用智能指针包装
struct auto_ptr {
    void *ptr;
    void (*cleanup)(void*);
};

void auto_ptr_cleanup(struct auto_ptr *ap)
{
    if (ap->ptr && ap->cleanup) {
        ap->cleanup(ap->ptr);
        ap->ptr = NULL;
    }
}
```

2. **引用计数**
```c
struct ref_counted_obj {
    atomic_t refcount;
    void *data;
};

struct ref_counted_obj *obj_get(struct ref_counted_obj *obj)
{
    if (atomic_inc_not_zero(&obj->refcount))
        return obj;
    return NULL;
}

void obj_put(struct ref_counted_obj *obj)
{
    if (atomic_dec_and_test(&obj->refcount))
        kfree(obj);
}
```

3. **内存屏障**
```c
// 使用内存屏障确保操作顺序
void secure_memory_ops(void)
{
    // 写入数据
    write_data();
    wmb();  // 写内存屏障

    // 更新状态
    update_status();

    // 读取数据
    rmb();  // 读内存屏障
    read_data();
}
```

## 7. 调试技巧

### 7.1 内存检查工具
```c
// 检查内存状态
void debug_memory_state(void *ptr, size_t size)
{
    // 检查内存是否可访问
    if (access_ok(VERIFY_WRITE, ptr, size)) {
        // 检查内存内容
        if (memory_is_poisoned(ptr))
            pr_err("Memory is poisoned\n");

        // 检查内存标记
        if (page_is_marked(ptr))
            pr_err("Memory is marked\n");
    }
}
```

### 7.2 堆栈回溯
```c
// 打印堆栈回溯
void print_stack_trace(void)
{
    struct stack_trace trace;
    unsigned long entries[32];

    trace.nr_entries = 0;
    trace.entries = entries;
    trace.max_entries = ARRAY_SIZE(entries);
    trace.skip = 1;

    save_stack_trace(&trace);
    print_stack_trace(&trace, 0);
}
```

## 8. 参考资料

1. Linux内核安全子系统文档
2. KASAN文档和源码
3. SLUB分配器实现细节
4. CVE数据库中的相关漏洞报告
5. Linux内核邮件列表中的安全讨论