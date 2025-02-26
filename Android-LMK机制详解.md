# Android Low Memory Killer (LMK) 机制详解

## 1. 工作原理

Low Memory Killer (LMK) 是Android系统中的一个内存管理机制，用于在系统内存不足时，通过终止优先级较低的进程来释放内存。其工作原理如下：

- 系统持续监控可用内存状态
- 当内存压力达到特定阈值时触发
- 根据进程优先级选择终止目标
- 通过信号方式终止进程

### 1.1 核心组件

- LMK驱动（lowmemorykiller.c）
  - 实现进程筛选和终止逻辑
  - 维护内存阈值和优先级映射
  - 提供sysfs接口用于配置

- Activity Manager Service
  - 管理应用进程生命周期
  - 动态更新进程oom_adj值
  - 响应内存压力事件

- Process Priority Manager
  - 计算进程优先级得分
  - 处理优先级继承关系
  - 维护进程状态转换

### 1.2 工作流程

### 1.3 源码实现

核心数据结构：
```c
// lowmemorykiller.c
static struct task_struct *lowmem_deathpending;
static unsigned long lowmem_deathpending_timeout;

static int lowmem_minfree[6] = {
    3 * 512,    /* Foreground App */
    2 * 1024,   /* Visible App */
    4 * 1024,   /* Secondary Server */
    16 * 1024,  /* Hidden App */
    28 * 1024,  /* Content Provider */
    32 * 1024   /* Empty App */
};
```

关键函数实现：
```c
static unsigned long lowmem_scan(int other_free, int other_file)
{
    struct task_struct *tsk;
    struct task_struct *selected = NULL;
    unsigned long rem = 0;
    int tasksize;
    int i;
    int min_score_adj = OOM_SCORE_ADJ_MAX + 1;
    int selected_tasksize = 0;
    int selected_oom_score_adj;
    int array_size = ARRAY_SIZE(lowmem_adj);
    int other_free_pages = other_free;
    int other_file_pages = other_file;

    // 遍历进程列表
    rcu_read_lock();
    for_each_process(tsk) {
        // 计算进程内存占用
        tasksize = get_mm_rss(tsk->mm);
        // 获取oom_score_adj
        oom_score_adj = tsk->signal->oom_score_adj;
        // 选择最佳终止目标
        if (selected) {
            if (oom_score_adj < selected_oom_score_adj)
                continue;
            if (oom_score_adj == selected_oom_score_adj &&
                tasksize <= selected_tasksize)
                continue;
        }
        selected = tsk;
        selected_tasksize = tasksize;
        selected_oom_score_adj = oom_score_adj;
    }
    // 发送终止信号
    if (selected && selected_oom_score_adj > min_score_adj) {
        send_sig(SIGKILL, selected, 0);
        rem += selected_tasksize;
    }
    rcu_read_unlock();
    return rem;
}
```

1. 系统定期检查可用内存
   - 通过meminfo接口获取内存状态
   - 计算可用内存比例
   - 评估swap使用情况

2. 当可用内存低于设定阈值时触发LMK
   - 读取/proc/meminfo获取系统内存信息
   - 检查各个内存阈值配置
   - 确定当前内存压力等级

3. 扫描进程列表确定候选进程
   - 遍历/proc目录获取进程信息
   - 读取进程oom_adj值
   - 计算进程评分（oom_score）
   - 考虑进程内存占用

4. 发送SIGKILL信号终止选中进程
   - 选择得分最高的进程
   - 通过kill系统调用发送信号
   - 等待进程退出完成
   - 验证内存释放效果

## 2. 进程优先级（oom_adj）

Android系统使用oom_adj值来表示进程的优先级，范围从-17到15，值越小优先级越高：

- -17：系统进程
  - init进程
  - kernel线程
  - 系统关键服务

- -16：persistent进程
  - SystemServer
  - 电话服务
  - 蓝牙服务

- -12：前台应用
  - 当前可见且可交互的应用
  - 正在播放音乐的应用
  - 具有前台服务的应用

- -8：可见应用
  - 部分UI可见但不可交互
  - 处于暂停状态的应用
  - Dialog的背景应用

- 0：后台应用
  - 对用户不可见的应用
  - 已被暂停的应用
  - 缓存在内存中的应用

- 7：内容提供者
  - Content Provider
  - 后台服务

- 8：空进程
  - 不含任何活动组件
  - 仅作为重启加速的缓存

### 2.1 优先级调整

- 进程状态变化时自动调整
  - Activity生命周期变化
  - Service启动和停止
  - Broadcast接收处理
  - Provider访问情况

- 通过ActivityManager动态管理
  - 监控应用使用频率
  - 统计应用切换模式
  - 响应用户操作意图

- 考虑用户交互情况
  - 最近使用的应用
  - 常用应用列表
  - 用户习惯分析

### 2.2 优先级继承

某些情况下，进程会继承关联进程的优先级：

1. Service绑定
   - 被绑定的Service继承调用者优先级
   - 多个调用者取最高优先级

