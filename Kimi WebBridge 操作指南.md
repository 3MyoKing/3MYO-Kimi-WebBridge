# Kimi WebBridge-win操作说明

## 环境说明

WebBridge 守护进程运行在 `http://127.0.0.1:10086`，通过 HTTP POST 发送 JSON 命令操控用户真实浏览器。

### 两种运行场景

| 场景 | 守护进程管理 | 注意事项 |
|------|------------|---------|
| Kimi Desktop App 内置 agent | 随 App 打开自动启动、关闭自动停止 | 不要用 CLI 命令 start/stop/restart，会与 Desktop App 冲突 |
| 其他本地 agent（OpenClaw、Claude Desktop 等） | 需独立安装，agent 自行管理生命周期 | 安装：Windows irm https://cdn.kimi.com/webbridge/install.ps1 \| iex，macOS/Linux curl -fsSL https://cdn.kimi.com/webbridge/install.sh \| bash |

- 二进制位置：~/.kimi-webbridge/bin/kimi-webbridge.exe（Windows: %USERPROFILE%.kimi-webbridge\bin\kimi-webbridge.exe）

### 支持范围

- **浏览器**：Chrome 和 Edge（建议最新版）。两者扩展独立，需分别在各自应用商店安装
- **Agent**：所有 Local Agent — Kimi Desktop/Kimi Code、Claude Code、Codex CLI、Cursor、Hermes Claw 等。配置指令相同

---

## 标准工作流

```
接到任务 → 确认浏览器配置 → 健康检查 → 导航+等待 → 操作+验证 → 收尾报告
```

### 1. 确认浏览器配置文件（必做）

> **⚠️ 多浏览器冲突**：如果 Edge 和 Chrome 都装了扩展，**WebBridge 会连接到任意一个**，导致操作发生在错误的浏览器上。**切换浏览器前必须关闭另一个！**

**流程：检测进程 → 关闭非目标浏览器 → 用户选择 Profile → 按需重启 → 截图验证**

#### 检测浏览器与配置文件

```bash
# 检查进程（两个都检查）
tasklist.exe | grep -i "msedge.exe" || echo "EDGE_NOT_RUNNING"
tasklist.exe | grep -i "chrome.exe" || echo "CHROME_NOT_RUNNING"

# 列出配置文件
echo "=== Edge ===" && ls -la "$LOCALAPPDATA/Microsoft/Edge/User Data/" | grep -E "^d.*(Default|Profile)" | awk '{print $NF}'
echo "=== Chrome ===" && ls -la "$LOCALAPPDATA/Google/Chrome/User Data/" | grep -E "^d.*(Default|Profile)" | awk '{print $NF}'
```

**检测结果对应的处理：**

| 检测结果 | 操作 |
|---------|------|
| 只有 Edge 运行 | 列出 Edge Profile，让用户选择 |
| 只有 Chrome 运行 | 列出 Chrome Profile，让用户选择 |
| **两者都运行** | **⚠️ 必须先关闭非目标浏览器**，再让用户选 Profile |
| 都没运行 | 列出两个浏览器的 Profile，让用户选浏览器和 Profile |

> **为什么必须关闭另一个浏览器？**
> WebBridge 守护进程只能连接一个浏览器扩展。如果 Edge 和 Chrome 都运行且都装了扩展，WebBridge 会随机连接到其中一个，导致后续操作发生在错误的浏览器上。

#### 关闭非目标浏览器

```bash
# 如果目标是 Chrome，先关闭 Edge（正常关闭）
taskkill.exe /IM msedge.exe > /dev/null 2>&1
sleep 8

# 检查是否还有残留进程
tasklist.exe | grep -i "msedge.exe" > /dev/null
if [ $? -eq 0 ]; then
  echo "⚠️ Edge 无法正常关闭，请手动关闭 Edge 浏览器后再继续"
  echo "等待用户手动操作后，按 Enter 继续..."
  read -p ""
fi

# 如果目标是 Edge，先关闭 Chrome（正常关闭）
taskkill.exe /IM chrome.exe > /dev/null 2>&1
sleep 8

# 检查是否还有残留进程
tasklist.exe | grep -i "chrome.exe" > /dev/null
if [ $? -eq 0 ]; then
  echo "⚠️ Chrome 无法正常关闭，请手动关闭 Chrome 浏览器后再继续"
  echo "等待用户手动操作后，按 Enter 继续..."
  read -p ""
fi
```

