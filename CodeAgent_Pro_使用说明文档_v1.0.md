# CodeAgent Pro 使用说明指导文档

> 版本: v1.0 | 日期: 2026-08-05 | 适用: 终端用户 / 运维
> 配套文档：`CodeAgent_Pro_参数配置指导书_v1.0.md`、`CodeAgent_Pro_功能说明文档_v1.0.md`

---

## 一、产品简介

CodeAgent Pro v1.0.0 是一款 AI 驱动的代码助手（AI-Powered Code Assistant），提供：

- **交互式 CLI**：聊天、Shell、文件操作、会话管理、撤销回滚
- **多模型接入**：DeepSeek / OpenAI / Anthropic / 智谱 / 通义 / 百川 / Moonshot / 本地 Ollama、LM Studio
- **工具生态**：文件、代码、Shell、Git、Web、搜索、数据库、安全、OCR、语音 10 类内置工具
- **API 服务**：REST + WebSocket + GraphQL 完整服务端（可配合 Docker 部署）

---

## 二、安装

### 方式 A：pip 安装 wheel（推荐，Python 3.12+）

```powershell
# 创建虚拟环境
python -m venv venv
.\venv\Scripts\activate

# 安装
pip install codeagent_pro-1.0.0-py3-none-any.whl

# 验证
codeagent --version      # 或 codeagent-pro --version
```

> `codeagent` 与 `codeagent-pro` 为同一入口的两个命令名，指向同一程序。

### 方式 B：Windows 免安装单文件 exe

```powershell
# 解压后直接运行（无需 Python 环境）
.\CodeAgentPro.exe --version
.\CodeAgentPro.exe --help
```

> exe 大小约 76MB，为单文件自包含可执行程序，首次运行解压较慢属正常现象。

### 方式 C：Docker 部署 API 服务

```bash
# 构建并启动（含 PostgreSQL + Redis）
docker compose up -d

# 仅构建 API 镜像
docker build -t codeagent-api .
```

### 方式 D：源码安装（开发）

```powershell
pip install -e ".[dev]"
```

---

## 三、首次配置（必读）

1. **配置 API Key**：在运行目录创建 `.env`（参考 `CodeAgent_Pro_参数配置指导书`），最少配置一个：

   ```ini
   DEEPSEEK_API_KEY=sk-xxxx
   DEFAULT_PROVIDER=deepseek
   DEFAULT_MODEL=deepseek-chat
   ```

2. **验证连通性**：

   ```powershell
   codeagent chat "你好，介绍一下你自己"
   ```

3. **查看当前配置**：

   ```powershell
   codeagent config --list
   ```

---

## 四、CLI 命令速查

### 4.1 全局选项

| 选项 | 说明 |
|------|------|
| `--version` | 显示版本号 |
| `--debug` | 启用调试模式（提升 codeagent 日志级别） |
| `--config PATH` | 指定配置文件路径 |
| `--help` | 显示帮助 |

### 4.2 命令一览

| 命令 | 功能 |
|------|------|
| `codeagent init` | 初始化新项目（生成 `.codeagent.json` + `pyproject.toml` 配置段） |
| `codeagent chat` | 与 AI Agent 聊天（单次或交互式） |
| `codeagent session` | 会话管理：创建/列表/加载/删除 |
| `codeagent shell` | 交互式 Shell 环境 |
| `codeagent read` | 读取文件 |
| `codeagent write` | 写入文件（含撤销备份） |
| `codeagent undo` | 撤销/重做文件操作 |
| `codeagent config` | 查看/设置配置 |
| `codeagent status` | 显示版本/Python/项目信息 |
| `codeagent help` | 显示命令帮助 |

### 4.3 常用命令详解

#### `codeagent init [项目路径] [--name 名称]`

在指定目录（默认当前目录）生成项目骨架配置，已存在的文件会跳过（不覆盖）。

```powershell
codeagent init my-project --name MyProject
```

#### `codeagent chat [消息...]`

```powershell
# 单次提问
codeagent chat "给这段代码写单元测试"

# 指定模型与 Agent 配置
codeagent chat -m deepseek-chat -a default --no-stream "分析当前目录"

# 交互模式（不带消息参数）
codeagent chat
```

交互模式内置命令（输入 `/help` 查看）：

| 命令 | 功能 |
|------|------|
| `/help` | 显示帮助 |
| `/status` | 当前状态 |
| `/clear` | 清空对话 |
| `/reset` | 重置会话 |
| `/model` | 查看/切换模型 |
| `/config` | 查看/修改配置 |
| `/read <path>` / `/write <path>` | 文件读写 |
| `/ls [path]` / `/mkdir <path>` / `/delete <path>` | 目录操作 |
| `/browser launch|close|goto` | 浏览器控制 |
| `exit` / `quit` / `q` / Ctrl+C | 退出 |

