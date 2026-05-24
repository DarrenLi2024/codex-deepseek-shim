# codex-shim

在 **Codex Desktop** 中使用你的自定义模型（DeepSeek、Claude、OpenAI 等），绕过官方模型白名单限制。

这是一个轻量本地 Python 代理服务器，它会伪装成 OpenAI Responses API 端点，Codex 将请求发到 shim 后，shim 根据配置将请求转发给真实的上游 API（OpenAI Chat Completions / Anthropic Messages），同时完成协议转换和适配。

> 基于 [0xSero/codex-shim](https://github.com/0xSero/codex-shim) 分支，专为 **DeepSeek V4** 等非 OpenAI 模型深度适配优化。

---

## 项目简介

Codex Desktop 默认只显示服务端 Statsig 配置中白名单内的模型。如果你有自己的 API key（DeepSeek、Claude、OpenAI、Z.ai、OpenRouter 等）并希望它们在模型选择器中以一等公民出现，这个 shim 可以帮你实现。

它的工作原理：

```
Codex Desktop ── /v1/responses ──▶ codex-shim (127.0.0.1:8765)
                                     │
                                     ├── provider "openai" ──▶ baseUrl/chat/completions
                                     ├── provider "anthropic" ──▶ baseUrl/messages
                                     └── provider "generic-…" ──▶ baseUrl/chat/completions
```

---

## 安装

### 前提条件

- Python 3.11+
- Codex Desktop（macOS arm64 测试通过，Linux/Windows 理论上也可用）

### 安装步骤

```bash
# 1. 克隆仓库
git clone https://github.com/DarrenLi2024/codex-shim ~/Documents/codex-shim
cd ~/Documents/codex-shim

# 2. 安装 Python 依赖
pip install aiohttp certifi

# 3. 创建命令行快捷方式
ln -s "$PWD/bin/codex-shim" ~/.local/bin/codex-shim
ln -s "$PWD/bin/codex-app"  ~/.local/bin/codex-app
ln -s "$PWD/bin/codex-model" ~/.local/bin/codex-model
```

确保 `~/.local/bin` 在 PATH 中。

---

## 使用方法

### 1. 配置模型

编辑 `~/.factory/settings.json`（如果目录和文件不存在则创建）：

```json
{
  "customModels": [
    {
      "model": "deepseek-v4-flash",
      "provider": "openai",
      "baseUrl": "https://api.deepseek.com/v1",
      "apiKey": "sk-你的DeepSeek密钥",
      "displayName": "DeepSeek V4 Flash",
      "noImageSupport": true,
      "maxContextLimit": 1000000
    },
    {
      "model": "deepseek-v4-pro",
      "provider": "openai",
      "baseUrl": "https://api.deepseek.com/v1",
      "apiKey": "sk-你的DeepSeek密钥",
      "displayName": "DeepSeek V4 Pro",
      "noImageSupport": true,
      "maxContextLimit": 1000000
    }
  ]
}
```

支持的上游类型：

| provider | 上游接口 |
|---|---|
| `openai` | `/v1/chat/completions`（如 DeepSeek、OpenAI） |
| `generic-chat-completion-api` | OpenAI 格式的通用聊天补全 |
| `anthropic` | `/v1/messages`（如 Claude、DeepSeek Anthropic 端点） |

模型列表中的**第一个**模型会自动成为默认模型（当 Codex 发送 gpt-5.5 请求时会被重定向到此模型）。

请注意：**API key 只保存在此配置文件中，shim 不会将其写入生成的文件。**

### 2. 生成目录并启动

```bash
# 生成模型目录（读取 settings.json，写入 catalog 和 config）
codex-shim generate

# 启动 shim 守护进程（后台运行于 127.0.0.1:8765）
codex-shim start

# 查看已注册的模型
codex-shim list

# 检查运行状态
codex-shim status
```

### 3. 启动 Codex Desktop 并连接 Shim

```bash
codex-shim app .
```

该命令会：
1. 确保 shim 正在运行
2. 向 Codex 临时配置中注入 shim 相关的模型提供商和目录
3. 启动 Codex Desktop

启动后，在 Codex 的模型选择器中即可看到你在 settings.json 中配置的所有模型。

### 4. 切换默认模型（可选）

```bash
# 查看可用模型
codex-model list

# 切换默认模型为 DeepSeek V4 Pro
codex-model deepseek-v4-pro

# 重启 Codex 使生效
codex-app
```

### 5. 停止

```bash
codex-shim stop          # 停止 shim 守护进程
codex-shim disable       # 停止并恢复 Codex 原始配置
```

---

## 适配与优化

本分支相对上游 0xSero/codex-shim 做了以下深度适配，重点解决 **DeepSeek V4 兼容性**问题：

### 1. DeepSeek V4 Thinking 模式兼容

DeepSeek V4 默认启用 thinking（推理）模式，API 会返回 `reasoning_content` 字段，并要求下一次请求时必须将推理内容回传。但 Codex 在截断对话历史时会丢掉这些推理内容，导致报错。

**解决方案**：在发送给 DeepSeek 的请求中主动禁用 thinking 模式，避免需要 round-trip 的推理内容。

### 2. reasoning_content 保留与传递

即使禁用了 thinking，translate 层仍完整实现了 `reasoning_content` 的解析和传递功能：
- `chat_completion_to_response`：从 DeepSeek 响应中提取 `reasoning_content`，转为 Responses API 的 `reasoning` 输出项
- `_responses_input_to_messages`：将推理内容暂存并附加到下一个 assistant 消息中，同时适配 tool call 场景

### 3. SSL 证书兼容

Python 3.14+ 默认不加载 macOS 系统信任存储，导致访问 HTTPS API 时出现 `[SSL: CERTIFICATE_VERIFY_FAILED]`。

**解决方案**：使用 `certifi` 库提供的 CA 证书包创建 SSL 上下文，应用到所有上游连接。

### 4. gpt-5.5 请求重定向

Codex Desktop 无论配置什么模型，首次请求总是发送 `gpt-5.5`。原始 shim 将其转发到 ChatGPT passthrough，需要 ChatGPT 订阅才能使用。

**解决方案**：将 `gpt-5.5` 请求自动重定向到 settings.json 中配置的**第一个自定义模型**，无需 ChatGPT 订阅。

### 5. Developer 角色映射

OpenAI Responses API 使用 `developer` 角色，但 DeepSeek 只支持 `system/user/assistant/tool`。

**解决方案**：在协议翻译层将 `developer` 自动映射为 `system`。

### 6. 模型选择器补丁（针对 macOS）

如果 Codex Desktop 的模型下拉列表只显示 "default" 而看不到自定义模型，说明 Statsig 服务端白名单启用了隐藏。可以通过修改 ASAR 包来关闭这个白名单：

```bash
sudo codex-shim patch-app
```

如需恢复原始 ASAR：

```bash
sudo codex-shim restore-app
```

> 注意：macOS SIP/TCC 可能阻止直接修改 `/Applications/Codex.app`，如遇权限问题建议备份并在恢复模式下调整。

---

## 命令参考

| 命令 | 说明 |
|---|---|
| `codex-shim generate` | 重新生成模型目录和配置 |
| `codex-shim start` | 启动后台守护进程 |
| `codex-shim stop` | 停止守护进程 |
| `codex-shim restart` | 重启守护进程 |
| `codex-shim status` | 健康检查 + 模型数量 |
| `codex-shim list` | 列出所有模型和路由 |
| `codex-shim model list` | 列出当前可用模型 |
| `codex-shim model use <slug>` | 设置默认模型 |
| `codex-shim app [path]` | 启动 Codex Desktop 并连接 shim |
| `codex-shim codex -- <args>` | 通过 shim 运行 Codex CLI |
| `codex-shim patch-app` | 修补 ASAR 以显示自定义模型 |
| `codex-shim restore-app` | 恢复原始 ASAR |

所有命令均支持 `--settings <path>` 和 `--port <port>`。

---

## 项目结构

```
codex_shim/              Python 源码（服务器 + CLI + 协议转换）
bin/codex-shim           主入口脚本
bin/codex-app            codex-shim app 的快捷方式
bin/codex-model          codex-shim model 的快捷方式
.codex-shim/             生成的目录、配置、日志、PID（已 gitignore）
tests/                   测试用例
```

---

## 安全说明

- API key **只存储**在 `~/.factory/settings.json` 中，不会出现在代码仓库中
- Shim 每次请求时实时从配置文件读取密钥，不会持久化到其他文件
- 本仓库所有提交均已检查不含任何敏感信息

---

## 许可

MIT — 参见 `LICENSE`。

Codex Desktop 是 OpenAI 的商标。本项目与其无关。