2. Provider访问
   - Provider进程继承访问者优先级
   - 保持到访问结束

3. 进程组关系
   - 子进程继承父进程优先级
   - 同组进程共享优先级

## 3. 内存压力等级

Android定义了多个内存压力等级，每个等级对应不同的处理策略：

### 3.1 压力等级定义

- PRESSURE_NONE：无压力
  - 系统运行流畅
  - 内存充足
  - 无需干预

- PRESSURE_MODERATE：中等压力
  - 后台应用较多
  - 内存使用率上升
  - 开始后台清理

- PRESSURE_CRITICAL：严重压力
  - 可用内存不足
  - 系统响应变慢
  - 强制回收内存

- PRESSURE_EXTREME：极端压力
  - 系统濒临崩溃
  - 主要功能受影响
  - 激进式内存回收

### 3.2 触发阈值

每个压力等级都有对应的内存阈值：

```
PRESSURE_NONE: > 剩余内存50%
PRESSURE_MODERATE: 剩余内存30-50%
PRESSURE_CRITICAL: 剩余内存20-30%
PRESSURE_EXTREME: < 剩余内存20%
```

阈值计算方法：
```java
// 计算当前内存压力等级
private int calculatePressureLevel() {
    long totalMem = getTotalMemory();
    long availMem = getAvailableMemory();
    float memRatio = (float)availMem / totalMem;
    
    if (memRatio > 0.5) {
        return PRESSURE_NONE;
    } else if (memRatio > 0.3) {
        return PRESSURE_MODERATE;
    } else if (memRatio > 0.2) {
        return PRESSURE_CRITICAL;
    } else {
        return PRESSURE_EXTREME;
    }
}
```

### 3.3 压力检测机制

1. 定时检测
   - 系统每隔一定时间检查内存状态
   - 默认检测间隔可配置
   - 支持动态调整检测频率

2. 事件触发
   - 大型应用启动时
   - 系统服务内存申请失败
   - 应用报告内存压力

3. 阈值动态调整
   - 基于设备总内存
   - 考虑系统运行时间
   - 适应用户使用模式

## 4. 与标准Linux OOM Killer的区别

### 4.1 触发机制

- LMK：主动监控，提前干预
  - 持续监控内存状态
  - 设置多级预警阈值
  - 渐进式内存回收
  - 可预测的处理流程

- OOM Killer：被动触发，紧急响应
  - 等待系统内存耗尽
  - 仅一次性阈值判断
  - 强制杀死进程
  - 不可预测的处理时机

### 4.2 进程选择

- LMK：基于Android特有的优先级体系
  - 使用oom_adj值
  - 考虑应用可见性
  - 重视用户体验
  - 保护系统关键进程

- OOM Killer：基于进程的内存占用和运行时间
  - 使用badness值
  - 偏重内存占用大小
  - 考虑进程运行时间
  - 计算资源消耗得分

### 4.3 执行方式

- LMK：渐进式，可控制
  - 分级处理策略
  - 可配置处理参数
  - 支持策略调整
  - 保留恢复机制

- OOM Killer：紧急处理，不可控
  - 一次性处理
  - 固定处理逻辑
  - 不可中断
  - 无法精细控制

### 4.4 使用场景

- LMK：移动设备内存管理
  - 资源受限环境
  - 用户交互频繁
  - 应用切换频繁
  - 电池续航敏感

- OOM Killer：通用Linux系统
  - 服务器环境
  - 资源相对充足
  - 任务稳定运行
  - 性能优先考虑

### 4.5 配置灵活性

- LMK：高度可配置
  - 阈值可调整
  - 优先级可定制
  - 策略可修改
  - 支持动态调整

- OOM Killer：配置有限
  - 参数较少
  - 策略相对固定
  - 调整空间小
  - 主要依赖系统默认值

## 5. 实现细节

### 5.1 关键数据结构

```c
// 进程描述符
struct task_struct {
    ...
    int oom_adj;           // 进程优先级调整值
    int oom_score_adj;     // OOM评分调整值
    unsigned long mm_rss;  // 进程实际物理内存占用
    ...
};

// 内存区域描述符
struct mem_cgroup {
    ...
    unsigned long usage_in_bytes;  // 内存使用量
    unsigned long memsw_usage_in_bytes;  // 内存+交换区使用量
    ...
};

// LMK配置参数
struct lowmem_tune {
    int minfree[

### 5.2 核心算法

```c
static unsigned long lowmem_scan(int other_free, int other_file)
{
    // 扫描进程列表
    // 计算oom分数
    // 选择终止进程
}
```

### 5.3 调试接口

1. procfs接口
```bash
# 查看当前LMK配置
cat /proc/sys/vm/lowmem_minfree
cat /proc/sys/vm/lowmem_adj

# 查看进程oom_score
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj

# 查看LMK统计信息
cat /proc/lowmemorykiller/stat
```

2. sysfs接口
```bash
# 调整LMK参数
echo "1024,2048,4096,8192,16384,32768" > /sys/module/lowmemorykiller/parameters/minfree
echo "-17,-12,-8,0,7,15" > /sys/module/lowmemorykiller/parameters/adj

