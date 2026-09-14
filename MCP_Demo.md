# MCP 动手实践：基于 Dify 1.17.1 的 MCP Server 端到端验证

> **文档性质**：动手实验教程
> **适用读者**：Linux 应用运维工程师、SRE、AI Infra / LLMOps 工程师、AI Agent 应用开发者
> **环境基准**：Dify 1.17.1、MCP Python SDK（当前官方版本）、MCP 协议版本 2025-06-18
> **文档作者**：马哥教育（http://www.magedu.com）
> **版权说明：**原创文档，转载必须经过作者同意

---

## 前言

本实验的目标是验证一条完整的 MCP 调用闭环“**Dify → MCP Server → Tool Discovery → Tool Call → 返回结果**”，基于 Dify 1.17.1 的原生 MCP 能力编写。Dify 1.17.x 已将 MCP 实现为工作区级原生 Integration，而且 V1.17.1 进一步修复了 MCP Server URL、工具 metadata 刷新等问题，因此**不再需要**早期教程中的 MCP SSE Plugin、Fetch MCP Tools、Call MCP Tool 等插件方案。

实验采用的总体方案为：**极简 Python MCP Server + Streamable HTTP + Dify 1.17.1 原生 MCP + Agent App + `/v1/chat-messages` 手工调用**。全文结构如下：

- **第 1 章**说明实验架构与分层验证策略；
- **第 2 章**完成 Calculator MCP Server 的构建与启动；
- **第 3 章**在接入 Dify 之前，直接对 MCP Server 进行协议级验证；
- **第 4 章**处理 Docker 部署下的网络连通性问题；
- **第 5 章**在 Dify 中注册 MCP Server 并完成工具发现；
- **第 6 章**创建 Agent 应用并在 Dify UI 内完成端到端验证；
- **第 7 章**通过 Dify Service API 完成最终的自动化闭环验证；
- **第 8 章**给出标准化的故障排查路径与常见故障分析；
- **第 9 章**总结最小实验路径，并说明向 Prometheus MCP 等真实场景迁移的方法。

---

# 第一章　实验架构与验证策略

## 1.1 总体测试架构

整个测试链路如下：

```text
curl
  │
  │ POST /v1/chat-messages
  ▼
Dify 1.17.1
  │
  │ Agent / Function Calling
  ▼
MCP Tool
  │
  │ Streamable HTTP
  │ POST /mcp
  ▼
Calculator MCP Server
  │
  ├── add(a,b)
  ├── subtract(a,b)
  ├── multiply(a,b)
  └── divide(a,b)
```

## 1.2 分层验证策略

实验中需要严格区分两个验证层次：

```text
第一层：直接测试 MCP Server
curl / MCP Inspector
        ↓
确认 MCP Server 自身工作正常

第二层：通过 Dify 调用 MCP
curl → Dify API → Agent → MCP Server
        ↓
确认真正的 Dify + MCP 闭环
```

必须严格按照"先第一层、后第二层"的顺序执行。这样做的意义在于：当第二层出现故障时，可以确保问题不在 MCP Server 侧，从而将排查范围收敛到网络、Dify MCP Client 或 LLM Tool Calling 环节。

---

# 第二章　构建 Calculator MCP Server

当前 MCP 官方推荐远程 Server 使用 **Streamable HTTP** 传输。Streamable HTTP 自 MCP 2025-03-26 起取代旧的 HTTP+SSE Transport，本文不再使用 SSE 方案。

## 2.1 创建 Python 环境

Python虚拟环境可使用python3自带的venv，也可以使用conda。但简单的代码实例，venv更加轻量。

```bash
mkdir -p ~/calculator-mcp
cd ~/calculator-mcp

python3 -m venv .venv
source .venv/bin/activate
```

我们这里使用新发布的MCP 2.X，首先要安装官方 MCP Python SDK：

```bash
pip install -U "mcp[cli]"
```

> 注意：2026 年 7 月 28 日 MCP Python SDK 发布了 v2.0.0 稳定版，pip install mcp 现在默认安装 2.x。v2 是一次重大重构：FastMCP 被重命名为 MCPServer（虽然装饰器 API 保持不变），且 mcp.server.fastmcp 的导入路径已被指向迁移指南。官方明确建议：如果项目还没完成 v2 迁移，必须给依赖加 <2 上限，例如 mcp>=1.28,<2。

