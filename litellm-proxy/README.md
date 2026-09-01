# Litellm-Proxy 实践教程



LiteLLM Proxy（也称为 LiteLLM AI Gateway 或 LLM Proxy Server）是一款知名的开源的代理服务器和 AI 网关工具，由 BerriAI 开发。它允许开发者通过统一的 OpenAI 兼容 API 接口调用超过 100 个大型语言模型（LLM）提供商，包括 OpenAI、Azure、Anthropic、Hugging Face、Bedrock、TogetherAI、VertexAI、Cohere、Sagemaker、VLLM 和 NVIDIA NIM 等。 该工具的核心目标是简化多提供商 LLM 的集成、管理和监控，支持负载均衡、成本跟踪、预算控制和日志记录等功能。它本质上是一个 LLM 网关，能够将不同的 LLM API 标准化为 OpenAI 格式的输入/输出，从而减少开发者在切换模型时的适配工作。



**配置文件的配置段说明**

| Param Name              | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `model_list`            | List of supported models on the server, with model-specific configs |
| `router_settings`       | litellm Router settings, example `routing_strategy="least-busy"` |
| `litellm_settings`      | litellm Module settings, example `litellm.drop_params=True`, `litellm.set_verbose=True`, `litellm.api_base`, `litellm.cache` |
| `general_settings`      | Server settings, example setting `master_key: sk-my_special_key` |
| `environment_variables` | Environment Variables example, `REDIS_HOST`, `REDIS_PORT`    |



## LiteLLM-Proxy使用示例

本示例的核心目标将集中于LiteLLM-Proxy核心功能的演示上，包括模型代理、Virtual Key、输入过滤和缓存等。

![Deployment_Architecture](./images/Deployment_Architecture.png)



**相关的docker-compose文件说明：**

- docker-compose.yml：基础示例，它会通过卷读取config/litellm-proxy.yaml作为其配置文件；
- docker-compose-lilellm-proxy-ui.yml：启用LiteLLM-Proxy的UI，并激活Virtual Key；
- docker-compose-cache-redis.yml：基于Redis的请求缓存，精确匹配；它会通过卷读取config/litellm-config-with-cache.yaml作为其配置文件；



### 1. 基础示例

运行如下命令启动服务：

```bash
docker-compose up
```

运行如下命令，即可进行请求测试。

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-8b",
    "messages": [
      {
        "role": "user",
        "content": "请介绍一下马哥教育。"
      }
    ]
}'
```



### 2. UI和Virtual Key

启用UI并使用Virtual Key，通常需要部署并配置一个可用于的PostgreSQL作为LiteLLM-Proxy的存储后端。LiteLLM-Proxy通常基于环境变量来对接要访问PostgreSQL服务，例如：

```bash
DATABASE_URL=postgresql://llmproxy:dbmagedu-com@litellm-db:5432/litellm
STORE_MODEL_IN_DB=True
```



另外，UI的配置也可以通过环境变量进行设定。相关的几个关键环境变量如下：

```yaml
LITELLM_MASTER_KEY="magedu.com" # this is your master key for using the proxy server
UI_USERNAME=admin             # username to sign in on UI，默认为admin
UI_PASSWORD=Magedu-Com        # password to sign in on UI，默认为环境变量LITELLM_MASTER_KEY的值
SERVER_ROOT_PATH=/ui          # UI的访问路径，默认为/ui
```



运行如下命令启动服务：

```bash
docker-compose -f docker-compose-lilellm-proxy-ui.yml up
```

随后，通过LiteLLM-Proxy服务的“/ui”路径即可访问相关的服务，访问时会首先跳转至需要登录页面。其中的用户名和密码即为前面使用环境变量UI_USERNAME和UI_PASSWORD设置的值。若未设定此两项时，默认的用户名为admin，密码为环境变量LITELLM_MASTER_KEY的值。

![litellm-proxy_login](./images/litellm-proxy_login.png)



登录后即可进行UI，其默认的首屏为虚拟密钥界面。

![litellm-proxy_ui](./images/litellm-proxy_ui.png)



随后，我们需要先创建一个Virtual Key，以认证并访问指定的模型服务。

![litellm-proxy_create_virtual_key](/home/marion/git-repos/llm-in-practise/Deployment/litellm-proxy/images/litellm-proxy_create_virtual_key.png)



随后在弹出的对话框中会提示生成的密钥，要注意保存该密钥。随后，它代表持有该密钥的用户的访问ID，相关的配额、RPM等相关配置也与该密钥相关。

![litellm-proxy_virtual-key](./images/litellm-proxy_virtual-key.png)



接下来即可使用该密钥来请求被授予访问的模型qwen3-8b模型。

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA" \
  -d '{
    "model": "qwen3-8b",
    "messages": [
      {
        "role": "user",
        "content": "请介绍一下马哥教育。"
      }
    ]
}'
```



