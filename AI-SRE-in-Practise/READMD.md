# 基于Dify的AI-SRE（AgenticOps）闭环实践

> **文档性质**：动手实验教程
>
> **适用读者**：Linux 应用运维工程师、SRE、AI Infra / LLMOps 工程师、AI Agent 应用开发者
>
> **环境基准**：Dify 1.17.1、MCP Python SDK（当前官方版本）、MCP 协议版本 2025-06-18
>
> **文档作者**：马哥教育（http://www.magedu.com）
>
> **版权说明：**原创文档，转载必须经过作者同意



## 一、部署LLM推理模型

### 1. 前提

下载模型至模型存储目录中，例如 `/root/autodl-tmp/models/`。在数据盘上创建模型存储目录：

```bash
mkdir -pv /root/autodl-tmp/models/
```

创建专用于模型下载的虚拟python环境，并安装modelscope：

```bash
conda create -p /root/autodl-tmp/conda/download python=3.12
conda activate /root/autodl-tmp/conda/download
pip install --upgrade modelscop
```

下载LLM模型权重Qwen3.5-4B（其它模型类似）：

```bash
modelscope download --model Qwen/Qwen3.5-4B --local_dir /root/autodl-tmp/models/Qwen3.5-4B
```

下载嵌入模型权重Qwen3-Embedding：

```bash
modelscope download --model Qwen/Qwen3-Embedding-4B --local_dir /root/autodl-tmp/models/Qwen3-Embedding-4B
```

> **备注**：若要到HuggingFace上下载模型，则应该安装如下工具，并使用相应的命令下载。

```bash
pip install --upgrade "huggingface_hub[cli]"

hf download Qwen/Qwen3.5-4B --local-dir /root/autodl-tmp/models/Qwen3.5-4B
hf download Qwen/Qwen3-Embedding-4B --local-dir /root/autodl-tmp/models/Qwen3-Embedding-4B
```

### 2. 启动LLM模型推理服务

**第一步**：创建专用于的虚拟python环境：

```bash
conda create -p /root/autodl-tmp/conda/vllm29 python=3.12
conda activate /root/autodl-tmp/conda/vllm29
```

**第二步**：安装vllm，可以指定版本，本示例采用0.29.0版本。

```bash
pip install vllm==0.29.0
```

**第三步**：vLLM安装完成后，可以运行以下Python代码来快速验证是否安装成功以及相关的环境。尤其要关注其自动安装的CUDA Runtime的版本。

```bash
python - <<'PY'
import torch, vllm

print("PyTorch:", torch.__version__)
print("PyTorch CUDA Runtime:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
print("vLLM:", vllm.__version__)

if torch.cuda.is_available():
    for i in range(torch.cuda.device_count()):
        print(f"GPU {i}:", torch.cuda.get_device_name(i))
PY
```

**第四步**：手动安装编译好的缺失FlashInfer内核

FlashInfer 就是大模型推理的"底层引擎"，它已被几乎所有主流推理框架集成，包括vLLM、SGLang、MLC-Engine、TensorRT-LLM、TGI等。vLLM/SGLang负责请求调度、batching、KV Cache 管理策略、API 服务（上层框架），FlashInfer负责关键 GPU kernel 的执行效率（底层算子库）。FlashInfer 把 LLM 推理工作负载解构为四个核心算子家族，它们是Attention、GEMM、Communication、Sampling。

```bash
# 首先检查缺失的内核
flashinfer show-config

# 而后手动安装缺失的内核，注意其必须与此前pip安装时自动安装的CUDA Runtime版本一致
flashinfer download-kernels --cuda-version 13.0
```

如果前一种方式因网速等原因，代价太大，可暂时使用如下方式规避vllm 0.29.0的flashinfer的新要求：

```bash
export VLLM_USE_FLASHINFER_SAMPLER=0
```

**第五步**：启动模型推理服务。

