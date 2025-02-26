# Linux内核内存管理机制

## 3. 内存地址空间

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