### 3. 缓存

LiteLLM-proxy 支持两种类型的缓存机制：**精确匹配缓存 (Exact Match Cache)** 和 **语义缓存 (Semantic Cache)**。这两种机制可以配置不同的缓存后端 (Cache Backends) 来存储数据。

**精确匹配缓存（Exact-Match Response Cache）的工作原理**

- Key 生成：默认使用 model + messages + temperature + logit_bias 等哈希。可自定义 get_cache_key 函数
- 命中条件：精确字符串匹配（忽略顺序，但不处理语义相似）
- 存储格式：JSON（plaintext，未加密），包含响应 + 元数据（timestamp、cost）
- Proxy 特有：支持 /cache/ping 端点测试连接；高负载下可批量 Redis GET（减少延迟 20-30%）
- 性能：内存后端 <1ms 命中；Redis ~5ms

本示例将以redis cache为例，来测试LiteLLM-Proxy的精确缓存机制。运行如下命令即可启动LiteLLM-Proxy、inference-engine、guard-engine、llama-guard-wrapper和redis-cache等相关的服务。

```bash
docker-compose -f docker-compose-cache-redis.yml up
```

随后，可针对同一个Prompt连续发起多次请求进行验证。若两次请求完全相同，随后的响应可以直接命中缓存，并从缓存返回结果给用户。缓存是否命中的相关信息，可从LiteLLM-Proxy的日志中检索，LiteLLM-Proxy的UI也能展示缓存命中的相关信息。

![LiteLLM-Proxy_Response-Cache](./images/LiteLLM-Proxy_Response-Cache.png)





### 4. 语义缓存

语义缓存是部署于 AI Gateway 层（如 LiteLLM Proxy、Portkey、Cloudflare AI Gateway 等）的一种请求级拦截与复用机制。其核心目标是：在用户请求到达下游 LLM 之前，通过向量相似度判断该请求是否已有语义等价的缓存响应，从而避免重复推理、降低延迟、削减 Token 成本。

对于 LiteLLM Proxy 来说，语义缓存（Semantic Cache）非常适合部署在 LLM Gateway 层，尤其是大量 FAQ、RAG 问答、知识助手、标准化运维咨询等重复语义请求，可以明显降低后端 vLLM 的 GPU 请求量。截至 **2026 年 9 月**，LiteLLM 官方 Proxy 已明确支持以下三种语义后端：

- `redis-semantic`
- `qdrant-semantic`
- `valkey-semantic`

本示例采用**LiteLLM Proxy + Redis Semantic Cache + Embedding Model + vLLM Chat Model**这样一套架构来演示语义缓存的功能。

#### 语义缓存的工作机制

```mermaid
flowchart TD
	%% magedu.com
    A[User Prompt] --> B[Embedding Model]
    B --> C[Prompt Vector]
    C --> D[Vector Similarity Search]
    D --> E{Similarity >= Threshold?}
    
    E -- Yes --> F[Cache Response]
    E -- No --> G[LLM]
    
    G --> H[Store Prompt Vector & Response]
    
    %% 样式美化（可选）
    style E fill:#fff4e6,stroke:#ff9800,stroke-width:2px
    style F fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style G fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
    style H fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
```