```bash
export VLLM_USE_DEEP_GEMM=0
vllm serve ./Qwen3.5-4B \
    --host 0.0.0.0 \
    --port 6006 \
    --served-model-name qwen3 \
    --tensor-parallel-size 1 \
    --dtype auto \
    --max-model-len 262144 \
    --max-num-seqs 16 \
    --gpu-memory-utilization 0.90 \
    --enable-prefix-caching \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_xml \
    --default-chat-template-kwargs '{"enable_thinking":false}' \
    --speculative-config '{"method":"mtp","num_speculative_tokens":1}'
```

> **注意**：默认端口是8000，这里为了适配AutoDL主机远程访问需要，给改成了6006。

个别参数的作用说明：

- `--tensor-parallel-size 1`：张量并行度（TP），即把模型切分到多少张GPU上。
- `--dtype auto`：模型权重的计算精度。auto表示自动从模型config中读取（Qwen3.5系列通常为bfloat16）。这是推荐设置，可避免手动指定导致精度不匹配。可用的其它值有half(fp16)、bfloat16、float16、float32 等。
- `--max-model-len 262144`：模型支持的最大上下文长度（token数），262144=256K，即允许最长256K token的输入+输出。该值不能超过模型本身支持的上限，否则启动报错。
- `--max-num-seqs 16`：单次迭代中最多同时处理的序列数（即最大并发请求数）。调大会带来吞吐量的提升，但显存和延迟增加；调小能保证延迟更稳定，但吞吐下降。
- `--gpu-memory-utilization 0.90`：vLLM允许占用的GPU显存比例。0.90表示最多使用90%显存，剩余10%留给CUDA上下文、其他进程或碎片。
- `--reasoning-parser qwen3`：指定要使用的推理解析器。Qwen3系列支持“思考模式”，输出中会包含 ... 的推理过程。该解析器会自动把思考内容与最终回答分离，在API返回中分别放入reasoning_content和content字段。
- `--enable-auto-tool-choice`：启用自动工具调用（Function Calling）能力，从而允许模型根据请求中提供的tools定义，自主决定是否调用工具并生成结构化参数。这是开启tool use的前提开关。
- `--tool-call-parser qwen3_xml`：指定工具调用的输出解析器。Qwen3系列使用XML格式输出工具调用（如 `<tool_call>...</tool_call>`）。qwen3_xml解析器会把这些XML片段转换为OpenAI标准的tool_calls结构。
- `--default-chat-template-kwargs '{"enable_thinking":false}'`：传递给chat template的默认关键字参数。enable_thinking: false 示默认关闭思考模式。模型会直接输出答案，不生成 ... 推理过程。关闭思考模式可降低延迟、减少 token 消耗，适合不需要显式推理的对话场景。
- `--speculative-config '{"method":"mtp","num_speculative_tokens":1}'`：启用推测解码（Speculative Decoding）的配置。示例中这个配置表示使用模型自带的 MTP (Multi-Token Prediction) 头，无需额外草稿模型。

本地测试（注意端口要匹配）：

```bash
curl http://localhost:6006/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3",
    "messages": [
      {"role": "user", "content": "你好，请用一句话介绍马哥教育。"}
    ],
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

远程测试（注意修改其中的服务地址为你真实可用的地址）：

```bash
curl https://u492360-aab5-9b7211b4.westb.seetacloud.com:8443/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3",
    "messages": [
      {"role": "user", "content": "你好，请用一句话介绍马哥教育。"}
    ],
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

## 二、Open WebUI

1. 切换到目标目录

```bash
cd open-webui
```

2. 注意修改LLM服务端点

```text
OPENAI_API_BASE_URL=https://u492360-aab5-9b7211b4.westb.seetacloud.com:8443/v1
```

3. 获取 Image

```bash
docker compose pull
```

4. 启动容器服务

```bash
docker compose up
```

## 三、启动prometheus监控服务

### 1. 初始设置

```bash
# 切换到vllm-prometheus目录
cd vllm-prometheus

# 创建服务用到的数据目录
mkdir -p prometheus/data grafana/data alertmanager/data

# 设置其属主、属组，以确保指定的用户可以正常访问
chown 1000:1000 prometheus/data alertmanager/data
chown 472:472 grafana_data/
```