该 SDK 原生支持 stdio、Streamable HTTP 以及旧版 SSE 三种传输方式。安装完成后验证：

```bash
python -c "import mcp; print('MCP SDK OK')"
```

## 2.2 编写 MCP Server

创建 `calculator_server.py`：

```python
from mcp.server import MCPServer
from mcp.server.transport_security import TransportSecuritySettings


mcp = MCPServer("calculator-mcp")


@mcp.tool()
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b


@mcp.tool()
def subtract(a: float, b: float) -> float:
    """Subtract b from a."""
    return a - b


@mcp.tool()
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b


@mcp.tool()
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    if b == 0:
        raise ValueError("b must not be zero")
    return a / b


if __name__ == "__main__":
    security = TransportSecuritySettings(
        enable_dns_rebinding_protection=False
    )

    mcp.run(
        transport="streamable-http",
        host="0.0.0.0",
        port=9000,
        streamable_http_path="/mcp",
        stateless_http=True,
        json_response=True,
        transport_security=security,
    )
```

### 2.2.1 关于 DNS Rebinding Protection 的说明

上述代码在测试环境中**有意关闭了 DNS rebinding protection**：

```python
enable_dns_rebinding_protection=False
```

其原因并非 MCP 协议要求，而是避免 Dify 通过 Docker 容器、IP 或主机名访问时，因 `Host` 头不在 allowlist 中而直接收到如下错误：

```text
421 Misdirected Request
Invalid Host header
```

这是当前 MCP Python SDK 中一个常见的配置陷阱。MCP 官方文档明确要求：对外部署时应配置 `TransportSecuritySettings`。**生产环境必须配置明确的 `allowed_hosts`，而不是关闭保护**，例如：

```python
TransportSecuritySettings(
    allowed_hosts=[
        "mcp.magedu.com",
        "mcp.magedu.com:*"
    ]
)
```

## 2.3 启动 MCP Server

```bash
python calculator_server.py
```

预期输出类似：

```text
Uvicorn running on http://0.0.0.0:9000
```

实际的 MCP Endpoint 为：

```text
http://<MCP服务器IP>:9000/mcp
```

例如 `http://192.168.1.100:9000/mcp`。当前 MCP 的 Streamable HTTP 模式采用统一的 `/mcp` Endpoint，客户端通过 HTTP POST 发送 JSON-RPC 请求。

---

# 第三章　直接验证 MCP Server

## 3.1 验证策略说明

**在接入 Dify 之前，必须先独立完成 MCP Server 的协议级验证。** 否则当 Dify 侧报错时，无法区分问题究竟来自MCP Server、Dify、Dify MCP Client，还是LLM Tool Calling。

另外需要注意的是，`/mcp` 不是普通 REST API Endpoint，因此直接执行 `curl http://localhost:9000/mcp` 不会返回 `{"status":"ok"}` 之类的健康检查结果。正确的验证方式是按协议依次执行 `initialize`、`tools/list`、`tools/call`。

## 3.2 使用 curl 执行 MCP initialize

```bash
curl -i \
  -X POST \
  http://localhost:9000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-test",
        "version": "1.0"
      }
    }
  }'
```

此处协议版本使用 `2025-06-18`，原因是 Dify 1.17.1 的 MCP Client 源码中明确定义了 `LATEST_PROTOCOL_VERSION = "2025-06-18"`，使用该版本进行测试最接近 Dify 的实际行为。

正常响应类似：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18"
  }
}
```

具体字段可能因 MCP SDK 版本略有差异。

## 3.3 测试 tools/list（工具发现）

```bash
curl -s \
  -X POST \
  http://localhost:9000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
  }' | jq
```

正常情况下应返回四个工具：`add`、`subtract`、`multiply`、`divide`，大致结构如下：

```json
{
  "tools": [
    {
      "name": "add",
      "description": "Add two numbers.",
      "inputSchema": {}
    },
    {
      "name": "subtract"
    }
  ]
}
```

Dify 的 MCP Client 同样是通过标准的 `tools/list` 与 `tools/call` 完成工具发现与调用的。

## 3.4 测试 tools/call（工具调用）

以计算 12 + 8 为例：

```bash
curl -s \
  -X POST \
  http://localhost:9000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "add",
      "arguments": {
        "a": 12,
        "b": 8
      }
    }
  }' | jq
