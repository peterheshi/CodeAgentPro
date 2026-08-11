# CodeAgent Pro 参数配置指导书

> 版本: v1.0 | 日期: 2026-08-05 | 适用范围: 管理员 / 高级用户 / 部署运维
> 依据源码: `src/codeagent/config/settings.py`、`src/codeagent/core/config_manager.py`、`src/codeagent/adapters/llm/provider_manager.py`

---

## 一、配置体系总览

CodeAgent Pro 的配置共有 **4 类来源**，按优先级从低到高依次为：

| 优先级 | 配置来源 | 载体 | 加载入口 |
|--------|----------|------|----------|
| 1（低） | 代码内置默认值 | Python dataclass / Field 默认值 | `settings.py`、`config_manager.py` |
| 2 | 全局配置文件 | `~/.codeagent/config.yaml` | `ConfigManager._load_global_config()` |
| 3 | 用户配置文件 | `~/.codeagent/user.yaml` | `ConfigManager._load_user_config()` |
| 4 | 项目配置文件 | `<项目>/.codeagent/project.yaml` | `ConfigManager._load_project_config()` |
| 5 | 环境变量 | `.env` 文件 / 系统环境变量 | `pydantic-settings` + `_load_environment_config()` |
| 6（高） | 运行时修改 | `codeagent config set ...` 内存值 | `ConfigManager.set()` |

> 优先级说明：后加载的配置覆盖先加载的配置。环境变量优先级高于所有 YAML 文件，运行时设置（RUNTIME）最高。

配置值来源可通过 `codeagent config --list` 查看（每项显示 `(source)` 来源标识）。

---

## 二、配置来源 1：环境变量 / `.env` 文件

### 2.1 文件位置与加载规则

- **路径**：当前工作目录下的 `.env` 文件（`SettingsConfigDict(env_file=".env")`）。
- **模板**：项目根目录 `.env.example`（发布包内含）。
- **加载时机**：进程启动时一次性读取（`get_settings()` 带 `lru_cache`，运行中修改需调用 `clear_settings_cache()`）。
- **大小写**：`case_sensitive=True`，环境变量名必须与字段名**完全一致**。

### 2.2 完整参数表

#### （1）运行模式

| 参数名 | 类型 | 默认值 | 作用 |
|--------|------|--------|------|
| `APP_NAME` | str | `CodeAgent Pro` | 应用显示名称 |
| `APP_VERSION` | str | `1.0.0` | 应用版本号 |
| `DEBUG` | bool | `false` | 调试模式；为 `false` 时若 `SECRET_KEY` 为默认值会输出安全警告 |
| `ENVIRONMENT` | str | `development` | 环境标识，仅允许 `development / staging / production` |
| `HOST` | str | `0.0.0.0` | API 服务监听地址 |
| `PORT` | int | `8000` | API 服务监听端口（1-65535） |

#### （2）安全

| 参数名 | 类型 | 默认值 | 作用 |
|--------|------|--------|------|
| `SECRET_KEY` | str | `change-me-in-production-...` | JWT 签名密钥；**生产环境必须修改**，否则每次启动输出安全警告 |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | int | `60` | JWT 访问令牌有效期（分钟） |

#### （3）数据库与缓存（可选，未配置则用内存存储）

| 参数名 | 类型 | 默认值 | 作用 |
|--------|------|--------|------|
| `DATABASE_URL` | str | `None` | PostgreSQL 连接串，如 `postgresql://user:pass@host:5432/db` |
| `DATABASE_POOL_SIZE` | int | `5` | 数据库连接池大小 |
| `REDIS_URL` | str | `None` | Redis 连接串，用于缓存/会话 |

#### （4）LLM API Keys（至少配置一个）

| 参数名 | 作用 | 获取地址 |
|--------|------|----------|
| `DEEPSEEK_API_KEY` | DeepSeek（主力，推荐） | platform.deepseek.com/api_keys |
| `OPENAI_API_KEY` | OpenAI GPT 系列 | platform.openai.com/api-keys |
| `ANTHROPIC_API_KEY` | Anthropic Claude 系列 | console.anthropic.com/settings/keys |
| `ZHIPU_API_KEY` | 智谱 GLM 系列 | open.bigmodel.cn/usercenter/apikeys |
| `QWEN_API_KEY` | 通义千问 | dashscope.console.aliyun.com/apiKey |
| `BAICHUAN_API_KEY` | 百川 Baichuan | platform.baichuan-ai.com/console/apikey |
| `MOONSHOT_API_KEY` | Moonshot Kimi | platform.moonshot.cn/console/api-keys |