### 2. 修改目标Target（vllm）

本示例中，监控vLLM Server的配置通过基于文件的服务发现（file-based sd）进行，配置文件放置在 `prometheus/conf/file_sd_targets` 目录中的 `vllm-hosts.yml` 文件中，请修改vllm相关的目标Target的地址为容器化运行的Prometheus可达的地址。

```yaml
- targets:
    - "u492360-aab5-9b7211b4.westb.seetacloud.com:8443"       # 修改为vLLM Server监听地址和端口
  labels:
    env: "production"           # 可选：附加标签，会合并到采集到的所有指标上
    service: "vllm"             # 可选：便于在 Grafana 中按服务名过滤
```

另外，如有必要，还需要修改 `prometheus/conf/prometheus.yml` 文件中的vllm job，确保修改后的结果类似如下所示：

```yaml
  # 采集 vLLM 指标, 使用基于 YAML 文件的服务发现
  - job_name: "vllm"
    scrape_interval: 10s
    metrics_path: /metrics
    scheme: https     # 重点是该项
    file_sd_configs:
      - files:
          - "/etc/prometheus/file_sd_targets/*.yml"   # 服务发现目标文件路径
        refresh_interval: 30s                        # 文件变更检测间隔，无需重启即可生效
```

### 3. 启动服务

```bash
# 获取Image
docker compose pull

# 启动服务
docker compose up -d
```

### 4. 导入Grafana Dashboard

Grafana上vllm的Dashboard示例：`25502`

## 四、（可选）推理服务压测

### 1. 生成本地测试文件

```bash
cat <<EOF > /root/vllm-bench-prompts.jsonl
{"prompt":"请解释 Kubernetes 中 Deployment 和 StatefulSet 的区别。","output_tokens":2048}
{"prompt":"分析下面这段 Nginx 日志可能存在的问题：10.0.0.1 - - [16/Sep/2026:20:00:01] \"GET /api HTTP/1.1\" 502 173","output_tokens":2048}
{"prompt":"Prometheus 发现某节点 CPU 使用率持续超过 90%，请给出排查思路。","output_tokens":2048}
{"prompt":"请解释 vLLM 中 PagedAttention 的工作原理。","output_tokens":4096}
{"prompt":"请用一句话介绍一下主营产品为Linux应用运维、云计算和SRE及周边课程的马哥教育。","output_tokens":512}
{"prompt":"请分析 Kubernetes Pod 出现 CrashLoopBackOff 的常见原因以及排查步骤。","output_tokens":2048}
{"prompt":"请详细解释 vLLM 中 Continuous Baching 的工作原理。","output_tokens":4096}
{"prompt":"请写一个玄幻小说的目录规划，男主角精通SRE，不少于5000字。","output_tokens":8192}
EOF
```

### 2. 压测

```bash
conda activate /root/autodl-tmp/conda/vllm29/
cd /root/autodl-tmp/models/
```

使用vLLM官方压测工具“vllm bench serve”对大语言模型进行端到端服务性能测试：

```bash
vllm bench serve \
  --backend openai-chat   \
  --base-url http://localhost:6006   \
  --endpoint /v1/chat/completions   \
  --model qwen3   \
  --tokenizer ./Qwen3.5-4B   \
  --dataset-name custom   \
  --dataset-path /root/vllm-bench-prompts.jsonl   \
  --custom-output-len -1   \
  --num-prompts 200   \
  --max-concurrency 30   \
  --request-rate inf
```

各参数的详细解释：

**基础连接配置**
- `--backend openai-chat`：指定后端协议类型为 openai-chat，表示使用 OpenAI 的聊天接口格式进行请求。
- `--base-url http://localhost:6006`：指定 vLLM 服务的地址和端口为 localhost:6006。
- `--endpoint /v1/chat/completions`：指定要测试的具体 API 端点路径为 /v1/chat/completions（聊天补全接口）。

