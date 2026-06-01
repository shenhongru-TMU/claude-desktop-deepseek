# Claude Desktop + DeepSeek 完整配置指南

> 无需 Anthropic 订阅，在 Claude Desktop GUI 中使用 DeepSeek V4 Pro / Flash
> 
> 环境：macOS 15.5 · Claude Desktop v1.9659.2 · DeepSeek V4 · 2026年6月

---

## 架构

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Claude Desktop  │ ──▶ │  本地代理 :3200   │ ──▶ │  DeepSeek API    │
│  (GUI 应用)      │ ◀── │  mixed-proxy     │ ◀── │  anthropic 端点   │
│                  │     │  模型名映射+转发   │     │                  │
│ 显示: Sonnet 4.6 │     │ sonnet→v4-pro    │     │ deepseek-v4-pro  │
│ 实际: V4 Pro     │     │ haiku →v4-flash  │     │ deepseek-v4-flash │
└─────────────────┘     └─────────────────┘     └──────────────────┘
```

---

## 前提条件

| 条件 | 说明 |
|------|------|
| macOS 12+ | Windows 类似，路径不同 |
| Node.js 18+ | `brew install node` |
| DeepSeek API Key | [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) 获取并充值 |
| Claude Desktop | [claude.com/download](https://claude.com/download) 下载安装 |

---

## 安装步骤（一步完成）

```bash
echo "sk-你的DeepSeek-API-Key" | npx -y deepclaude-mixed-setup
```

### 这行命令内部做了什么

```
1. 检测系统环境（darwin/win32/linux）
2. 创建 ~/.deepclaude-mixed/mixed-proxy.mjs  代理脚本
3. 写入 ~/Library/LaunchAgents/com.deepclaude.proxy.plist  守护进程
4. 写入 ~/Library/Application Support/Claude-3p/claude_desktop_config.json
   └── deploymentMode: "3p"
