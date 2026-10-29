# GPU监控及常见故障处理



## 一、GPU应该监控哪些目标状态信息：

| 层次         | 监控内容                                                     | 主要识别的问题                     |
| ------------ | ------------------------------------------------------------ | ---------------------------------- |
| 设备可用性   | 预期 GPU UUID 是否存在、NVML 查询是否成功、PCIe 设备是否可访问 | 掉卡、驱动异常、设备失联           |
| 硬件健康     | 温度、功耗、限频原因、ECC、显存重映射、PCIe/NVLink 错误      | 散热、供电、显存、互连故障         |
| 系统与运行时 | 内核 Xid/AER 日志、CUDA 错误、进程退出、NCCL 超时            | 驱动、计算内核、通信故障           |
| 推理服务     | 实际生成请求成功率、TTFT、TPOT、队列、吞吐                   | GPU 看似正常但服务不可用或性能退化 |



## 二、GPU 健康状态监控

监控分两层：**轻量级日常巡检** 和 **集群级可观测体系**。

### 1. 轻量级：nvidia-smi

最基础、最常用的工具，适合单机巡检和脚本化快速检查。

```bash
# 实时轮询，每秒刷新
nvidia-smi -l 1

# 查看 GPU 详细信息（含 ECC、温度、功耗）
nvidia-smi -q

# 专项查看 ECC 错误
nvidia-smi -q -d ECC

# 专项查看温度/功耗
nvidia-smi -q -d TEMPERATURE,POWER

# 查看多卡拓扑和 NVLink 状态
nvidia-smi topo -m

# 查看 PCIe 总线上的 GPU 设备
lspci -d 10de:
```

**关键指标解读：**

| 指标     | 正常范围              | 告警阈值                |
| -------- | --------------------- | ----------------------- |
| GPU-Util | 70-100%（推理满载时） | 长期低于 70% 说明有瓶颈 |
| 温度     | < 80°C                | 持续 > 80°C 易触发降频  |
| 显存使用 | < 90%                 | > 90% 有 OOM 风险       |
| 功耗     | < 功率上限            | 读数 ≤ 0 或接近上限     |
| ECC 错误 | 0                     | 非零需立即关注          |

### 2. 集群级：DCGM + Prometheus + Grafana

这是生产推理集群的推荐方案。

**架构：** DCGM-Exporter（:9400） → Prometheus（采集存储） → Grafana（可视化） + Alertmanager（告警）

**部署 DCGM-Exporter：**

```bash
docker run -d \
  --name dcgm-exporter \
  --gpus all \
  --restart unless-stopped \
  -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:3.3.6-3.4.2-ubuntu22.04
```

**DCGM 相比 nvidia-smi 的优势：**

- 支持 150+ 种 GPU 指标（远超 nvidia-smi）
- 自带主动健康检查和硬件诊断
- 支持 GPU 分组管理，适合多租户
- 原生支持 Prometheus 格式，Grafana 开箱即用
- 提供 SM 占用率、Tensor Core 利用率等细粒度饱和度指标

**核心监控指标（DCGM 字段）：**

| 维度     | 指标                                 | 说明                    |
| -------- | ------------------------------------ | ----------------------- |
| 利用率   | `DCGM_FI_DEV_GPU_UTIL`               | GPU 利用率              |
| 显存     | `DCGM_FI_DEV_FB_USED / FREE / TOTAL` | 显存使用情况            |
| 温度     | `DCGM_FI_DEV_GPU_TEMP`               | GPU 温度                |
| 功耗     | `DCGM_FI_DEV_POWER_USAGE`            | 当前功耗                |
| 错误     | `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL`      | ECC 单比特错误计数      |
| XID 错误 | `DCGM_FI_DEV_XID_ERRORS`             | XID 错误码              |
| 深度分析 | `DCGM_FI_PROF_SM_ACTIVE`             | SM 活跃率（真实饱和度） |
| PCIe     | `DCGM_FI_PROF_PCIE_TX_BYTES`         | PCIe 发送带宽           |

**关键告警规则示例（Prometheus）：**

```yaml
groups:
- name: gpu-alerts
  rules:
  - alert: GPUHighTemperature
    expr: DCGM_FI_DEV_GPU_TEMP > 80
    for: 5m
    labels:
      severity: critical
  - alert: GPUMemoryPressure
    expr: (DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)) > 0.9
    labels:
      severity: warning
  - alert: GPUMissing  # 掉卡检测
    expr: absent(DCGM_FI_DEV_GPU_UTIL{gpu="0"})
    for: 1m
    labels:
      severity: critical
```



## 三、常见的故障

