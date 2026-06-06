# HolmesGPT 插件机制详解

> 本文档深入说明 HolmesGPT 的插件（Plugin）体系如何工作，并**以集成 Elasticsearch 为例**，从配置加载到工具调用，完整演示一个插件的全生命周期。

---

## 目录

1. [插件体系总览](#1-插件体系总览)
2. [工具集（Toolset）工作机制](#2-工具集toolset工作机制)
   - 2.1 什么是工具集
   - 2.2 工具的两种定义方式
   - 2.3 工具集的生命周期
3. [数据源插件（Source Plugin）工作机制](#3-数据源插件source-plugin工作机制)
4. [通知目标插件（Destination Plugin）工作机制](#4-通知目标插件destination-plugin工作机制)
5. [提示词插件（Prompt Plugin）工作机制](#5-提示词插件prompt-plugin工作机制)
6. [案例：Elasticsearch 工具集深度解析](#6-案例elasticsearch-工具集深度解析)
   - 6.1 功能概览
   - 6.2 配置类设计
   - 6.3 前提检查机制
   - 6.4 工具定义详解
   - 6.5 请求执行流程
   - 6.6 多实例路由
   - 6.7 JSON 过滤机制
   - 6.8 错误处理策略
   - 6.9 两个工具集的分工
7. [YAML 工具集与自定义工具集](#7-yaml-工具集与自定义工具集)
8. [最佳实践与设计原则](#8-最佳实践与设计原则)

---

## 1. 插件体系总览

HolmesGPT 的插件体系分为四种类型，它们共同构成了系统的可扩展性基础：

```
插件体系
├── 工具集（Toolset）       — 核心扩展点，为 LLM 提供可调用的工具
│   ├── Python 工具集        — 用 Python 类实现的工具集（Prometheus、Grafana、Elasticsearch...）
│   ├── YAML 工具集          — 用 YAML 定义的工具集（bash 命令方式）
│   ├── MCP 工具集           — 通过 MCP 协议连接的远程工具集
│   ├── HTTP 工具集          — 封装 REST API 的工具集
│   └── Database 工具集      — SQL 数据库查询工具集
│
├── 数据源插件（Source Plugin）  — 从外部系统获取事件/告警
│   ├── AlertManager
│   ├── Jira
│   ├── GitHub
│   ├── PagerDuty
│   └── OpsGenie
│
├── 通知目标插件（Destination Plugin） — 将结果发送到外部系统
│   ├── Slack
│   └── PagerDuty
│
└── 提示词插件（Prompt Plugin） — Jinja2 模板化的提示词片段
    ├── generic_ask.jinja2       — 主系统提示词模板
    ├── base_user_prompt.jinja2  — 用户提示词模板
    └── _ticket_additions.jinja2 — 工单调查附加内容
```

**最常用也最核心的是工具集**，它是 LLM 与外部世界交互的桥梁。本文重点介绍工具集的机制，并以 Elasticsearch 为完整案例进行说明。

---

## 2. 工具集（Toolset）工作机制

### 2.1 什么是工具集

工具集（Toolset）是一组相关工具的集合，每个工具都是 LLM 可以调用的一个"能力"。比如 Elasticsearch 工具集包含了搜索文档、查询集群健康、列出索引等多个工具。

从 LLM 的视角看，工具就像函数调用（function calling）：

```
LLM 思考 → "我需要查看集群健康状态"
LLM 调用 → elasticsearch_cluster_health(elasticsearch_instance="prod-eu")
工具返回 → {"cluster_name": "prod-eu", "status": "yellow", ...}
LLM 阅读结果 → "状态是 yellow，有未分配的分片"
LLM 再调用 → elasticsearch_cat(elasticsearch_instance="prod-eu", endpoint="shards")
...直到得出结论...
```

### 2.2 工具的两种定义方式

**方式一：Python 类（推荐用于复杂集成）**

每个工具是一个继承 `Tool` 基类的 Python 类，核心方法 `_invoke(params, context)` 接收参数并返回 `StructuredToolResult`：

```python
class MyTool(Tool):
    def __init__(self, toolset):
        super().__init__(
            name="my_tool",
            description="做什么用的",
            parameters={
                "param1": ToolParameter(type="string", required=True, description="..."),
            },
        )
        self._toolset = toolset

    def _invoke(self, params, context) -> StructuredToolResult:
        # 执行逻辑...
        return StructuredToolResult(status=StructuredToolResultStatus.SUCCESS, data=result)
```

**方式二：YAML 定义（适用于简单 bash 命令）**

在 YAML 文件中定义工具，适合调用 shell 命令执行查询：

```yaml
toolsets:
  my_toolset:
    enabled: true
    prerequisites:
      - env: [API_ENDPOINT]  # 前置条件：环境变量
    tools:
      - name: my_tool
        description: "查询外部系统"
        command: "curl -s '{{api_endpoint}}/api/v1/status'"
```

YAML 方式适合简单的 HTTP 调用或 shell 命令组合，不需要复杂的错误处理和参数校验。

### 2.3 工具集的生命周期

工具集从加载到使用，经历以下阶段：

```
加载阶段
────────────────────────────────────────────
1. 启动时：ToolsetManager 收集所有工具集
   ├── 内置工具集（Python 类 + YAML 文件）
   ├── 配置文件中的工具集（覆盖内置配置）
   ├── 自定义工具集（用户提供的 YAML）
   └── CLI 参数传递的工具集


前提检查阶段
────────────────────────────────────────────
2. 并行检查所有工具集的前提条件
   ┌─────────────────────────────────────┐
   │ ToolsetManager.check_toolset_prerequisites()
   │   ├── ThreadPoolExecutor 并行执行
   │   ├── 每个工具集独立超时（默认 20 秒）
   │   └── 超时的工具集标记为 FAILED
   └─────────────────────────────────────┘

   每个工具集的检查逻辑不同：
   ├── Elasticsearch：尝试连接 _cluster/health 端点
   ├── Prometheus：检查 PromQL API 是否可访问
   ├── MCP 工具集：连接 MCP 服务器并加载工具列表
   └── YAML 工具集：检查环境变量是否设置


结果缓存阶段
────────────────────────────────────────────
3. 状态写入 toolsets_status.json 缓存
├── ENABLED：前提通过，工具可用
├── FAILED：前提失败（连接不上、认证失败等）
└── DISABLED：用户显式禁用了


懒加载优化
────────────────────────────────────────────
4. 大部分工具集在启动时只做快速配置校验
├── 实际连接检查（callable prerequisite）延迟到首次调用
├── MCP 工具集除外：必须立即初始化以获取工具列表
└── 前提检查超时的工具集标记失败但不阻塞启动


工具调用阶段
────────────────────────────────────────────
5. Agentic Loop 中 LLM 调用工具
┌─────────────────────────────────────────┐
│ 1. LLM 决定调用工具           (思考)      │
│ 2. LLM 生成 tool_call + 参数  (函数调用)  │
│ 3. ToolExecutor 解析参数并调用 _invoke()  │
│ 4. 返回 StructuredToolResult   (结果)     │
│ 5. LLM 阅读结果，继续推理      (再思考)    │
└─────────────────────────────────────────┘
```

---

## 3. 数据源插件（Source Plugin）工作机制

数据源插件解决的是"从哪里获取需要调查的问题"。当用户执行 `holmes investigate alertmanager` 时，系统不直接调用工具集获取数据，而是先通过 Source Plugin 拉取待处理的告警或工单列表，然后对每条逐一分析。

**接口定义：**

```python
class SourcePlugin:
    def fetch_issues(self) -> List[Issue]:
        """获取所有待处理的事件列表"""
        pass

    def fetch_issue(self, id: str) -> Issue:
        """获取单个事件的详细信息"""
        pass

    def write_back_result(self, issue_id, result):
        """将 AI 分析结果写回源系统（可选）"""
        pass
```

**完整调用流程（以 AlertManager 为例）：**

```
用户输入：holmes investigate alertmanager "检查所有告警"

Step 1: 创建数据源
  Config.create_alertmanager_source()
  → AlertManagerSource(url="http://alertmanager:9093")

Step 2: 获取告警列表
  source.fetch_issues()
  → GET /api/v2/alerts
  → 返回 List[Issue]（每条包含告警名称、标签、描述）

Step 3: 逐条分析
  for issue in issues:
    _investigate_issue(ai, issue, config)
    → 构建提示词：system_prompt + "调查这个告警"
    → LLM 调用工具集查询 Prometheus 指标、日志等
    → 输出诊断结论

Step 4: 可选写回
  source.write_back_result(issue.id, llm_result)
  → 将分析结果作为评论添加到 Jira 工单
```

**现有的数据源实现：**

| 插件 | 用途 | 是否支持写回 |
|------|------|-------------|
| AlertManager | 获取 Prometheus 告警 | 否 |
| Jira | 获取 Jira 工单 | 是（添加评论） |
| GitHub | 获取 GitHub Issue | 是（添加评论） |
| PagerDuty | 获取 PagerDuty 事件 | 是（更新事件） |
| OpsGenie | 获取 OpsGenie 告警 | 是 |

---

## 4. 通知目标插件（Destination Plugin）工作机制

通知目标插件决定了分析结果的去向。默认情况下结果输出到终端（CLI 模式），但也可以配置发送到外部系统。

**接口定义：**

```python
class DestinationPlugin:
    def send_issue(self, issue: Issue, result: LLMResult):
        """将分析结果发送到目标系统"""
        pass
```

**使用场景：**

```
在定时健康检查中：
  Check 失败 + mode=alert + destinations=[{type: "slack", config: {...}}]
  → Checks API 创建 SlackDestination 实例
  → slack.send_issue(issue, llm_result)
  → 发送到指定 Slack 频道

在 CLI 中：
  holmes ask "检查集群状态" --destination slack --slack-token xoxb-... --slack-channel #ops
  → handle_result() 调用 SlackDestination
  → 结果发送到 Slack 频道
```

---

## 5. 提示词插件（Prompt Plugin）工作机制

提示词插件是 Jinja2 模板文件，位于 `holmes/plugins/prompts/` 目录。它们控制 LLM 的行为指令。

**模板加载方式：**

```python
# builtin://前缀 → 从内置 prompts 目录加载
prompt = load_and_render_prompt("builtin://generic_ask.jinja2", context={...})

# file://前缀 → 从文件系统加载
prompt = load_and_render_prompt("file:///path/to/template.jinja2")

# 纯字符串 → 直接使用
prompt = load_and_render_prompt("你是一个运维专家，请帮我分析...")
```

**提示词组件化控制：**

系统的提示词被拆分为多个组件，每个组件可以独立启用或禁用：

| 组件 | 控制开关 | 内容 |
|------|----------|------|
| `INTRO` | 默认启用 | 模型自我介绍 |
| `ASK_USER` | 默认启用 | 允许反问用户 |
| `TODOWRITE_INSTRUCTIONS` | 默认启用 | TodoWrite 工具使用说明 |
| `TOOLSET_INSTRUCTIONS` | 默认启用 | 可用工具列表和描述 |
| `STYLE_GUIDE` | 默认启用 | 输出风格要求 |
| `CLUSTER_NAME` | 默认启用 | 集群名称信息 |
| `FILES` | 默认启用 | 附件文件内容注入 |
| `TODOWRITE_REMINDER` | 默认启用 | 每轮提醒使用 TodoWrite |

可以通过环境变量 `ENABLED_PROMPTS` 或 `prompt_component_overrides` 参数控制：

```python
# 快速模式：跳过 TodoWrite 提示
prompt_component_overrides = {
    PromptComponent.TODOWRITE_INSTRUCTIONS: False,
    PromptComponent.TODOWRITE_REMINDER: False,
}
```

**关键模板文件：**

| 文件 | 用途 | 渲染时机 |
|------|------|---------|
| `generic_ask.jinja2` | 主系统提示词 | 每次请求开始时 |
| `base_user_prompt.jinja2` | 用户问题封装 | 每次请求开始时 |
| `_ticket_additions.jinja2` | 工单调查额外指令 | 调查 Jira/PagerDuty 时 |
| `_investigation_additions.jinja2` | 告警调查额外指令 | 调查 AlertManager 时 |

---

## 6. 案例：Elasticsearch 工具集深度解析

### 6.1 功能概览

Elasticsearch 工具集实际上是**两个工具集**，它们共享相同的底层代码但提供不同粒度的工具：

```
elasticsearch/data（数据工具集）
├── elasticsearch_search          — Query DSL 搜索文档
├── elasticsearch_mappings        — 查看索引字段映射
├── elasticsearch_list_indices    — 列出索引
└── elasticsearch_data_list_instances — 列出 ES 实例

elasticsearch/cluster（集群工具集）
├── elasticsearch_cluster_health     — 集群健康状态
├── elasticsearch_cat                — _cat API（shards/indices/nodes...）
├── elasticsearch_index_stats        — 索引统计
├── elasticsearch_allocation_explain — 分片分配说明
├── elasticsearch_nodes_stats        — 节点统计
└── elasticsearch_cluster_list_instances — 列出 ES 实例
```

这种分工的原因：数据工具集只需要索引级别的读权限，而集群工具集需要集群级别的管理权限。用户可以根据实际权限情况选择启用哪一个。

### 6.2 配置类设计

Elasticsearch 的配置使用 Pydantic 模型，支持多种认证方式和多实例路由。

**配置文件示例（单实例）：**

```yaml
# ~/.holmes/config.yaml
toolsets:
  elasticsearch/data:
    enabled: true
    config:
      api_url: "https://my-cluster.es.cloud.io"
      api_key: "base64encodedapikey=="
      timeout_seconds: 30
```

**多实例配置：**

```yaml
toolsets:
  elasticsearch/data:
    enabled: true
    config:
      username: "elastic"
      password: "{{ env.ES_GLOBAL_PASSWORD }}"  # 全局默认密码
      verify_ssl: true
      timeout_seconds: 30
      instances:
        - name: prod-eu
          api_url: "https://prod-eu.es.internal:9200"
        - name: prod-us
          api_url: "https://prod-us.es.internal:9200"
          password: "{{ env.ES_US_PASSWORD }}"   # 覆盖全局密码
```

**配置类的加载规则：**

配置类 `ElasticsearchConfig` 通过 Pydantic 的 `model_validator` 实现了一系列智能处理：

1. **向后兼容**：废弃的字段名自动映射到新名称（如 `url` → `api_url`）
2. **多实例归一化**：单实例配置自动包装为 `instances` 列表中的一条（名为 "default"）
3. **全局默认继承**：多实例模式下，如果某个实例没有设置认证信息，自动从顶层继承
4. **认证互斥校验**：`api_key` 和 `username+password` 只能选一种
5. **mTLS 全局继承**：`client_cert` 和 `client_key` 自动下推到未设置的实例
6. **实例名称唯一性**：检查 `instances` 中是否有重名

### 6.3 前提检查机制

当工具集加载时，系统调用 `prerequisites_callable` 来验证配置是否有效、服务是否可达：

```python
def prerequisites_callable(self, config):
    # 第一步：验证配置格式
    self.config = ElasticsearchConfig(**config)

    # 第二步：对每个配置的实例做健康检查
    for instance in instances:
        try:
            data = requests.get(f"{instance.api_url}/_cluster/health", ...)
            # 连接成功，记录集群名称和状态
        except HTTPError as e:
            if e.status == 401:
                # 认证失败：给出明确提示
            elif e.status == 403:
                # 权限不足：需要集群级权限
        except SSLError:
            # SSL 错误：检查证书配置
        except ConnectionError:
            # 连接失败：检查网络和 URL
        except Timeout:
            # 超时：建议增加 timeout_seconds

    # 第三步：容忍判定（只要有一个实例可达就算通过）
    if successes_count > 0:
        return True  # 部分成功也算成功
    return False     # 全部失败才算失败
```

这种"容忍性检查"的设计原因：如果你的集群有多个 ES 实例，其中一个暂时不可用不应该让整个工具集不可用。健康的实例仍然可以服务请求。

**前提检查可能的状态消息：**

```
✓ [prod-eu] Connected to 'production-cluster' (status: green)
✗ [prod-us] HTTP 401 from https://prod-us.es.internal:9200: auth failed
✗ [staging] Failed to connect: Connection refused

→ 检查结果：部分成功（prod-eu 可用，prod-us 认证失败，staging 不可达）
→ 工具集状态：ENABLED（因为至少有一个实例可用）
```

### 6.4 工具定义详解

我们以 `ElasticsearchSearch` 工具为例，完整解析一个工具是如何定义的：

```python
class ElasticsearchSearch(BaseElasticsearchTool):
    def __init__(self, toolset):
        super().__init__(
            toolset=toolset,
            name="elasticsearch_search",                          # 工具名称（LLM 调用的标识）
            description=(                                        # 工具描述（LLM 决定何时使用）
                "Execute an Elasticsearch search query using Query DSL. "
                "Supports full Query DSL including bool queries, aggregations, and filters."
            ),
            parameters={                                         # 参数定义（LLM 知道传什么）
                "elasticsearch_instance": ToolParameter(          # 多实例路由用
                    type="string", required=False,
                    description="Name of the ES instance to query",
                ),
                "index": ToolParameter(                          # 必填：索引名
                    type="string", required=True,
                    description="Index name or pattern to search",
                ),
                "query": ToolParameter(                          # Query DSL 查询体
                    type="object", required=False,
                    description='Example: {"bool": {"must": [{"match": {"level": "ERROR"}}]}}',
                ),
                "size": ToolParameter(                           # 返回条数
                    type="integer", required=False,
                    description="Max documents to return (default: 100)",
                ),
                "sort": ToolParameter(                           # 排序
                    type="array", required=False,
                    description='Example: [{"@timestamp": "desc"}]',
                ),
                "source": ToolParameter(                         # 字段过滤
                    type="object", required=False,
                    description="Fields to include/exclude in response",
                ),
                "aggregations": ToolParameter(                   # 聚合查询
                    type="object", required=False,
                    description='Example: {"by_service": {"terms": {"field": "service.keyword"}}}',
                ),
            },
        )
```

**工具定义的三个关键要素：**

1. **name**（工具名称）：LLM 用这个名字来调用工具。命名风格使用 `工具集_功能` 的蛇形命名法。
2. **description**（描述）：最重要！LLM 阅读这个描述来决定"当前情况是否应该调用这个工具"。描述要写清楚工具的用途、适用场景和限制。
3. **parameters**（参数定义）：每个参数的类型、是否必填、描述约束了 LLM 传入什么样的数据。描述要写清楚格式和示例。

**工具执行时发生了什么：**

当 LLM 决定调用 `elasticsearch_search` 工具时，系统内部经历：

```
LLM 生成 tool_call:
  elasticsearch_search(index="logs-*", query={...}, size=50)

↓

ToolExecutor 查找工具定义
  → 在 elasticsearch/data 工具集中找到 elasticsearch_search

↓

调用 _invoke(params, context)
  params = {"index": "logs-*", "query": {...}, "size": 50}
  context = ToolInvokeContext(llm, tool_name, tool_call_id, ...)

↓

执行 HTTP 请求
  实例: prod-eu (由 _get_instance 从 params 解析)
  URL: https://prod-eu.es.internal:9200/logs-*/_search
  请求体: {"query": {...}, "size": 50}

↓

返回 StructuredToolResult
  status: SUCCESS
  data: {"hits": {"total": 1234, "hits": [...]}}
  params: {"index": "logs-*", ...}

↓

结果格式化为 LLM 消息
  LLM 阅读结果 → "日志中有 1234 条匹配记录，其中..."
  LLM 继续推理或调用下一个工具
```

### 6.5 请求执行流程

Elasticsearch 工具集的 HTTP 请求流程如下：

```python
def _make_request(self, instance, method, endpoint, params, body=None):
    # 1. 构建 URL
    url = f"{instance.api_url.rstrip('/')}/{endpoint.lstrip('/')}"

    # 2. 构建认证头
    headers = {"Accept": "application/json", "Content-Type": "application/json"}
    if instance.api_key:
        headers["Authorization"] = f"ApiKey {instance.api_key}"

    # 3. 可选 mTLS 证书
    cert = (instance.client_cert, instance.client_key) if ... else None

    # 4. 执行 HTTP 请求
    response = requests.request(
        method=method, url=url,
        headers=headers,
        auth=HTTPBasicAuth(instance.username, instance.password),
        cert=cert,
        json=body,
        timeout=instance.timeout_seconds or 10,
        verify=instance.verify_ssl,
    )
    response.raise_for_status()

    # 5. 返回 JSON
    return response.json()
```

**发起 HTTP 请求的整个过程还包括：**

```
_make_request 后的结构化返回
  ↓
HTTP 成功 → StructuredToolResult(status=SUCCESS, data=response_json)

HTTP 4xx → StructuredToolResult(
    status=ERROR,
    error="[prod-eu] HTTP 404: index "logs-202401" not found"
)

HTTP 5xx → StructuredToolResult(
    status=ERROR,
    error="[prod-eu] Elasticsearch request failed: HTTP 503: No nodes available"
)

连接超时 → StructuredToolResult(
    status=ERROR,
    error="[prod-eu] Elasticsearch request timed out for endpoint '_cluster/health'"
)

SSL 错误 → StructuredToolResult(
    status=ERROR,
    error="[prod-eu] SSL error: certificate verify failed"
)
```

### 6.6 多实例路由

当配置了多个 Elasticsearch 实例时，LLM 需要知道应该查询哪个实例。系统通过 `elasticsearch_instance` 参数实现路由：

```python
def _get_instance(self, params):
    if len(self._instances) == 1:
        # 只有一个实例，自动选择
        return next(iter(self._instances.values()))

    # 多个实例，必须指定
    requested = params.get("elasticsearch_instance")
    if not requested:
        raise ValueError("elasticsearch_instance is required (prod-eu, prod-us)")
    if requested not in self._instances:
        raise ValueError(f"Unknown instance '{requested}'. Available: prod-eu, prod-us")

    return self._instances[requested]
```

为了让 LLM 知道有哪些实例可用，`ElasticsearchListInstances` 工具提供了实例发现能力：

```
LLM 调用: elasticsearch_data_list_instances()
返回: {"instances": [
    {"name": "prod-eu", "api_url": "https://prod-eu.es.internal:9200"},
    {"name": "prod-us", "api_url": "https://prod-us.es.internal:9200"}
]}

LLM 推理: "我需要查询 prod-eu 集群的日志"
LLM 调用: elasticsearch_search(
    elasticsearch_instance="prod-eu",
    index="logs-*",
    query={"match": {"level": "ERROR"}}
)
```

当只有一个实例时，`elasticsearch_instance` 参数和实例列表工具**自动隐藏**，减少 LLM 的决策负担，也节省 token。

### 6.7 JSON 过滤机制

Elasticsearch 的某些工具返回大量 JSON 数据（如索引映射、索引列表），可能触发 token 溢出。`JsonFilterMixin` 提供了两种客户端过滤方式：

**限制嵌套深度：**

```
LLM 调用: elasticsearch_mappings(index="orders", max_depth=2)
返回:
  {"orders": {"mappings": {"properties": {...truncated at depth 2}}}}
```

**使用 jq 表达式提取特定字段：**

```
LLM 调用: elasticsearch_list_indices(pattern="logs-*", jq=".[].index")
返回:
  ["logs-20240101", "logs-20240102", "logs-20240103"]
```

**过滤参数自动注入：** 工具通过 `JsonFilterMixin.extend_parameters()` 在原有参数基础上追加 `max_depth` 和 `jq` 两个参数：

```python
class ElasticsearchMappings(BaseElasticsearchTool, JsonFilterMixin):
    def __init__(self, toolset):
        super().__init__(
            ...
            parameters=JsonFilterMixin.extend_parameters({
                "elasticsearch_instance": ...,
                "index": ToolParameter(required=True, ...),
            }),
        )

    def _invoke(self, params, context):
        result = self._make_request(instance, "GET", path, params)
        return self.filter_result(result, params)  # 应用过滤
```

### 6.8 错误处理策略

Elasticsearch 工具集采用了"详细错误信息"策略，这是 HolmesGPT 工具开发的一个核心原则：

**工具必须返回详细的错误信息，包括执行的查询、参数、API 错误详情，以便 LLM 自我纠正。**

```python
# ✅ 好的错误处理：返回足够的信息让 LLM 能自己修正
except HTTPError as e:
    return StructuredToolResult(
        status=ERROR,
        error=(
            f"[{instance.name}] Elasticsearch request failed for endpoint '{endpoint}': "
            f"HTTP {e.response.status_code}: "
            f"{json.dumps(error_body.get('error', e.response.text[:500]))}"
        ),
        params=params,  # 返回原始参数，方便 LLM 回顾自己传了什么
    )
```

**真实场景示范：**

```
LLM 调用: elasticsearch_search(index="my_index", query={...})

返回错误:
  [prod-eu] Elasticsearch request failed for endpoint 'my_index/_search':
  HTTP 404: index "my_index" not found

LLM 推理: "索引不存在，可能是我拼写错了，让我先列出索引看看"
LLM 调用: elasticsearch_list_indices(pattern="*")

返回:
  ["orders-202401", "orders-202402", "products-202401"]

LLM 推理: "原来叫 orders-202401，我修正查询"
LLM 调用: elasticsearch_search(index="orders-202401", query={...})

返回成功 → 最终给出答案
```

这种设计让 LLM 可以在不依赖人工干预的情况下自我修正，是 Agent 系统稳定运行的关键。

### 6.9 两个工具集的分工

ElasticsearchDataToolset 和 ElasticsearchClusterToolset 都继承自同一个基类，但行为有所不同：

```python
# 文件末尾注册
class ElasticsearchDataToolset(ElasticsearchBaseToolset):
    def __init__(self):
        super().__init__(
            name="elasticsearch/data",     # 名字带斜杠，便于分类
            description="查询 ES 索引中的数据",
            tools=[
                ElasticsearchListInstances(self),
                ElasticsearchSearch(self),       # 查询文档
                ElasticsearchMappings(self),     # 查看字段映射
                ElasticsearchListIndices(self),  # 列出索引
            ],
        )

class ElasticsearchClusterToolset(ElasticsearchBaseToolset):
    def __init__(self):
        super().__init__(
            name="elasticsearch/cluster",
            description="诊断集群健康问题",
            tools=[
                ElasticsearchListInstances(self),
                ElasticsearchCat(self),             # _cat API
                ElasticsearchClusterHealth(self),    # 集群健康
                ElasticsearchIndexStats(self),       # 索引统计
                ElasticsearchAllocationExplain(self),# 分片分配
                ElasticsearchNodesStats(self),       # 节点统计
            ],
        )
```

两个工具集共享同一个配置类 `ElasticsearchConfig` 和同样的 `_make_request` / `_get_instance` 等基础设施。用户在配置文件中可以分别启用或禁用：

```yaml
toolsets:
  elasticsearch/data:
    enabled: true          # 启用数据查询工具
    config:
      api_url: "..."
      api_key: "..."

  elasticsearch/cluster:
    enabled: false         # 禁用集群诊断工具
```

**为什么要分成两个工具集？**

1. **权限分离**：数据查询只需要索引级别权限，集群诊断需要集群管理权限
2. **工具数量控制**：如果不拆分，一个工具集会包含 10 个工具，token 消耗更大
3. **按需启用**：用户可以根据自己的权限和需求选择启用哪一组
4. **职责隔离**：数据工具集用于日常查询，集群工具集用于故障诊断

---

## 7. YAML 工具集与自定义工具集

除了 Python 工具集，用户还可以通过 YAML 配置文件自定义工具集。

**YAML 工具集示例：**

```yaml
# ~/.holmes/config.yaml 或自定义工具集文件
toolsets:
  my_custom_api:
    enabled: true
    type: http
    config:
      base_url: "https://api.example.com"
      api_key: "{{ env.MY_API_KEY }}"
      timeout_seconds: 30
    tools:
      - name: query_orders
        description: "查询订单数据"
        path: "/api/v1/orders"
        method: GET
        params:
          - name: status
            type: string
            required: false
            description: "订单状态过滤"
```

**自定义 YAML 工具集文件：**

将 YAML 保存为文件，然后在配置中引用：

```yaml
# ~/.holmes/config.yaml
custom_toolsets: ["/path/to/my_toolsets.yaml"]
```

```yaml
# /path/to/my_toolsets.yaml
toolsets:
  my_bash_tool:
    enabled: true
    description: "执行自定义脚本"
    prerequisites:
      - env: [SCRIPT_PATH]
    tools:
      - name: run_health_check
        description: "运行健康检查脚本"
        command: "bash {{env.SCRIPT_PATH}}/health_check.sh {{ params.service }}"
        params:
          - name: service
            type: string
            required: true
            description: "要检查的服务名"
```

YAML 工具集适合：
- 快速集成一个已有的 API 或脚本
- 不需要复杂错误处理的简单查询
- 用户想快速实验而不写 Python 代码的场景

Python 工具集适合：
- 需要复杂参数校验和错误处理
- 需要多步骤逻辑（如获取 token → 查询 → 处理结果）
- 需要集成第三方 SDK 或库
- 需要多实例路由和高级配置

---

## 8. 最佳实践与设计原则

### 工具必须返回详细的错误信息

这是 HolmesGPT 工具开发的**第一原则**。LLM 通过错误信息自我修正，所以每个错误响应都应该包含足够的信息：

```python
# ✅ 好：详细的错误信息
error="[prod-eu] HTTP 400: search_phase_execution_exception: 
       Field 'unknown_field' is not searchable. Available fields: @timestamp, level, message, service.name"

# ❌ 不好：模糊的错误信息  
error="Query failed"
```

### API 端必须支持服务端过滤

避免返回无界数据导致 token 溢出。Elasticsearch 工具集通过 `index` 参数强制用户指定索引：

```python
# ✅ 好：需要指定索引
elasticsearch_cat(endpoint="shards", index="logs-*")
# 只返回 logs-* 索引的分片信息

# ❌ 假设中的反例：返回所有分片（可能导致 25K+ tokens）
elasticsearch_cat(endpoint="shards")
```

当服务端过滤不可用时，使用 `JsonFilterMixin` 提供客户端过滤：

```python
elasticsearch_list_indices(pattern="*", columns="index,health,store.size", sort="store.size:desc")
# 只返回需要的列，按大小排序，限制前几名
```

### 配置向后兼容

当工具集配置字段发生变化时，使用 `extra="allow"` + `model_validator` 处理废弃字段：

```python
class ElasticsearchConfig(ToolsetConfig):
    model_config = {"extra": "allow"}  # 接受未定义的字段

    _deprecated_mappings = {"url": "api_url", "timeout": "timeout_seconds"}

    @model_validator(mode="after")
    def handle_deprecated(self):
        extra = self.model_extra or {}
        if "url" in extra:
            self.api_url = extra["url"]
            logging.warning("'url' is deprecated, use 'api_url' instead")
        return self
```

### 优先使用 requests 库

Python 工具集应使用 `requests` 库而不是专门的 SDK（如 `opensearchpy`）。这样：

- 依赖更少，维护更简单
- 更直接地控制错误处理
- 更容易模拟测试（通过 `responses` 库）
- 不引入额外的依赖冲突风险

Elasticsearch 工具集就是遵循这个原则的典型例子：所有 HTTP 调用都通过 `requests.request()`，而不是使用 `elasticsearch-py`。

### 工具描述要清晰准确

工具的描述是 LLM 决定是否调用的唯一依据。好的描述应该：

```python
# ✅ 好的描述：说明用途、适用场景、注意事项
description=(
    "Get detailed statistics for indices including document count, "
    "store size, indexing rate, and search rate. "
    "Use '_all' for all indices."
),

# ❌ 差的描述：模糊不清
description="Get index stats",
```

参数描述同样重要：

```python
# ✅ 好的参数描述：包含格式示例和使用提示
"query": ToolParameter(
    description=(
        "Elasticsearch Query DSL query object. Example: "
        '{"bool": {"must": [{"match": {"level": "ERROR"}}]}}. '
        "For full-text search use 'match', for exact matches use 'term'."
    ),
    type="object",
),
```

### 单实例时自动简化

当只配置一个实例时，自动隐藏实例选择和实例发现工具：

```python
def _prune_tools_for_single_instance(self):
    if len(self._instances) != 1:
        return                          # 多实例：保留所有工具
    self.tools = [t for t in self.tools
                  if not isinstance(t, ElasticsearchListInstances)]  # 移除发现工具
    for tool in self.tools:
        tool.parameters.pop("elasticsearch_instance", None)          # 移除实例参数
```

这样 LLM 的工具列表更简洁，token 消耗更少，决策路径更短。

### 测试策略

Elasticsearch 工具集的测试使用 `responses` 库模拟 HTTP：

```python
import responses

@responses.activate
def test_elasticsearch_cluster_health():
    responses.add(
        responses.GET,
        "https://es.example.com/_cluster/health",
        json={"cluster_name": "test", "status": "green"},
        status=200,
    )
    # ... 创建工具集、调用工具、验证结果
```

---

> 本文档基于 2026 年 6 月代码库状态编写。Elasticsearch 工具集位于 `holmes/plugins/toolsets/elasticsearch/`，可作为 Python 工具集的参考实现。