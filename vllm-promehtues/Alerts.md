# vLLM 告警规则

为 vLLM 推理服务设置告警规则，核心是关注服务可用性、推理性能、资源利用率和请求队列这四个维度。下面是一套可以直接在 Prometheus 中使用的告警规则示例，在使用时可以根据实际的 SLO 目标调整阈值。

###  Prometheus 告警规则示例

将以下内容保存为 vllm-alerts.yml，并在 prometheus.yml 中通过 rule_files 加载。

```yaml
groups:
  - name: vllm_alerts
    rules:
      # ---------- 服务可用性 ----------
      - alert: vLLMServiceDown
        expr: up{job="vllm"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "vLLM 服务实例 {{ $labels.instance }} 已宕机"
          description: "vLLM 服务已超过1分钟无法访问。"

      - alert: vLLMMetricsMissing
        expr: absent(vllm:num_requests_running)
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "vLLM 指标缺失"
          description: "已连续5分钟未采集到 vLLM 的运行中请求指标，服务可能已停止上报。"

      # ---------- 请求延迟 ----------
      - alert: HighVLLMTimeToFirstToken
        expr: histogram_quantile(0.99, rate(vllm:time_to_first_token_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "vLLM 首字延迟（TTFT）P99 过高"
          description: "过去5分钟内，P99 首字延迟持续超过2秒。当前值: {{ $value | printf \"%.2f\" }}s"

      - alert: HighVLLME2ELatency
        expr: histogram_quantile(0.95, rate(vllm:e2e_request_latency_seconds_bucket[5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "vLLM 端到端请求延迟过高"
          description: "过去5分钟内，95%的请求端到端延迟超过10秒。当前值: {{ $value | printf \"%.2f\" }}s"

      # ---------- 请求错误率 ----------
      - alert: HighVLLMErrorRate
        expr: sum(rate(vllm:request_failure_counter[5m])) / sum(rate(vllm:request_counter[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "vLLM 请求错误率过高"
          description: "过去5分钟内，请求错误率超过5%。当前错误率: {{ $value | printf \"%.2f\" }}%"

      # ---------- 资源利用率 ----------
      - alert: HighVLLMGPUMemoryUsage
        expr: vllm:gpu_memory_usage_bytes / vllm:gpu_memory_total_bytes > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "vLLM GPU 显存使用率过高"
          description: "GPU 显存使用率持续5分钟超过90%，存在 OOM 风险。当前使用率: {{ $value | printf \"%.2f\" }}%"

      - alert: HighVLLMKVCacheUsage
        expr: vllm:kv_cache_usage_perc > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "vLLM KV Cache 使用率过高"
          description: "KV Cache 使用率持续5分钟超过90%，可能引发请求抢占。当前使用率: {{ $value | printf \"%.2f\" }}%"

      # ---------- 请求队列 ----------
      - alert: VLLMRequestQueueBacklog
        expr: vllm:num_requests_waiting > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "vLLM 请求队列积压"
          description: "等待处理的请求数持续超过10个。当前队列深度: {{ $value }}"
```



### 关键指标与阈值说明

下表汇总了上述规则中使用的关键指标、常用阈值及其业务含义，方便你根据实际场景调整：

| 告警名称                 | 关键指标                            | 建议阈值  | 业务含义                              |
| :----------------------- | :---------------------------------- | :-------- | :------------------------------------ |
| vLLMServiceDown          | `up{job="vllm"}`                    | `== 0`    | 服务进程不可达，实例宕机              |
| vLLMMetricsMissing       | `absent(vllm:num_requests_running)` | 持续5分钟 | 指标上报中断，可能引擎内部故障        |
| HighVLLMTimeToFirstToken | `vllm:time_to_first_token_seconds`  | P99 > 2s  | 首字延迟过高，用户可感知卡顿          |
| HighVLLME2ELatency       | `vllm:e2e_request_latency_seconds`  | P95 > 10s | 端到端延迟过高，影响整体体验          |
| HighVLLMErrorRate        | `vllm:request_failure_counter`      | > 5%      | 请求错误率过高，服务异常              |
| HighVLLMGPUMemoryUsage   | `vllm:gpu_memory_usage_bytes`       | > 90%     | 显存即将耗尽，有 OOM 崩溃风险         |
| HighVLLMKVCacheUsage     | `vllm:kv_cache_usage_perc`          | > 90%     | KV Cache 耗尽将触发请求抢占，影响吞吐 |
| VLLMRequestQueueBacklog  | `vllm:num_requests_waiting`         | > 10      | 排队请求过多，服务处理能力已达瓶颈    |

> 注意：vLLM 的指标名称可能因版本而异。例如，GPU 内存指标在部分版本中为 vllm:gpu_cache_usage_perc，错误率指标可能为 vllm:request_failure_counter 或基于 vllm:request_duration_seconds_count{status_code=~"5.."} 计算。请在 Prometheus 的 /graph 页面中确认实际指标名。