| 故障或异常          | 典型表现                             | 优先排查方向                                      |
| ------------------- | ------------------------------------ | ------------------------------------------------- |
| 显存不足 OOM        | 模型加载失败、请求失败、进程退出     | 模型大小、上下文、并发、Batch、KV Cache、其他进程 |
| CUDA 非法访存       | `illegal memory access`、worker 崩溃 | 算子、模型实现、运行时和版本兼容                  |
| 散热异常            | 温度升高、热限频、吞吐下降           | 风扇、风道、灰尘、机房进风温度                    |
| 供电或连接异常      | 高负载时重启、失联、掉卡             | PSU、供电线、接头、板卡及平台                     |
| 显存硬件错误        | ECC、重映射、计算失败                | 显存健康、故障重复性                              |
| PCIe 链路异常       | AER、Replay 增长、设备失联           | 插槽、转接线、主板、链路稳定性                    |
| NVLink/多卡通信异常 | NCCL 超时、TP 实例卡死               | 互连、某一卡先发生故障、驱动及通信配置            |
| 驱动或 GSP 异常     | NVML 失败、RPC 超时、GPU 无响应      | 驱动、固件、内核及硬件状态                        |
| 容器设备暴露异常    | 宿主机正常，容器看不到 GPU           | Container Toolkit、设备映射、权限、Device Plugin  |



## 四、GPU 掉卡排查与处理

"掉卡"指 `nvidia-smi` 识别不到 GPU 或 GPU 显示 `rev ff`，通常伴随推理服务异常。

### 1. 快速检测掉卡

```bash
# 检测 GPU 掉卡（输出不为空则有掉卡）
lspci -d 10de: | grep "rev ff"

# 检查 GPU 数量是否与预期一致
lspci -d 10de: | grep -v 1af1

# 检查内核日志中的 Xid 错误
dmesg | grep -i xid

# 检查 PCIe 总线错误
dmesg | grep -i "AER\|PCIe Bus Error\|NVRM"

# 检查是否有 nouveau 驱动冲突
lsmod | grep nouveau
```

### 2. 常见掉卡原因与 Xid 错误码

| Xid 错误        | 含义                              | 处理方式                     |
| --------------- | --------------------------------- | ---------------------------- |
| **Xid 79**      | GPU has fallen off the bus        | 重启实例；仍不行联系平台换卡 |
| **Xid 48**      | Double Bit ECC Error（不可纠正）  | 重启实例；反复出现直接换卡   |
| **Xid 119/120** | GSP RPC Timeout / GSP Error       | 升级驱动到最新版；或关闭 GSP |
| **Xid 92**      | 高单比特 ECC 错误率               | 监控趋势，持续增长需换卡     |
| **Xid 94/95**   | ECC 错误（Contained/Uncontained） | 重启应用或重启实例           |
| **Xid 74**      | NVLink ERROR                      | 检查 NVLink 桥接器和多卡互联 |

### 3. 掉卡应急处理流程

```
发现掉卡
  ├── 1. 立即止损：将流量切走（负载均衡摘除该节点）
  ├── 2. 确认现象：
  │     ├── dmesg | grep -i xid     → 定位 Xid 错误码
  │     ├── lspci -d 10de:          → 确认硬件识别
  │     └── nvidia-smi              → 驱动是否能通信
  ├── 3. 尝试恢复：
  │     ├── 重启实例（大多数情况可恢复）
  │     ├── 若反复出现 → 检查 BMC 日志的 AER 错误
  │     ├── 检查插槽/Riser/供电
  │     └── 检查驱动版本兼容性
  ├── 4. 无法恢复 → 联系云平台/硬件供应商换卡
  └── 5. 记录归档：保留 dmesg、Xid 信息、复现路径
```

### 4. 预防性措施

- **开启 Persistence Mode**：减少 GPU 掉卡和带宽降低问题
  ```bash
  sudo nvidia-smi -pm 1  # 所有 GPU 开启常驻模式
  ```
- **ECC 错误趋势监控**：日常巡检纳入 ECC 增量分析，提前预警
- **DCGM 在线诊断**：`dcgmi diag -r 3` 可执行三级硬件诊断
- **Kubernetes 环境自动隔离**：ACK 的 NPD 组件可自动检测 Xid 错误并隔离异常 GPU 卡
- **固件/驱动版本管理**：定期灰度升级驱动和固件，保持版本匹配
- **散热巡检**：机房风道、风扇积尘、导热膏老化定期检查



## 五、nvidia-smi的常用命令

nvidia-smi 是 NVIDIA 驱动自带的 GPU 管理工具，日常运维用得非常多。下面按场景分类整理：

### 基础查看

