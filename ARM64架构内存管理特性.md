# ARM64架构内存管理特性

## 1. ARM64内存管理硬件特性

### 1.1 内存属性

在ARM64架构中，内存属性通过MAIR_ELx寄存器配置，主要包括：

1. **设备内存属性**
   - Device-nGnRnE：非收集、非重排序、非提前
   - Device-nGnRE：非收集、非重排序、允许提前
   - Device-nGRE：非收集、允许重排序、允许提前
   - Device-GRE：允许收集、允许重排序、允许提前

2. **普通内存属性**
   - Normal Non-cacheable
   - Normal Write-Through Cacheable
   - Normal Write-Back Cacheable

### 1.2 访问权限控制

ARM64提供细粒度的访问权限控制：

```
+----------------+-------------------+
| AP[2:1] | EL0  | EL1/EL2/EL3      |
+----------------+-------------------+
|   00    | None | Read/Write       |
|   01    | RW   | Read/Write       |
|   10    | None | Read-only        |
|   11    | RO   | Read-only        |
+----------------+-------------------+
```

### 1.3 内存管理单元(MMU)

1. **转换表基地址寄存器**
   - TTBR0_EL1：用户空间地址转换
   - TTBR1_EL1：内核空间地址转换

2. **转换控制寄存器**
   ```c
   // TCR_EL1配置示例
   tcr_el1 = (16UL << TCR_T0SZ_SHIFT)   // 48位用户空间
           | (16UL << TCR_T1SZ_SHIFT)   // 48位内核空间
           | (TCR_TG0_4K)               // 4KB页大小
           | (TCR_TG1_4K)               // 4KB页大小
           | (TCR_IRGN0_WBWA)           // 内部缓存策略
           | (TCR_ORGN0_WBWA)           // 外部缓存策略
           | (TCR_SH0_INNER)            // 内部共享
           | (TCR_EL1_EPD1_BIT);        // 启用TTBR1_EL1
   ```

## 2. ARM64启动阶段内存管理

### 2.1 早期内存初始化

1. **设备树解析**
   ```c
   void __init early_init_dt_scan_memory(void)
   {
       // 解析设备树中的memory节点
       of_scan_flat_dt(early_init_dt_scan_memory_arch, NULL);
       // 设置memblock初始内存布局
       memblock_allow_resize();
   }
   ```

2. **页表初始化**
   ```c
   void __init paging_init(void)
   {
       // 设置页表基地址
       cpu_set_reserved_ttbr0();
       // 清除TLB
       local_flush_tlb_all();
       // 设置内存属性
       cpu_set_mair(MAIR_EL1_VALUE);
       // 设置转换控制
       cpu_set_tcr(TCR_EL1_VALUE);
   }
   ```

### 2.2 内存布局初始化

1. **线性映射区域设置**
   ```c
   void __init arm64_memblock_init(void)
   {
       // 预留内核镜像区域
       memblock_reserve(__pa(_text), _end - _text);
       // 预留设备树区域
       memblock_reserve(__pa(initial_boot_params),
                       initial_boot_params_size);
       // 设置线性映射区域
       arm64_mm_init_memblock();
   }
   ```

2. **虚拟内存布局**
   ```
   VA_START             未定义区域
   |                    |
   |  用户空间          | (0000000000000000 - 0000ffffffffffff)
   |                    |
   |--------------------|  
   |  内核空间          | (ffff000000000000 - ffffffffffffffff)
   |   - 模块区域       |
   |   - vmalloc区域    |
   |   - 直接映射区域   |
   VA_END              |
   ```

## 3. ARM64特有的内存管理特性

### 3.1 内存一致性模型

1. **共享属性**
   - Non-shareable：不共享
   - Inner Shareable：内部共享
   - Outer Shareable：外部共享

2. **屏障指令**
   ```assembly
   // 数据同步屏障
   dsb sy      // 全系统屏障
   dsb ish     // 内部共享屏障
   dsb osh     // 外部共享屏障
   
   // 数据内存屏障
   dmb sy      // 全系统屏障
   dmb ish     // 内部共享屏障
   dmb osh     // 外部共享屏障
   ```

### 3.2 内存管理扩展

1. **PAC (Pointer Authentication)**
   ```c
   // PAC使能配置
   static void __init init_cpu_pac(void)
   {
       // 设置PAC密钥
       if (system_supports_pac()) {
           set_pac_keys();
           enable_pac_features();
       }
   }
   ```