```

预期返回：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "20"
      }
    ]
  }
}
```

至此，以下各项均已验证通过：

```text
MCP transport     OK
MCP initialize    OK
tools/list        OK
tools/call        OK
calculator        OK
```

即 MCP Server 自身的问题已经完全排除。

---

# 第四章　Docker 环境下的网络连通性

## 4.1 关键问题：容器内的 localhost

Dify 通常以 Docker Compose 方式部署。此时，`http://localhost:9000/mcp` **不能直接填入 Dify**——对 Dify API 容器而言，`localhost` 指向的是容器自身，而非宿主机。这是 Dify + MCP 配置失败的最常见原因之一。

## 4.2 推荐的三种地址方案

假设 Calculator MCP Server 运行在 Dify 宿主机上：

### 4.2.1 Docker Desktop 环境

通常可以直接使用：

```text
http://host.docker.internal:9000/mcp
```

Dify 自身的配置文件同样以 `host.docker.internal` 作为容器访问宿主机服务的标准方式。

### 4.2.2 Linux Docker 环境：使用宿主机 LAN IP

Linux 环境下若无 `host.docker.internal`，可直接使用宿主机局域网 IP：

```text
http://192.168.1.100:9000/mcp
```

### 4.2.3 推荐方案：加入 Dify Docker 网络

更规范的做法是将 Calculator MCP 容器加入 Dify 的 Docker network，然后直接使用服务名访问：

```text
http://calculator-mcp:9000/mcp
```

## 4.3 从 Dify API 容器验证连通性

在进入 Dify UI 配置之前，强烈建议先从 Dify API 容器内部验证网络连通性。查找 API 容器：

```bash
docker ps
```

假设容器名为 `docker-api-1`，进入容器：

```bash
docker exec -it docker-api-1 bash
# 或
docker exec -it docker-api-1 sh
```

在容器内执行 initialize 测试：

```bash
curl -i \
  -X POST \
  http://host.docker.internal:9000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{
      "protocolVersion":"2025-06-18",
      "capabilities":{},
      "clientInfo":{
        "name":"dify-network-test",
        "version":"1.0"
      }
    }
  }'
```

只有该测试成功后，才应进入 Dify UI 注册 MCP。这样排障路径非常清晰：

```text
Host → MCP            OK
Dify Container → MCP  OK
然后才验证：
Dify MCP Client → MCP
```


---

# 第五章　在 Dify 中注册 MCP Server

## 5.1 进入 MCP 集成入口

Dify 1.17.x 中，MCP 已属于工作区级原生 Integration，UI 入口为“Dify” → “Workspace” →  “Integrations” →  “MCP”。官方 UI 对该功能的描述为：

> Connect and manage MCP servers to give your apps access to external tools and services.

选择 **Add MCP Server** 即可开始注册。

## 5.2 填写 Server 信息

```text
Name:
Calculator MCP

Server URL:
http://host.docker.internal:9000/mcp
```

根据 MCP Server 的实际部署位置，URL 也可能是类似如下这样格式的地址：

```text
http://192.168.1.100:9000/mcp      # 局域网内其他主机
https://mcp.magedu.com/mcp        # HTTPS 域名
```

需要特别注意的是，**URL 必须指向 `/mcp` 本身**，也就是必须也只能附件 /mcp PATH。

```text
错误：http://host:9000
错误：http://host:9000/sse
正确：http://host:9000/mcp
```

早期 Dify MCP 曾使用 HTTP+SSE，而现代 MCP 已推荐 Streamable HTTP，不应再依据 2025 年早期教程创建或使用 `/mcp/sse` 端点。

## 5.3 认证配置

本实验的 Calculator MCP Server 未启用认证，因此选择：

```text
Authentication: None / No Authentication
```

在最小验证阶段，不建议引入 OAuth、Bearer Token、API Key 等认证机制，以免不必要地增加问题的复杂度。