# 开启/关闭调试日志
echo 1 > /sys/module/lowmemorykiller/parameters/debug_level
```

3. EventLog接口
```java
// 监听LMK事件
adb logcat -b events | grep lowmemorykiller

// 分析内存压力
adb shell dumpsys meminfo
```

## 6. 调优建议

### 6.1 系统级优化

1. 内存阈值调整
   - 根据设备总内存量调整minfree值
   - 考虑用户使用场景设置合理的阈值
   - 避免过于激进的回收策略
   ```bash
   # 示例：4GB内存设备的推荐配置
   echo "18432,23040,27648,32256,36864,46080" > /sys/module/lowmemorykiller/parameters/minfree
   ```

2. 优先级策略优化
   - 合理分配oom_adj值
   - 保护关键系统服务
   - 考虑应用使用频率
   ```java
   // 示例：调整应用优先级
   Process.setOomAdj(pid, Process.FOREGROUND_APP_ADJ);
   ```

3. 监控机制完善
   - 启用内存使用统计
   - 记录LMK触发日志
   - 分析系统内存趋势
   ```bash
   # 开启详细日志
   echo 2 > /sys/module/lowmemorykiller/parameters/debug_level
   ```

### 6.2 应用级优化

1. 内存使用优化
   - 及时释放不需要的资源
   - 使用弱引用管理缓存
   - 避免内存泄漏
   ```java
   // 示例：使用WeakReference
   WeakReference<Bitmap> weakBitmap = new WeakReference<>(bitmap);
   ```

2. 生命周期管理
   - 正确实现onTrimMemory()
   - 响应内存压力事件
   - 主动释放缓存
   ```java
   @Override
   public void onTrimMemory(int level) {
       if (level >= TRIM_MEMORY_MODERATE) {
           clearMemoryCache();
       }
   }
   ```

3. 后台服务优化
   - 使用WorkManager替代Service
   - 避免长期运行的后台任务
   - 合理设置服务优先级
   ```java
   // 示例：使用WorkManager
   WorkManager.getInstance(context)
       .enqueueUniqueWork("work", ExistingWorkPolicy.KEEP,
           OneTimeWorkRequest.from(MyWorker.class));
   ```

## 7. 常见问题解决

### 7.1 应用被意外终止

1. 问题表现
   - 应用无预警退出
   - 后台服务频繁重启
   - 数据丢失

2. 排查方法
   ```bash
   # 查看系统日志
   adb logcat | grep -i kill
   
   # 分析内存状态
   adb shell dumpsys meminfo
   
   # 查看进程优先级
   adb shell ps -A -o PID,PPID,USER,STATE,OOM,CMD
   ```

3. 解决方案
   - 优化应用内存使用
   - 调整进程优先级
   - 实现数据保存机制

### 7.2 系统响应延迟

1. 问题表现
   - UI操作卡顿
   - 应用启动缓慢
   - 系统动画不流畅

2. 排查方法
   ```bash
   # 监控内存状态
   adb shell top -m 10 -s rss
   
   # 分析CPU使用
   adb shell vmstat 1
   
   # 查看swap使用情况
   adb shell cat /proc/meminfo
   ```

3. 解决方案
   - 调整LMK阈值
   - 限制后台进程数量
   - 优化系统缓存策略

### 7.3 内存泄漏问题

1. 问题表现
   - 应用内存持续增长
   - 频繁触发LMK
   - 系统内存压力大

2. 排查方法
   ```bash
   # 使用Android Studio Memory Profiler
   
   # 获取heap dump
   adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
   
   # 分析内存泄漏
   adb pull /data/local/tmp/heap.hprof ./
   ```

3. 解决方案
   - 修复内存泄漏
   - 实现内存监控
   - 优化资源释放

### 7.4 优先级设置问题

1. 问题表现
   - 重要进程被终止
   - 服务保活失败
   - 进程优先级不合理

2. 排查方法
   ```bash
   # 查看进程优先级
   adb shell ps -A -o PID,PPID,USER,STATE,OOM,CMD
   
   # 分析优先级继承
   adb shell dumpsys activity processes
   ```

3. 解决方案
   - 调整oom_adj值
   - 优化服务设计
   - 实现合理的保活机制

## 8. 最佳实践

### 8.1 开发建议

1. 内存使用
   - 避免大对象分配
   - 使用内存缓存池
   - 及时回收临时对象

2. 进程管理
   - 合理设置进程优先级
   - 避免过多后台进程
   - 实现进程间通信机制

3. 性能优化
   - 监控内存使用趋势
   - 响应系统内存事件
   - 实现平滑的降级策略

### 8.2 运维建议

1. 监控策略
   - 部署内存监控系统
   - 收集LMK触发日志
   - 分析系统性能指标

2. 告警机制
   - 设置内存使用阈值
   - 监控进程存活状态
   - 跟踪系统响应时间

3. 优化方案
   - 定期评估系统性能
   - 调整LMK配置参数
   - 优化应用运行策略