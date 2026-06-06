# HolmesGPT 本地开发调试指南

> 本文档面向 HolmesGPT 项目的开发者，涵盖环境搭建、开发流程、测试运行、调试技巧和常见问题排查。

---

## 目录

1. [环境要求](#1-环境要求)
2. [快速开始](#2-快速开始)
3. [项目结构速览](#3-项目结构速览)
4. [配置详解](#4-配置详解)
5. [开发与调试](#5-开发与调试)
6. [测试体系](#6-测试体系)
7. [Docker 部署](#7-docker-部署)
8. [常见问题](#8-常见问题)

---

## 1. 环境要求

### 系统要求

- **Python**：^3.10（推荐 3.11+）
- **包管理器**：[Poetry](https://python-poetry.org/)（推荐全局安装）
- **操作系统**：macOS / Linux / Windows（Windows 下建议使用 WSL2）

### 安装 Poetry

```bash
# macOS / Linux
curl -sSL https://install.python-poetry.org | python3 -

# Windows (PowerShell)
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -

# 验证
poetry --version
```

### 必要的 API Key

至少需要以下之一：

| 提供商 | 环境变量 | 获取方式 |
|--------|----------|----------|
| OpenAI | `OPENAI_API_KEY` | https://platform.openai.com/api-keys |
| Anthropic | `ANTHROPIC_API_KEY` | https://console.anthropic.com/ |
| OpenRouter | `OPENROUTER_API_KEY` | https://openrouter.ai/keys （推荐用于测试） |

---

## 2. 快速开始

### 2.1 克隆并安装依赖

```bash
# 克隆项目
git clone https://github.com/robusta-dev/holmesgpt.git
cd holmesgpt

# 安装所有依赖（含 dev 和 otel 可选组）
poetry install --with dev,otel

# 激活虚拟环境
poetry shell
```

### 2.2 设置 API Key

```bash
# 基础配置
export OPENAI_API_KEY="sk-..."    # 或 ANTHROPIC_API_KEY

# 推荐：使用 OpenRouter 统一接口（成本更低）
export OPENROUTER_API_KEY="sk-or-..."
export MODEL="openrouter/openai/gpt-4.1-mini"
export CLASSIFIER_MODEL="openrouter/openai/gpt-4.1"

# 可选：使用 OpenTelemetry 追踪
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
```

### 2.3 运行第一个命令

```bash
# 直接提问（非交互模式）
poetry run holmes ask "What version of Python is this?" --no-interactive

# 带详细日志的交互模式
poetry run holmes ask "List the pods in my cluster" --verbose
```

如果看到类似 `Model: gpt-4.1, 200K context, 16K max response` 的输出，说明环境配置成功。

### 2.4 创建本地配置文件

```bash
# 创建配置文件目录
mkdir -p ~/.holmes

# 复制示例配置
cp config.example.yaml ~/.holmes/config.yaml

# 编辑配置文件（可选）
vim ~/.holmes/config.yaml
```

配置文件位置由环境变量 `HOLMES_CONFIGPATH_DIR` 控制，默认为 `~/.holmes`。配置文件路径为 `~/.holmes/config.yaml`。

---

## 3. 项目结构速览

```
holmesgpt/
├── holmes/                    # 核心引擎（主要代码库）
│   ├── main.py                # CLI 入口 (Typer)
│   ├── config.py              # 配置系统 (Pydantic)
│   ├── interactive.py         # 交互模式 (prompt_toolkit)
│   ├── toolset_config_tui.py  # 工具集配置编辑器
│   ├── core/
│   │   ├── tool_calling_llm.py  # Agentic Loop 引擎
│   │   ├── tools.py             # 工具抽象基类
│   │   ├── toolset_manager.py   # 工具集管理器
│   │   ├── prompt.py            # 提示词构建系统
│   │   ├── llm.py               # LLM 抽象与模型注册表
│   │   ├── tools_utils/         # 工具执行器、文件存储
│   │   ├── truncation/          # 上下文压缩
│   │   ├── conversations/       # 对话管理
│   │   └── ...                  # OAuth、追踪、反馈等
│   ├── plugins/
│   │   ├── toolsets/            # 所有工具集实现
│   │   ├── prompts/             # Jinja2 提示词模板
│   │   ├── sources/             # 数据源插件
│   │   └── destinations/        # 通知目标插件
│   ├── checks/               # 健康检查系统
│   ├── common/               # 公共配置和工具函数
│   └── utils/                # 工具函数
├── holmes_operator/          # Kubernetes Operator
│   ├── operator.py            # kopf 入口
│   ├── models.py              # CRD 数据模型
│   ├── context.py             # 全局上下文
│   ├── handlers/              # CRD 事件处理器
│   ├── scheduler/             # APScheduler 定时调度
│   └── client/                # Holmes API HTTP 客户端
├── tests/                    # 测试目录
│   ├── llm/                  # LLM 评估测试
│   ├── holmes_operator/      # Operator 测试
│   ├── ...                   # 单元/集成测试
├── docs/                     # 文档
├── server.py                 # FastAPI 服务端入口
└── Dockerfile                # Docker 构建文件
```

---

## 4. 配置详解

### 4.1 配置加载顺序

配置从多个来源合并（后覆盖前）：

1. **默认值**：定义在 `Config` 类的字段默认值中
2. **YAML 配置文件**：`~/.holmes/config.yaml`
3. **CLI 选项**：`--model`, `--api-key` 等
4. **环境变量**：`$MODEL`, `$OPENAI_API_KEY` 等

### 4.2 关键环境变量

#### LLM 模型配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `OPENAI_API_KEY` | OpenAI API Key | — |
| `ANTHROPIC_API_KEY` | Anthropic API Key | — |
| `MODEL` | 模型名称 | `gpt-4.1` |
| `CLASSIFIER_MODEL` | 分类器模型（OpenRouter 时必填） | = MODEL |
| `OPENROUTER_API_KEY` | OpenRouter API Key | — |
| `OPENROUTER_API_BASE` | OpenRouter 端点 | `https://openrouter.ai/api/v1` |
| `TEMPERATURE` | LLM 温度参数 | `0.00000001` |
| `LLM_REQUEST_TIMEOUT` | LLM 请求超时（秒） | `600` |
| `THINKING` | Anthropic 思考预算（token 数） | — |
| `REASONING_EFFORT` | 推理努力程度（`low`/`medium`/`high`） | — |

#### 工具集与运行配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ENABLED_BY_DEFAULT_TOOLSETS` | 默认启用的工具集 | `kubernetes/core,kubernetes/logs,robusta,internet` |
| `DISABLE_PROMETHEUS_TOOLSET` | 禁用 Prometheus | `False` |
| `HOLMES_TOOLSET_PREREQ_TIMEOUT_SECONDS` | 工具集前提检查超时 | `20` |
| `HOLMES_DISABLE_VISION` | 禁用多模态视觉 | `False` |
| `HOLMES_DISABLE_STRICT_TOOL_CALLS` | 禁用严格工具调用 | `False` |
| `BASH_TOOL_UNSAFE_ALLOW_ALL` | 允许所有 bash 命令 | `False` |

#### 服务端配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `HOLMES_HOST` | 服务绑定地址 | `0.0.0.0` |
| `HOLMES_PORT` | 服务端口 | `5050` |
| `LOG_LEVEL` | 日志级别 | `INFO` |
| `DEVELOPMENT_MODE` | 开发模式 | `False` |
| `ENABLE_TELEMETRY` | 启用遥测 | `False` |
| `HOLMES_TRACE_BACKEND` | 追踪后端（`braintrust`/`otel`） | — |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTel 导出端点 | — |

#### 对话与调度

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ENABLE_CONVERSATION_HISTORY_COMPACTION` | 启用对话压缩 | `True` |
| `ENABLE_CONVERSATION_WORKER` | 启用对话工作器 | `True` |
| `ENABLED_SCHEDULED_PROMPTS` | 启用定时提示词 | `True` |
| `TOOLSET_STATUS_REFRESH_INTERVAL_SECONDS` | 工具集刷新间隔 | `300` |
| `MCP_RETRY_BACKOFF_SCHEDULE` | MCP 重试退避 | `[30, 60, 120]` |

#### Operator 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `HOLMES_API_URL` | Holmes API 地址 | `http://holmes-api:80` |
| `HOLMES_API_TIMEOUT` | API 超时（秒） | `300` |
| `LOG_LEVEL` | 日志级别 | `INFO` |
| `MAX_HISTORY_ITEMS` | 历史记录保留数 | `10` |
| `CLEANUP_COMPLETED_CHECKS` | 清理已完成检查 | `False` |
| `COMPLETED_CHECK_TTL_HOURS` | 完成后保留时长 | `24` |

### 4.3 配置文件示例

```yaml
# ~/.holmes/config.yaml
model: "gpt-4.1"
# api_key: "..."  # 也可通过环境变量设置

# 快速模型（用于工具结果摘要）
# fast_model: "gpt-4o-mini"

# 自定义工具集
# custom_toolsets: ["examples/custom_toolset.yaml"]

# AlertManager 连接
# alertmanager_url: "http://localhost:9093"

# Jira 连接
# jira_username: "user@company.com"
# jira_api_key: "..."
# jira_url: "https://your-company.atlassian.net"

# Slack 通知
# slack_token: "..."
# slack_channel: "#general"

# GitHub 集成
# github_owner: "robusta-dev"
# github_pat: "..."
# github_repository: "holmesgpt"
```

### 4.4 工具集注册与标签

工具集通过标签（`ToolsetTag`）区分 CLI 和服务端模式：

- `ToolsetTag.CORE`：所有模式都包含的核心工具集
- `ToolsetTag.CLI`：仅在 CLI 模式下加载的工具集（如 bash、kubectl-run）
- `ToolsetTag.CLUSTER`：仅在服务端模式下加载的工具集

创建 `ToolCallingLLM` 时通过 `toolset_tag_filter` 控制：

```python
# CLI 模式
config.create_toolcalling_llm(
    toolset_tag_filter=[ToolsetTag.CORE, ToolsetTag.CLI],
    enable_all_toolsets_possible=True,
)

# 服务端模式
config.create_toolcalling_llm(
    toolset_tag_filter=[ToolsetTag.CORE, ToolsetTag.CLUSTER],
    enable_all_toolsets_possible=False,  # 只加载显式启用的工具集
)
```

---

## 5. 开发与调试

### 5.1 CLI 调试

#### 增加日志详细程度

```bash
# -v 一次显示 INFO 级别日志
# -v -v 两次显示 DEBUG 级别日志
poetry run holmes ask "..." --verbose --verbose
```

#### 使用 OpenRouter 降低调试成本

```bash
export OPENROUTER_API_KEY="sk-or-..."
export MODEL="openrouter/openai/gpt-4.1-mini"
export CLASSIFIER_MODEL="openrouter/openai/gpt-4.1"
```

#### 快速模式跳过 TodoWrite

```bash
poetry run holmes ask "..." --no-interactive --fast-mode
```

#### 查看工具调用输出

```bash
poetry run holmes ask "..." --no-interactive --show-tool-output
```

#### 保存 JSON 输出

```bash
poetry run holmes ask "..." --no-interactive --json-output-file ./output.json
```

#### 禁用所有工具审批

```bash
# 在沙箱环境中
poetry run holmes ask "..." --bash-always-allow
```

#### 计算成本

```bash
poetry run holmes ask "..." --no-interactive --log-costs
```

### 5.2 服务端调试

#### 本地启动 FastAPI 服务

```bash
# 方法 1：直接运行 server.py
poetry run python server.py

# 方法 2：使用 uvicorn（热重载）
poetry run uvicorn server:app --reload --port 5050 --host 0.0.0.0
```

#### 测试 API 端点

```bash
# 健康检查
curl http://localhost:5050/healthz

# 执行健康检查（模拟 Operator 调用）
curl -X POST http://localhost:5050/api/checks/execute \
  -H "Content-Type: application/json" \
  -d '{
    "query": "检查集群节点状态",
    "name": "dev-check",
    "timeout": 30,
    "mode": "monitor"
  }'

# 使用脚本测试
bash scripts/test-api.sh
```

### 5.3 Operator 调试

#### 本地运行 Operator

```bash
# 需要先确保 Holmes API 服务在运行
export HOLMES_API_URL="http://localhost:5050"

# 使用 kubeconfig 连接集群
export KUBECONFIG=~/.kube/config

# 运行 Operator（需要集群已安装 CRD）
poetry run python -m holmes_operator.operator
```

#### 安装 CRD 到集群

Operator 使用的 CRD 定义通常在 `helm/` 目录中：

```bash
# 通过 Helm 部署
helm install holmes-operator ./helm/holmes-operator

# 或直接应用 CRD
kubectl apply -f ./helm/holmes-operator/crds/
```

#### 创建测试 HealthCheck

```bash
kubectl create -f - <<EOF
apiVersion: holmesgpt.dev/v1alpha1
kind: HealthCheck
metadata:
  name: test-check
  namespace: default
spec:
  query: "检查 default 命名空间中是否有异常 Pod"
  timeout: 30
  mode: monitor
EOF

# 查看执行结果
kubectl get healthcheck test-check -o yaml
```

### 5.4 IDE 调试

#### VS Code 配置（CLI 调试）

创建 `.vscode/launch.json`：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Holmes Ask",
            "type": "debugpy",
            "request": "launch",
            "module": "holmes.main",
            "args": ["ask", "检查集群节点状态", "--no-interactive"],
            "console": "integratedTerminal",
            "env": {
                "OPENAI_API_KEY": "${input:openai_key}",
                "LOG_LEVEL": "DEBUG"
            }
        },
        {
            "name": "Holmes Server",
            "type": "debugpy",
            "request": "launch",
            "module": "server",
            "console": "integratedTerminal",
            "env": {
                "OPENAI_API_KEY": "${input:openai_key}",
                "HOLMES_HOST": "0.0.0.0",
                "HOLMES_PORT": "5050"
            }
        },
        {
            "name": "Holmes Operator",
            "type": "debugpy",
            "request": "launch",
            "module": "holmes_operator.operator",
            "console": "integratedTerminal",
            "env": {
                "HOLMES_API_URL": "http://localhost:5050",
                "KUBECONFIG": "${env:HOME}/.kube/config"
            }
        }
    ],
    "inputs": [
        {
            "type": "promptString",
            "id": "openai_key",
            "description": "OpenAI API Key"
        }
    ]
}
```

#### PyCharm 配置

注意：当使用 PyCharm 调试器附加时，`sys.stdin.isatty()` 返回 `false`，CLI 会自动检测到 `PYCHARM_HOSTED` 环境变量并正确处理。

1. 创建 Run Configuration：`Module` = `holmes.main`，参数 = `ask "your question"`
2. 在环境变量中设置 `OPENAI_API_KEY`、`LOG_LEVEL=DEBUG`

### 5.5 追踪与监控

#### Braintrust 追踪

```bash
export BRAINTRUST_API_KEY="..."
export HOLMES_TRACE_BACKEND="braintrust"

poetry run holmes ask "..." --trace braintrust
```

#### OpenTelemetry 追踪

```bash
# 需要先安装 otel 可选依赖
poetry install --with otel

# 启动本地 OTel Collector（可选）
docker run -p 4318:4318 otel/opentelemetry-collector-contrib:latest

# 设置环境变量
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318"
export HOLMES_TRACE_BACKEND="otel"

# 运行
poetry run holmes ask "..." --trace otel
```

### 5.6 交互模式 Slash 命令

在交互模式（`holmes ask --interactive`）中可使用以下命令：

| 命令 | 功能 |
|------|------|
| `/debug` | 切换调试日志 |
| `/toolsets` | 查看工具集状态 |
| `/refresh` | 刷新工具集 |
| `/config` | 打开工具集配置编辑器 |
| `/feedback` | 提交反馈 |
| `/documents` | 加载文档/Runbook |
| `/skills` | 管理技能 |
| `/model <name>` | 切换模型 |
| `/bash-allow` / `/bash-deny` | 切换 bash 审批模式 |
| `/exit` | 退出 |

### 5.7 自定义工具集开发

开发一个新的 Python 工具集（参考 `servicenow_tables/`）：

```bash
# 创建工具集目录
mkdir holmes/plugins/toolsets/my_toolset/
touch holmes/plugins/toolsets/my_toolset/__init__.py
```

工具集实现示例：

```python
# holmes/plugins/toolsets/my_toolset/my_toolset.py
from typing import Optional
from pydantic import BaseModel, Field
from holmes.core.tools import (
    PythonToolset, ToolsetStatusEnum, StructuredTool, ToolParam, ParamType,
    StructuredToolResult, StructuredToolResultStatus, ToolsetTag,
)


class MyToolsetConfig(BaseModel):
    model_config = {"extra": "allow"}  # 向后兼容
    api_url: str = Field(default="http://localhost:8080", description="API 地址")
    api_key: Optional[str] = None


class MyToolset(PythonToolset):
    def __init__(self, name="my_toolset"):
        super().__init__(
            name=name,
            description="我的自定义工具集",
            icon="my_toolset",
            docs_url="https://example.com/docs",
            prerequisites_callable=self._prerequisites,
            config_class=MyToolsetConfig,
            tags=[ToolsetTag.CORE, ToolsetTag.CLI],
            tools=[
                StructuredTool(
                    name="my_query",
                    description="查询我的 API 获取数据",
                    params={
                        "query": ToolParam(
                            type=ParamType.STRING,
                            required=True,
                            description="查询参数",
                        ),
                    },
                    callback=self.my_query,
                ),
            ],
        )

    def _prerequisites(self) -> bool:
        # 检查配置是否有效
        config: MyToolsetConfig = self.config
        if not config.api_key:
            self.error = "缺少 API Key"
            return False
        return True

    def my_query(self, params: dict, context) -> StructuredToolResult:
        import requests
        config: MyToolsetConfig = self.config
        try:
            response = requests.get(
                f"{config.api_url}/data",
                params={"q": params["query"]},
                headers={"Authorization": f"Bearer {config.api_key}"},
                timeout=10,
            )
            response.raise_for_status()
            return StructuredToolResult(
                status=StructuredToolResultStatus.SUCCESS,
                data=response.text,
            )
        except Exception as e:
            return StructuredToolResult(
                status=StructuredToolResultStatus.ERROR,
                error=f"查询失败: {e}",
                params=params,
            )
```

然后在 `holmes/plugins/toolsets/__init__.py` 的 `load_python_toolsets()` 中注册：

```python
from holmes.plugins.toolsets.my_toolset.my_toolset import MyToolset

def load_python_toolsets(dal=None, additional_search_paths=None):
    toolsets = [
        CoreInvestigationToolset(),
        # ... 已有工具集 ...
        MyToolset(),  # <-- 添加这行
    ]
    return toolsets
```

---

## 6. 测试体系

### 6.1 测试分类

| 类型 | 标记 | 说明 | 是否需要 LLM API |
|------|------|------|------------------|
| 单元测试 | — | 纯逻辑测试，不依赖外部服务 | 否 |
| 集成测试 | `integration` | 依赖外部服务（运行中的 Server、真实 API） | 部分 |
| LLM 评估 | `llm` | 端到端测试，依赖真实 LLM API | 是 |
| 回归测试 | `llm and regression` | 核心回归套件，必须稳定通过 | 是 |

### 6.2 运行测试

```bash
# 运行所有非 LLM 测试（快速）
poetry run pytest tests -m "not llm"

# 使用 Makefile
make test-without-llm

# 运行特定测试文件
poetry run pytest tests/test_config.py -v

# 运行特定测试函数
poetry run pytest tests/test_config.py::test_config_load -v

# 运行单个 LLM 评估（指定 eval 编号）
poetry run pytest -k "09_crashpod" --no-cov

# 运行回归套件
poetry run pytest -m "llm and regression" --no-cov -n 4

# 运行所有 LLM 评估并行
poetry run pytest tests/llm/ -n 6 --no-cov

# 带覆盖率报告
poetry run pytest tests -m "not llm" --cov=holmes --cov-report=html

# 查看测试标记列表
poetry run pytest --markers
```

### 6.3 测试选项

| 选项 | 说明 |
|------|------|
| `-n auto` | 并行运行（pytest-xdist） |
| `--no-cov` | 跳过覆盖率检查（运行更快） |
| `--skip-setup` | 跳过 LLM 评估的前置部署步骤 |
| `--skip-cleanup` | 测试后不清理资源（便于调试） |
| `--only-setup` | 仅执行前置部署，不运行测试 |
| `--only-cleanup` | 仅执行清理 |
| `-v` / `-vv` | 显示更详细的输出 |
| `-k "pattern"` | 按名称过滤测试 |
| `-m "marker"` | 按标记过滤 |
| `--tb=long` / `--tb=short` | 控制回溯输出长度 |

### 6.4 LLM 评估测试结构

LLM 评估测试位于 `tests/llm/`，每个测试包含：

```
tests/llm/
├── conftest.py               # 全局配置和 fixture
├── test_ask_holmes.py        # 主测试文件
├── fixtures/
│   ├── shared/               # 共享基础设施（Prometheus、Grafana 等）
│   └── 09_crashpod/          # 单个测试的完整目录
│       ├── test_case.yaml    # 测试定义（prompt、expected_output、infra 配置）
│       ├── manifests.yaml    # K8s 资源配置
│       └── toolsets.yaml     # （可选）自定义工具集配置
└── utils/                    # 工具函数
```

`test_case.yaml` 结构：

```yaml
# tests/llm/fixtures/09_crashpod/test_case.yaml
name: "09_crashpod"
description: "测试 Pod CrashLoopBackOff 诊断"
prompts:
  - "检查 default 命名空间中是否有状态异常的 Pod，分析原因并给出修复建议"
expected_output:
  - "Must mention CrashLoopBackOff"
  - "Must identify the application crash reason"
before_test:
  - apply manifests.yaml
after_test:
  - delete namespace app-09
cleanup: namespace app-09
timeout: 120
tags: [llm, kubernetes, regression, easy]
include_tool_calls: true
```

### 6.5 Operator 测试

```bash
# 运行 Operator 单元测试
poetry run pytest tests/holmes_operator/ -v

# 需要运行 Holmes API 服务的集成测试
poetry run pytest tests/holmes_operator/ -m integration
```

测试使用 `respx` 库模拟 HTTP 调用（而非 `unittest.mock`），以更真实地模拟网络层。

### 6.6 Eval 测试最佳实践

**调试 eval 测试时：**

```bash
# 1. 只在特定测试上运行 setup（快速迭代）
poetry run pytest -k "09_crashpod" --only-setup --no-cov

# 2. 运行完整测试
poetry run pytest -k "09_crashpod" --no-cov

# 3. 跳过 setup 重新运行（基础设施已就绪时）
poetry run pytest -k "09_crashpod" --skip-setup --no-cov

# 4. 跳过 cleanup 以便检查资源状态
poetry run pytest -k "09_crashpod" --skip-cleanup --no-cov

# 5. 手动清理
poetry run pytest -k "09_crashpod" --only-cleanup --no-cov
```

---

## 7. Docker 部署

### 7.1 构建本地镜像

```bash
# 构建 Holmes API 镜像
docker build -t holmes:local .

# 构建 Operator 镜像
docker build -t holmes-operator:local -f Dockerfile.operator .
```

### 7.2 使用 Docker Compose 本地运行

```bash
# 启动 Holmes API 服务
export OPENAI_API_KEY="sk-..."
docker compose up -d

# 查看日志
docker compose logs -f

# 停止
docker compose down
```

`docker-compose.yaml` 默认配置：
- 镜像：`us-central1-docker.pkg.dev/genuine-flight-317411/devel/holmes`
- 端口：`5050:5050`
- 自动挂载 `~/.holmes`、`~/.kube/config`、`~/.aws` 等
- 自动修正 K8s API 地址为 `host.docker.internal`

### 7.3 本地 K8s 沙箱测试

使用 `scripts/setup-sandbox-k8s.sh` 可以在本地快速启动 K3s 沙箱：

```bash
# 启动 K3s 集群（需要 Docker）
bash scripts/setup-sandbox-k8s.sh

# 设置环境变量
export KUBECONFIG=/tmp/k3s-output/kubeconfig.yaml
export RUN_LIVE=true
export OPENAI_API_KEY="sk-..."
export MODEL="openrouter/openai/gpt-4.1-mini"
export CLASSIFIER_MODEL="openrouter/openai/gpt-4.1"

# 运行回归测试
poetry run pytest tests/llm/test_ask_holmes.py \
  -m "llm and regression and not network" \
  --no-cov -n 4 -p no:cacheprovider
```

---

## 8. 常见问题

### 8.1 安装问题

**Q: Poetry 安装慢或超时**
```bash
# 使用国内镜像
poetry source add --priority=default mirrors https://pypi.tuna.tsinghua.edu.cn/simple/
poetry install

# 或设置超时
export POETRY_REQUESTS_TIMEOUT=120
poetry install
```

**Q: mypy / ruff 版本冲突**
```bash
# 只安装核心依赖
poetry install --no-root

# 后续按需安装 dev 组
poetry install --with dev
```

### 8.2 运行时问题

**Q: "No LLM models were loaded"**
```bash
# 检查 API Key 是否设置
echo $OPENAI_API_KEY

# 检查 MODEL 环境变量
echo $MODEL

# 尝试指定模型
poetry run holmes ask "hello" --model openrouter/openai/gpt-4.1-mini --no-interactive
```

**Q: "The Azure model you chose is not supported"**
- 这是 Azure 模型版本过低的问题，需要模型版本 1106 及以上。

**Q: 交互模式排版混乱 / 闪屏**
- 检查终端宽度：Rich 的 Live 渲染对终端宽度敏感
- 尝试减小终端宽度后重新运行
- 确认不使用 `screen` 或 `tmux` 的分割窗格

**Q: 工具执行超时 / 工具集标记为 FAILED**
```bash
# 增加前提检查超时
export HOLMES_TOOLSET_PREREQ_TIMEOUT_SECONDS=60

# 禁用不需要的工具集（在 config.yaml 中）
toolsets:
  prometheus:
    enabled: false
  elasticsearch:
    enabled: false
```

**Q: Token 限制 / 上下文溢出**
```bash
# 增加工具结果截断阈值
export TOOL_MAX_ALLOCATED_CONTEXT_WINDOW_TOKENS=10000
export MAX_OUTPUT_TOKEN_RESERVATION=8192

# 禁用一个大的工具集
export DISABLE_PROMETHEUS_TOOLSET=true
```

**Q: 运行在 WSL2 中时 K8s 连接失败**
```bash
# 确保 kubectl 配置正确
export KUBECONFIG=/mnt/c/Users/yourname/.kube/config

# 或在 Kubernetes 集群内运行 Service + Operator
```

### 8.3 调试技巧

**Q: 如何查看 LLM 实际发送的 prompts？**

使用 Braintrust 追踪：
```bash
export BRAINTRUST_API_KEY="..."
poetry run holmes ask "..." --trace braintrust --no-interactive
```
Braintrust 会记录每次 LLM 调用的完整提示词和响应。

或启用详细日志：
```bash
export LOG_LEVEL=DEBUG
poetry run holmes ask "..." -vv --no-interactive
```

**Q: 如何模拟 HTTP 请求进行测试？**

项目使用 `responses` 库拦截 HTTP 请求：
```python
import responses

@responses.activate
def test_my_tool():
    responses.add(
        responses.GET,
        "https://api.example.com/data",
        json={"result": "ok"},
        status=200,
    )
    # ... 测试代码
```

对于 httpx 异步客户端，使用 `respx` 库（如 `tests/holmes_operator/conftest.py` 所示）。

**Q: 如何快速验证一个工具集是否正常工作？**

```bash
# 列出工具集状态
poetry run holmes toolset list

# 刷新并查看详细状态
poetry run holmes toolset refresh
```

### 8.4 代码质量

```bash
# 格式化代码（仅当被要求时）
poetry run ruff format

# 检查代码（仅当被要求时）
poetry run ruff check --fix

# 类型检查（仅当被要求时）
poetry run mypy
```

---

> 本文档基于 2026 年 6 月代码库状态编写。随着项目发展，部分环境变量默认值和配置路径可能发生变化，请结合最新的代码和文档进行验证。