5. 写入 ~/Library/Application Support/Claude-3p/configLibrary/*.json
   └── inferenceModels: claude-sonnet-4-6, claude-haiku-4-5-20251001
6. 启动代理 （127.0.0.1:3200）
7. 重启 Claude Desktop
```

---

## 安装细节与踩坑记录

### 瓶颈 1：`launchctl setenv` 对 GUI 应用无效

**现象**：在终端设置 `launchctl setenv ANTHROPIC_BASE_URL ...`，Claude Desktop 读不到。

**原因**：macOS 从 10.10 起，`launchctl setenv` 只影响 launchd 启动的进程。从 Dock/Finder 打开的 GUI 应用由 `loginwindow` 启动，不会继承这些变量。

**解决**：
- **方案 A**（推荐）：从终端直接启动 Claude Desktop，变量由 shell 继承：
  ```bash
  ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic" \
  /Applications/Claude.app/Contents/MacOS/Claude &
  ```
- **方案 B**：使用 `deepclaude-mixed-setup`，它通过 LaunchAgent + 本地代理绕过这个问题。

---

### 瓶颈 2：`chatIn3p` 功能不可用

**现象**：Claude Desktop 进程参数显示 `"chatIn3p":{"status":"unavailable"}`。

**原因**：`chatIn3p`（Chat in Third-Party）是 Anthropic 服务器下发的功能开关。未开启 Developer Mode 时默认为 `unavailable`。

**解决**：启用 Developer Mode。
1. 打开 Claude Desktop
2. 菜单栏 → **Help** → **Troubleshooting** → **Enable Developer Mode**
3. App 重启后菜单栏出现 **Developer** 菜单

> 如果找不到这个选项，可以用 AppleScript 自动化：
> ```bash
> osascript -e 'tell app "Claude" to activate' -e '
> tell app "System Events" to tell process "Claude"
>   click menu bar item "Help" of menu bar 1
>   delay 0.3
>   click menu item "Troubleshooting" of menu 1 of menu bar item "Help" of menu bar 1
>   delay 0.3
>   click menu item "Enable Developer Mode" of menu 1 of ¬
>     menu item "Troubleshooting" of menu 1 of menu bar item "Help" of menu bar 1
> end tell'
> ```

---

### 瓶颈 3：模型名白名单（核心瓶颈）

**现象**：Claude Desktop 报错：
```
configured model "deepseek-v4-pro" is not an Anthropic model.
Gateway deployments require an Anthropic model from the provider catalog.
```

**原因**：2026年5月6日起，Anthropic 在 Claude Desktop 1.6259.1+ 中加入模型名白名单。Gateway 模式只接受精确匹配的官方模型名（如 `claude-sonnet-4-6`、`claude-haiku-4-5-20251001`）。

**尝试过的方法及结果**：

| 方法 | 结果 |
|------|------|
| 直接用 `deepseek-v4-pro` | ❌ 被白名单拦截 |
| 加 `claude-` 前缀 `claude-deepseek-v4-pro` | ❌ 同样被拦截（精确匹配，非前缀匹配） |
| 添加 `displayName` 字段 | ❌ 被忽略，不显示 |
| 修改 `~/Library/Application Support/Claude/claude_desktop_config.json` | ❌ 不是这个文件的职责 |
| 修改 `Claude-3p/configLibrary/*.json` | ❌ 名字改了但校验不过 |
| 降级到旧版 Claude Desktop | ✅ 但失去新功能 |
| 二进制补丁（ModelIDPatch） | ⚠️ 有安全风险 |

**最终方案**：接受 Anthropic 官方模型名，通过代理做底层映射。显示名无法修改，但请求已经正确路由到 DeepSeek。

---

### 瓶颈 4：代理 /v1/models 端点

**现象**：修改了 `inferenceModels` 但重启后被覆盖为 Anthropic 名称。

**原因**：deepclaude-mixed-setup 在首次运行时写入了配置，后续 Claude Desktop 会从网关拉取模型列表。

**解决**：在代理中添加 `/v1/models` 端点，返回正确的模型列表：
```javascript
// ~/.deepclaude-mixed/mixed-proxy.mjs 中添加
if (req.method === 'GET' && req.url === '/v1/models') {
    res.writeHead(200, { 'content-type': 'application/json' });
    res.end(JSON.stringify({
        data: [
            { id: 'claude-sonnet-4-6', type: 'model', display_name: 'DeepSeek V4 Pro' },
            { id: 'claude-haiku-4-5-20251001', type: 'model', display_name: 'DeepSeek V4 Flash' },
        ]
    }));
    return;
}
```

---

## 代理路由逻辑

```javascript
// ~/.deepclaude-mixed/mixed-proxy.mjs 核心路由
function pickRoute(model, { deepseekKey }) {
    const m = (model || '').toLowerCase();
    
    if (m.includes('flash') || m.includes('haiku')) {
        // → DeepSeek V4 Flash
        return { host: 'api.deepseek.com', remap: 'deepseek-v4-flash' };
    }
    // sonnet / opus / deepseek / v4-pro / 默认
    // → DeepSeek V4 Pro
    return { host: 'api.deepseek.com', remap: 'deepseek-v4-pro' };
}
```

请求流程：
1. Claude Desktop 发送请求，`model: "claude-sonnet-4-6"`
2. 代理收到请求，匹配到 `sonnet` → 路由到 DeepSeek
3. 代理将 `model` 重写为 `deepseek-v4-pro`
4. 请求转发到 `https://api.deepseek.com/anthropic/v1/messages`
5. DeepSeek 返回响应 → 代理透传给 Claude Desktop

---

## 关键文件清单

| 文件 | 作用 |
|------|------|
| `~/.deepclaude-mixed/mixed-proxy.mjs` | 代理脚本（模型映射 + 请求转发） |
| `~/Library/LaunchAgents/com.deepclaude.proxy.plist` | 开机自启守护进程 |
| `~/Library/Application Support/Claude-3p/claude_desktop_config.json` | 3P 部署模式配置 |
| `~/Library/Application Support/Claude-3p/configLibrary/*.json` | 模型列表（必须用 Anthropic 名） |
| `~/Library/Application Support/Claude-3p/configLibrary/_meta.json` | 活跃配置索引 |
| `~/.zshrc` | CLI 环境变量（可选） |

---

## 验证

```bash
# 检查代理状态
curl -s http://127.0.0.1:3200/_proxy/status
# {"mode":"mixed","deepseekKey":"set","routes":{"sonnet":"deepseek-v4-pro","haiku":"deepseek-v4-flash"}}

# 检查模型列表
curl -s http://127.0.0.1:3200/v1/models | python3 -m json.tool

# 检查代理进程
pgrep -fl mixed-proxy

# 查看代理日志
tail -f /tmp/com.deepclaude.proxy.out.log
```

---

## CLI 环境变量（终端使用）

写入 `~/.zshrc`：

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

| 问题 | 原因 | 解决 |
|------|------|------|
| Desktop 不显示任何模型 | Developer Mode 未开启 | Help → Troubleshooting → Enable Developer Mode |
| 提示 "not an Anthropic model" | inferenceModels 用了非 Anthropic 名 | 改为 `claude-sonnet-4-6` 等官方名称 |
| 模型列表为空 | 代理未运行 | `launchctl kickstart -k gui/$(id -u)/com.deepclaude.proxy` |
| 401 认证错误 | API Key 问题 | 检查 DeepSeek Key 是否正确、余额是否充足 |
| 402 Payment Required | 余额不足 | 去 platform.deepseek.com 充值（¥1 即可） |
| 代理未启动 | LaunchAgent 加载失败 | 查看日志 `tail -f /tmp/com.deepclaude.proxy.out.log` |
| Developer 菜单消失 | 3P 配置覆盖了设置 | 重新启用 Developer Mode |
| 代理端口被占用 | 其他程序占用了 3200 | `lsof -i :3200` 检查并释放 |
| 想切换回 Anthropic | 需要 Anthropic 订阅 | `claude-mode 1p` 或删除 3P 配置 |

---

## 模型映射对照表

| Claude Desktop 显示 | 代理路由 | DeepSeek 实际模型 | 上下文 | 适用场景 |
|---|---|---|---|---|
| Claude Sonnet 4.6 | sonnet → v4-pro | deepseek-v4-pro | 1M | 主力编码、复杂推理 |
| Claude Haiku 4.5 | haiku → v4-flash | deepseek-v4-flash | 1M | 快速问答、简单任务 |

---

## 成本对比

| 方案 | 月费 | 年费 |
|------|------|------|
| Anthropic Pro 订阅 | $20/月 | $240/年 |
| DeepSeek API 按量 | ¥0.5-2/百万 token | 取决于用量，通常 <¥50/月 |

---

## 参考

- [deepclaude-mixed-setup (npm)](https://www.npmjs.com/package/deepclaude-mixed-setup)
- [DeepSeek API - Anthropic 兼容端点](https://api-docs.deepseek.com/guides/anthropic_api)
- [Claude Code Desktop 官方文档](https://code.claude.com/docs/en/desktop-quickstart)
- [Claude Desktop 3P Configuration](https://claude.com/docs/cowork/3p/configuration)