2. **MTE (Memory Tagging Extension)**
   ```c
   // MTE配置示例
   void init_mte(void)
   {
       if (system_supports_mte()) {
           // 设置MTE控制寄存器
           sysreg_clear_set(sctlr_el1, SCTLR_ELx_TCF_MASK,
                           SCTLR_ELx_TCF_SYNC);
           // 使能MTE
           sysreg_set_bit(sctlr_el1, SCTLR_ELx_ATA_BIT);
       }
   }
   ```

### 3.3 性能优化特性

1. **硬件预取器控制**
   ```c
   // 配置硬件预取器
   void arm64_set_prefetch_policy(void)
   {
       // L1预取器控制
       sysreg_clear_set(cpufeature, 
                       ARM64_HW_PREFETCH_MASK,
                       ARM64_HW_PREFETCH_ENABLE);
   }
   ```

2. **TLB管理优化**
   ```c
   // TLB维护操作
   static inline void __tlbi_vmalle1(void)
   {
       // VMALLE1指令：使所有EL1的TLB条目无效
       asm volatile("tlbi vmalle1\n"
                    "dsb ish\n"
                    "isb\n" : : : "memory");
   }
   ```

## 4. 调试与性能监控

### 4.1 内存调试特性

1. **内存访问追踪**
   ```c
   // 配置内存访问追踪
   static void enable_memory_tracing(void)
   {
       // 设置PMU事件计数器
       armv8pmu_pmcr_write(ARMV8_PMCR_E |    // 使能
                          ARMV8_PMCR_P |    // 事件计数复位
                          ARMV8_PMCR_C);    // 周期计数复位
   }
   ```

2. **故障诊断**
   ```c
   // 数据中止处理
   static int __kprobes do_data_abort(unsigned long addr,
                                     unsigned int esr,
                                     struct pt_regs *regs)
   {
       // 分析中止原因
       unsigned int ec = ESR_ELx_EC(esr);
       if (ec == ESR_ELx_EC_DABT_LOW)
           // 处理数据访问错误
           return do_data_abort_fault(addr, esr, regs);
       return -EFAULT;
   }
   ```

### 4.2 性能监控

1. **PMU事件监控**
   ```c
   // 配置PMU计数器
   static void arm64_pmu_configure_counter(int idx, u32 event)
   {
       // 选择要监控的事件
       armv8pmu_select_counter(idx);
       // 配置事件类型
       armv8pmu_write_evtype(idx, event);
       // 使能计数器
       armv8pmu_enable_counter(idx);
   }
   ```

2. **缓存性能监控**
   ```c
   // 监控缓存miss
   static void monitor_cache_misses(void)
   {
       // 配置L1D缓存miss计数器
       arm64_pmu_configure_counter(0,
           ARMV8_PMUV3_PERFCTR_L1D_CACHE_REFILL);
       // 配置L2缓存miss计数器
       arm64_pmu_configure_counter(1,
           ARMV8_PMUV3_PERFCTR_L2D_CACHE_REFILL);
   }
   ```

## 5. 最佳实践

### 5.1 内存布局优化

1. **对齐考虑**
   ```c
   // 页面对齐的内存分配
   void *alloc_aligned_memory(size_t size)
   {
       void *ptr;
       // 使用页面对齐的分配
       ptr = memalign(PAGE_SIZE, size);
       if (!ptr)
           return NULL;
       // 设置内存属性
       set_memory_attributes(ptr, size);
       return ptr;
   }
   ```

2. **大页面使用**
   ```c
   // 配置大页面支持
   static void setup_hugepage_support(void)
   {
       if (cpu_has_feature(CPU_FEATURE_HUGE_PAGE)) {
           // 使能大页面支持
           write_sysreg(read_sysreg(tcr_el1) |
                       TCR_TG0_16K | TCR_TG1_16K,
                       tcr_el1);
       }
   }
   ```

### 5.2 性能优化建议

1. **缓存优化**
   - 合理使用预取指令
   - 注意数据对齐
   - 避免频繁的缓存失效操作

2. **TLB优化**
   - 使用大页面减少TLB条目
   - 合理使用TLB广播操作
   - 避免不必要的地址空间切换

3. **内存访问模式优化**
   - 保持内存访问的局部性
   - 减少跨NUMA节点的访问
   - 使用适当的内存屏障指令