```bash
# 查看 GPU 当前状态概览（最常用，默认输出）
nvidia-smi

# 每秒自动刷新
nvidia-smi -l 1

# 以表格形式持续监控，每 2 秒刷新
watch -n 2 nvidia-smi
```

### 详细信息

```bash
# 查看某张卡的完整详情（温度、功耗、ECC、时钟频率等）
nvidia-smi -q -i 0

# 查看所有卡的完整详情
nvidia-smi -q

# 只看温度和功耗
nvidia-smi -q -d TEMPERATURE,POWER

# 只看 ECC 错误信息
nvidia-smi -q -d ECC

# 只看显存页退休状态（用于判断 ECC 故障）
nvidia-smi -q -d PAGE_RETIREMENT

# Ampere 及以上架构看显存行重映射状态
nvidia-smi -q -d ROW_REMAPPER
```

### 格式输出（适合脚本解析）

```bash
# CSV 格式输出 GPU 编号、名称、显存使用、利用率、温度
nvidia-smi --query-gpu=index,name,memory.used,utilization.gpu,temperature.gpu --format=csv

# 查看每张卡的 ECC 未纠正错误数
nvidia-smi --query-gpu=ecc.errors.uncorrected.volatile.sram --format=csv

# 查看每张卡退休页数量
nvidia-smi --query-gpu=retired_pages.single_bit_ecc.count --format=csv

# 包含时间戳的持续监控（适合写日志）
nvidia-smi --query-gpu=timestamp,gpu_name,utilization.gpu,memory.used,temperature.gpu \
  --format=csv -l 5
```

### 进程管理

```bash
# 查看当前占用 GPU 的进程
nvidia-smi pmon -c 1

# 持续监控进程级 GPU 使用
nvidia-smi pmon

# 查看某张卡的进程（默认输出底部就有，也可以用 -q 看更详细）
nvidia-smi -i 0 -q -d COMPUTE
```

### 拓扑与互联

```bash
# 查看多卡拓扑关系和 NVLink 状态（多卡推理/训练必查）
nvidia-smi topo -m

# 查看 NVLink 详细状态
nvidia-smi nvlink -s

# 查看 NVLink 错误统计
nvidia-smi nvlink -e
```

### 配置与调优

```bash
# 开启 Persistence Mode（常驻模式，减少掉卡风险，推荐生产环境开启）
sudo nvidia-smi -pm 1

# 设置 GPU 时钟频率锁定（固定高性能）
sudo nvidia-smi -lgc 1500

# 设置功耗上限（单位：瓦）
sudo nvidia-smi -pl 300 -i 0

# 设置计算模式（独占进程模式，防止多进程冲突）
nvidia-smi -c EXCLUSIVE_PROCESS -i 0

# 重置 GPU（掉卡后尝试恢复）
sudo nvidia-smi -r -i 0

# 开启 ECC
nvidia-smi -i 0 -e 1
```

### 实时设备监控

```bash
# 设备级持续监控（利用率、显存、温度、功耗、时钟等）
nvidia-smi dmon -s ucmtp

# 带延迟的持续监控，每 5 秒采样
nvidia-smi dmon -d 5 -s ucmtp

# 统计模式（汇总一段时间的数据）
nvidia-smi stats -d
```

### 诊断与排障

```bash
# 生成完整诊断报告（提交 NVIDIA 支持时用）
sudo nvidia-bug-report.sh

# 查看支持的 GPU 特性
nvidia-smi --query-supported-clocks --format=csv

# 查看驱动版本
nvidia-smi -q | grep "Driver Version"

# 查看 CUDA 版本
nvidia-smi | grep "CUDA Version"
```

### 实用技巧速查

| 场景               | 命令                                                         |
| ------------------ | ------------------------------------------------------------ |
| 快速看一眼状态     | `nvidia-smi`                                                 |
| 持续盯着看         | `nvidia-smi -l 1` 或 `watch -n1 nvidia-smi`                  |
| 查哪张卡温度最高   | `nvidia-smi --query-gpu=index,temperature.gpu --format=csv`  |
| 查哪张卡显存快满了 | `nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv` |
| 看进程谁在吃 GPU   | `nvidia-smi pmon`                                            |
| 检查掉卡           | `nvidia-smi -L` 看是否所有卡都在                             |
| 看 NVLink 是否正常 | `nvidia-smi topo -m`                                         |
| 查 ECC 有没有问题  | `nvidia-smi -q -d ECC`                                       |
| 生成排障报告       | `sudo nvidia-bug-report.sh`                                  |

对于生产推理集群，建议把 `nvidia-smi -pm 1` 加入开机自启脚本，并将关键指标（温度、显存、ECC 错误）接入 DCGM + Prometheus 做持续监控和告警，而不是只靠人工 `nvidia-smi` 巡检。