**模型与分词器配置**
- `--model qwen3`：指定要测试的模型名称为 qwen3。
- `--tokenizer ./Qwen3.5-4B`：指定本地分词器（Tokenizer）的路径为 ./Qwen3.5-4B。压测工具需要它来计算 Token 数量和生成延迟指标。

**数据集配置**
- `--dataset-name custom`：指定数据集类型为自定义（custom）。
- `--dataset-path /root/vllm-bench-prompts.jsonl`：指定自定义数据集的文件路径。压测工具会读取该 JSONL 文件中的 prompt 作为请求内容。
- `--custom-output-len -1`：设置自定义数据集的输出长度。设置为 -1 通常表示不强制截断输出，而是使用数据集中自带的 output_tokens 字段（即你之前生成的 2048）或生成到结束符为止。

**流量与并发控制**
- `--num-prompts 200`：指定本次压测总共发送 200 个请求。
- `--max-concurrency 30`：限制最大并发请求数为30。即使请求发送得再快，同时处于处理状态的请求也不会超过30个。
- `--request-rate inf`：设置请求发送速率为无限大（inf）。这意味着压测工具会尽可能快地（瞬间）发送所有请求，以测试服务在最大并发压力下的极限吞吐能力。

## 五、启动嵌入模型

### 1. 启动Embedding模型

切换至vllm的虚拟环境

```bash
conda activate /root/autodl-tmp/conda/vllm29/
```

模型文件位于 `/root/autotl-tmp/models/Qwen3-Embedding-4B` 目录下：

```bash
cd /root/autotl-tmp/models
export VLLM_USE_DEEP_GEMM=0 CUDA_VISIBLE_DEVICES=1
vllm serve ./Qwen3-Embedding-4B \
  --dtype float16 \
  --max-model-len 32768 \
  --max-num-seqs 16 \
  --gpu-memory-utilization 0.90 \
  --tensor-parallel-size 1 \
  --served-model-name qwen3-embedding \
  --port 6008 \
  --trust-remote-code \
  --enforce-eager
```

> **注意**：同样是为了适配AutoDL远程端口开放的需要，这里给修改为了6008。默认为8000。

### 2. 测试文本嵌入

```bash
curl -X POST http://localhost:6008/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-embedding",
    "input": "马哥出品，必属精品。"
  }'
```

### 3. 添加到Prometheus的监控中

修改 `prometheus/conf/file_sd_targets` 目录中的 `vllm-hosts.yml` 文件，添加一个新的目标Target的地址。

```yaml
- targets:
    - "u492360-a5b8-a3bf5850.westc.seetacloud.com:8443"       # 修改为vLLM Server监听地址和端口
    - "uu492360-a5b8-a3bf5850.westc.seetacloud.com:8443"
  labels:
    env: "production"           # 可选：附加标签，会合并到采集到的所有指标上
    service: "vllm"             # 可选：便于在 Grafana 中按服务名过滤
```

## 六、Dify和RAG 

### 1. 克隆Dify仓库

为了可复现，这里指定1.17.1这个版本：

```bash
git clone --branch 1.17.1 https://github.com/langgenius/dify.git
```

### 2. 初始化配置

```bash
cd dify/docker 
cp .env.example .env
```

### 3. 下载镜像，启动服务

```bash
docker compose pull 
docker compose up -d 
```

### 4. 访问Dify服务

随后即可访问Dify主机的上服务，默认端口为80：

```text
http://dify.magedu.com 
```

### 5. 添加LLM和Embedding模型

“集成” --> “模型供应商” --> 搜索“openai-api-compatible”并安装

**添加LLM模型的关键配置：**

- 模型名称：qwen3
- 模型类型：LLM
- 凭据名称：vLLM-Qwen3
- 模型显示名称：Qwen3.5-4B
- API Key：留空（如果 vLLM 未设置 `--api-key`）
- API Base URL（修改为具体值）： `http://<vLLM-IP>:8000/v1`
- API endpoint中的模型名称（必须与vllm上的模型名称保持一致，留空则同“模型名称”）：qwen3
- 对话类型：对话
- 模型上下文长度：262144（生产可以主动限制成 65536/131072）
- 最大 token 上限：4096 或 8192
- 思考模式支持：两种模型都支持
- API 类型：Chat Completions API (/chat/completions)
- 函数调用类型：工具调用

