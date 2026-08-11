# CodeAgent Pro 功能说明文档

> 版本: v1.0 | 日期: 2026-08-05 | 适用: 架构评审 / 功能验收 / 二次开发
> 依据源码：`src/codeagent/`（约 250 个模块），配套 `CodeAgent_Pro_参数配置指导书_v1.0.md`

---

## 一、系统架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    交互层（Presentation）                │
│  CLI (click+rich) │ REST API │ WebSocket │ GraphQL      │
│  Desktop │ 可视化 panels                                 │
├─────────────────────────────────────────────────────────┤
│                    核心引擎层（core）                    │
│  Agent 运行时 (core/agent, 1 主类 + 12 Mixin)           │
│  上下文管理 │ Token 计数 │ 工具注册/调度 │ 规划/编排     │
│  安全沙箱/权限 │ 熔断/重试 │ 状态机 │ 会话存储          │
├─────────────────────────────────────────────────────────┤
│                    工具层（tools）                      │
│  文件 │ 代码 │ Shell │ Git │ Web │ 搜索 │ 数据库 │ 安全 │
│  OCR │ 语音(ASR)                                        │
├─────────────────────────────────────────────────────────┤
│                    适配层（adapters）                   │
│  LLM Provider: DeepSeek/OpenAI/Anthropic/智谱/通义/百川/│
│  Moonshot/Ollama/LM Studio │ 多模态适配 │ 记忆适配      │
├─────────────────────────────────────────────────────────┤
│                    增强模块（可配置开关）               │
│  reflection(反思) │ meta_cognition(元认知) │ evolution(进化) │
│  engineering(规划/并行/路由/多Agent) │ multimodal(多模态)  │
│  skills(技能) │ visualization(可视化)                    │
└─────────────────────────────────────────────────────────┘
```

---

## 二、交互层功能

### 2.1 CLI（`codeagent` / `codeagent-pro`）

入口链：`pyproject.toml` scripts → `codeagent.__main__:run` → `codeagent.cli.main:main`（Click 组）。

| 命令 | 功能 | 关键能力 |
|------|------|----------|
| `init` | 项目初始化 | 生成 `.codeagent.json` 与 `pyproject.toml [tool.codeagent]` |
| `chat` | AI 对话 | 单次提问/交互模式；支持流式/非流式；内置 `/` 命令（文件、浏览器、模型） |
| `session` | 会话管理 | 新建/列出/加载/删除（存于 `~/.codeagent/sessions/`） |
| `shell` | 交互式 Shell | 子进程隔离执行、60s 超时自动杀、Ctrl+C 进程树终止、内置 ls/cd/pwd/cat |
| `read` / `write` | 文件读写 | 项目目录沙箱限制 + 系统目录黑名单 |
| `undo` | 操作回滚 | 文件写前自动备份、多步撤销、重做、操作历史 |
| `config` | 配置管理 | 10 个键读写、来源追踪、持久化到 user.yaml |
| `status` / `help` | 状态与帮助 | 版本/Python/项目信息 |

### 2.2 API 服务（FastAPI + Uvicorn）

15 组路由（`api/routes/`）：`agents`、`auth`、`commands`、`context`、`files`、`graphql`、`health`、`messages`、`mobile`、`models`、`oauth`、`permissions`、`sessions`、`skills`、`websocket`。

- **认证**：JWT（`SECRET_KEY` 签名、`ACCESS_TOKEN_EXPIRE_MINUTES` 有效期）+ API Key 头（`X-API-Key`），见 `api/middleware/auth.py`
- **健康检查**：`/api/v1/health` 聚合数据库、Redis、LLM Key 连通性
- **GraphQL**：`/graphql` 端点
- **实时通信**：WebSocket

### 2.3 其他界面

- **Desktop**（`desktop/`）：桌面端壳
- **Visualization**（`visualization/` + `panels/`）：数据可视化面板

---

## 三、核心引擎功能

### 3.1 Agent 运行时（`core/agent/`）

主类 `core.py` 采用 **Mixin 组合架构**，12 个能力插件：

| Mixin | 能力 |
|-------|------|
| `llm_mixin` | LLM 调用与流式响应 |
| `tool_mixin` | 工具注册/调用/参数校验 |
| `loop_mixin` | 推理循环（ReAct 式迭代） |
| `planning_mixin` | 任务规划 |
| `reflection_mixin` | 结果反思与自我修正 |
| `session_mixin` | 会话生命周期 |
| `hitl_mixin` | 人工介入（Human-in-the-Loop） |
| `experience_mixin` | 经验库调用 |
| `collaboration_mixin` | 多 Agent 协作 |
| `init_mixin` | 初始化流程 |
| `config` | Agent 配置模型 |

### 3.2 上下文管理（`core/context*.py`）

- **ContextCache**：上下文缓存，`get_stats()` 输出条目统计
- **ContextCompressor**：增量/聚类压缩（`_cluster_by_similarity` 关键词重叠聚类，`max_clusters = min(10, len(segments)//3)`）
- **IncrementalCompressor**：增量压缩器
- **Token 体系**：`token_counter` / `async_token_counter`（计数）、`token_budget`（预算）、`context_window`（窗口）、`context_alert`（阈值预警）、`context_retrieval`（检索）、`context_config`
- **ResourceConfig**：`max_context_tokens`（默认 1,000,000）、`max_tool_iterations`（默认 30 次）

### 3.3 工具体系（`core/tool_*.py`）

- `tool_registry`：全局注册表
- `tool_types`：工具类型定义
- `tool_cache`：工具结果缓存
- `tool_display`：结果展示格式化
- `tool_retry`：工具失败重试

### 3.4 执行与编排

- `engine.py` / `executor.py`：执行引擎
- `orchestrator.py`：任务编排
- `loop_executor.py`：循环执行器
- `scheduler.py`：调度器
- `planner.py`：任务规划器
- `state_machine.py`：Agent 状态机（IDLE/RUNNING/WAITING/ERROR/INTERRUPTED）

### 3.5 安全与可靠性

- **沙箱**：`sandbox.py`、`sandbox_policy.py`、`sandbox_resource.py`、`agent_sandbox.py`
- **权限**：`permission.py`、`confirmation.py`（默认 `ask` 模式，白/黑名单命令）
- **熔断**：`circuit_breaker.py`（连续失败自动熔断，测试/运行自动复位）
- **重试**：`retry.py`、`error_handler.py`
- **安全工具**：`security.py`（含安全操作检查）

### 3.6 数据与状态

- `session.py` / `session_store.py` / `agent_session.py`：会话模型与持久化
- `snapshot.py`：快照
- `rewind_manager.py`：回滚管理（对应 CLI `undo`）
- `file_tracker.py`：文件变更追踪
- `project_scanner.py` / `symbol_index.py`：项目扫描与符号索引
- `multi_file_editor.py`：多文件编辑
- `code_reviewer.py`：代码审查

---

## 四、内置工具（`core/tools/`，10 类）

| 工具 | 能力 |
|------|------|
| `file_ops` | 文件读写/目录操作 |
| `code_ops` | 代码检索/修改/生成 |
| `shell_ops` | Shell 命令执行 |
| `git_ops` | Git 状态/提交/差异 |
| `web_ops` | 网页抓取/浏览器操作 |
| `search_ops` | SerpAPI/Bing/SearXNG 搜索 |
| `db_ops` | 数据库操作 |
| `security_ops` | 安全检查 |
| `ocr_ops` | 图片文字识别（配合 multimodal/ocr） |
| `asr_ops` | 语音转文字（配合 multimodal/asr） |

---

## 五、LLM 多提供商适配（`adapters/llm/`）

- **ProviderManager**：9+ 家提供商统一管理（DeepSeek/OpenAI/Anthropic/智谱/通义/百川/Moonshot + Ollama/LM Studio 本地）
- **统一接口**：`LLMConfig`（model/api_key/base_url/temperature/max_tokens/top_p/stop_sequences/timeout/max_retries）、`LLMResponse`（含 token 统计）、`StreamEvent`（结构化流式事件：content/tool_call/done/usage）
- **原生 Function Calling**：支持流式 tool_calls 累积
- **自定义提供商**：YAML 配置（`LLM_PROVIDER_CONFIG`），支持 `${ENV}` 变量引用
- **DirectAdapter**：OpenAI 兼容协议的本地模型直连

---

## 六、增强模块（默认关闭，YAML/环境变量开关启用）

| 模块 | 目录 | 功能 |
|------|------|------|
| 反思闭环 | `reflection/` | 生成结果质量评分（阈值 0.90）后自动反思修正 |
| 元认知 | `meta_cognition/` | 策略选择器、质量评分器（`strategy_selector` / `quality_scorer`） |
| 进化闭环 | `evolution/` | 经验库（`experience_store`）、策略规则（`strategy_rules`）、护栏（`guardrail`） |
| 智能工程 | `engineering/` | 规划（planning）、并行（parallel）、路由（routing）、多 Agent（multi_agent） |
| 多模态 | `multimodal/` | 视觉（vision）、OCR、语音（asr）、图表（diagram）、渲染（renderer） |
| 技能系统 | `skills/` | 技能定义/加载/执行（11 模块） |
| 持久记忆 | `core/memory/` | 持久化记忆 |
| 主动感知 | `proactive_perception/` | 主动感知 |
| 反馈学习 | `feedback_loop/` | 反馈闭环 |
| 目标监控 | `goal_monitor/` | 目标追踪 |
| HITL | `hitl/` | 人工介入 |
| MCP | `core/mcp_protocol.py` | MCP 协议支持（`MCP_SERVERS` 配置） |

> 开关方式见《参数配置指导书》第三章（`enable_reflection`、`enable_planning` 等 16 个模块开关）。

---

## 七、数据模型（`models/`）

| 实体 | 关键字段 |
|------|----------|
| Agent | id、session_id（必填）、status（IDLE/RUNNING/WAITING/ERROR/INTERRUPTED） |
| Session | id、user_id（必填）、title、status（ACTIVE/PAUSED/COMPLETED/ARCHIVED） |
| Message | id、role、content（非空校验）、session_id |
| Skill / Tool | 名称、描述、参数 schema、执行函数 |
| Event | 领域事件（供观察者/审计） |

---

## 八、安全特性汇总

1. **JWT 认证**：令牌签名 + 过期（默认 60 分钟）
2. **项目沙箱**：CLI 文件读写限制在当前项目目录，系统目录黑名单拦截
3. **命令权限**：ask/auto/restricted 三模式 + 命令白/黑名单
4. **安全密钥检查**：生产环境默认 SECRET_KEY 告警
5. **熔断器**：异常风暴自动熔断保护上游
6. **Shell 隔离**：子进程执行、60s 超时强杀、进程树终止
7. **撤销备份**：文件写操作自动备份，可回滚
8. **CORS 配置**：跨域白名单可配置（默认 `*`）

---

## 九、技术栈与运行要求

| 项 | 说明 |
|----|------|
| Python | >= 3.12（当前构建基于 3.14） |
| 关键依赖 | aiohttp 3.14、FastAPI 0.141、pydantic 2.13（pydantic-settings 2.14）、click 8.4、typer、rich、structlog、psutil 7.2、uvicorn |
| 数据库 | 可选 PostgreSQL（DATABASE_URL），默认内存存储 |
| 缓存 | 可选 Redis（REDIS_URL） |
| 部署形态 | wheel / sdist / Windows 单文件 exe / Docker（含 compose） |

---

## 十、版本信息

- 版本：1.0.0（`src/codeagent/__init__.py:__version__`）
- 变更记录：`CHANGELOG.md`
- 发布说明：`docs/人工核查/CodeAgent_Pro_发布说明_v1.0.0.md`
- 发布评估结论：建议发布 v1.0.0（见《版本发布评估报告_v1.1》）