## 5.4 注册过程的实际行为

点击连接时，Dify 内部会自动执行一些功能，其逻辑大致为：

```text
Dify
 │
 ├─ initialize
 │
 └─ tools/list
       │
       ▼
MCP Server
       │
       ├─ add
       ├─ subtract
       ├─ multiply
       └─ divide
```

注册成功后，Dify 的 MCP Server 页面应显示工具列表，但此阶段尚未涉及 LLM 调用，它仅证明链路成功：

```text
Dify MCP Client → MCP Server → tool discovery
```

---

# 第六章　构建 Agent 应用并完成 UI 内验证

## 6.1 创建最小 Agent App

```text
Studio
  ↓
Create App
  ↓
Agent
```

应用名称示例：`MCP Calculator Test`。

## 6.2 为 Agent 添加 MCP 工具

进入 Agent 的工具配置区域，选择 **Add Tool**，应能看到已注册的 `Calculator MCP`。勾选全部四个工具：

```text
Agent

Model
  └── 支持 Tool Calling 的模型

Tools
  ├── Calculator MCP.add
  ├── Calculator MCP.subtract
  ├── Calculator MCP.multiply
  └── Calculator MCP.divide
```

测试阶段建议全部勾选，避免遗漏。

## 6.3 模型必须支持 Tool Calling

这一步非常关键：**MCP Server 注册成功，并不意味着 LLM 一定会调用 MCP 工具**。真实链路是：

```text
用户
 ↓
LLM
 ↓
判断是否调用 tool
 ↓
tool_call
 ↓
Dify
 ↓
MCP Server
```

因此所选模型必须可靠支持 Function Calling / Tool Calling。例如，基于vLLM运行Qwen3.5时，必须使用“--enable-auto-tool-choice”和“--tool-call-parser qwen3_xml”选项才能适合本实验。

## 6.4 推荐的测试用 System Prompt

最小验证阶段不应让模型自由发挥，应通过明确的 System Prompt 约束其行为：

```text
You are a calculator test agent.

For any arithmetic request involving addition, subtraction,
multiplication, or division, you MUST use the corresponding
calculator MCP tool.

Do not calculate arithmetic yourself.

Available operations:
- addition: use add
- subtraction: use subtract
- multiplication: use multiply
- division: use divide

Return the final calculation result after the tool call.
```

其目的是将"MCP 链路是否正常"与"模型是否愿意调用工具"两个问题尽量分离，便于定位故障。

## 6.5 在 Dify UI 内执行首次端到端测试

输入：

```text
请计算 123 + 456。
必须使用工具完成，不要自己计算。
```

理想的执行轨迹为：

```text
User
 ↓
LLM
 ↓
tool_call

name: add
arguments: {"a": 123, "b": 456}

 ↓
Dify MCP Client
 ↓
POST /mcp
tools/call
add(123,456)
 ↓
579
 ↓
LLM
 ↓
最终答案：579
```

## 6.6 检查 Dify Run Log

这是非常关键的验证步骤。进入 Logs / Run History / Trace，应能在 Agent 执行过程中看到 Tool Call 记录：

```json
{
  "name": "add",
  "arguments": {
    "a": 123,
    "b": 456
  }
}
```

以及返回结果 `579`。如果 MCP Server 控制台同时输出了对应的请求日志，则形成两端相互印证的证据链：

```text
Dify Trace + MCP Server Log
```

## 6.7 发布 Agent 并获取 API 凭证

UI 测试成功后，执行 **Publish**，然后进入 Access → API（或 API Access），创建 API Key，例如：

```text
app-xxxxxxxxxxxxxxxx
```

同时确认 Dify Service API 地址。假设 API 地址为 `http://192.168.1.20/v1`，则：

```text
DIFY_BASE_URL=http://192.168.1.20/v1
```

---

# 第七章　通过 Dify API 完成最终闭环验证

## 7.1 调用 /v1/chat-messages

Dify 的 Chat / Agent 应用通过 `POST /v1/chat-messages` 提供服务。设置环境变量：

```bash
export DIFY_URL="http://172.29.0.165/v1"
export DIFY_API_KEY="app-xxxxxxxxxxxxxxxx"
```