**添加Embedding模型的关键配置：**

- 模型名称：qwen3-embedding
- 模型类型：Text Embedding
- 凭据名称：Qwen3-Embedding-vLLM
- 模型显示名称：Qwen3 Embedding
- API Key：未启用认证则留空
- API Base URL：`http://embedding-server:8001/v1`
- API endpoint中的模型名称：qwen3-embedding
- 模型每批的最大分块数：16
- 模型上下文长度：按模型真实上限填写，例如 8192

## 七、Dify RAG 测试

### 1. 创建知识库

“知识库” --> “创建即用时知识库” --> 配置知识库 --> 选择要添加到知识库的文档（这里以dify-rag目录给定的文件为例）。

### 2. 关键配置项

**父子分段**

- 父块类型：段落
- 父块分隔符：`\n### `
- 父块最大长度：2048 characters
- 子块分隔符：`\n\n`
- 子块最大长度：512 characters
- 替换连续空格、换行符和制表符：开启
- 删除 URL 和电子邮件地址：关闭
- 摘要自动生成：关闭

**索引方式**：高质量

**混合检索**：开启

**排序方式**：权重设置
- 语义：0.5
- 关键词：0.5

**Top K**：5

**Score 阈值**：开启
- Score Threshold：0.45～0.5

**Rerank**：关闭

> **注意**：上面的父块分隔符是“`\n### `”，后面一个空格，这个主要是为了匹配markdown的三级标题的前缀。

### 3. 召回测试

“前往知识库” --> “召回测试”

**告警名精确路由**

| #    | 测试查询                                  | 预期命中                                         |
| ---- | ----------------------------------------- | ------------------------------------------------ |
| A1   | 收到告警 VLLMHighKVCacheUsage，如何处理？ | §5.4，排名第一                                   |
| A2   | LiteLLMProxyDown 告警的影响面和排查步骤   | §5.11（注意不能命中成 §5.1——同构条目区分度测试） |
| A3   | GPUXidError 和 GPULost 的鉴别命令是什么   | §5.3                                             |

**故障现象语义描述**

考察向量通道：告警里没有告警名，只有现象。

| #    | 测试查询                                                     | 预期命中                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| B1   | "vLLM 服务起不来，日志里一直在编译 kernel，等了半个多小时"   | §5.1 + 案例 INC-20260901-01 + §6.1                           |
| B2   | "凌晨高负载时推理服务崩了，nvidia-smi 少了一张卡，dmesg 有 Xid 79" | §5.3 + 案例 INC-20260905-02（Xid 79 原文命中）               |
| B3   | "Dify 上所有对话都失败，但 vLLM 的健康检查是正常的"          | §5.10 或 §5.11 + 案例 INC-20260910-03（**网关层归因能力的关键测试**） |
| B4   | "首 token 突然变慢，TTFT 的 P99 到了 8 秒"                   | §5.6                                                         |
| B5   | "磁盘使用率 92%，有哪些地方可以清理"                         | §5.9                                                         |

**边界与负例**

考察不误召、不臆断。

| #    | 测试查询                                   | 预期行为                                                     |
| ---- | ------------------------------------------ | ------------------------------------------------------------ |
| E1   | "如何给 Kubernetes 集群配置 HPA？"         | 知识库无此内容，召回得分应低于阈值；不得编造                 |
| E2   | "vLLM 0.28 的启动参数和 0.29 有什么差异？" | 文档未覆盖版本对比，同上应走人工升级                         |
| E3   | "如何扩容 vLLM 到多机多卡？"               | 文档明确单机单卡（`tensor-parallel-size 1`），应如实回答"本文档环境为多卡不并行"，而非泛化多机方案 |