> ⚠️ **禁止强制关闭浏览器**（`taskkill /F`）。如果正常关闭失败，**必须暂停并提示用户手动关闭**，等待用户确认后再继续。

#### 让用户确认 Profile

> 您选择了 **Edge + Profile 1**，确认吗？
>
> 选项：1. 控制当前活跃窗口（不关闭） / 2. 关闭后用指定 Profile 重启

#### 关闭浏览器（需重启时）

```bash
# 正常关闭（给浏览器保存 Cookie/登录态的时间）
taskkill.exe /IM msedge.exe > /dev/null 2>&1   # 或 chrome.exe > /dev/null 2>&1
sleep 8

# 检查残留进程
tasklist.exe | grep -i "msedge.exe" > /dev/null
if [ $? -eq 0 ]; then
  echo "⚠️ 浏览器无法正常关闭，请手动关闭浏览器后再继续"
  echo "等待用户手动操作后，按 Enter 继续..."
  read -p ""
fi
```

> ⚠️ **禁止强制关闭浏览器**（`taskkill /F`）。强制终止会导致 Cookie/登录状态丢失。
>
> **如果正常关闭失败，必须暂停并提示用户手动关闭**，等待用户确认后再继续。绝不能直接执行 `taskkill /F`。

> ⚠️ 正常关闭后建议等 8-10 秒再启动，数据量大或磁盘慢时尤其注意。

> ⚠️ `taskkill` 输出是 GBK 编码，Git Bash 会显示乱码。建议加 `> /dev/null 2>&1` 忽略输出。

#### 启动指定 Profile

**⚠️ 推荐方式：CMD（最可靠，窗口必弹出）**

```bash
# Edge (Profile 1)
cmd.exe /c "start \"\" \"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe\" --profile-directory=\"Profile 1\""

# Chrome (Default)
cmd.exe /c "start \"\" \"C:\Program Files\Google\Chrome\Application\chrome.exe\" --profile-directory=\"Default\""
```

**备选方式：PowerShell**（如果 CMD 方式窗口不弹出）

```bash
# Edge (Profile 1)
powershell -Command "Start-Process 'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe' -ArgumentList '--profile-directory=Profile 1'"

# Chrome (Default)
powershell -Command "Start-Process 'C:\Program Files\Google\Chrome\Application\chrome.exe' -ArgumentList '--profile-directory=Default'"
```

> ⚠️ **启动后必须等待 8-10 秒**，让浏览器和扩展完全加载，再检查 `extension_connected` 状态。

> ⚠️ **从沙盒环境启动可能不稳定**（进程启动但窗口不弹出）。如果 `extension_connected` 始终为 `false`，需让用户手动打开浏览器。

> ⚠️ **Profile 参数格式**：
> - Edge: `--profile-directory="Profile 1"`（带引号，空格）
> - Chrome: `--profile-directory=Profile 1`（不带引号）
> 具体格式取决于浏览器版本，如果启动后扩展未连接，尝试调整引号。

**验证启动成功：**

```bash
# 等待 8-10 秒后检查
sleep 10 && curl.exe -s http://127.0.0.1:10086/status

# 期望结果：extension_connected: true
# 如果仍是 false，让用户手动打开浏览器或检查扩展是否已安装
```

Chrome 路径通常为 `C:\Program Files\Google\Chrome\Application\chrome.exe`，Profile 命名与 Edge 相同（Default / Profile 1 / Profile 2...）。验证方式：截图确认图标（Chrome 彩色 vs Edge 蓝绿色），或访问 `chrome://version/` 查看「个人资料路径」。

#### 验证连接正确

启动后务必验证：1. `screenshot` 截图看浏览器外观和登录账号；2. `snapshot` 检查页面内容；3. 不对则停下来让用户手动切换。

### 2. 健康检查（任务开始前必做）

```bash
curl.exe -s http://127.0.0.1:10086/status
```

| 状态 | 处理 |
|------|------|
| `running: true` + `extension_connected: true` | ✅ 正常，继续 |
| 无响应 | Desktop App：检查 App 是否打开。其他 agent：`kimi-webbridge.exe start` |
| `running: true` 但 `extension_connected: false` | 桌面版：重启 Kimi 桌面版。其他 Agent：重装 `curl -fsSL https://kimi-web-img.moonshot.cn/webbridge/install_skill.sh \| bash -s -- -y` 后重启 Agent |