#### （5）搜索工具（可选）

| 参数名 | 默认值 | 作用 |
|--------|--------|------|
| `SERPAPI_KEY` | `None` | SerpAPI 网页搜索 |
| `BING_SEARCH_KEY` | `None` | Bing Web Search API |
| `BING_SEARCH_ENDPOINT` | `https://api.bing.microsoft.com/v7.0/search` | Bing 搜索端点 |
| `SEARXNG_URL` | `None` | 自建 SearXNG 实例地址 |

#### （6）其他集成

| 参数名 | 默认值 | 作用 |
|--------|--------|------|
| `GITHUB_TOKEN` | `None` | GitHub Token（技能市场访问） |
| `OLLAMA_BASE_URL` | `http://localhost:11434/v1` | 本地 Ollama 服务地址 |
| `LMSTUDIO_BASE_URL` | `http://localhost:1234/v1` | 本地 LM Studio 服务地址 |
| `MCP_SERVERS` | `""` | 逗号分隔的 MCP 服务器 URL（工具发现） |

#### （7）默认 LLM 行为

| 参数名 | 类型 | 默认值 | 作用 |
|--------|------|--------|------|
| `DEFAULT_PROVIDER` | str | `deepseek` | 默认 LLM 提供商名称 |
| `DEFAULT_MODEL` | str | `deepseek-chat` | 默认模型标识 |
| `LLM_MAX_TOKENS` | int | `4096` | 单次请求最大 token 数（≥1） |
| `LLM_TEMPERATURE` | float | `0.7` | 采样温度（0.0-2.0） |
| `LLM_TIMEOUT_SECONDS` | int | `60` | 请求超时（秒，≥1） |
| `LLM_RETRY_COUNT` | int | `3` | 失败重试次数（≥0） |
| `LLM_STREAM_ENABLED` | bool | `true` | 是否启用流式响应 |
| `LLM_PROVIDER_CONFIG` | str | `None` | 自定义 Provider 配置 YAML 文件路径（见第四章） |

#### （8）日志与跨域

| 参数名 | 类型 | 默认值 | 作用 |
|--------|------|--------|------|
| `LOG_LEVEL` | str | `INFO` | 仅允许 `DEBUG/INFO/WARNING/ERROR/CRITICAL` |
| `LOG_FORMAT` | str | `%(asctime)s ...` | 日志格式串 |
| `CORS_ORIGINS` | list | `["*"]` | 允许的跨域来源 |
| `MAX_UPLOAD_SIZE` | int | `10485760` | 上传文件大小上限（字节，默认 10MB） |

### 2.3 ⚠️ 兼容性注意（.env.example 与代码字段名不一致项）

以下 `.env.example` 中出现的参数名**与代码字段名不一致，写入 .env 不生效**（`extra="ignore"` 静默忽略，不报错）：

| .env.example 中的名称 | 代码实际字段名 | 说明 |
|------------------------|----------------|------|
| `DEFAULT_LLM_PROVIDER` | `DEFAULT_PROVIDER` | 用 `DEFAULT_PROVIDER` 或 `CODEAGENT_LLM_PROVIDER` |
| `MAX_TOKENS` | `LLM_MAX_TOKENS` | 用 `LLM_MAX_TOKENS` 或 `CODEAGENT_LLM_MAX_TOKENS` |
| `TEMPERATURE` | `LLM_TEMPERATURE` | 用 `LLM_TEMPERATURE` 或 `CODEAGENT_LLM_TEMPERATURE` |
| `API_KEY_HEADER` | （无此字段） | 代码中未实现，Header 认证名固定为 `X-API-Key` |

---

## 三、配置来源 2：多级 YAML 配置文件

### 3.1 文件路径

| 层级 | 路径 | 用途 |
|------|------|------|
| 全局 | `~/.codeagent/config.yaml` | 所有项目的通用配置 |
| 用户 | `~/.codeagent/user.yaml` | 个人偏好（`codeagent config set` 默认写入此文件） |
| 项目 | `<项目>/.codeagent/project.yaml` | 仅作用于当前项目 |

### 3.2 配置结构（顶层键）

