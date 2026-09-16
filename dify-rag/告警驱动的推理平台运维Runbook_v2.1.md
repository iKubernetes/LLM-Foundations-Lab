# 告警驱动的推理平台运维 Runbook（Alert-Driven RCA 版）

> **适用范围**：vLLM 0.29.0 推理平台（Qwen3.5 系列 LLM + Qwen3-Embedding-4B），经 LiteLLM Proxy 统一代理后，为 Dify 1.17.1 提供 Agent 与 RAG 推理能力；含宿主机、GPU、监控栈的全栈告警处置与根因分析。
>
> **消费方**：Dify Workflow（Webhook 接收 AlertManager 告警 → 代码执行节点规范化 → Agent 节点结合本 RAG 自动判定根因）。
>
> **文档版本**：v2.1（由 v2.0 演进：新增 LiteLLM Proxy 推理网关层与全量主机命名约定）
> **最后更新**：2026-09-14
> **维护角色**：AI Infra / SRE

---

## 目录

- [1. 系统全景与告警链路](#1-系统全景与告警链路)
- [2. Dify Workflow 集成契约](#2-dify-workflow-集成契约)
- [3. 告警目录与分诊矩阵](#3-告警目录与分诊矩阵)
- [4. 根因判定决策树](#4-根因判定决策树)
- [5. 告警级 Runbook（按告警逐条处置）](#5-告警级-runbook按告警逐条处置)
- [6. 服务层 Runbook：vLLM 部署与运维](#6-服务层-runbookvllm-部署与运维)
- [7. 推理网关层 Runbook：LiteLLM Proxy](#7-推理网关层-runbooklitellm-proxy)
- [8. 宿主机与 GPU 层 Runbook](#8-宿主机与-gpu-层-runbook)
- [9. Dify 平台层 Runbook](#9-dify-平台层-runbook)
- [10. 历史故障案例库（模板与示例）](#10-历史故障案例库模板与示例)
- [11. 附录：主机、端口与参数速查表](#11-附录主机端口与参数速查表)

---

## 1. 系统全景与告警链路

### 1.1 主机清单与命名约定

| 主机 | FQDN:端口 | 角色 |
|---|---|---|
| Prometheus | `prometheus.magedu.com:9090` | 指标采集与告警规则评估 |
| AlertManager | `alertmanager.magedu.com:9093` | 告警分组/静默/路由，Webhook 推送至 Dify |
| Grafana | `grafana.magedu.com:3000` | 监控可视化 |
| vLLM 推理主机 | `vllm.magedu.com`（LLM :8000 / Embedding :8001） | 独立主机，运行两类推理服务 |
| LiteLLM Proxy | `litellm.magedu.com:4000` | 推理网关，统一代理 LLM 与 Embedding |
| Dify | `dify.magedu.com:80` | Agent / RAG / Workflow 平台 |
| prometheus-mcp | `prometheus-mcp.magedu.com:9100` | Prometheus MCP 服务，供 Agent 实时查询指标 |

### 1.2 监控对象五层模型

| 层级 | 对象 | 关键 Exporter / 数据源 |
|---|---|---|
| L1 基础设施层 | 各宿主机 CPU / 内存 / 磁盘 / 网络 | node_exporter |
| L2 GPU 层 | vllm.magedu.com 上的 RTX 3090 / 4090D / 4090（显存、温度、功耗、掉卡） | dcgm_exporter 或 nvidia_gpu_exporter |
| L3 推理服务层 | vLLM LLM（vllm.magedu.com:8000）、vLLM Embedding（vllm.magedu.com:8001） | vLLM 内置 `/metrics`（Prometheus 格式） |
| L4 推理网关层 | LiteLLM Proxy（litellm.magedu.com:4000） | LiteLLM `/metrics`、健康检查 `/health/liveliness` |
| L5 平台应用层 | Dify 1.17.1（dify.magedu.com:80） | Dify 日志 / 自定义指标 / 黑盒探测 |

### 1.3 告警处置自动化链路

```
被监控对象 → Exporter → Prometheus（prometheus.magedu.com:9090，告警规则评估）
    → AlertManager（alertmanager.magedu.com:9093，分组/静默/路由）
    → Webhook → Dify Workflow（dify.magedu.com:80，v1.17.1）
        ├─ 起始节点：Webhook（接收 AlertManager 告警载荷）
        ├─ 代码执行节点：告警信息规范化（统一字段 schema，见 §2.2）
        └─ Agent 节点：本地 LLM（经 LiteLLM 调用 qwen3，工具调用已开启）
             ├─ 检索本 RAG 知识库（Qwen3-Embedding-4B 向量化）
             ├─ 可经 prometheus-mcp（:9100）实时查询指标佐证
             ├─ 按 §4 决策树推理根因
             └─ 输出结构化根因判定 + 处置建议（schema 见 §2.3）

推理调用链（业务面）：
Dify ──→ LiteLLM Proxy（litellm.magedu.com:4000，模型名 qwen3 / qwen3-embedding）
        ──→ vLLM LLM（vllm.magedu.com:8000）或 vLLM Embedding（vllm.magedu.com:8001）
```

**注意（自指依赖）**：Agent 节点自身依赖 `qwen3` 完成根因分析。当 LLM 推理链（vLLM 或 LiteLLM）整体故障时，自动诊断 Workflow 本身也会失效——此时以 AlertManager 直接通知人的备用渠道兜底，恢复顺序应为 vLLM → LiteLLM → Dify Workflow 验证。

### 1.4 推理服务基线（摘要）

| 服务 | 地址 | LiteLLM 模型名 / served-model-name | GPU | 关键约束 |
|---|---|---|---|---|
| LLM（Qwen3.5-4B/9B/27B-FP8） | vllm.magedu.com:8000 | `qwen3` | 默认 GPU 0 | 工具调用必需；`enable_thinking:false`；`max-model-len 262144`；`max-num-seqs 8` |
| Embedding（Qwen3-Embedding-4B） | vllm.magedu.com:8001 | `qwen3-embedding` | `CUDA_VISIBLE_DEVICES=1` | `--enforce-eager`；`--trust-remote-code`；`max-model-len 32768` |

两服务均要求启动前已执行 `flashinfer download-kernels --cuda-version 13.0`，且环境变量 `VLLM_USE_DEEP_GEMM=0`。详见 §6。**Dify 侧只感知 LiteLLM（:4000）上的模型名 `qwen3` / `qwen3-embedding`**，LiteLLM 后端指向 vLLM 时沿用同名 served-model-name。

---

## 2. Dify Workflow 集成契约

> 本章是 RAG 语料与 Workflow 各节点之间的"接口文档"。Agent 节点收到的输入字段、应输出的字段，均以本章为准；知识库文档中的术语必须与字段名对齐，才能保证检索命中率。

### 2.1 AlertManager Webhook 原始载荷（起始节点输入）

```json
{
  "receiver": "dify-webhook",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "VLLMHighKVCacheUsage",
        "severity": "warning",
        "instance": "vllm.magedu.com:8000",
        "job": "vllm",
        "service": "llm"
      },
      "annotations": {
        "summary": "vLLM KV Cache 使用率过高",
        "description": "实例 vllm.magedu.com:8000 KV Cache 使用率 96%，持续 5 分钟",
        "runbook_hint": "5.4"
      },
      "startsAt": "2026-09-14T08:30:00Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus.magedu.com:9090/graph?..."
    }
  ],
  "groupLabels": {"alertname": "VLLMHighKVCacheUsage"},
  "externalURL": "http://alertmanager.magedu.com:9093"
}
```

### 2.2 代码执行节点规范化输出（Agent 节点输入）

| 字段 | 类型 | 来源/规则 | 示例 |
|---|---|---|---|
| `alert_name` | string | `labels.alertname` | `VLLMHighKVCacheUsage` |
| `severity` | enum | `labels.severity`，归一化为 `critical / warning / info` | `warning` |
| `layer` | enum | 按 §3 矩阵映射：`infra / gpu / vllm / litellm / dify` | `vllm` |
| `instance` | string | `labels.instance`（主机名遵循 §1.1 约定） | `vllm.magedu.com:8000` |
| `service` | enum | 端口映射：8000→`llm`，8001→`embedding`，4000→`litellm`，其余→`host` | `llm` |
| `summary` | string | `annotations.summary` | vLLM KV Cache 使用率过高 |
| `description` | string | `annotations.description` | （原文） |
| `runbook_hint` | string | `annotations.runbook_hint`，指向本 Runbook 章节号 | `5.4` |
| `started_at` | string | `startsAt` 转本地时区 | `2026-09-14 16:30:00 +08:00` |
| `status` | enum | `firing / resolved` | `firing` |

### 2.3 Agent 节点输出契约

Agent 必须输出如下 JSON（供下游通知/工单节点消费）：

```json
{
  "alert_name": "VLLMHighKVCacheUsage",
  "root_cause": "长上下文请求集中到达，max-num-seqs=8 下 KV Cache 被打满",
  "confidence": "high | medium | low",
  "evidence": ["引用的 Runbook 章节与检索到的文档片段"],
  "immediate_actions": ["通过 prometheus-mcp 观察 num_requests_waiting 是否 > 4", "必要时临时上调 max-num-seqs"],
  "escalation": "是否需人工介入（true/false）及理由"
}
```

**判定原则（写入 Agent 系统提示词）**：
- 优先按 §4 决策树逐层排除；**先排除下层（GPU/主机），再排除中间层（vLLM/LiteLLM），最后归因应用层（Dify）**；
- 可调用 prometheus-mcp（prometheus-mcp.magedu.com:9100）实时查询指标验证假设；
- 每条根因结论必须引用本知识库中的至少一个文档片段作为证据；
- `confidence` 为 low 时必须 `escalation=true`，禁止臆断。

---

## 3. 告警目录与分诊矩阵

> 本矩阵是 Agent 的"告警语义字典"。`runbook_ref` 指向 §5 的处置条目。部署新告警规则时，**必须同步在本表登记并在 annotations 中写 `runbook_hint`**。

| alertname | 层级 | 严重度 | 含义 | 可能根因（按概率） | runbook_ref |
|---|---|---|---|---|---|
| `VLLMServiceDown` | vllm | critical | vllm.magedu.com:8000 健康检查失败或进程消失 | ① 进程崩溃（OOM/CUDA error）② 掉卡 ③ 宿主机故障 | 5.1 |
| `VLLMEmbeddingDown` | vllm | critical | vllm.magedu.com:8001 不可用，RAG 全链路中断 | 同 `VLLMServiceDown` | 5.1 |
| `VLLMHighKVCacheUsage` | vllm | warning | `gpu_cache_usage_perc` > 0.95 持续 5min | ① 长上下文请求集中 ② 并发打满 ③ 显存碎片化 | 5.4 |
| `VLLMRequestQueueBuildup` | vllm | warning | `num_requests_waiting` > 4 持续 5min | ① 突发流量 ② 单请求过长占用槽位 ③ `max-num-seqs` 过小 | 5.5 |
| `VLLMHighTTFT` | vllm | warning | TTFT P99 > 5s | ① 队列堆积 ② prefix cache 命中率下降 ③ GPU 降频 | 5.6 |
| `LiteLLMProxyDown` | litellm | critical | litellm.magedu.com:4000 健康检查失败 | ① Proxy 进程崩溃 ② 配置错误导致启动失败 ③ 宿主机/网络故障 | 5.11 |
| `LiteLLMUpstreamError` | litellm | warning | LiteLLM 对上游 vLLM 调用 5xx/超时率升高 | ① vLLM 端故障（转 §5.1）② LiteLLM 超时配置过短 ③ 网关与 vLLM 主机间网络 | 5.12 |
| `LiteLLMHighLatency` | litellm | warning | 经网关的请求端到端延迟显著高于 vLLM 直连 | ① 网关资源不足（CPU/连接数）② 网关队列/限流生效 ③ 上游 TTFT 劣化（转 §5.6） | 5.12 |
| `GPUMemoryPressure` | gpu | warning | 显存剩余 < 10%（排除 vLLM 配额后） | ① 其他进程挤占 ② vLLM 显存泄漏 ③ 多服务同卡 | 5.2 |
| `GPUTemperatureHigh` | gpu | warning | 温度 > 85°C 持续 5min | ① 机箱风道/积灰 ② 风扇故障 ③ 持续满载 | 5.7 |
| `GPUXidError` / `GPULost` | gpu | critical | 内核日志 Xid 错误或 `nvidia-smi` 无输出 | ① 掉卡（硬件/供电/驱动）② 驱动崩溃 | 5.3 |
| `HostMemoryHigh` | infra | warning | 内存使用率 > 90% | ① 页缓存外进程泄漏 ② Dify 组件（Celery）堆积 | 5.8 |
| `HostDiskFull` | infra | critical | 磁盘 > 90% | ① 模型权重/日志膨胀 ② Docker 镜像堆积 | 5.9 |
| `DifyWorkflowFailure` | dify | warning | Workflow 执行失败率升高 | ① LiteLLM 端点不可达（先查网关）② Webhook 载荷异常 ③ 代码节点报错 | 5.10 |
| `PrometheusTargetDown` | infra | warning | 某 exporter 抓取失败 | ① 被监控进程挂了 ② 端口/防火墙变更 | 5.1/5.8 |

---

## 4. 根因判定决策树

> Agent 收到规范化告警后按此树推理。每一层"先排除更底层"——GPU 掉了，vLLM、LiteLLM、Dify 的所有指标告警都只是表象。

```
告警到达
  │
  ├─ status == resolved？ → 记录闭环，输出"已自愈"，结束
  │
  ├─ 第一步：定位层级（layer 字段），自底向上排除
  │     ├─ gpu 层告警 → 先查 §5.3（掉卡）与 §5.7（温度）；GPU 层问题可解释其上方全部告警
  │     ├─ infra 层告警 → §5.8 / §5.9；宿主机资源耗尽可解释上方全部告警
  │     ├─ vllm 层告警 → 进入第二步
  │     ├─ litellm 层告警 → 先查上游 vLLM 健康（curl vllm.magedu.com:8000/8001 /health），
  │     │     上游健康再查网关自身（§5.11/§5.12）
  │     └─ dify 层告警 → 按调用链自近及远：Dify → LiteLLM（:4000 /health/liveliness）
  │           → vLLM（:8000/:8001 /health），逐段 curl 定位断点（§5.10）
  │
  ├─ 第二步（vllm 层）：服务存活？
  │     ├─ 否 → §5.1 进程消失处置链
  │     └─ 是 → 区分容量问题还是质量问题：
  │           ├─ 队列/Cache 类（waiting↑、cache_usage↑） → §5.4 / §5.5（容量）
  │           └─ 延迟类（TTFT↑、TPOT↑） → §5.6（性能，注意回溯 GPU 降频）
  │
  └─ 第三步：时间关联
        ├─ 告警前是否有变更（升级 vLLM / 换模型 / 改参数 / 改 LiteLLM 配置 / Dify 发版）？
        │     → 是：优先怀疑变更引入，引用 §6.4 / §7.3 回滚流程
        └─ 是否多告警同时触发？
              → 是：按"下层级告警是上层级告警的根因"原则归因到最低层
```

---

## 5. 告警级 Runbook（按告警逐条处置）

### 5.1 `VLLMServiceDown` / `VLLMEmbeddingDown`：推理服务不可用

- **影响面**：LLM 中断 → LiteLLM 上游报错 → Dify Agent 与自动诊断 Workflow 全部失败；Embedding 中断 → RAG 召回失败。
- **根因候选**：① vLLM 进程崩溃（启动 OOM、CUDA error、flashinfer 算子缺失）；② GPU 掉卡（伴随 `GPULost` 告警）；③ vllm.magedu.com 宿主机故障（伴随 `PrometheusTargetDown`）。
- **鉴别命令**（在 vllm.magedu.com 上执行）：

```bash
pgrep -af "vllm serve"                 # 进程是否存活
curl -s -m 5 http://localhost:8000/health; echo
curl -s -m 5 http://localhost:8001/health; echo
nvidia-smi                             # 掉卡时该命令报错或卡数不对
dmesg | tail -50 | grep -i xid         # 内核层 GPU 错误
# 查看服务日志末尾 100 行（tmux 会话或日志文件），重点搜 ERROR / OOM / CUDA error
```

- **处置**：
  1. 进程崩溃 → 按 §6.2/§6.3 以标准模板重启；若日志提示 flashinfer kernel 缺失 → 执行 `flashinfer download-kernels --cuda-version 13.0` 后重启（见 §6.1）。
  2. 伴随掉卡 → 转 §5.3，GPU 恢复前不要在故障卡上重启服务。
  3. 宿主机故障 → 转 §5.8/§5.9。
- **恢复验证**：`/health` 返回 200 + §6.5 冒烟通过 + 经 LiteLLM（:4000）的端到端调用通过（§7.4）。

### 5.2 `GPUMemoryPressure`：显存压力

- **根因候选**：① 其他进程挤占（常见于训练任务/残留 python 进程）；② LLM 与 Embedding 被配到同一张卡（`CUDA_VISIBLE_DEVICES` 设置错误）；③ vLLM 运行期显存缓慢增长（罕见泄漏）。
- **鉴别命令**：`nvidia-smi` 查看各进程显存占用；比对 vLLM 配额（24GB × 0.90 ≈ 21.6GB/卡）。
- **处置**：清退无关进程；确认 LLM 在 GPU 0、Embedding 在 GPU 1 的隔离；确认泄漏则滚动重启对应服务并记录案例（§10）。
- **注意**：vLLM 按 `gpu-memory-utilization` 预占显存是**设计行为**，进程级显存高不等于异常，须结合 `gpu_cache_usage_perc` 判断（§5.4）。

### 5.3 `GPUXidError` / `GPULost`：掉卡

- **根因候选**：① 供电不足/电源线松动（消费级卡高负载常见）；② 散热失效过热保护；③ 驱动崩溃；④ 硬件故障。
- **鉴别命令**：

```bash
nvidia-smi                             # 是否少卡/报错
dmesg | grep -i "xid\|nvrm" | tail -20 # Xid 编号对照 NVIDIA 文档
nvidia-smi -q | grep -iE "power|temp"  # 掉卡前的功耗/温度线索
```

- **处置**：先迁移服务（停掉故障卡上的 vLLM，必要时单机单服务降级运行，并同步在 LiteLLM 中摘除该上游）；软件层面 `sudo nvidia-smi --gpu-reset` 或重启宿主机；反复掉卡 → 检查 PSU 功率余量（4090 瞬时功耗高，建议 1000W+ 优质电源）与 PCIe 供电线插接；仍复发判定硬件故障走维保。
- **事后**：必须写故障案例（§10），Xid 编号是检索关键词。

### 5.4 `VLLMHighKVCacheUsage`：KV Cache 使用率过高

- **含义**：`vllm:gpu_cache_usage_perc > 0.95`。KV Cache 接近耗尽时新请求将排队，是 `VLLMRequestQueueBuildup` 的前兆。
- **根因候选**：① 长上下文请求集中（`max-model-len 262144` 下单请求可占大量 Cache）；② 并发达到 `max-num-seqs 8` 上限；③ 异常客户端（含 LiteLLM 重试风暴）发送超长 prompt。
- **鉴别命令**：

```bash
curl -s http://vllm.magedu.com:8000/metrics | grep -E "num_requests_(running|waiting)|gpu_cache_usage|prompt_tokens"
# 同步查看 LiteLLM 侧是否有大量重试（网关日志或 §7.2 指标）
```

- **处置**：短期——无须立即动作，观察队列是否消化；持续告警则临时重启清空 Cache（有损，低峰执行）。中期——显存有余量时上调 `--max-num-seqs`；或在 LiteLLM/Dify 侧限制最大上下文与并发。
- **关联知识**：prefix caching 已开启（`--enable-prefix-caching`），多轮对话场景可显著缓解本告警，若命中率骤降见 §5.6。

### 5.5 `VLLMRequestQueueBuildup`：请求队列堆积

- **含义**：`vllm:num_requests_waiting > 4` 持续 5 分钟，说明到达速率超过服务能力。
- **根因候选**：① 突发流量（批量知识库重建打到 Embedding、定时任务集中触发 Agent）；② 少量超长请求长时间占用并发槽位；③ `max-num-seqs` 对当前负载过于保守；④ LiteLLM 重试放大流量（一次上游超时触发多次重试）。
- **处置**：确认流量来源（LiteLLM 请求日志按 API Key/应用维度统计）；能削峰的削峰（错峰执行批量任务）；确认显存余量（`gpu_cache_usage_perc` 长期 < 0.7）后上调 `max-num-seqs`（LLM 8→12 试）；Embedding 并发上限 16，批量重建索引时在 Dify 侧限速；如为重试风暴，调整 LiteLLM 重试次数与超时（§7.3）。

### 5.6 `VLLMHighTTFT`：首 Token 延迟过高

- **根因候选**：① 队列堆积（先按 §5.5 排除）；② prefix cache 命中率下降（对话前缀频繁变化、服务重启后冷启动）；③ GPU 降频（温度/功耗墙，回溯 §5.7）。
- **鉴别命令**：

```bash
curl -s http://vllm.magedu.com:8000/metrics | grep -E "time_to_first_token|prefix_cache|time_per_output_token"
nvidia-smi -q -d CLOCK | grep -A4 "Clocks Event Reasons"   # 降频原因
```

- **处置**：队列问题治队列；冷启动导致的一过性升高观察即可；降频按 §5.7 处理。**区分网关开销与引擎开销**：对比 LiteLLM 端到端延迟与 vLLM `/metrics` 中的 TTFT，差值大则问题在网关（转 §5.12）。

### 5.7 `GPUTemperatureHigh`：GPU 过热

- **根因候选**：① 机箱风道不良/积灰（消费级卡开放散热在多卡机箱中易积热）；② 风扇故障；③ 持续满载且环境温度过高。
- **处置**：`nvidia-smi -q -d TEMPERATURE,POWER` 确认各卡温度与风扇转速；清灰、调整风道、必要时降 `gpu-memory-utilization` 配合限功耗（`nvidia-smi -pl`）临时降压；持续 > 90°C 触发降频会连带引发 `VLLMHighTTFT`。

### 5.8 `HostMemoryHigh`：宿主机内存高

- **根因候选**：① Dify 组件内存增长（Celery worker 任务堆积常见于知识库批量重建）；② LiteLLM 进程内存增长（长连接/大日志缓冲）；③ 其他进程泄漏。
- **鉴别**：`free -g`、`ps aux --sort=-%mem | head`、`docker stats`；按 instance 字段确认是哪台主机（vllm / litellm / dify）。
- **处置**：定位到具体主机与组件后重启对应服务并排查任务队列；系统性不足则加内存或拆分部署。

### 5.9 `HostDiskFull`：磁盘将满

- **根因候选**：① 模型权重文件大（多版本模型并存于 vllm.magedu.com，27B-FP8 约 30GB）；② vLLM/Dify/LiteLLM 日志膨胀；③ Docker 镜像与构建缓存堆积。
- **处置**：`df -h` + `du -sh /models/* /var/lib/docker/* | sort -h`；清理旧版本模型、`docker system prune`、配置日志轮转（logrotate / Docker json-file max-size）。

### 5.10 `DifyWorkflowFailure`：Dify Workflow 失败率升高

- **根因候选**（按调用链自近及远排查）：① LiteLLM 端点不可达或鉴权失败（litellm.magedu.com:4000，最常见）；② vLLM 端点不可达（经网关表现为 5xx）；③ Webhook 载荷结构变化导致代码执行节点解析失败；④ 工具调用失效（vLLM 侧 `--enable-auto-tool-choice` / `--tool-call-parser qwen3_xml` 缺失）；⑤ 上下文超限。
- **鉴别命令（逐段定位断点）**：

```bash
curl -s -m 5 http://litellm.magedu.com:4000/health/liveliness   # 网关存活
curl -s -m 5 http://vllm.magedu.com:8000/health                 # LLM 引擎
curl -s -m 5 http://vllm.magedu.com:8001/health                 # Embedding 引擎
# 经网关端到端调用（模型名 qwen3）
curl -s -m 30 http://litellm.magedu.com:4000/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <LITELLM_KEY>" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"ping"}],"max_tokens":8}'
```

- **处置**：断点在网关 → §5.11/§5.12；断点在 vLLM → §5.1；载荷问题修代码节点解析逻辑（对照 §2.2 schema）；工具调用与上下文问题 → §6.6。

### 5.11 `LiteLLMProxyDown`：推理网关不可用

- **影响面**：**全局单点**。Dify 所有模型调用（Agent、RAG 嵌入、自动诊断 Workflow 自身）全部中断，但 vLLM 引擎可能完全健康。
- **根因候选**：① LiteLLM 进程崩溃或 OOM；② 配置文件错误（模型名/上游地址/鉴权）导致启动失败；③ litellm.magedu.com 宿主机或网络故障；④ 网关依赖的数据库/Redis（如启用虚拟 Key 管理）不可达。
- **鉴别命令**：

```bash
curl -s -m 5 http://litellm.magedu.com:4000/health/liveliness
systemctl status litellm 2>/dev/null || docker ps | grep litellm   # 按实际部署方式
# 查看网关日志末尾，重点搜 config / prisma / redis / upstream 报错
curl -s -m 5 http://vllm.magedu.com:8000/health   # 确认上游健康——上游健康则问题纯在网关层
```

- **处置**：配置错误 → 回滚最近配置变更并重启（§7.3）；依赖组件故障 → 恢复依赖后重启网关；宿主机故障 → 转 §5.8/§5.9。
- **恢复验证**：§7.4 端到端冒烟（经网关分别调 `qwen3` 与 `qwen3-embedding`）。

### 5.12 `LiteLLMUpstreamError` / `LiteLLMHighLatency`：网关上游错误/高延迟

- **根因候选**：① 上游 vLLM 故障或饱和（先转 §5.1/§5.5 排除）；② LiteLLM 超时配置短于 vLLM 长上下文生成耗时（256K 上下文下单请求可能分钟级）；③ 网关重试策略放大故障；④ 网关自身 CPU/连接数瓶颈。
- **鉴别**：对比三段延迟——Dify 观测的端到端延迟、LiteLLM 日志中的上游响应耗时、vLLM `/metrics` 的 TTFT/TPOT。哪两段之差大，问题就在哪段链路上。
- **处置**：超时配置按 `max-model-len 262144` 的最坏生成时间调大；重试次数收敛（建议 ≤ 2 且仅对连接错误重试，避免对生成中超时重试）；网关瓶颈则扩容或调优 worker 数。

---

## 6. 服务层 Runbook：vLLM 部署与运维

> 部署于独立主机 **vllm.magedu.com**。本章供告警处置中"重启/变更/参数调整"引用。

### 6.1 前置条件（每次新环境或升级 vLLM 后必做）

```bash
# flashinfer 算子内核默认不随包分发，必须预下载（--cuda-version 与 vLLM 的 CUDA 后端一致）
flashinfer download-kernels --cuda-version 13.0
python -c "import torch; print(torch.version.cuda)"   # 不确定版本时先确认
```

不执行会导致启动时 nvcc 现场编译（极慢且易失败），表现为 `VLLMServiceDown` 告警后服务长时间无法拉起。

### 6.2 LLM 服务标准启动模板

```bash
export VLLM_USE_DEEP_GEMM=0

vllm serve ./Qwen3.5-4B \
    --host 0.0.0.0 --port 8000 \
    --served-model-name qwen3 \
    --tensor-parallel-size 1 --dtype auto \
    --max-model-len 262144 --max-num-seqs 8 \
    --gpu-memory-utilization 0.90 \
    --enable-prefix-caching \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice --tool-call-parser qwen3_xml \
    --chat-template-kwargs '{"enable_thinking":false}'
```

切换模型（Qwen3.5-9B / Qwen3.8-27B-FP8）仅需改模型路径；27B-FP8 在 24GB 卡上若 OOM，按序下调 `max-model-len`（→65536）、`max-num-seqs`（→4）、`gpu-memory-utilization`（→0.85）。**模型名对 LiteLLM 与 Dify 透明，均为 `qwen3`**。

**Dify Agent 依赖的三个关键点**：`--enable-auto-tool-choice`、`--tool-call-parser qwen3_xml`、`enable_thinking:false`（思考输出会污染工具调用解析，且挤占上下文）。

### 6.3 Embedding 服务标准启动模板

```bash
export VLLM_USE_DEEP_GEMM=0 CUDA_VISIBLE_DEVICES=1

vllm serve ./Qwen3-Embedding-4B \
  --dtype float16 \
  --max-model-len 32768 --max-num-seqs 16 \
  --gpu-memory-utilization 0.90 --tensor-parallel-size 1 \
  --served-model-name qwen3-embedding \
  --port 8001 --trust-remote-code --enforce-eager
```

### 6.4 变更与回滚流程

1. 变更前存档：vLLM 版本、模型路径、完整启动命令、LiteLLM 中对应上游配置。
2. 停旧 → 更新 → **vLLM 升级后重新下载 flashinfer 内核** → 启动。
3. 执行 §6.5 冒烟 → §7.4 经网关端到端验证 → Dify 侧抽测 Agent 与 RAG。
4. 任一步失败：按存档命令回滚。

### 6.5 冒烟验证脚本（vllm.magedu.com 本机直连）

```bash
# 健康与模型列表
curl -s http://localhost:8000/health && curl -s http://localhost:8000/v1/models
curl -s http://localhost:8001/health

# LLM 对话（响应不应包含 thinking 标签）
curl -s http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"一句话介绍你自己"}],"max_tokens":128}'

# 工具调用（tool_calls 必须非空，arguments 为合法 JSON）
curl -s http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"北京今天天气"}],
       "tools":[{"type":"function","function":{"name":"get_weather","description":"查天气",
       "parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}],
       "tool_choice":"auto"}'

# Embedding（返回 200 且向量维度正常、无 NaN）
curl -s http://localhost:8001/v1/embeddings -H "Content-Type: application/json" \
  -d '{"model":"qwen3-embedding","input":"vLLM 是高吞吐推理引擎"}'
```

### 6.6 关键监控指标（告警规则的数据源）

| 指标 | 地址 | 告警规则 | 关联条目 |
|---|---|---|---|
| `vllm:gpu_cache_usage_perc` | vllm.magedu.com:8000/metrics | > 0.95 持续 5min → `VLLMHighKVCacheUsage` | §5.4 |
| `vllm:num_requests_waiting` | :8000/:8001/metrics | > 4 持续 5min → `VLLMRequestQueueBuildup` | §5.5 |
| `vllm:time_to_first_token_seconds` P99 | :8000/metrics | > 5s → `VLLMHighTTFT` | §5.6 |
| `up{job="vllm"}` | prometheus.magedu.com | == 0 → `VLLMServiceDown` / `VLLMEmbeddingDown` | §5.1 |

---

## 7. 推理网关层 Runbook：LiteLLM Proxy

### 7.1 定位与职责

LiteLLM Proxy（litellm.magedu.com:4000）是 Dify 与 vLLM 之间的统一推理网关：对 Dify 暴露 OpenAI 兼容接口与统一模型名（`qwen3`、`qwen3-embedding`），承担虚拟 Key 鉴权、路由、限流、重试、用量统计。**Dify 侧不直接配置 vLLM 地址**，所有模型调用均指向网关。

### 7.2 参考配置（config.yaml 要点）

```yaml
model_list:
  - model_name: qwen3
    litellm_params:
      model: openai/qwen3                     # vLLM 的 OpenAI 兼容模式
      api_base: http://vllm.magedu.com:8000/v1
      api_key: EMPTY
    model_info:
      mode: chat
      supports_function_calling: true         # Dify Agent 依赖，必须声明
  - model_name: qwen3-embedding
    litellm_params:
      model: openai/qwen3-embedding
      api_base: http://vllm.magedu.com:8001/v1
      api_key: EMPTY
    model_info:
      mode: embedding

litellm_settings:
  request_timeout: 600        # 需覆盖 256K 上下文的最坏生成耗时，过短会引发 LiteLLMUpstreamError
  num_retries: 2              # 仅对连接级错误重试；生成中超时不应重试（防重试风暴）
general_settings:
  master_key: sk-xxxx         # 虚拟 Key 管理（Dify 使用独立 virtual key，便于按来源统计与限流）
```

### 7.3 变更与回滚

1. 变更前备份 config.yaml 与虚拟 Key 清单。
2. 修改后 `litellm --config config.yaml` 重启（或对应 systemd/docker 方式），先看日志确认两个 model 加载成功。
3. §7.4 端到端冒烟；失败则恢复备份配置并重启。
4. **注意**：模型名 `qwen3` / `qwen3-embedding` 是对 Dify 的契约，改名必须同步 Dify 配置，否则触发 `DifyWorkflowFailure`。

### 7.4 端到端冒烟（经网关，模拟 Dify 调用路径）

```bash
GW=http://litellm.magedu.com:4000
KEY=<Dify使用的LITELLM虚拟KEY>

curl -s -m 5 $GW/health/liveliness                              # 网关存活
curl -s $GW/v1/models -H "Authorization: Bearer $KEY"           # 应列出 qwen3 与 qwen3-embedding

# 经网关的 LLM 对话
curl -s -m 60 $GW/v1/chat/completions -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"ping"}],"max_tokens":16}'

# 经网关的工具调用（tool_calls 必须非空）
curl -s -m 60 $GW/v1/chat/completions -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"北京今天天气"}],
       "tools":[{"type":"function","function":{"name":"get_weather","description":"查天气",
       "parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}],
       "tool_choice":"auto"}'

# 经网关的 Embedding
curl -s -m 30 $GW/v1/embeddings -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-embedding","input":"litellm 网关冒烟测试"}'
```

**工具调用透传是网关层特有的验证点**：确认经 LiteLLM 转发后 `tool_calls` 结构未被改写；若直连 vLLM 正常而经网关异常，检查 LiteLLM 版本对 `qwen3_xml` 解析结果的兼容性。

### 7.5 监控指标

| 指标/信号 | 来源 | 告警规则 | 关联条目 |
|---|---|---|---|
| 健康检查 | litellm.magedu.com:4000/health/liveliness | 失败 → `LiteLLMProxyDown` | §5.11 |
| 上游 5xx/超时率 | LiteLLM `/metrics` 或日志 | 升高 → `LiteLLMUpstreamError` | §5.12 |
| 端到端延迟 vs vLLM TTFT | LiteLLM 日志 + vLLM `/metrics` | 差值持续过大 → `LiteLLMHighLatency` | §5.12 |

---

## 8. 宿主机与 GPU 层 Runbook

| 检查项 | 命令 | 基线 |
|---|---|---|
| GPU 总览（vllm.magedu.com） | `nvidia-smi` | 卡数正确、无 ERR、温度 < 85°C |
| 掉卡/Xid | `dmesg \| grep -i xid` | 无输出 |
| 降频原因 | `nvidia-smi -q -d CLOCK` | 无 Thermal/Power 降频标志 |
| 内存 | `free -g` | 可用 > 10% |
| 磁盘 | `df -h` | 各分区 < 90% |
| Docker 占用 | `docker system df` | 定期 prune |

处置细节分别见 §5.2、§5.3、§5.7、§5.8、§5.9。消费级 GPU 特别注意：4090/4090D 无 NVLink，`tensor-parallel-size` 恒为 1，任何多卡并行告警均属配置错误；电源余量不足是高负载掉卡的首因之一。

---

## 9. Dify 平台层 Runbook

1. **模型端点配置（经 LiteLLM）**：Dify（dify.magedu.com:80）中以 OpenAI-API-compatible 方式接入，API Endpoint 统一为 `http://litellm.magedu.com:4000/v1`，API Key 填 LiteLLM 签发的虚拟 Key；模型名分别为 `qwen3`（LLM，勾选 Function Calling，上下文 262144）与 `qwen3-embedding`（Text Embedding，上下文 32768）。**Dify 不直接配置 vllm.magedu.com 的地址**。
2. **切换 Embedding 模型后必须重建知识库索引**，否则向量维度不匹配报错（`DifyWorkflowFailure` 的常见根因之一）。
3. **Workflow 自检顺序（按调用链自近及远）**：LiteLLM 健康（:4000）→ vLLM 健康（:8000/:8001）→ Webhook 载荷结构 → 代码执行节点输出是否符合 §2.2 → Agent 节点 LLM 配置（模型名、工具调用）。
4. **RAG 知识库维护**：本 Runbook 更新后需重新向量化；新增告警规则时同步更新 §3 矩阵并重建索引。
5. **自动诊断的自指依赖**：诊断 Workflow 自身走 LiteLLM→vLLM 链路，该链路整体故障时自动诊断不可用，需 AlertManager 直接通知人作为兜底渠道。

---

## 10. 历史故障案例库（模板与示例）

> 案例库是 RCA 类 RAG 检索命中率最高的语料。每次故障处置后按模板补录，告警 `alertname` 与关键词（Xid 编号、报错原文、主机名）务必原样保留。

### 10.1 案例模板

```
【案例编号】INC-YYYYMMDD-NN
【关联告警】alertname（与 §3 矩阵一致）
【现象】监控表现 + 用户侧表现 + 关键报错原文
【定位过程】执行的命令、各步排除了什么
【根因】一句话根因 + 分类（容量/硬件/配置/变更/外部依赖）
【处置】实际执行的动作
【预防】规则、流程或文档的改进
```

### 10.2 示例案例

**INC-20260901-01｜关联告警：VLLMServiceDown**
- 现象：vllm.magedu.com 上 vLLM 0.28.1 → 0.29.0 升级后服务启动卡在初始化 40 分钟，健康检查超时，`VLLMServiceDown` 触发；同时 LiteLLM 报 `LiteLLMUpstreamError`。
- 定位：日志显示 flashinfer 正在调用 nvcc 现场编译内核；`python -c "import torch; print(torch.version.cuda)"` 确认新后端为 CUDA 13.0，而预下载的内核是 12.x 版本。
- 根因：升级后未按新 CUDA 版本重新下载 flashinfer 预编译内核（配置/变更类）；网关告警为连带表象。
- 处置：中断启动，执行 `flashinfer download-kernels --cuda-version 13.0`，重启后 90 秒内就绪。
- 预防：将"重下内核"写入 §6.4 变更流程强制步骤。

**INC-20260905-02｜关联告警：GPULost + VLLMServiceDown（同时触发）**
- 现象：凌晨高负载时段 vllm.magedu.com:8000 服务崩溃，随后 `nvidia-smi` 只剩 1 张卡，dmesg 出现 Xid 79（GPU has fallen off the bus）。
- 定位：按 §4 决策树归因到 GPU 层；检查电源发现 4090 的 12VHPWR 转接线未完全插紧。
- 根因：高负载下供电接触不良导致掉卡（硬件类）；vLLM 崩溃为连带表象。
- 处置：重插供电线并更换原生线材，重启宿主机，按 §6.2 拉起服务，§7.4 验证网关链路。
- 预防：巡检增加供电接口检查项；电源升级为 1200W。

**INC-20260910-03｜关联告警：LiteLLMProxyDown**
- 现象：Dify 全部 Agent 对话失败，自动诊断 Workflow 自身也无法执行；但 vllm.magedu.com:8000/8001 健康检查正常。
- 定位：§5.10 逐段排查——vLLM 健康，LiteLLM `/health/liveliness` 无响应；网关日志末尾报 config 解析错误，定位到当天上午为新增模型修改 config.yaml 时 YAML 缩进错误。
- 根因：LiteLLM 配置变更未做语法校验即重启，进程启动失败（变更类）。
- 处置：回滚 config.yaml 备份，重启网关，§7.4 冒烟通过。
- 预防：§7.3 变更流程增加"配置语法校验 + 备份"强制步骤；为 `LiteLLMProxyDown` 配置 AlertManager 直接通知人的兜底渠道（自动诊断 Workflow 在网关故障时不可用）。

---

## 11. 附录：主机、端口与参数速查表

### 11.1 主机与端口

| 主机 | 地址 | 用途 |
|---|---|---|
| Prometheus | prometheus.magedu.com:9090 | 指标与告警规则 |
| AlertManager | alertmanager.magedu.com:9093 | 告警路由，Webhook → Dify |
| Grafana | grafana.magedu.com:3000 | 可视化 |
| vLLM LLM | vllm.magedu.com:8000（`qwen3`） | 推理引擎（独立主机，GPU 0） |
| vLLM Embedding | vllm.magedu.com:8001（`qwen3-embedding`） | 嵌入引擎（GPU 1） |
| LiteLLM Proxy | litellm.magedu.com:4000 | 推理网关，Dify 唯一入口 |
| Dify | dify.magedu.com:80 | Agent / RAG / Workflow |
| prometheus-mcp | prometheus-mcp.magedu.com:9100 | Agent 实时查指标 |

### 11.2 关键约定

| 项目 | 约定 |
|---|---|
| 模型名契约 | Dify 与 LiteLLM 均使用 `qwen3` / `qwen3-embedding`，与 vLLM `--served-model-name` 一致；改任何一环必须三环同步 |
| 调用链 | Dify → LiteLLM（:4000）→ vLLM（:8000/:8001）；排障按此链自近及远逐段 curl |
| GPU 分配 | GPU 0 → LLM；GPU 1 → Embedding（`CUDA_VISIBLE_DEVICES=1`） |
| 必需环境变量 | `VLLM_USE_DEEP_GEMM=0`（两服务均需要） |
| 前置依赖 | `flashinfer download-kernels --cuda-version 13.0`（新环境/升级后必做） |
| 显存紧张降级顺序 | `gpu-memory-utilization`↓ → `max-num-seqs`↓ → `max-model-len`↓ → `--enforce-eager` |
| 自指依赖兜底 | LiteLLM/vLLM 整体故障时自动诊断 Workflow 不可用，AlertManager 须有直接通知人的备用渠道 |

---

> **变更记录**
>
> | 版本 | 日期 | 内容 |
> |---|---|---|
> | v1.0 | 2026-09-14 | 首版：vLLM 服务自身的部署与运维 Runbook |
> | v2.0 | 2026-09-11 | 面向 AlertManager → Dify Workflow → Agent 自动根因分析场景重构：新增系统全景、Workflow 集成契约、告警目录与分诊矩阵、根因判定决策树、告警级 Runbook、案例库 |
> | v2.1 | 2026-09-10 | 新增 LiteLLM Proxy 推理网关层：全量主机命名约定（§1.1）、五层监控模型、调用链改为 Dify→LiteLLM→vLLM；告警矩阵新增 `LiteLLMProxyDown` / `LiteLLMUpstreamError` / `LiteLLMHighLatency`（§3）；决策树插入网关层排除步骤（§4）；新增告警条目 §5.11/§5.12 与网关章节 §7（配置、变更、端到端冒烟、指标）；Dify 端点改指 litellm.magedu.com:4000（§9）；新增网关配置故障案例；补充"自动诊断自指依赖"风险与兜底说明 |