**阶段 1：请求接入与向量化**（Prompt Vector → Vector Similarity Search）

- 用户请求（Prompt）经由 Gateway 拦截后，不直接转发至 LLM，而是首先送入 Embedding 模型（如 qwen3-embedding、text-embedding-3-small、bge-large 等）。
- Embedding 模型将自然语言文本映射为固定维度的稠密向量（如 1536 维），该向量即为 Prompt Vector，是后续相似度检索的唯一索引键。

> 此步骤的计算开销极低（通常 < 10ms），远低于一次完整 LLM 推理。

**阶段 2：向量相似度检索**（Prompt Vector → Vector Similarity Search）

- Gateway 将生成的 Prompt Vector 提交至后端向量存储（如 Redis Vector Search、Qdrant、Milvus、pgvector 等）。
- 检索算法通常采用 余弦相似度（Cosine Similarity） 或 内积（Inner Product），在已缓存的历史 Prompt Vector 集合中执行 Top-K 近邻搜索（ANN）。
- 返回结果包含：最相似历史向量的相似度分数（score ∈ [0, 1]）及其关联的缓存响应体。

**阶段 3：阈值判定与路由决策**

```
Similarity >= Threshold ?
    ├── Yes → Cache Response（直接返回）
    └── No  → LLM（透传至模型）
```

- 阈值（Threshold） 是系统的关键超参数，典型取值范围为 0.92 ~ 0.98。
  - 阈值过高（如 0.99）：缓存命中率极低，形同虚设。
  - 阈值过低（如 0.85）：可能将语义不同的请求误判为等价，导致响应污染（Response Contamination）。
- Yes 分支：相似度达标，Gateway 直接返回缓存的 LLM 响应，请求生命周期终止。下游 LLM 不被调用，Token 消耗为零。
- No 分支：相似度未达标，判定为新请求，Gateway 将原始 Prompt 透传至下游 LLM 进行完整推理。

**阶段 4：缓存写入（仅 No 分支触发）**

```
LLM Response → Store Prompt Vector + Response
```

LLM 返回响应后，Gateway 执行异步写入操作，将以下三元组持久化至向量存储：

| 字段            | 内容                              | 用途                   |
| :-------------- | :-------------------------------- | :--------------------- |
| `prompt_vector` | 本次请求的 Embedding 向量         | 后续相似度检索的索引键 |
| `response`      | LLM 完整响应体                    | 命中时直接返回的载荷   |
| `metadata`      | 时间戳、模型名、TTL、token 用量等 | 过期淘汰与审计         |

- 写入操作通常置于异步队列中执行，不阻塞响应返回路径。

#### 关键设计约束与工程考量

1. 缓存失效策略（Invalidation）
- TTL（Time-To-Live）：设定缓存有效期（如 24h），过期自动淘汰，防止陈旧响应被返回。
- 模型版本绑定：缓存条目须与生成它的模型版本（如 gpt-4o-2024-08-06）强绑定。模型升级后，旧缓存应标记失效。
- 参数敏感性：temperature、top_p、system prompt 等参数差异应纳入缓存键的考量，否则可能返回不符合调用方预期的响应。
2. 阈值调优
- 阈值并非全局固定值，应根据业务场景分级设定：
- 高确定性场景（如 FAQ、知识问答）：阈值可放宽至 0.90。
- 创意生成 / 代码生成：阈值应收紧至 0.97+，或直接禁用语义缓存。
3. 非幂等请求的排除
- 含时间敏感信息（"今天的天气"）、用户上下文（"我的订单状态"）或随机性要求的请求，不应进入缓存检索流程。Gateway 需通过规则引擎或 Prompt 分类器进行前置过滤。
4. 安全性与隔离
- 多租户环境下，向量存储必须按 tenant_id / api_key 进行命名空间隔离，严禁跨租户缓存命中。