```yaml
llm:                     # LLM 覆盖项（合并进 settings 的 LLM 配置）
  default_provider: deepseek
  default_model: deepseek-chat
  max_tokens: 4096
  temperature: 0.7
  timeout_seconds: 60
  retry_count: 3
  stream_enabled: true

agent:                   # Agent 引擎参数
  max_parallel_agents: 8          # 并行 Agent 数上限（1-32）
  max_memory_per_agent_mb: 512    # 单 Agent 内存上限（MB）
  default_timeout_seconds: 300    # 默认超时（秒）
  enable_auto_recovery: true      # 自动恢复
  max_retries: 3                  # 重试次数
  checkpoint_interval_seconds: 60 # 检查点间隔

permission:              # 权限控制
  default_mode: ask               # ask / auto / restricted
  allowed_commands: []            # 允许的命令白名单
  denied_commands: []             # 禁止的命令黑名单
  auto_approve_safe_operations: false

logging:                 # 日志
  level: INFO
  format: json                    # json / text
  file_enabled: true
  console_enabled: true
  max_file_size_mb: 100
  backup_count: 5

ui:                      # 界面
  theme: dark                     # dark / light
  font_size: 14                   # 1-24
  font_family: Consolas
  show_line_numbers: true
  word_wrap: true
  auto_save: true
  auto_save_interval_seconds: 30

resource:                # 资源上限
  max_context_tokens: 1000000     # 上下文窗口（1000-10000000）
  max_tool_iterations: 30         # 单轮任务工具调用最大迭代次数

language: zh-CN          # 界面语言（zh-CN / en-US）
check_updates: true      # 启动时检查更新
telemetry_enabled: false # 遥测上报开关

# --- 模块融合开关（对应 ModuleConfig，顶层直接书写） ---
enable_reflection: false                 # 反思闭环
reflection_quality_threshold: 0.90       # 反思质量阈值
enable_circuit_breaker: false            # 熔断器
enable_persistent_memory: false          # 持久化记忆
mcp_servers: []                          # MCP 服务器列表
enable_planning: false                   # 任务规划
planning_complexity_threshold: 3         # 触发规划的任务复杂度阈值
enable_routing: false                    # 智能路由
enable_parallel: false                   # 并行执行
parallel_max_concurrency: 5              # 并行最大并发数
enable_experience_store: false           # 经验库
enable_strategy_engine: false            # 策略引擎
enable_feedback_learning: false          # 反馈学习
enable_goal_monitoring: false            # 目标监控
enable_multi_agent: false                # 多 Agent 协作
enable_proactive_perception: false       # 主动感知
```

### 3.3 模块开关对应的环境变量（等价写法）

| 环境变量 | 对应配置键 |
|----------|-----------|
| `CODEAGENT_AGENT_MAX_PARALLEL` | `agent.max_parallel_agents` |
| `CODEAGENT_AGENT_TIMEOUT` | `agent.default_timeout_seconds` |
| `CODEAGENT_LOG_LEVEL` | `logging.level` |
| `CODEAGENT_UI_THEME` | `ui.theme` |
| `CODEAGENT_LANGUAGE` | `language` |
| `CODEAGENT_ENABLE_REFLECTION` | `modules.enable_reflection` |
| `CODEAGENT_ENABLE_CIRCUIT_BREAKER` | `modules.enable_circuit_breaker` |
| `CODEAGENT_ENABLE_PERSISTENT_MEMORY` | `modules.enable_persistent_memory` |
| `CODEAGENT_ENABLE_PLANNING` | `modules.enable_planning` |
| `CODEAGENT_ENABLE_ROUTING` | `modules.enable_routing` |
| `CODEAGENT_ENABLE_PARALLEL` | `modules.enable_parallel` |
| `CODEAGENT_LLM_PROVIDER` | `llm.default_provider` |
| `CODEAGENT_LLM_MODEL` | `llm.default_model` |
| `CODEAGENT_LLM_TEMPERATURE` | `llm.temperature` |
| `CODEAGENT_LLM_MAX_TOKENS` | `llm.max_tokens` |

### 3.4 校验规则（配置不合法将抛出异常）

- `temperature` 必须在 0-2 之间
- `agent.max_parallel_agents` 必须在 1-32 之间
- `ui.font_size` 必须在 1-24 之间
- `permission.default_mode` 必须为 `ask / auto / restricted`

---

## 四、配置来源 3：LLM Provider 自定义配置（YAML）

### 4.1 启用方式

设置环境变量 `LLM_PROVIDER_CONFIG=<文件路径>`，加载逻辑见 `provider_manager.py:_load_config()`。

### 4.2 文件格式

```yaml
default_provider: my-provider     # 设为默认提供商

providers:
  my-provider:                     # 实例名
    type: openai                   # openai / anthropic / deepseek / zhipu / qwen / baichuan / moonshot ...
    api_key: ${DEEPSEEK_API_KEY}   # 支持 ${环境变量} 引用，也支持直接引用 settings 字段
    base_url: https://api.example.com/v1
    model: my-model
    enabled: true
    extra:
      timeout: 120
```

