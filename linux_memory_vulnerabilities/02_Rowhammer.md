# Rowhammer 漏洞分析

## 1. 漏洞概述

Rowhammer是一个硬件层面的漏洞，通过反复访问DRAM中的同一行(row)，可能导致相邻行的比特位发生翻转，从而破坏内存中的数据。

- **发现时间**: 2014年
- **CVE编号**: 多个，包括CVE-2016-6728等
- **影响范围**: DDR3/DDR4内存
- **危害等级**: 高

## 2. 漏洞原理

### 2.1 DRAM结构
```
Bank 0      Bank 1      Bank 2
+--------+  +--------+  +--------+
|Row 0   |  |Row 0   |  |Row 0   |
|Row 1   |  |Row 1   |  |Row 1   |
|Row 2   |  |Row 2   |  |Row 2   |
|...     |  |...     |  |...     |
+--------+  +--------+  +--------+
```

### 2.2 攻击原理

1. **电荷泄漏**
```c
// 示例：快速访问两个内存行
void hammer_rows(void *row1, void *row2)
{
    volatile uint64_t *ptr1 = (volatile uint64_t *)row1;
    volatile uint64_t *ptr2 = (volatile uint64_t *)row2;

    // 持续访问两行内存
    for (int i = 0; i < 1000000; i++) {
        *ptr1;
        *ptr2;
        _mm_clflush(ptr1);  // 刷新缓存
        _mm_clflush(ptr2);  // 刷新缓存
    }
}
```

2. **位翻转效应**
```c
// 检测位翻转
bool detect_bit_flips(void *memory, size_t size)
{
    uint8_t *ptr = (uint8_t *)memory;
    // 写入初始模式
    memset(ptr, 0x55, size);

    // 对相邻行进行hammer操作
    hammer_rows(ptr - 4096, ptr + 4096);

    // 检查是否发生位翻转
    for (size_t i = 0; i < size; i++) {
        if (ptr[i] != 0x55)
            return true;
    }
    return false;
}
```

## 3. 攻击方式

### 3.1 单行攻击(Single-Sided Rowhammer)
```c
void single_sided_attack(void *target_row)
{
    // 持续访问目标行
    for (int i = 0; i < 1000000; i++) {
        clflush(target_row);
        *((volatile char *)target_row);
    }
}
```

### 3.2 双行攻击(Double-Sided Rowhammer)
```c
void double_sided_attack(void *row1, void *row2)
{
    for (int i = 0; i < 1000000; i++) {
        clflush(row1);
        clflush(row2);
        *((volatile char *)row1);
        *((volatile char *)row2);
    }
}
```

## 4. 防护措施

### 4.1 硬件层面防护

1. **Target Row Refresh (TRR)**
```c
// 硬件自动检测和刷新被频繁访问的行
struct dram_refresh_ctrl {
    unsigned long refresh_interval;
    unsigned long target_row_threshold;
    bool trr_enabled;
};

void init_trr(struct dram_refresh_ctrl *ctrl)
{
    ctrl->refresh_interval = 64000;  // 64ms
    ctrl->target_row_threshold = 32000;
    ctrl->trr_enabled = true;
}
```

2. **ECC内存**
```c
// ECC检查和纠正
struct ecc_info {
    uint64_t data;
    uint8_t ecc_bits;
};

bool check_and_correct_ecc(struct ecc_info *info)
{
    uint8_t computed_ecc = compute_ecc(info->data);
    if (computed_ecc != info->ecc_bits) {
        return correct_single_bit_error(info);
    }
    return true;
}
```

### 4.2 软件层面防护

1. **增加内存访问间隔**
```c
// 实现访问频率限制
static atomic_t row_access_count[MAX_ROWS];
static unsigned long last_access_time[MAX_ROWS];

bool check_row_access(unsigned long row_index)
{
    unsigned long current_time = jiffies;
    if (atomic_inc_return(&row_access_count[row_index]) > ROW_HAMMER_THRESHOLD) {
        if (time_before(current_time,
            last_access_time[row_index] + ROW_HAMMER_INTERVAL)) {
            return false;  // 访问过于频繁
        }
        atomic_set(&row_access_count[row_index], 0);
    }
    last_access_time[row_index] = current_time;
    return true;
}
```

2. **页面隔离**
```c
// 实现物理页面隔离
struct page *alloc_isolated_page(void)
{
    struct page *page = alloc_page(GFP_KERNEL);
    if (page) {
        // 确保相邻页面不被分配
        set_page_guard(page - 1);
        set_page_guard(page + 1);
    }
    return page;
}
```

3. **内存访问监控**
```c
// 实现内存访问监控
struct memory_access_monitor {
    unsigned long *access_counts;
    unsigned long window_start;
    spinlock_t lock;
};

void monitor_memory_access(struct memory_access_monitor *mon,
                         unsigned long address)
{
    unsigned long row = get_dram_row(address);
    spin_lock(&mon->lock);
    mon->access_counts[row]++;

    if (mon->access_counts[row] > THRESHOLD) {
        // 触发保护机制
        trigger_row_protection(row);
    }
    spin_unlock(&mon->lock);
}
```

## 5. 检测方法

### 5.1 内存测试工具
```c
// 实现内存测试
void test_for_rowhammer(void)
{
    void *memory = mmap(NULL, 1024*1024*1024,
                       PROT_READ|PROT_WRITE,
                       MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

    // 初始化内存模式
    init_memory_pattern(memory);

    // 执行hammer测试
    perform_hammer_test(memory);

    // 检查位翻转
    check_bit_flips(memory);
}
```

### 5.2 系统监控
```c
// 实现系统监控
void monitor_memory_behavior(void)
{
    struct perf_event_attr attr = {
        .type = PERF_TYPE_RAW,
        .config = 0x01,  // 内存访问事件
        .disabled = 1,
        .exclude_kernel = 0,
    };

    // 创建性能计数器
    perf_event_create_kernel_counter(&attr, -1, NULL, NULL, NULL);
}
```

## 6. 性能影响

### 6.1 防护措施的开销
- TRR: 1-2% 性能影响
- ECC: 2-3% 性能影响
- 软件监控: 5-15% 性能影响

### 6.2 优化建议
```c
// 自适应保护级别
struct protection_level {
    bool trr_enabled;
    bool ecc_enabled;
    bool software_monitoring;
    int refresh_rate;
};

void adjust_protection(struct protection_level *prot)
{
    // 根据工作负载调整保护级别
    if (is_high_risk_workload()) {
        prot->refresh_rate = HIGH_REFRESH_RATE;
        prot->software_monitoring = true;
    } else {
        prot->refresh_rate = NORMAL_REFRESH_RATE;
        prot->software_monitoring = false;
    }
}
```

## 7. 最佳实践

1. 使用支持TRR的新型内存
2. 部署ECC内存在关键系统中
3. 启用软件防护机制
4. 定期进行内存测试
5. 监控异常的内存访问模式

## 8. 参考资料

1. "Flipping Bits in Memory Without Accessing Them" (2014)
2. "Rowhammer.js: A Remote Software-Induced Fault Attack in JavaScript" (2015)
3. JEDEC DDR4 规范
4. Linux内核内存管理文档
5. Intel和AMD的硬件缓解措施白皮书