### 3. 导航到目标页面

**⚠️ 强制规范：`group_title` 必须用英文命名（`Task.1`, `Task.2`, ...），禁止用中文！**

```bash
# 标准格式（必须遵循）
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true,"group_title":"Task.1"},"session":"task-name"}'
```

**命名规则：**
- `group_title`: `Task.1`, `Task.2`, `Task.3`...（按任务序号递增）
- `session`: 用英文描述任务（如 `baidu-open`, `github-login`, `jd-search`）
- **禁止用中文命名**，会导致乱码（`？？？？`）

**为什么不能用中文？**
- Git Bash 的 MinGW 会损坏命令行中的中文字符
- 即使通过文件传参，标签组名称仍可能显示异常
- 英文命名是经过验证的可靠方案

- 首次必用 `newTab:true`；`group_title` 只在首次设置，后续 navigate 不指定
- 一个任务 = 一个 `session` 名，**全程不变**

### 4. 等待页面加载

navigate 后必须等待（`sleep 3` 起），动态页面延长至 5-8 秒，否则 snapshot 可能拿到空页面。

### 5. 操作 + 验证（循环）

```
snapshot → 分析页面 → 决定下一步 → 等待 → snapshot 验证 → 继续/结束
```

**关键原则**：每次操作后都验证，不要连续做两个操作再验证。

### 6. 收尾（任务结束必做）

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"close_session","args":{},"session":"task-name"}'
```

---

## 工具速查

| 工具 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `navigate` | `url`, `newTab`(bool), `group_title` | `{success, url, tabId}` | 首次必用 `newTab:true` |
| `find_tab` | `url`, `active`(bool) | `{success, url, tabId}` | 按 **domain** 匹配；`active:true` 返回当前标签 |
| `snapshot` | — | `{url, title, tree}` + `@e` refs | **优先用 @e 引用**，比 CSS 选择器稳定 |
| `click` | `selector` (@e 或 CSS) | `{success, tag, text}` | 合成 `el.click()` |
| `fill` | `selector`, `value` | `{success, tag, mode}` | **替换**现有内容，不是追加。`<input>`/`<textarea>` 用原生 setter；`[contenteditable]` 用 `execCommand('insertText')` |
| `evaluate` | `code` (支持 async/await) | `{type, value}` | 用 IIFE `(() => { ... })()` 包裹；`JSON.stringify` **不加** `null, 2` 缩进 |
| `screenshot` | `format`(png/jpeg), `quality`, `selector` | `{format, dataLength, data}` | 用 `selector` 只截特定元素更快；**不要直接返回 base64**，会淹没上下文 |
| `network` | `cmd`(start/stop/list/detail), `filter`, `requestId` | request/response 数据 | 抓取网络请求 |
| `upload` | `selector`, `files`(string[]) | `{success, fileCount}` | 上传文件 |
| `save_as_pdf` | `paper_format`, `landscape`, `scale`, `print_background`, `file_name` | `{path, sizeBytes, mimeType, pageTitle}` | 保存到 `/tmp/kimi-webbridge-pdfs/`；上限 100MB |
| `list_tabs` | — | `{success, tabs:[...]}` | 列出当前会话标签 |
| `close_tab` | — | `{success, closed: bool}` | 关闭当前标签 |
| `close_session` | — | `{success, closed: int}` | 关闭会话所有标签，**任务结束必调用** |

---

## 调用格式

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true}}'
```

**Windows 注意**：
- 永远用 `curl.exe`（不是 `curl`，PowerShell 中 `curl` 是 `Invoke-WebRequest` 别名）
- 中文内容先写入 JSON 文件，再用 `--data-binary @文件路径` 传参（命令行直接内联中文会损毁）

---

## 截图处理

> **不要直接返回 screenshot 的 base64 数据**，会淹没上下文。

**Kimi 环境**：使用 `scripts/screenshot.sh` helper：

