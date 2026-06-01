# Claude Desktop + DeepSeek 配置指南

> 无需 Anthropic 订阅，在 Claude Desktop GUI 中使用 DeepSeek V4 Pro / Flash

## 最终效果

```
Claude Sonnet 4.6  →  DeepSeek V4 Pro  (1M context)
Claude Haiku 4.5   →  DeepSeek V4 Flash (1M context)
```

模型选择器显示 Anthropic 名称（受限于 Anthropic 白名单），但实际 API 请求走 DeepSeek。

---

## 前提条件

- macOS（Windows 类似，路径不同）
- Node.js 18+
- DeepSeek API Key（[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) 获取）
- Claude Desktop 已安装（[claude.com/download](https://claude.com/download)）

---

## 一键安装

```bash
echo "sk-你的DeepSeek-API-Key" | npx -y deepclaude-mixed-setup
```

### 这行命令自动做了什么：

| 步骤 | 说明 |
|------|------|
| 创建本地代理 | `127.0.0.1:3200`，拦截并转发 API 请求 |
| 模型路由映射 | Sonnet/Haiku/Opus → DeepSeek V4 Pro/Flash |
| 写入 3P 配置 | `deploymentMode: 3p`，Claude Desktop 进入第三方模式 |
| 安装守护进程 | LaunchAgent，开机自动启动代理 |
| 重启 Claude Desktop | 立即生效 |

---

## 关键文件

| 文件 | 作用 |
|------|------|
| `~/Library/Application Support/Claude-3p/claude_desktop_config.json` | 3P 部署模式 |
| `~/Library/Application Support/Claude-3p/configLibrary/*.json` | 模型列表 |
| `~/.deepclaude-mixed/mixed-proxy.mjs` | 代理脚本 |
| `~/Library/LaunchAgents/com.deepclaude.proxy.plist` | 开机自启 |

---

## 验证

```bash
# 检查代理状态
curl -s http://127.0.0.1:3200/_proxy/status

# 返回示例：
# {"mode":"mixed","deepseekKey":"set","routes":{"opus":"deepseek-v4-pro","sonnet":"deepseek-v4-pro","haiku":"deepseek-v4-flash"}}
```

---

## CLI 环境变量（可选）

写入 `~/.zshrc` 让终端 `claude` 命令也走 DeepSeek：

```bash
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-你的Key"
export ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
export CLAUDE_CODE_EFFORT_LEVEL="max"
```

---

## 故障排查

| 问题 | 解决 |
|------|------|
| Desktop 不显示模型 | 确认 `Developer` 菜单已出现（Help → Troubleshooting → Enable Developer Mode） |
| 模型列表为空 | 重启代理：`launchctl kickstart -k gui/$(id -u)/com.deepclaude.proxy` |
| 401 认证错误 | 检查 DeepSeek API Key 是否正确、余额是否充足 |
| 代理未启动 | 查看日志：`tail -f /tmp/com.deepclaude.proxy.out.log` |
| 想切换回 Anthropic | `claude-mode 1p`（需要 Anthropic 订阅） |

---

## 为什么模型显示名改不了

2026 年 5 月起，Anthropic 在 Claude Desktop 中加入模型名白名单，Gateway 模式只接受官方 Anthropic 模型名（`claude-sonnet-4-6` 等）。社区目前没有绕过方法，但这不影响实际使用——代理已将请求路由到 DeepSeek。

---

## 参考

- [deepclaude-mixed-setup (npm)](https://www.npmjs.com/package/deepclaude-mixed-setup)
- [DeepSeek API 文档 - Anthropic 兼容端点](https://api-docs.deepseek.com/guides/anthropic_api)
- [Claude Code 官方文档](https://code.claude.com/docs/en/desktop-quickstart)