#### 请求链

如下请求链的工作逻辑可严格概括为：应用请求 → 网关拦截 → 向量化 → 语义检索 → 阈值判定 → 命中直返 / 未命中推理并回写。它以 LiteLLM Proxy 为核心网关、集成 Redis Semantic Cache 的 LLM 推理服务链路。其设计目标是在应用层（Dify）与模型推理层（vLLM Qwen）之间插入一个语义感知的缓存拦截层，在保证响应正确性的前提下，最大化缓存命中率、降低推理成本与端到端延迟。其核心价值在于将 LLM 推理从"每次全量计算"转变为"按需计算 + 语义复用"的混合模式。

```mermaid
flowchart TD
    Dify["Dify"]
    LiteLLM["LiteLLM Proxy\n:4000"]
    EmbVLLM["Embedding vLLM"]
    Redis["Redis Semantic Cache"]
    HitResp["Response"]
    Qwen["vLLM Qwen"]
    CacheWrite["Cache Response"]
    Client["Client"]

    Dify --> LiteLLM
    LiteLLM -->|"Generate Embedding"| EmbVLLM
    EmbVLLM --> Redis
    Redis -->|"HIT"| HitResp
    Redis -->|"MISS"| Qwen
    Qwen --> CacheWrite
    HitResp --> Client
    CacheWrite --> Client

    %% 样式定义
    style Dify fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style LiteLLM fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style EmbVLLM fill:#ede7f6,stroke:#6a1b9a,stroke-width:2px
    style Redis fill:#fce4ec,stroke:#c62828,stroke-width:2px
    style HitResp fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style Qwen fill:#ede7f6,stroke:#6a1b9a,stroke-width:2px
    style CacheWrite fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style Client fill:#e0e0e0,stroke:#424242,stroke-width:2px
```



#### 本地 Embedding 模型

语义缓存的 Embedding 与传统 RAG Embedding 本质相同，都是把文本转换成向量。但语义缓存与 RAG 的目标稍有区别，RAG 更关注“查询请求能否找到相关文档”，Semantic Cache 更关注“两个请求是否足够等价，旧答案是否足够可用”。

因此 Semantic Cache 对 Embedding 的要求实际上很高，它不只是要求“相关内容靠得近”，还希望“看起来相关、但不能共用答案”的问题不要靠得太近。因此 Semantic Cache 最关键的模型能力之一，是**细粒度语义区分能力**，因此我们并不能单纯按照“Embedding 越小越便宜”来选。如果主要考虑开源、本地部署、中文与英文兼顾，可以分成几组。

| 模型                           | 参数规模 | 中文 | 英文 | 多语言 | 资源 | Semantic Cache |
| ------------------------------ | -------- | ---- | ---- | ------ | ---- | -------------- |
| Qwen3-Embedding-0.6B           | 0.6B     | 很好 | 很好 | 很好   | 低   | ★★★★☆          |
| Qwen3-Embedding-4B             | 4B       | 优秀 | 优秀 | 优秀   | 中   | **★★★★★**      |
| Qwen3-Embedding-8B             | 8B       | 优秀 | 优秀 | 优秀   | 高   | ★★★★★          |
| BGE-M3                         | ~0.6B    | 很好 | 很好 | 优秀   | 低   | ★★★★☆          |
| multilingual-e5-large-instruct | ~0.6B    | 好   | 很好 | 很好   | 低   | ★★★★           |
| gte-Qwen2-1.5B-instruct        | 1.5B     | 很好 | 很好 | 好     | 中低 | ★★★★           |
| gte-Qwen2-7B-instruct          | ~7.6B    | 很好 | 很好 | 好     | 高   | ★★★★☆          |