## 八、模型网关

LiteLLM Proxy 是一个开源的“大模型统一网关”，位于 Dify、应用程序与后端模型服务之间，对上提供统一的 OpenAI 兼容接口，对下连接 vLLM、OpenAI、Azure OpenAI、Claude 等不同模型服务。

### 1. 编辑配置

将启动的LLM模型和嵌入模型添加到配置文件中。配置文件为 `config/litellm-config.yaml`，修改其中的两个api-base的值为你的vllm提供服务的地址和端口：

```yaml
qwen3: 
  api_base: https://u492360-aab5-9b7211b4.westb.seetacloud.com:8443/v1

qwen3-embedding:
  api_base: https://uu492360-aab5-9b7211b4.westb.seetacloud.com:8443/v1
```

### 2. 启动服务 

切换到目标目录下：

```bash
cd litellm-proxy 
```

启动服务：

```bash
dokcer compose up -d 
```

访问LiteLLM Proxy UI：

```text
http://llm.magedu.com:4000/ui/
```

默认的用户名和密码： `admin/magedu.com` 

### 3. 创建Virtual Key并对模型发起访问测试

假设创建的Virual Key如下：

- open-webui-key: `sk-kh6ww9QLFnL47asgPMACZQ`
- dify-key: `sk-ewJ_IUAmKBAt6bGTP5pqEA`

请求LLM服务：

```bash
curl http://llm.magedu.com:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-PioDVBvK0Rfb-UV2vn2s-w" \
  -d '{
    "model": "qwen3",
    "messages": [
      {"role": "user", "content": "你好，请用一句话介绍马哥教育。"}
    ],
    "max_tokens": 256,
    "temperature": 0.7
  }'
```

请求嵌入模型服务：

```bash
curl -X POST http://llm.magedu.com:4000/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-PioDVBvK0Rfb-UV2vn2s-w" \
  -d '{
    "model": "qwen3-embedding",
    "input": "马哥出品，必属精品。"
  }'
```

### 4. 修改Dify接入配置

修改Dify通过LiteLLM Proxy接入两个模型，并进行测试。

## 九、prometheus-mcp 

prometheus/prometheus-mcp是由Prometheus官方组织提供的MCP Server，用于把Prometheus的查询能力转换成大模型可以调用的标准MCP工具。

### 1. 启动服务 

首先，配置后端Prometheus Server的地址：

编辑 `docker-compose.yml` 文件，修改环境变量 `PROMETHEUS_MCP_SERVER_PROMETHEUS_URL` 的值为后端Prometheus Server实际监听的地址。

而后，切换至目标目录：

```bash
cd prometheus-mcp 
```

启动服务：

```bash
docker compose up -d 
```

### 2. 测试 

MCP 客户端不能一上来就调用 tools/call。对于 Streamable HTTP，标准访问顺序是：
1. initialize
2. notifications/initialized
3. tools/list
4. tools/call（可重复多次）
5. DELETE /mcp（可选，结束会话）

#### initialize请求

下面是一个initilize请求：

```bash
curl -sS -N \
  -X POST http://prometheus-mcp.magedu.com:9100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {
        "name": "prometheus-mcp-curl-test",
        "version": "1.0.0"
      }
    }
  }'
```

HTTP 响应头中还可能返回 `Mcp-Session-Id: xxxxx`，后续的请求需要用到该ID。下面是一个可以保存Session ID的命令。

```bash
SESSION_ID=$(curl -i -N -sS   -X POST http://prometheus-mcp.magedu.com:9100/mcp   -H 'Content-Type: application/json'   -H 'Accept: application/json, text/event-stream'   -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-test",
        "version": "1.0"
      }
    }
  }' | awk '/Session/{print $2}' | tr -d '\r\n')
```

> **注意**：要确保SESSION_ID中没有结尾处的“\r”，可以用如下命令测试：
> ```bash
> printf '%q\n' "${SESSION_ID}"
> ```

#### 确认初始化完成