### 4.3 内置提供商默认信息（`DEFAULT_PROVIDERS`）

| 名称 | base_url | 默认模型 | 可用模型 |
|------|----------|----------|----------|
| openai | https://api.openai.com/v1 | gpt-4o | gpt-4o, gpt-4-turbo, gpt-4, gpt-3.5-turbo |
| anthropic | https://api.anthropic.com | claude-3-5-sonnet-20241022 | claude-3-opus, claude-3-sonnet 等 |
| deepseek | https://api.deepseek.com | deepseek-chat | deepseek-chat, deepseek-coder |
| zhipu | https://open.bigmodel.cn/api/paas/v4 | glm-4 | glm-4, glm-4-flash, glm-4-plus |
| qwen | https://dashscope.aliyuncs.com/compatible-mode/v1 | qwen-turbo | qwen-turbo, qwen-plus, qwen-max |
| baichuan | https://api.baichuan-ai.com/v1 | Baichuan4 | Baichuan4, Baichuan3-Turbo |
| 本地模型 | `OLLAMA_BASE_URL` / `LMSTUDIO_BASE_URL` | - | 本地部署模型 |

---

## 五、配置来源 4：CLI 运行时配置（`codeagent config`）

### 5.1 命令语法

```bash
codeagent config --list              # 列出全部配置及来源
codeagent config get <KEY>           # 读取单项
codeagent config set <KEY> <VALUE>   # 写入（保存到 ~/.codeagent/user.yaml）
codeagent config KEY                 # 简写读取
codeagent config KEY VALUE           # 简写写入
```

### 5.2 支持键与类型

| CLI 键 | 对应配置 | 类型转换 |
|--------|----------|----------|
| `llm.default_model` | llm.default_model | str |
| `llm.max_tokens` | llm.max_tokens | int |
| `llm.temperature` | llm.temperature | float |
| `llm.timeout_seconds` | llm.timeout_seconds | int |
| `llm.retry_count` | llm.retry_count | int |
| `llm.stream_enabled` | llm.stream_enabled | true/false/1/0/yes/no |
| `agent.max_context_tokens` | resource.max_context_tokens | int |
| `agent.max_tool_iterations` | resource.max_tool_iterations | int |
| `ui.theme` | ui.theme | str |
| `language` | language | str |

---

## 六、项目级配置（`codeagent init` 生成）

在项目目录执行 `codeagent init` 生成两个文件：

**`.codeagent.json`**

```json
{
  "project_name": "my-project",
  "version": "0.1.0",
  "model": "deepseek-chat",
  "temperature": 0.7,
  "max_tokens": 4096,
  "session_history": true,
  "undo_enabled": true
}
```

**`pyproject.toml` 的 `[tool.codeagent]` 段**

```toml
[tool.codeagent]
model = "deepseek-chat"
temperature = 0.7
```

---

## 七、运行数据目录（非配置，但运维相关）

| 路径 | 内容 |
|------|------|
| `~/.codeagent/sessions/*.json` | 会话记录（`codeagent session` 管理） |
| `~/.codeagent/undo/history.json` | 文件操作撤销历史（`codeagent undo` 管理） |
| `~/.codeagent/undo/backup_*` | 写入前的文件备份 |

---

## 八、推荐配置顺序（首次部署清单）

1. 复制 `.env.example` 为 `.env`，配置至少一个 LLM API Key（推荐 `DEEPSEEK_API_KEY`）。
2. 按需修改 `DEFAULT_PROVIDER`、`DEFAULT_MODEL`、`LLM_TEMPERATURE` 等默认行为。
3. 生产环境：设置强 `SECRET_KEY`、`ENVIRONMENT=production`、`LOG_LEVEL=INFO`。
4. 多模型/私有化网关：配置 `LLM_PROVIDER_CONFIG` 指向自定义 YAML。
5. 需要搜索工具：配置 `SERPAPI_KEY` 或 `BING_SEARCH_KEY`。
6. 验证：`codeagent config --list` 检查各项来源与取值，`codeagent chat "你好"` 验证连通。

---

## 九、安全注意事项

- **API Key 与 SECRET_KEY 为敏感信息**，`.env` 已在 `.gitignore` 中排除，勿提交仓库。
- 生产环境若 `SECRET_KEY` 仍为默认值，启动时会持续输出安全警告（当前 1.0.0 行为，非 bug）。
- 配置文件支持热重载（`ConfigManager.start_hot_reload()`，轮询间隔 1 秒），修改 `config.yaml`/`project.yaml` 后自动生效；修改 `.env` 需重启进程。