```bash
bash "$(dirname "$SKILL_PATH")/scripts/screenshot.sh"          # 默认路径
bash "$(dirname "$SKILL_PATH")/scripts/screenshot.sh" -s task  # 指定 session
bash "$(dirname "$SKILL_PATH")/scripts/screenshot.sh" -o /tmp/page.png  # 自定义路径
bash "$(dirname "$SKILL_PATH")/scripts/screenshot.sh" -f jpeg -q 60     # JPEG 格式
```

**其他本地 agent**：保存 base64 到文件后用 Read 查看：

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"screenshot","args":{"format":"png"},"session":"my-task"}' > /tmp/screenshot.json

node -e "const fs=require('fs');const d=JSON.parse(fs.readFileSync('/tmp/screenshot.json','utf8'));fs.writeFileSync('/tmp/page.png',Buffer.from(d.data.data,'base64'));console.log('Saved')"
```

---

## 常见场景速查

### 打开页面并获取内容

```bash
curl.exe -s http://127.0.0.1:10086/status  # 健康检查
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true},"session":"demo"}'
sleep 5
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"demo"}'
# 任务结束
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"close_session","args":{},"session":"demo"}'
```

### 点击元素（含中文传参）

```bash
# 中文先写文件
echo '{"action":"navigate","args":{"url":"https://example.com","newTab":true},"session":"demo"}' > /tmp/req.json
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" --data-binary @/tmp/req.json

# snapshot 找 @e 引用
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"demo"}'

# 点击 + 验证
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"click","args":{"selector":"@e10"},"session":"demo"}'
sleep 3
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"demo"}'
```

### 表单填写（分步验证）

```bash
# 逐个填写，每次 fill 后建议 snapshot 验证
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"fill","args":{"selector":"@e5","value":"myusername"},"session":"demo"}'
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"fill","args":{"selector":"@e6","value":"mypassword"},"session":"demo"}'
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"click","args":{"selector":"@e7"},"session":"demo"}'
sleep 5
# 验证
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"demo"}'
```

### evaluate 提取数据

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" \
  -d '{"action":"evaluate","args":{"code":"(() => { const items=document.querySelectorAll(\".product-item\"); return JSON.stringify([...items].map(i=>({name:i.querySelector(\".title\").innerText,price:i.querySelector(\".price\").innerText}))); })()"},"session":"demo"}'
```

---

## 官方使用案例