```bash
curl -i -N \
  -X POST http://prometheus-mcp.magedu.com:9100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{
    "jsonrpc": "2.0",
    "method": "notifications/initialized"
  }'
```

#### 列出支持的工具

```bash
curl -sS -N \
  -X POST http://prometheus-mcp.magedu.com:9100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
  }' | grep '^data:' | cut -d' ' -f2- | jq -r '.result.tools[].name'
```

#### 调用 query 验证 Prometheus

```bash
curl -sS -N \
  -X POST http://prometheus-mcp.magedu.com:9100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-03-26' \
  -H "Mcp-Session-Id: ${SESSION_ID}" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "query",
      "arguments": {
        "query": "up"
      }
    }
  }'
```

它会返回类似如下内容，表明通过MCP Server进行指标查询成功：

```text
data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"{\"result\":\"up{instance=\\\"localhost:9090\\\", job=\\\"prometheus\\\"} =\\u003e 1 @[1789456855.716]\\nup{env=\\\"production\\\", instance=\\\"u492360-a5b8-a3bf5850.westc.seetacloud.com:8443\\\", job=\\\"vllm\\\", service=\\\"vllm\\\"} =\\u003e 1 @[1789456855.716]\\nup{env=\\\"production\\\", instance=\\\"uu492360-a5b8-a3bf5850.westc.seetacloud.com:8443\\\", job=\\\"vllm\\\", service=\\\"vllm\\\"} =\\u003e 1 @[1789456855.716]\",\"warnings\":null}"}]}}
```

## 十、集成Dify和promtheus-mcp 

### 1. 开放Dify对内网中的MCP Server的访问

为防范SSRF（服务器端请求伪造）攻击，Dify默认禁止SSRF代理访问任何私有IP地址。这时，Dify的功能（如工具调用、网页读取，以及私网中的MCP Server等）尝试请求一个指向内网IP的地址时，请求会被直接拦截并拒绝。这是最安全的配置，用于保护内网环境。因此，为了集成监听于私网地址的prometheus-mcp，我们首先需要允许对其的访问。建议遵循最小权限原则，仅开班需要访问的目标主机的IP地址。

```text
SSRF_PROXY_ALLOW_PRIVATE_IPS=172.16.0.0/12
```

### 2. 添加MCP服务

集成 --> 工具 --> MCP --> 添加MCP服务（HTTP）

> **注意**，地址是 `http://prometheus-mcp.magedu.com:9100/mcp`

### 3. 集成和测试

（略）

## 十一、AlertManager触发自动问题诊断

### 1. 修改AlertManager的配置文件

将其中的Webhook更改为Workflow Webhook发布后的URL。

### 2. 启动AlertManager服务

```bash
cd vllm-prometheus 
docker compose --profile alertmanager up -d alertmanager
```

### 3. 导入Workflow的定义

（略）

### 4. 手动测试

发布Workflow后（或通过“测试运行”进行），向其Webhook的地址使用curl命令发起测试请求，请求体使用标准的AlertManager的告警信息格式。

下面的示例手动构造了一个模拟“vLLM KV Cache 使用率过高”的告警，注意修改其中的Webhook地址为实际的Dify Workflow Webbook的地址，还要修改其中的instance为真实的vllm服务器的地址。