每个需要执行 Semantic Cache Lookup 的请求，都先要调用一次 Embedding。所以 embedding 太重的话，会导致调用成本过高。最好的规格，是4B的版本，因为4B 实际上处在一个非常好的“甜点位”：性能已经非常接近 8B，但参数规模只有它的一半。




## 路由功能与示例

LiteLLM Proxy Router 是 LiteLLM Proxy 的“大脑”和“心脏”，它不是一个独立的服务，而是一个内置在LiteLLM Proxy实例中的高性能路由引擎。它负责把每一个进来的 OpenAI 格式请求（/v1/chat/completions、/v1/embeddings 等），根据定义的规则，智能、快速、安全地路由到“最合适”的后端 LLM 部署上去。

LiteLLM Proxy Router 是企业级 LLM 流量调度中枢，它让一个 Proxy 能同时管理几十甚至几百个真实的 LLM 部署（Azure、OpenAI、Anthropic、vLLM、Ollama、Groq、Together、Fireworks、Bedrock…），并实现负载均衡、自动重试、自动 fallback、成本优化、区域亲和性、会话粘性等生产级能力。

### 路由决策机制

下图展示了LiteLLM-Router的路由决策机制。

```mermaid
graph TD
    %% 1. 客户端请求
    A["客户端 (Client)"] -->|"请求 (v1/chat/completions)"| B["LiteLLM Proxy API 接口"]

    %% 2. 预处理与身份鉴权
    subgraph "预处理与鉴权 (Pre-Processing & Auth)"
        B --> C["API Key / Token 校验"]
        C --> D["速率限制与预算检查<br>(RPM/TPM Rate Limit & Budget)"]
    end

    %% 3. 路由与决策引擎
    D --> E{"Router 路由决策引擎<br>(Routing Engine)"}

    subgraph "路由策略与配置 (Routing Strategies)"
        E --> F1["负载均衡<br>(Simple Shuffle / Weighted)"]
        E --> F2["最低延迟优先<br>(Lowest Latency)"]
        E --> F3["最低成本优先<br>(Lowest Cost)"]
        E --> F4["故障转移机制<br>(Fallbacks / Cooldowns)"]
    end

    %% 4. 模型选择与代理调用
    F1 --> G["确定目标部署模型<br>(Target Deployment Picked)"]
    F2 --> G
    F3 --> G
    F4 --> G

    G --> H["转换请求格式<br>(Transform Request for Provider)"]

    %% 5. 上游 LLM 服务商
    subgraph "LLM 提供商 (LLM Providers)"
        H --> I1["OpenAI API"]
        H --> I2["Azure OpenAI"]
        H --> I3["Anthropic / Bedrock"]
        H --> I4["其他模型 (Ollama/Groq等)"]
    end

    %% 6. 响应与后处理
    I1 --> J["接收上游响应<br>(LLM Response)"]
    I2 --> J
    I3 --> J
    I4 --> J

    J --> K["异步日志与费用统计<br>(Redis / Postgres Audit & Cost Logging)"]
    K --> L["返回结果给客户端"]

    %% 样式定义
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style E fill:#ffa,stroke:#333,stroke-width:2px
    style G fill:#bbf,stroke:#333,stroke-width:2px
    style L fill:#afa,stroke:#333,stroke-width:2px
```



**下表展示了Router支持的各负载均衡算法：**

| Strategy                       | Description                                                  | When to Use                |
| ------------------------------ | ------------------------------------------------------------ | -------------------------- |
| **simple-shuffle** (推荐)      | round_robin 是基础策略，Weighted Pick 和 simple-shuffle 则是它的优化实现 | 通用生产场景，部署性能相近 |
| **least_busy**                 | 将请求发送给当前活跃请求数 (Active Requests) 最少的部署      | 高并发多实例集群           |
| **usage_based** (bad for perf) | 路由到当前 TPM/RPM 用量最低，或速率限制剩余最多的部署；也称为Rate-Limit Aware 或 Rate-Limit Aware v2 (Async) | 多 API key 限流严格        |
| **latency_based**              | 选择最近平均延迟最低的部署                                   | 延迟敏感（如实时聊天）     |
| **cost_based**                 | 选择每千 Token 成本最低的部署                                | 预算优化、混合云           |