> 来自 [Kimi WebBridge 帮助文档](https://www.kimi.com/zh-cn/help/kimi-webbridge/kimi-webbridge-use-cases)，展示 WebBridge 配合 Skill 和 CLI 的典型用法。

**使用技巧**：任务描述越具体越准确（明确目标网站、筛选条件、输出格式）；页面复杂时先截图确认状态。

| 类别 | 场景 | 说明 | 相关工具 |
|------|------|------|---------|
| **信息查询** | 旅游攻略规划 | 多平台跳转，对比价格时间，整理行程和预算 | Skill: `travel-planning`，CLI: `ctrip-cli`、`booking-cli` |
| **信息查询** | 租房信息筛选 | 多平台搜索，按条件筛选，统一整理推荐排序 | Skill: `rental-assistant`，CLI: `58-cli`、`anjuke-cli` |
| **内容调研** | 文献调研 | 搜索学术文献，提取摘要方法结论，输出综述 | Skill: `paper-research`，CLI: `scholar-cli` |
| **内容调研** | 话题深度搜索 | 自动搜索抓取正文，综合摘要或保留原文 | CLI: `google-cli`、`baidu-cli` |
| **日常办公** | 电商比价 | 多平台同商品比价，对比规格评价，整理最优方案 | — |
| **日常办公** | 网页数据提取 | 提取表格列表等结构化数据，整理成指定格式 | — |

**CLI/Skill 安装**：从 [Releases](https://github.com/better-world-ai/x-cli/releases) 下载 → `npx skills add better-world-ai/x-cli --skill <skill名>` → 在本地 Agent 中发送 prompt

---

## 已知限制与错误处理

### 已知限制

| 限制 | 说明 | 应对 |
|------|------|------|
| **isTrusted 检查** | 银行门户/验证码网站检查 `event.isTrusted`，`fill`/`click` 是 synthetic events 会被拒 | 产品边界，无法绕过 |
| **跨域 iframe** | `fill`/`click`/`evaluate`/`snapshot` 只能操作 top frame | 直接导航到 iframe 的 URL |
| **插件冲突** | 爬虫类、录屏类、AI 辅助类插件可能导致操作一直失败 | 关闭其他插件 → 只保留 WebBridge → 重启浏览器 → 逐个恢复定位冲突插件 |
| **动态页面** | 网页结构复杂或动态加载导致操作失败 | 简化指令，先截图确认页面状态 |

### 错误处理速查

| 错误情况 | 原因 | 处理 |
|---------|------|------|
| `status` 无响应 | daemon 未启动 | Desktop App：检查是否打开。其他 agent：`kimi-webbridge.exe start` |
| `extension_connected: false` | 扩展未安装/断开 | 桌面版：重启 Kimi。其他 Agent：重装 `curl -fsSL https://kimi-web-img.moonshot.cn/webbridge/install_skill.sh \| bash -s -- -y` |
| `navigate` 成功但 snapshot 为空 | 页面未加载完 | 延长 sleep 至 5-10 秒 |
| `click`/`fill` 返回 `success: false` | 元素不可操作/已消失 | snapshot 重新定位；fill 检查 role 和 tag |
| 页面报错/卡死 | 脚本错误或验证码 | 需用户手动操作 |
| 返回 `Please update the Kimi WebBridge extension` | 扩展版本过旧 | 访问 https://kimi.com/features/webbridge 更新，不要重试 |
| 端口 10086 被占用 | Desktop App 和其他 agent 同时运行 | 关闭其一，或共用已有守护进程 |
| 操作发生在错误 Profile | 多 Profile 都装了扩展 | 按第 1 步流程确认，截图验证 |
| WebBridge 连到了另一个浏览器 | Edge 和 Chrome 都装了扩展且都运行 | 关闭非目标浏览器 |
| `taskkill` 报错 "无效参数 - 'F:/'" | Git Bash MinGW 路径转换 | 加 `MSYS_NO_PATHCONV=1` |
| Cookie/登录状态丢失 | `taskkill /F` 或正常关闭后等待不足 | 先正常关闭等 8-10 秒；或加 `--restore-last-session` |
| Chrome 找不到扩展 | Chrome 扩展未安装 | https://kimi.com/features/webbridge 安装后重启 Chrome |
| 安装扩展提示"无法从该网站添加应用" | 未从官方商店安装 | 从 Chrome Web Store 或 Edge Add-ons 安装，或用手动安装方式 |
| Windows 安装报错 | 用了错误的安装命令 | 用 PowerShell：`irm https://kimi-web-img.moonshot.cn/webbridge/install.ps1 \| iex` |
| 运行连接指令无反应 | 网络问题 | 检查网络；重启 Kimi Claw Desktop 后重试 |

### 版本检查

```bash
curl.exe -s http://127.0.0.1:10086/status | jq '{version, extension_version}'
```

---

## 操作速记

| 类别 | 要点 |
|------|------|
| **多浏览器/多 Profile** | 先确认目标浏览器和配置文件，关闭非目标浏览器，截图验证连接正确 |
| **操作验证** | 每次操作后 snapshot/screenshot 验证，不连续做两个操作再验证 |
| **元素引用** | 优先用 @e 引用，比 CSS 选择器稳定 |
| **浏览器关闭** | 先 `taskkill /IM` 正常关闭等 8-10 秒；残留再用 `taskkill /F` |
| **浏览器启动** | 优先 CMD `start` 或 PowerShell；Git Bash `&` 后台可能窗口不弹出 |
| **evaluate** | IIFE 包裹，compact JSON（不加 `null, 2` 缩进） |
| **fill** | 替换不是追加；追加需先 evaluate 读取再拼接 |
| **session** | 一个任务一个 session 名，全程不变 |
| **收尾** | 任务结束必 `close_session` |
| **curl** | Windows 永远用 `curl.exe` |
| **中文传参** | 先 Write JSON 到文件，再 `--data-binary @路径` |
| **截图** | 不要直接返回 base64，用 helper 或保存到文件 |
| **插件冲突** | 操作异常时排查其他插件（爬虫/录屏/AI 类） |
| **任务描述** | 越具体越准确——明确网站、条件、输出格式 |

---

## 安全与隐私

- **所有操作在本地完成**：登录态和网页内容不会离开设备
- **Agent 只获取授权结果**：不会主动泄露信息
- **每台电脑需单独配置**