```bash
curl -i -X POST \
  'http://dify.magedu.com/triggers/webhook/jrd4B9rpTmbfnIHkw7B93wme' \
  -H 'Content-Type: application/json' \
  --data-binary '{
    "version": "4",
    "groupKey": "{}:{alertname=\"HighVLLMKVCacheUsage\",instance=\"u492360-aab5-9b7211b4.westb.seetacloud.com:8443\"}",
    "truncatedAlerts": 0,
    "status": "firing",
    "receiver": "dify-webhook",

    "groupLabels": {
      "alertname": "VLLMHighKVCacheUsage"
    },

    "commonLabels": {
      "alertname": "VLLMHighKVCacheUsage",
      "instance": "u492360-aab5-9b7211b4.westb.seetacloud.com:8443",
      "job": "vllm",
      "service": "vllm",
      "env": "production",
      "model_name": "qwen3",
      "severity": "warning"
    },

    "commonAnnotations": {
      "summary": "vLLM KV Cache usage is too high",
      "description": "The vLLM KV Cache usage has exceeded the configured warning threshold.",
      "runbook_hint": "5.4"
    },

    "externalURL": "http://alertmanager.magedu.com:9093",

    "alerts": [
      {
        "status": "firing",

        "labels": {
          "alertname": "VLLMHighKVCacheUsage",
          "instance": "u492360-aab5-9b7211b4.westb.seetacloud.com:8443",
          "job": "vllm",
          "service": "vllm",
          "env": "production",
          "model_name": "qwen3",
          "severity": "warning"
        },

        "annotations": {
          "summary": "vLLM KV Cache usage is too high",
          "description": "The vLLM KV Cache usage has exceeded the configured warning threshold.",
          "runbook_hint": "5.4"
        },

        "startsAt": "2026-09-15T15:45:00+08:00",
        "endsAt": "0001-01-01T00:00:00Z",

        "generatorURL": "http://prometheus.magedu.com:9090/graph?g0.expr=vllm%3Akv_cache_usage_perc%7Bjob%3D%22vllm%22%2Cinstance%3D%22u492360-a5b8-a3bf5850.westc.seetacloud.com%3A8443%22%7D",

        "fingerprint": "vllm-kv-cache-qwen3-test-001"
      }
    ]
  }'
```

若以上命令的响应结果包含类似如下内容，则表示请求成功：

```json
{"status":"success","message":"Webhook processed successfully"}
```

### 5. 自动触发测试

向vLLM上的qwen3发起压测请求，使得Prometheus触发TTFT、E2E或者等待队列等任何指标的告警，而后即可通过Dify Workflow上的日志来了解其RCA诊断结果。



## 附录：Dify的环境变量配置

> **提示**：实验环境，默认的各项配置即可启动服务。但生产中要做多项调整，至少如下敏感配置要修改。为了节约时间，本示例不再修改。

```text
DB_PASSWORD=difyai123456
REDIS_PASSWORD=difyai123456

CODE_EXECUTION_API_KEY=dify-sandbox
SANDBOX_API_KEY=dify-sandbox

PLUGIN_DAEMON_KEY=...
PLUGIN_DIFY_INNER_API_KEY=...

DIFY_AGENT_API_TOKEN=dify-agent-run-token-for-dev-only
DIFY_AGENT_SERVER_SECRET_KEY=MDEyMzQ1...
```

`DIFY_AGENT_SERVER_SECRET_KEY` 也明确要求生产环境替换，官方当前源码也再次强调这两个值不能继续使用开发默认值。可以统一生成高强度随机值：

```bash
python -c 'import secrets; print(secrets.token_urlsafe(48))'
```

另外，还有一些公网URL需要配置，它们的默认值均为localhost，应该适配到Dify主机的可用外部地址，例如 `dify.magedu.com`：

```text
CONSOLE_API_URL=
CONSOLE_WEB_URL=
SERVICE_API_URL=
TRIGGER_URL=http://localhost
APP_API_URL=
APP_WEB_URL=
FILES_URL=

ENDPOINT_URL_TEMPLATE=http://localhost/e/{hook_id}
NEXT_PUBLIC_SOCKET_URL=ws://localhost
```

修改为：

```text
CONSOLE_API_URL=http://dify.magedu.com
CONSOLE_WEB_URL=http://dify.magedu.com

SERVICE_API_URL=http://dify.magedu.com
APP_API_URL=http://dify.magedu.com
APP_WEB_URL=http://dify.magedu.com

FILES_URL=http://dify.magedu.com

TRIGGER_URL=http://dify.magedu.com
ENDPOINT_URL_TEMPLATE=http://dify.magedu.com/e/{hook_id}

NEXT_PUBLIC_SOCKET_URL=wss://dify.magedu.com
```