各算法具体的配置实例，请参考[官方文档](https://docs.litellm.ai/docs/routing)。

### 可用性保障体系

从 SRE 视角看，**LiteLLM Router 本质上就是一个面向 LLM 推理流量的“智能 L7 负载均衡器 + 容错控制器”**。区别只是它除了传统的健康状态和并发状态之外，还理解 **Model、Token、TPM、RPM、模型组、模型级 Fallback** 等 LLM 特有语义。下图是**四层 Router 决策模型**：先过滤哪些 Deployment 不能用，再通过 Routing Strategy 决定用哪个；调用失败后通过 Retry 和 Cooldown 解决实例级故障，如果整个 Model Group 都无法服务，再通过 Fallback 切换到其他模型。

```mermaid
flowchart TD

    A["客户端请求<br/>Model Group"] --> B["第一层：候选过滤<br/>Candidate Filtering"]

    B --> B1["排除：<br/>Cooldown<br/>TPM/RPM 超限<br/>并发超限<br/>不健康实例"]

    B1 --> C["第二层：负载均衡<br/>Routing Strategy"]

    C --> C1["从健康 Deployments 中<br/>选择一个实例"]

    C1 --> D["第三层：实例级容错<br/>Retry + Cooldown"]

    D --> E{"当前 Model Group<br/>是否还能处理请求？"}

    E -->|是| F["返回响应"]

    E -->|否| G["第四层：模型级容错<br/>Fallback"]

    G --> H["切换至备用<br/>Model Group"]

    H --> B
```

1. **请求首先进入 Model Group，而不是直接指定某个实例**

   客户端通常请求的是一个逻辑模型名，例如“model = qwen3.8-27b”，我们可将这个 `qwen3.8-27b` 看作一个 **Model Group**。在 LiteLLM Proxy 后面，它可能对应多个实际 Deployment，所以客户端看到的是一个模型，而 Router 看到的是一组候选后端。这也是 LiteLLM Router 能做负载均衡、高可用和容灾的基础。

2. **Router 先找到所有候选 Deployment**

   找到 Model Group 后，Router 会取得这个模型组对应的所有后端实例。但这时尚且不能立即选一个来转发请求，因为这些 Deployment 当前状态可能不同。例如，有些实例正常，也有些实例RPM已经耗尽（达到调用限制），还有些实例刚刚连续调用失败，所以后面还要经过一层过滤。

3. **Pre-call Checks：先过滤“不应该调用”的实例**

   这是生产环境里非常重要的一步。Router 在真正发起模型调用之前，会根据运行状态过滤掉不适合作为当前候选目标的 Deployment。典型过滤条件包括Cooldown、TPM 限制、RPM 限制、Concurrency Limit和Health 状态，Router需要先排除明显不能用的，再从健康实例中进行挑选。

4. **Routing Strategy：从健康实例中决定“选谁”**

   过滤完成之后，如果还有多个健康 Deployment，Router 才真正进入负载均衡阶段。

5. **选中 Deployment 后，真正发送模型请求**

6. **如果调用失败，先进行 Deployment 级故障处理**

   如果当前实例调用失败，Router 会记录失败状态。此时并不会立即认为 qwen3.8-27b 整个模型都不可用了，因为可能只是这个 Deployment 有问题。于是就引出了两个很重要的机制：Retry和Cooldown。

7. **Failure Threshold：判断该 Deployment 是否已经“不可信”**

   偶发的一次超时，并不一定需要马上摘掉节点，这时一般会通过 Failure Threshold 判断“失败次数是否已经超过阈值”，若已经超过，Router 会认为该节点暂时不应该继续接收流量，于是就进入cooldown状态。

8. **Cooldown：临时把故障实例移出候选池**

   Cooldown是当于暂时摘除这个 Deployment，而非永久删除。经过一段时间后，它还可能重新恢复到候选集合。因此它类似传统负载均衡器中的“Backend Temporary Ejection”，或者可以类比 Nginx / Envoy / Service Mesh 中的“故障节点临时摘除”。

9. **Retry：当前实例失败后，尝试其他 Deployment**

   如果当前 Deployment 失败，而同一个 Model Group 中还有其他健康节点，那么 Router 可以 Retry。需要特别强调说明的是，Retry 通常仍然是在同一个 Model Group 内解决问题。因此 Retry 本质上属于实例级容错。

10. **若整个 Model Group 都不可用，则会进入 Fallback**

    Retry 是 Deployment 级容错，Fallback 是 Model Group 级容错。例如，若qwen3.8-27这一个Model Scope中的实例A1请求失败，Retry会将请求发往A2或A3，若其中的A1、A2和A3全部故障，则Fallback会将请求切换至其它Model Group，例如qwen3.5-9b。

    

进一步把相关概念映射到传统 SRE/基础设施概念，应该会非常容易理解：

| LiteLLM Router     | SRE / 基础设施中的对应概念 |
| ------------------ | -------------------------- |
| Model Group        | Service / Backend Pool     |
| Deployment         | Backend Instance / Pod     |
| Pre-call Check     | Backend Eligibility Check  |
| Routing Strategy   | Load Balancing Algorithm   |
| Retry              | Request Retry              |
| Cooldown           | Temporary Backend Ejection |
| Fallback           | Service-Level Failover     |
| TPM/RPM            | Rate Limit / Capacity      |
| Healthy Deployment | Healthy Backend            |

### Router使用示例

#### 配置说明

本示例会启动LiteLLM-Proxy服务，以及四个模型服务deepseek-r1-01、deepseek-r1-02、qwen3-4b和llama3。其中：

- deepseek-r1 组（2 个部署）：统一别名 deepseek-r1，对应两个本地 vLLM 实例（端口 8001/8002）。
  - input_cost_per_token/output_cost_per_token: 0.0000028 用于 cost-based-routing 计算总成本（输入 + 输出 token 费用，单位：美元/token）。
  - rpm: 3000/2000 + tpm: 1200000 作为限流/均衡参考（usage-based tie-breaker）。
  - weight: 0.5/0.3 在 shuffle 时加权分发（总权重 0.8，剩余 0.2 隐式均匀）。
  - priority: 100 让它在 least-busy 等策略中优先。
  - metadata.region: cn-south-1 可用于自定义路由规则（e.g., 区域亲和）。
- qwen3（备用）：成本 0.000012（高于 deepseek），rpm/tpm 较低，weight: 0.2 作为高性能后备（e.g., 长上下文场景）。
- llama3（兜底）：成本最低（0.0000008），rpm/tpm 高，fallback: true 标记为全局备用。适合廉价保底。



**router_settings 配置：**

- 负载均衡（routing_strategy: cost-based-routing）：优先选总成本最低部署（基于 input/output_cost_per_token * token 数）。若成本相等，fallback 到 least-busy（当前活跃请求最少）。
- 重试机制（num_retries / fallback_retries / retry_policy）：
  - 全局重试 3 次（含指数退避 + jitter）。
  - fallback_retries: true 让备用模型也重试。
  - retry_policy 精细控制：429 重试 5 次（限流宽容）、内部错误 3 次、超时 4 次。
  - 排除不可重试错误（如 401/内容违规）。
- 冷却机制（allowed_fails / cooldown_time）：
  - 60 秒窗口内失败 6 次 → 冷却 120 秒（部署级隔离，从可用池移除）。
  - 内置 429 自动 5 秒轻量冷却（瞬时故障快速恢复）。
- 回退机制（fallbacks / default_fallbacks）：
  - 模型特定链：deepseek-r1 失败 → qwen3 → llama3；qwen3 → llama3。
  - default_fallbacks: ["llama3"] 全局兜底（所有模型失败时用）。
- 上下文窗口回退（litellm_settings.context_window_fallbacks）：
  - deepseek-r1 超上下文 → 直接切 qwen3（需 qwen3 支持更大窗口）。
  - 结合 enable_pre_call_checks: true，预验证避免无效调用。
- 其他调优（health_check_interval / timeout / enable_pre_call_checks）：
  - 30 秒健康检查（/health 端点）
  - 90 秒超时
  - 预检查（上下文/参数验证）



#### 启动服务并进行测试

**启动服务：**

```bash
docker-compose -f docker-compose-router-lb.yaml up
```



**验证效果：**

以下所有命令假设：

- Proxy 运行在 http://localhost:4000
- 已设置Master Key，或者已经配置好Virtual Key，例如"sk-zlrZs48TQ7GkkT761QjfAA"
- 四个后端容器（deepseek-r1-01/qwen3/llama3）均已经正常运行



1. 验证负载均衡（cost-based-routing + weight）生效

   ```bash
   curl -X POST 'http://0.0.0.0:4000/chat/completions' \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer magedu.com' \
   -d '{
     "model": "deepseek-r1",
     "messages": [
           {"role": "user", "content": "请用三句话介绍一下你自己。"}
       ]
   }'
   ```

   

2. 验证重试机制（Retry）生效

   ```bash
   curl -X POST 'http://0.0.0.0:4000/chat/completions' \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA' \
   -d '{
     "model": "deepseek-r1",
     "messages": [
           {"role": "user", "content": "请用三句话介绍一下你自己。"}
       ],
       "mock_testing_rate_limit_error": true
   }'
   ```

   

3. 验证冷却机制（Cooldown）生效

   先测试deepseek-r1的fallback机制：

   ```bash
   curl -X POST 'http://0.0.0.0:4000/chat/completions' \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA' \
   -d '{
     "model": "deepseek-r1",
     "messages": [
           {"role": "user", "content": "请用三句话介绍一下你自己。"}
       ],
       "mock_testing_fallbacks": true
   }'
   ```

   

   再测试qwen3的fallback机制：

   ```bash
   curl -X POST 'http://0.0.0.0:4000/chat/completions' \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA' \
   -d '{
     "model": "qwen3",
     "messages": [
           {"role": "user", "content": "请用三句话介绍一下你自己。"}
       ],
       "mock_testing_fallbacks": true
   }'
   ```

   

4. 验证全局兜底（default_fallbacks）生效

   ```bash
   # 让所有模型(deepseek-r1-01、deepseek-r1-02和qwen3-4b)都挂掉，看是否最终落到 llama3
   curl -X POST 'http://0.0.0.0:4000/chat/completions' \
   -H 'Content-Type: application/json' \
   -H 'Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA' \
   -d '{
     "model": "deepseek-r1",
     "messages": [
           {"role": "user", "content": "请用三句话介绍一下你自己。"}
       ]
   }'
   ```

   

5. 验证上下文窗口 Fallback 生效

   ```bash
   curl -X POST http://localhost:4000/v1/chat/completions \
     -H 'Content-Type: application/json' \
     -H "Authorization: Bearer sk-zlrZs48TQ7GkkT761QjfAA" \
     -d '{
       "model": "deepseek-r1",
       "messages": [{"role": "user", "content": "'$(printf "马哥教育LLM超长文本测试%.0s" {1..300})'"}],
       "mock_testing_context_window_fallback": true
     }'
   ```

   

## 版权声明

本文档由[马哥教育](www.magedu.com)开发，允许自由转载，但必须保留马哥教育及相关的一切标识。另外，商用需要征得马哥教育的书面同意。