#### `codeagent session [会话ID] [选项]`

```powershell
codeagent session                    # 新建会话
codeagent session --list             # 列出所有会话
codeagent session <ID>               # 加载查看会话详情
codeagent session <ID> --load        # 加载会话
codeagent session <ID> --delete      # 删除会话
```

#### `codeagent shell [--working-dir 路径]`

交互式 Shell：直接执行系统命令，支持 `!` 前缀显式执行、内置 `ls/cd/pwd/cat/clear/exit`。命令默认 60 秒超时自动终止，Ctrl+C 取消运行中命令，连续两次 Ctrl+C 强制退出。

```powershell
codeagent shell
codeagent shell --working-dir D:\Projects\demo
```

#### `codeagent read <路径> [--lines N]`

安全限制：仅可读取当前项目目录内文件，系统目录（C:\Windows、Program Files 等）被拦截。

#### `codeagent write <路径> [内容...] [--append]`

- 写入前自动备份原文件到 `~/.codeagent/undo/`，可通过 `codeagent undo` 回滚
- 不提供内容时从标准输入读取
- 与 read 相同的安全限制

#### `codeagent undo [步数] [--list] [--redo]`

```powershell
codeagent undo            # 撤销最近 1 次
codeagent undo 3          # 撤销最近 3 次
codeagent undo --list     # 查看操作历史
codeagent undo --redo     # 重做最近一次撤销（仅支持 file_delete）
```

#### `codeagent config`

```powershell
codeagent config --list
codeagent config set llm.temperature 0.3
codeagent config get llm.default_model
codeagent config llm.max_tokens 8192
```

> 支持键清单与类型转换见 `CodeAgent_Pro_参数配置指导书_v1.0.md` 第五章。

#### `codeagent status`

显示版本号、Python 版本、当前项目路径。

---

## 五、快速上手（3 分钟场景）

```powershell
# 1. 初始化项目
codeagent init demo

# 2. 写入文件（自动备份，可撤销）
codeagent write demo\main.py "print('hello')"

# 3. 会话中让 AI 操作
codeagent chat
You: 读一下 main.py 并加一个 main() 函数
Agent: （生成代码并调用工具）

# 4. 误操作回滚
codeagent undo --list
codeagent undo

# 5. 清理会话
codeagent session --list
codeagent session <ID> --delete
```

---

## 六、API 服务模式（可选）

除 CLI 外，程序提供完整 REST API 服务端：

```powershell
# 启动 API 服务（端口见 PORT 配置，默认 8000）
python -m uvicorn codeagent.api.main:app --host 0.0.0.0 --port 8000
# 或 docker compose up -d
```

主要端点：

| 端点 | 说明 |
|------|------|
| `GET /api/v1/health` | 健康检查（含数据库/Redis/LLM 连通状态） |
| `/api/v1/auth/*` | JWT 登录/注册/刷新令牌 |
| `/api/v1/agents/*` | Agent 管理与调度 |
| `/api/v1/sessions/*` | 会话管理 |
| `/api/v1/messages/*` | 消息发送与历史 |
| `/api/v1/context/*` | 上下文管理 |
| `/api/v1/skills/*` | 技能管理 |
| `/api/v1/commands/*` | 命令执行（含 `/settings` 别名） |
| `/api/v1/files/*` | 文件操作 |
| `/api/v1/models/*` | 模型查询 |
| `/api/v1/permissions/*` | 权限管理 |
| `/api/v1/oauth/*` | OAuth 登录 |
| `/api/v1/mobile/*` | 移动端接口 |
| GraphQL `/graphql` | GraphQL 查询 |
| WebSocket `/ws` | 实时通信 |

---

## 七、常见问题（FAQ）

| 问题 | 处理 |
|------|------|
| 启动时提示 `SECURITY: SECRET_KEY is using the default...` | 在 `.env` 中设置强 `SECRET_KEY`（生产必做） |
| 中文乱码 | CLI 已自动执行 `chcp 65001` 并设置 UTF-8，若终端仍乱码请改用 Windows Terminal |
| 聊天无响应 | 检查 API Key 是否有效、网络连通、`LLM_TIMEOUT_SECONDS` 是否过小 |
| exe 首次启动慢 | onefile 模式每次启动需解压到临时目录，属正常现象 |
| 提示 `Not a directory` / `File not found` | CLI 受项目目录沙箱限制，请确认路径在当前项目内 |
| 配置修改不生效 | YAML 配置支持热重载（1 秒轮询）；`.env` 修改需重启进程 |
| Shell 命令无输出 | 命令通过子进程执行，某些交互式程序（如 vim）不支持 |

---

## 八、卸载

```powershell
# pip 安装
pip uninstall codeagent-pro

# 清理用户数据（谨慎，含会话/配置/备份）
Remove-Item -Recurse $HOME\.codeagent
```