发起调用：

```bash
curl -N \
  -X POST \
  "${DIFY_URL}/chat-messages" \
  -H "Authorization: Bearer ${DIFY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {},
    "query": "请计算 123 * 456，必须使用计算器工具。",
    "response_mode": "streaming",
    "conversation_id": "",
    "user": "mcp-test-user"
  }'
```

## 7.2 为什么必须使用 streaming 模式

这是 Dify 1.17.1 中需要特别注意的一点。对于新版 Agent App，`"response_mode": "blocking"` 可能直接失败——Dify 后端代码明确注明：

```text
New Agent app mode supports streaming only.
```

因此必须使用 `"response_mode": "streaming"`，不要沿用旧教程中的 blocking 模式。同时，curl 命令中的 `-N` 参数用于禁用输出缓冲，确保 SSE 流式数据实时可见。

## 7.3 正常的流式返回

正常情况会看到一系列 SSE 数据块：

```text
data: {...}

data: {...}

data: {...}
```

最终汇总的 answer 为：

```text
56088
```

也可能是多个 message chunk 分段输出（如 `"56"`、`"088"`），拼接后得到完整结果。

## 7.4 如何证明结果不是 LLM 自行计算的

不能仅凭答案 `56088` 判断链路正确，必须同时检查 Dify Trace，确认其中存在真实的 Tool Call：

```text
Agent

Thought / Tool Call

multiply
{"a": 123, "b": 456}

Tool Result

56088
```

同时，MCP Server 日志中应出现对应的 `POST /mcp` 记录。至此，实际验证的完整链路为：

```text
curl
 ↓
Dify Service API
 ↓
Agent Runtime
 ↓
LLM Tool Calling
 ↓
Dify MCP Client
 ↓
MCP Streamable HTTP
 ↓
multiply(123,456)
 ↓
56088
 ↓
LLM
 ↓
Dify SSE
 ↓
curl
```

这才是真正完整的端到端闭环。

## 7.5 四个确定性测试

建议对四种运算分别执行确定性验证：

```text
123 + 456  → 579
1000 - 333 → 667
12 * 25    → 300
100 / 8    → 12.5
```

以除法为例：

```bash
curl -N \
  -X POST \
  "${DIFY_URL}/chat-messages" \
  -H "Authorization: Bearer ${DIFY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {},
    "query": "请使用 divide 工具计算 100 / 8。",
    "response_mode": "streaming",
    "user": "mcp-test-user"
  }'
```

预期返回 `12.5`。

## 7.6 错误处理测试

建议补充异常场景测试，例如除零：

```bash
curl -N \
  -X POST \
  "${DIFY_URL}/chat-messages" \
  -H "Authorization: Bearer ${DIFY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {},
    "query": "必须使用工具计算 100 / 0。",
    "response_mode": "streaming",
    "user": "mcp-test-user"
  }'
```

预期行为是 MCP Tool 返回错误，而不是导致 MCP Server 崩溃。该测试同时验证了三个层面：

```text
正常 Tool Call
+
Tool Error
+
Dify Agent Error Handling
```


---

# 第八章　故障排查

## 8.1 十层排查链

当 Dify 添加 MCP Server 或调用失败时，不建议盲目翻查日志，而应严格按照以下顺序逐层定位：

```text
① MCP Server 是否监听？
   ss -lntp | grep 9000

② Host 本机 initialize 是否成功？
   curl → localhost:9000/mcp

③ Dify API 容器能否访问 MCP Server？
   docker exec dify-api ...
   curl → MCP_IP:9000/mcp

④ initialize 是否成功？

⑤ tools/list 是否成功？

⑥ Dify MCP 页面是否显示 4 个 Tools？

⑦ Agent 是否已加载 Tools？

⑧ LLM 是否生成 tool_call？

⑨ MCP Server 是否收到 tools/call？

⑩ Dify 是否收到 Tool Result？
```

通过这十层检查，可以快速将问题定位到具体环节。

## 8.2 三个最常见的故障

### 8.2.1 localhost 指向错误

典型现象：Dify 中填写 `localhost:9000` 后连接失败。原因是 Dify 容器内的 `localhost` 指向容器自身。解决方法是改用 `host.docker.internal`、宿主机 LAN IP，或将 MCP Server 加入 Dify 的 Docker 网络后使用服务名。

