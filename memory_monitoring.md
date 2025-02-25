### 9.3 内存监控框架与工具

随着系统复杂性增加，全面的内存监控变得至关重要：

#### 9.3.1 统一内存监控框架

整合各种内存监控机制：

**核心组件**：
- eBPF内存分析器
- 统一内存事件接口
- 自动异常检测

**使用示例**：
```bash
# 使用BPF跟踪页面回收
bpftrace -e 'kprobe:shrink_page_list { @pages = count(); }'

# 监控内存压力事件
# 基于PSI(Pressure Stall Information)
cat /proc/pressure/memory
```

**详细监控指标解析**：
```bash
# PSI内存压力指标格式
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# 含义解释
# some: 至少有一个任务因内存不足而延迟
# full: 所有任务因内存不足而停滞
# avg10/60/300: 10秒/60秒/300秒平均压力百分比
# total: 自系统启动以来累计压力时间(微秒)
```

**高级内存事件监控**：
```bash
# 使用eBPF监控大页分配失败
bpftrace -e 'kretprobe:alloc_hugepage_vma /retval == 0/ { printf("Hugepage allocation failed\n"); }'

# 跟踪内存碎片化状态
cat /proc/buddyinfo

# 监控NUMA节点间内存平衡
numastat -m
```

**自动化监控脚本示例**：
```python
#!/usr/bin/env python3
import time
import subprocess

def monitor_memory_pressure():
    while True:
        # 读取内存压力信息
        with open('/proc/pressure/memory', 'r') as f:
            pressure = f.read().strip()

        # 解析压力值
        some_pressure = float(pressure.split('avg10=')[1].split()[0])

        # 如果压力超过阈值，触发警报
        if some_pressure > 20.0:
            print(f"警告: 内存压力过高 ({some_pressure}%)")
            # 这里可以添加发送警报的代码

        time.sleep(5)

if __name__ == "__main__":
    monitor_memory_pressure()
```

#### 9.3.2 内存可视化工具

现代内存监控依赖可视化工具提供直观视图：

**常用工具组合**：
- Prometheus + Grafana：时序数据收集与可视化
- Netdata：实时系统监控
- Datadog：企业级内存性能监控

**Grafana内存监控面板示例**：
```json
{
  "panels": [
    {
      "title": "内存使用率",
      "type": "graph",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "100 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100)",
          "legendFormat": "内存使用百分比"
        }
      ]
    },
    {
      "title": "内存压力",
      "type": "graph",
      "targets": [
        {
          "expr": "node_pressure_memory_some_avg10",
          "legendFormat": "10秒平均压力"
        }
      ]
    }
  ]
}
```

**内存异常检测配置**：
```yaml
# Prometheus告警规则示例
groups:
- name: memory_alerts
  rules:
  - alert: HighMemoryUsage
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "内存使用率过高"
      description: "服务器 {{ $labels.instance }} 内存使用率超过90%已持续5分钟"

  - alert: MemoryPressureHigh
    expr: node_pressure_memory_some_avg10 > 40
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "内存压力过高"
      description: "服务器 {{ $labels.instance }} 内存压力超过40%已持续2分钟"
```

#### 9.3.3 内存分析最佳实践

有效的内存监控需要遵循一些最佳实践：

**监控策略**：
- 建立基线：记录正常工作负载下的内存指标
- 多维度监控：同时监控使用率、压力和应用指标
- 趋势分析：关注长期内存使用模式变化

**告警配置**：
- 分级告警：根据严重程度设置不同级别
- 避免告警风暴：设置合理的告警间隔和聚合
- 上下文信息：告警中包含足够的诊断信息

**自动化响应**：
```bash
#!/bin/bash
# 内存压力自动响应脚本

# 监控内存压力
while true; do
  pressure=$(cat /proc/pressure/memory | grep some | awk '{print $2}' | cut -d= -f2)

  # 转换为数值
  pressure_value=$(echo $pressure | sed 's/\..*//')

  if [ $pressure_value -gt 50 ]; then
    echo "检测到高内存压力: $pressure_value%"

    # 执行紧急内存释放
    echo 1 > /proc/sys/vm/drop_caches

    # 如果压力仍然很高，限制非关键进程
    if [ $pressure_value -gt 80 ]; then
      echo "执行非关键进程限制"
      for pid in $(pgrep -f "non-critical-process"); do
        echo $pid > /sys/fs/cgroup/memory/limited/tasks
      done
    fi
  fi

  sleep 10
done
```