### 8.2.2 HTTP 421 Misdirected Request

如果出现：

```text
421 Misdirected Request
Invalid Host header
```

这是 MCP Python SDK 的 DNS rebinding protection 触发，并非 Dify 或 MCP 协议本身的错误。处理方式：

- **实验环境**：`TransportSecuritySettings(enable_dns_rebinding_protection=False)`；
- **生产环境**：配置明确的 `allowed_hosts`（见 2.2.1 节），不得直接关闭保护。

### 8.2.3 工具可发现但 Agent 不调用

现象：MCP 注册成功、工具列表正常，但 Agent 不产生 `tool_call`。这说明：

```text
MCP        OK
网络       OK
tools/list OK
```

问题范围已收敛到 **LLM Tool Calling** 环节。Dify 社区中亦有同类案例：MCP tool discovery 成功，但模型未产生工具调用。此时应首先检查 Dify Run Log 中的原始 LLM 输出，并确认：

1. 模型本身支持 Function Calling / Tool Calling；
2. 推理服务（如 vLLM）已启用 `--enable-auto-tool-choice` 及匹配的 tool-call parser；
3. System Prompt 已明确要求 `MUST use calculator tools. Do not calculate by yourself.`

---

# 第九章　实验总结与进阶路径

## 9.1 推荐的最小实验路径

如果实验的最终目标是构建"Alertmanager → Dify Webhook → Workflow / Agent → Prometheus MCP"的 AI-SRE 自动诊断链路，不建议直接编写复杂的 MCP Server，而应先使用 Calculator MCP 完成以下五个递进实验：

```text
实验 1：curl → MCP → tools/list

实验 2：curl → MCP → tools/call

实验 3：Dify → MCP → Tool Test

实验 4：Dify Agent UI → LLM → MCP

实验 5：curl → Dify /v1/chat-messages → LLM → MCP → calculator
```

当实验 5 成功后，再将 Calculator MCP 替换为 Prometheus MCP。对 Dify 而言，二者的差异仅在于 `tools/list` / `tools/call` 返回的 Tool Schema 不同，链路与验证方法完全一致。

## 9.2 最终架构回顾

对于 Dify 1.17.1，合理的 MCP 验证方案不再是旧式的"安装 MCP SSE 插件 → Fetch MCP Tools → Call MCP Tool"，而是：

```text
                    ┌── add
                    ├── subtract
MCP Server /mcp ────┼── multiply
                    └── divide
        │
        │ Streamable HTTP
        ▼
Dify 1.17.1
Integrations → MCP
        │
        ▼
Agent
        │
        ▼
LLM Tool Calling
        │
        ▼
POST /v1/chat-messages
```

这套链路可以作为后续 **Dify + Prometheus MCP + AI-SRE 自动诊断** 的"Hello World"。Dify V1.17.1 本身包含针对 MCP Server URL、metadata 刷新等问题的修复，直接基于其原生 MCP 能力开展实验是当前最合适的选择。

---

# 附录　参考资料

1. Dify 仓库 UI 文案（Workspace MCP Integrations）：<https://github.com/langgenius/dify/blob/main/web/i18n/en-US/common.json>
2. MCP Python SDK 运行与传输文档：<https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/index.md>
3. MCP Python SDK 项目主页：<https://github.com/modelcontextprotocol/python-sdk>
4. MCP Python SDK 故障排查文档（DNS rebinding protection）：<https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/troubleshooting.md>
5. Dify MCP Client 协议版本定义：<https://github.com/langgenius/dify/blob/main/api/core/mcp/types.py>
6. Dify 环境变量示例（host.docker.internal）：<https://github.com/langgenius/dify/blob/main/api/.env.example>
7. Dify Service API chat-messages 实现：<https://github.com/langgenius/dify/blob/main/api/controllers/service_api/app/completion.py>
8. Dify 社区讨论：Agent 未执行 tool call：<https://github.com/langgenius/dify/discussions/31751>
9. Dify Releases（V1.17.1 修复说明）：<https://github.com/langgenius/dify/releases>
