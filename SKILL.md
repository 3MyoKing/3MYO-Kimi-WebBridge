---
name: 3MYO-Kimi-WebBridge
description: |
  通过本地守护进程控制用户真实浏览器（含登录态），支持导航、点击、输入、截图等操作。当用户想操作网页、自动化浏览器任务、抓取网页内容，或提及"浏览器""打开网址""截图"等关键词时使用此技能。
metadata:
  version: "1.0.4"
  enforcement: strict
---

# 3MYO-Kimi-WebBridge（强制规范版）

> **执行声明**：使用本技能时，必须严格遵循本文档中的检查清单。每完成一步，必须输出 `[✓]` 标记和简短结果。任何步骤失败，必须立即停止并按要求格式报告，禁止自行跳过或变通。

通过本地守护进程 `http://127.0.0.1:10086` 控制用户真实浏览器（保留登录会话）。

---

## 最终输出模板（必须严格遵循）

任务结束后，无论成功/失败/部分成功，必须按以下 JSON 结构输出，不得添加额外解释。

> 所有字段必须基于实际观察结果，禁止推测、编造或预填未验证信息。

```json
{
  "status": "success | failure | partial",
  "session": "本次任务 session 名",
  "steps_completed": ["步骤1", "步骤2", "..."],
  "failed_step": null | "步骤X",
  "error": null | "具体错误描述",
  "result": {
    "url": "当前页面 URL",
    "title": "当前页面标题",
    "summary": "任务结果摘要（一句话，仅基于已验证状态）"
  },
  "screenshots": ["C:/path/1.png", "..."],
  "files": ["C:/path/file.pdf", "..."],
  "notes": "需要用户关注的事项，无则留空"
}
```

**字段约束**：
- `steps_completed`：仅包含已完成且验证通过的步骤。
- `failed_step`：第一个失败步骤的名称。若成功，填 `null`。
- `result.summary`：必须只描述已确认的事实。失败时以「失败：」开头。
- `screenshots` / `files`：只填写实际存在、已用 Read 工具确认的文件。禁止预填路径。

---

## 核心规则（禁止违反）

| 规则 | 说明 |
|------|------|
| **一个任务 = 一个 session** | 选定后全程不变，跨网站也不换 |
| **每步必须验证** | 每次 click/fill/evaluate 后必须 snapshot/screenshot 确认，否则不得进入下一步 |
| **优先用 @e 引用** | snapshot 返回的可访问性树引用比 CSS 选择器更稳定。详见「优先用 snapshot」 |
| **Windows 必须用 `curl.exe`** | PowerShell 中 `curl` 是 `Invoke-WebRequest` 别名，会炸 |
| **Windows 中文必须文件传参** | 中文 JSON 必须写入临时文件后用 `--data-binary @路径` 提交。详见「调用格式」 |
| **Windows `group_title` 必须英文** | 用 `Task.1`、`Task.2`…中文在 Git Bash 下变 `????` |
| **任务结束必须关 session** | 调用 `close_session`；仅当用户明确说"等下还要用"时才保留 |
| **截图 = 文件路径** | 守护进程返回路径不是图片本身，必须用 Read 或 present_files 工具打开确认 |
| **evaluate = IIFE + 紧凑 JSON** | `(() => { ... })()` + `JSON.stringify(data)` 不带格式化 |
| **fill = 替换非追加** | 要追加 → 先 evaluate 读当前值 → 拼接 → fill |
| **taskkill 禁止 `/F`** | 先正常关闭等 8-10s；强制杀进程必须用户确认 |
| **仅保留一个会话窗口** | 用户确认目标后，关闭多余浏览器/窗口，避免 WebBridge 连错 |

---

## 标准工作流（必须逐项执行）

```
收到任务 → 系统自检 → 健康检查 → 导航+等待 → 操作+验证（循环） → 清理 → 输出最终 JSON
```

---

## 步骤 1：系统自检（每次任务前必须执行）

> **目的**：输出当前运行环境信息，让用户了解系统配置并选择浏览器配置。

### 1.1 检测系统环境

必须执行：

```bash
# 检测运行中的浏览器
tasklist.exe | grep -i "msedge.exe" || echo "EDGE_NOT_RUNNING"
tasklist.exe | grep -i "chrome.exe" || echo "CHROME_NOT_RUNNING"

# 列出所有可用 Profile
echo "=== Edge Profiles ===" && ls -la "$LOCALAPPDATA/Microsoft/Edge/User Data/" | grep -E "^d.*(Default|Profile)" | awk '{print $NF}'
echo "=== Chrome Profiles ===" && ls -la "$LOCALAPPDATA/Google/Chrome/User Data/" | grep -E "^d.*(Default|Profile)" | awk '{print $NF}'

# 检测核心命令行工具
echo "=== curl.exe ===" && (curl.exe --version | head -n 1 || echo "CURL_NOT_FOUND")
echo "=== tasklist.exe ===" && (tasklist.exe /? > /dev/null 2>&1 && echo "TASKLIST_OK" || echo "TASKLIST_NOT_FOUND")
echo "=== taskkill.exe ===" && (taskkill.exe /? > /dev/null 2>&1 && echo "TASKKILL_OK" || echo "TASKKILL_NOT_FOUND")
echo "=== bash ===" && (bash --version | head -n 1 || echo "BASH_NOT_FOUND")

# 检测可选辅助工具（用于 JSON 预检等优化场景）
echo "=== python ===" && (python --version 2>&1 || echo "PYTHON_NOT_FOUND")
echo "=== node ===" && (node --version 2>&1 || echo "NODE_NOT_FOUND")

# 检测 Kimi WebBridge 守护进程二进制
echo "=== WebBridge binary ===" && (test -f "$USERPROFILE/.kimi-webbridge/bin/kimi-webbridge.exe" && echo "WEBBRIDGE_BINARY_OK" || echo "WEBBRIDGE_BINARY_NOT_FOUND")
```

**完成标准**：获得浏览器运行状态 + Profile 列表 + 核心工具可用性状态。

### 1.2 输出系统自检报告

必须按以下格式完整输出给用户：

```
系统配置：
- 系统：Windows
- 平台：WorkBuddy
可选浏览器配置：
- Edge：1. Default  2. Profile 1  ...
- Chrome：3. Default  4. Profile 1  ...
（标注当前正在运行的浏览器）

工具状态：
- curl.exe：已安装 / 未找到
- tasklist.exe：已安装 / 未找到
- taskkill.exe：已安装 / 未找到
- bash：已安装 / 未找到
- python：已安装 / 未找到（可选，用于 JSON 预检）
- node：已安装 / 未找到（可选，用于 JSON 预检）
- Kimi WebBridge 二进制：已安装 / 未找到

> 注意：不得猜测哪个 Profile 有扩展。必须通过后续 status 检查验证。
```

**完成标准**：用户收到完整报告。

### 1.3 用户选择浏览器配置

必须等待用户选择**编号或名称**指定浏览器 + Profile。常见场景：

| 场景 | 用户选择后操作 |
|------|--------------|
| 目标浏览器已在运行 | 跳到步骤 2 健康检查 |
| 目标浏览器未运行 | 启动该浏览器指定 Profile |
| 多个浏览器在运行 | 关闭非目标浏览器后启动目标 |

**快速通道（必须同时满足以下所有条件）**：

- 自检后**只有一个可用浏览器 + Profile**
- 该浏览器**已在运行**
- **步骤 2 健康检查已通过**，即 `running: true` 且 `extension_connected: true`

满足以上条件时，可直接使用该配置继续，同时在报告中注明：

```
[✓] 步骤1.3：自动选择唯一可用配置：Edge Profile 1（用户未指定其他配置）
```

并在最终输出 `notes` 中记录：「已自动选择唯一可用浏览器配置，如需更换请重新指定」。

**若以上任一条件不满足，必须停下来让用户选择。**

**完成标准**：用户已明确指定浏览器 + Profile，或已按快速通道自动选定。

### 1.4 仅保留一个会话窗口（关键规则）

> **⚠️ 多浏览器冲突**：如果 Edge 和 Chrome 都装了 WebBridge 扩展且同时运行，守护进程会随机连到其中一个。操作前必须统一到一个浏览器窗口。

**必须按以下顺序执行**：

1. 提示用户：「将关闭 [非目标浏览器/多余窗口]，仅保留 [目标浏览器 + Profile]，是否继续？」
2. 获得确认后，安全关闭其他窗口：
   ```bash
   # 正常关闭（禁止 /F）
   taskkill.exe /IM msedge.exe > /dev/null 2>&1   # 或 chrome.exe
   sleep 8
   # 检查残留 — 如仍存在，提示用户手动关闭
   tasklist.exe | grep -i "msedge.exe" > /dev/null && echo "⚠️ 仍有进程残留，请手动关闭"
   ```
3. 等待 8-10s 确保登录态和 Cookie 正常保存
4. 如只剩下目标浏览器但仍有多个窗口/Profile 标签页，提示用户手动关闭多余窗口

**完成标准**：系统中只有一个目标浏览器窗口在运行。

### 1.5 启动指定 Profile 的浏览器

若目标浏览器未运行，必须启动指定 Profile：

```bash
# 推荐：CMD（窗口弹出最稳定）
cmd.exe /c "start \"\" \"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe\" --profile-directory=\"Profile 1\""
cmd.exe /c "start \"\" \"C:\Program Files\Google\Chrome\Application\chrome.exe\" --profile-directory=\"Default\""

# 备选：PowerShell
powershell -Command "Start-Process 'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe' -ArgumentList '--profile-directory=Profile 1'"
```

启动后必须等待 8-10s，然后进入步骤 2 验证 `extension_connected: true`。

**完成标准**：目标浏览器已启动并等待足够时间。

---

## 步骤 2：健康检查（每次任务前必须执行）

必须执行：

```bash
curl.exe -s http://127.0.0.1:10086/status
```

| 状态 | 强制操作 |
|------|---------|
| `running: true` + `extension_connected: true` | ✅ 输出 `[✓] 步骤2：健康检查通过` |
| `status` 无响应（二进制已存在） | 启动守护进程：`& "$env:USERPROFILE\.kimi-webbridge\bin\kimi-webbridge.exe" start`，然后重试 status |
| `status` 无响应（二进制不存在） | **自动安装**（见下方「守护进程自动安装」），装完启动后重试 status |
| `running: true` + `extension_connected: false` | ❌ **立即停止**，向用户报告：扩展未连接，需打开浏览器或重启 Kimi Desktop |

**完成标准**：`running: true` 且 `extension_connected: true`。

---

## 步骤 3：导航 + 等待

首次导航必须设置 session 和 group_title。等待后必须 snapshot 验证页面已加载，未完成则延长等待：

| 页面类型 | 初始等待 | 验证失败时延长 |
|---------|---------|--------------|
| 静态页面 / 已缓存页面 | 3s | 延长至 5s |
| 普通动态页面（抖音、百度等） | 5s | 延长至 8s |
| 复杂首屏 / 大量资源 | 5-8s | 延长至 10s |

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true,"group_title":"Task.1"},"session":"my-task"}'
sleep 3
# 然后必须调用 snapshot 验证页面加载完成
```

**完成标准**：导航返回 `success: true`，且 snapshot 验证页面已加载完成。

---

## 步骤 4：操作 + 验证（循环执行直到任务完成）

每次循环必须遵循：

```
snapshot 读页面 → 分析定位 → 操作（click/fill/evaluate）→ 等待 → snapshot 确认 → 输出 [✓] 标记 → 重复或结束
```

### 4.1 读取页面（必须用 snapshot）

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"my-task"}'
```

**完成标准**：获得 `{url, title, tree}`，能识别目标元素 `@e` 引用。

### 4.2 定位元素

- **优先使用 `@e` 引用**（来自 snapshot 的可访问性树）
- 仅当 snapshot 中没有目标元素时，才回退到 CSS 选择器
- 严禁使用模糊、易变的 class name（如 `._3xkj`）

**回退条件**：详见「优先用 snapshot，而非 CSS/JS 选择器」章节。

**完成标准**：明确知道要操作哪个元素。

### 4.3 执行操作

| 操作 | 命令示例 | 完成后必须 |
|------|---------|-----------|
| 点击 | `click` with `selector:"@e10"` | 等待 2-3s，然后 snapshot 确认 |
| 填写 | `fill` with `selector:"@e5"`, `value:"xxx"` | snapshot 或 evaluate 确认值已填入 |
| 执行 JS | `evaluate` with IIFE | 验证返回值符合预期 |
| 截图 | `screenshot` | 用 Read 或 present_files 打开确认 |

**完成标准**：操作返回 `success: true`，且下一步验证通过。

### 4.4 验证操作结果

操作后必须立即验证：

```bash
# 等待后重新 snapshot
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"snapshot","args":{},"session":"my-task"}'
```

验证内容：
- 页面标题是否变化
- URL 是否变化
- 目标文本是否出现/消失
- 目标元素是否可继续操作

**若验证失败**：
1. 立即截图
2. 输出 `[✗]` 标记
3. 进入「失败处理」流程
4. **禁止继续下一步**

---

## 步骤 5：清理（任务结束时必须执行）

必须执行：

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"close_session","args":{},"session":"my-task"}'
```

例外：用户明确说"等下还要用"或"先不关"，才可保留 session。保留时必须告知用户。

**完成后必须**：删除过程中产生的所有临时 JSON 文件。

---

## 工具速查

| 工具 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `navigate` | `url`, `newTab`(bool), `group_title` | `{success, url, tabId}` | 首次调用开新标签页；`group_title` 设置标签组名 |
| `find_tab` | `url`, `active`(bool) | `{success, url, tabId}` | 切换到已打开的标签页 |
| `snapshot` | — | `{url, title, tree}` 含 `@e` 引用 | **读取页面内容的主要方式** — 可访问性树 |
| `click` | `selector`（@e 引用或 CSS） | `{success, tag, text}` | 模拟 `el.click()` |
| `fill` | `selector`, `value` | `{success, tag, mode}` | 对 `<input>`/`<textarea>` 写入值；对 `[contenteditable]` 写入文本。具体返回字段以实际响应为准，必须通过 snapshot 验证内容是否已变更。 |
| `evaluate` | `code`（支持 async/await） | `{type, value}` | 页面上下文中执行 JS |
| `cdp` | `method`, `params` | CDP 原始响应 | Chrome DevTools Protocol 直通，低级后门 |
| `screenshot` | `format`(png/jpeg), `quality`, `selector`(@e/CSS), `path` | `{format, path, sizeBytes, mimeType}` | 返回文件路径 — 用 Read 工具打开 |
| `network` | `cmd`(start/stop/list/detail), `filter`, `requestId` | 请求/响应数据 | 抓取网络请求 |
| `upload` | `selector`, `files`(string[]) | `{success, fileCount}` | 上传文件 |
| `save_as_pdf` | `paper_format`, `landscape`, `scale`, `print_background`, `path` | `{path, sizeBytes, mimeType, pageTitle}` | PDF 上限 100 MB |
| `list_tabs` | — | `{success, tabs:[...]}` | 查看当前 session 所有标签页 |
| `close_tab` | — | `{success, closed: bool}` | 关闭当前标签页 |
| `close_session` | — | `{success, closed: int}` | 关闭 session 全部标签页 |

### 标签页与当前标签页

单标签页工具（`snapshot`、`click`、`fill`、`screenshot`、`save_as_pdf`）作用于**当前标签页** — 即最近用 `navigate` 打开或用 `find_tab` 选中的标签页。

- **开新页面**：需要页面共存（对比、交叉引用）时用 `newTab:true`；不设则复用当前标签页。
- **切回之前的标签页**：用 `find_tab` + 完整 URL。`active:true` 选用户正在看的标签页。
- 如果 `find_tab` 返回"no open tab found" → 用 `navigate` + `newTab:true`。

---

## 会话管理

**一个任务 = 一个 session。**`session` 是请求体的**顶层字段**（不在 `args` 内），用于在 WebBridge 守护进程中区分不同任务。

- 任务开始时选定 session 名，全程不变 — 哪怕切换不同网站。
- 按**任务**命名（如 `camping-research`、`phone-compare`），不要按网站命名。
- **`group_title` 只是浏览器标签分组名称**，与 `session` 是不同概念。`group_title` 可选，Windows 下只允许英文（`Task.1`、`Task.2`…）。Git Bash MinGW 会将中文损毁为 `????`。macOS/Linux 可用用户语言。

```bash
# 首标签页：设置 session + group_title（Windows 用英文）
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true,"group_title":"Task.1"},"session":"camping-research"}'

# 同一任务，不同网站 → 相同 session
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -d '{"action":"navigate","args":{"url":"https://other.com","newTab":true},"session":"camping-research"}'
```

---

## 调用格式

每条命令携带 `session`。以下示例省略仅为简洁。

**macOS / Linux** — 直接内联 JSON：

```bash
curl -s -X POST http://127.0.0.1:10086/command \
  -H 'Content-Type: application/json' \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true,"group_title":"My task"},"session":"my-task"}'
```

**Windows** — **必须 `curl.exe`**，禁止裸 `curl`。Shell 会损毁非 ASCII 字符：

1. 用 Write 工具写入唯一命名的临时文件（禁止 shell echo/heredoc）
2. 用 `curl.exe --data-binary @路径` 提交
3. 提交后立即删除临时文件

```powershell
# 无中文 — 可直接内联：
curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" -d '{"action":"snapshot","args":{},"session":"my-task"}'

# 有中文 — 必须走文件传参：
# 1. Write 工具创建 C:\Users\3MYO\AppData\Local\Temp\webbridge-req-abc123.json
# 2. curl.exe -s -X POST http://127.0.0.1:10086/command -H "Content-Type: application/json" --data-binary "@C:\Users\3MYO\AppData\Local\Temp\webbridge-req-abc123.json"
# 3. 删除临时文件
```

---

## 操作模式

### 打开页面并读取内容

```bash
curl.exe -s http://127.0.0.1:10086/status                                    # 健康检查
curl.exe ... -d '{"action":"navigate","args":{"url":"...","newTab":true,"group_title":"Task.1"},"session":"demo"}'
sleep 5
curl.exe ... -d '{"action":"snapshot","args":{},"session":"demo"}'            # 读页面
curl.exe ... -d '{"action":"close_session","args":{},"session":"demo"}'       # 清理
```

### 点击元素 + 验证

```bash
curl.exe ... -d '{"action":"snapshot","args":{},"session":"demo"}'             # 找 @e 引用
curl.exe ... -d '{"action":"click","args":{"selector":"@e10"},"session":"demo"}'
sleep 3
curl.exe ... -d '{"action":"snapshot","args":{},"session":"demo"}'             # 确认结果
```

### 表单填写（逐步，每步验证）

```bash
curl.exe ... -d '{"action":"fill","args":{"selector":"@e5","value":"username"},"session":"demo"}'
curl.exe ... -d '{"action":"snapshot","args":{},"session":"demo"}'             # 确认填写
curl.exe ... -d '{"action":"fill","args":{"selector":"@e6","value":"password"},"session":"demo"}'
curl.exe ... -d '{"action":"click","args":{"selector":"@e7"},"session":"demo"}' # 提交
sleep 5
curl.exe ... -d '{"action":"snapshot","args":{},"session":"demo"}'             # 确认结果
```

### 填入中文参数（Windows 关键模板）

> **核心规则**：任何包含中文的 JSON 都不能内联到 curl 命令中，Shell 会将中文损毁为 `?`。必须走文件传参。详见「调用格式」章节。

```bash
# 1. 用 Write 工具写 JSON 到临时文件（路径用绝对路径）
#    → C:\Users\3MYO\AppData\Local\Temp\webbridge-req-xxxxx.json
#    内容：{"action":"fill","args":{"selector":"@e3","value":"世界杯"},"session":"my-task"}

# 2. 用 --data-binary 提交文件
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  --data-binary "@C:\Users\3MYO\AppData\Local\Temp\webbridge-req-xxxxx.json"

# 3. 提交后立即删除临时文件
rm -f "C:\Users\3MYO\AppData\Local\Temp\webbridge-req-xxxxx.json"
```

**兜底方案**：如果 `fill` 返回 `"Uncaught"`，直接用 URL 参数搜索：

```bash
# 直接导航到搜索结果页（中文在 URL 中也会被 curl 编码，走文件传参最安全）
# Write 工具写入文件：{"action":"navigate","args":{"url":"https://www.baidu.com/s?wd=世界杯"},"session":"my-task"}
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  --data-binary "@C:\Users\3MYO\AppData\Local\Temp\webbridge-nav.json"
rm -f "C:\Users\3MYO\AppData\Local\Temp\webbridge-nav.json"
```

### 用 evaluate 提取数据

```bash
curl.exe ... -d '{"action":"evaluate","args":{"code":"(() => { const items = document.querySelectorAll(\".product-item\"); return JSON.stringify([...items].map(i => ({name: i.querySelector(\".title\").innerText, price: i.querySelector(\".price\").innerText}))); })()"},"session":"demo"}'
```

---

## 执行效率优化（在稳定性前提下提速）

### 1. 减少不必要的 snapshot（慎用）

> **默认规则仍是每步操作后验证。以下优化仅在同一页面同一操作已连续成功 3 次以上，且用户明确同意时才可用。**

- 单步操作成功且明显无副作用时，可跳过单独验证 snapshot。
- 组合操作（fill + click）后，统一用一次 snapshot 验证结果。
- 仅当页面状态不确定、或操作失败时，才逐步 snapshot 排查。

### 2. 缩短等待时间

- 静态元素：`sleep 2-3`
- 普通动态页面：`sleep 3-5`
- 复杂页面 / 网络慢：`sleep 5-8`

避免所有步骤都固定 `sleep 5`。

### 3. evaluate 用文件传参并预检 JSON

复杂 JS 代码容易因引号嵌套导致 JSON 错误。写入文件前，先在本地检查 JSON 是否可解析（可用 python/node 快速校验），避免往返重试。

### 4. 直播间进入策略（需用户确认）

抖音搜索结果的「进入直播间」链接有时对合成点击不响应。可尝试以下替代路径：

1. 进入直播搜索页面，并验证页面加载正常。
2. 用 `evaluate` 提取页面中所有包含 `live.douyin.com` 的链接。
3. **将提取到的 URL 完整输出给用户确认，禁止直接 navigate。**
4. 用户确认后，再 navigate 到该 URL。

> 注意：不得自行构造 `live.douyin.com/{room_id}`，必须从当前页面实际提取。

---

## 优先用 snapshot，而非 CSS/JS 选择器

`snapshot` 返回基于语义角色/名称的 `@e` 引用，不受 CSS 类名变化影响。

仅在以下情况回退到 `evaluate`（JS）：
- 目标元素在 snapshot 中没有 `@e` 引用
- 需要 snapshot 不包含的属性（如 `href`）
- 需要派发复杂事件序列或滚动操作

---

## Evaluate 技巧

- **IIFE 包裹**：`(() => { ... })()` — 避免跨调用 `const`/`let` 重复声明报错。
- **紧凑 JSON**：`JSON.stringify(data)` — 绝不用 `null, 2` 格式化。缩进和换行会使响应膨胀数倍导致截断。

---

## 文本输入 — 用 `fill`

适用于 `<input>`/`<textarea>` 和 `[contenteditable]` 富文本编辑器（如 ProseMirror/TipTap/Lexical）。无论返回字段如何，都必须用 snapshot 验证实际内容是否已变更。

**fill 是清空后填入** — 会替换已有内容。要追加：先 evaluate 读当前值 → 拼接 → fill。

---

## 表单提交 / 特殊按键

直接点击提交按钮。派发按键事件（如 Escape 关闭弹窗）：

```bash
{"action":"evaluate","args":{"code":"document.activeElement.dispatchEvent(new KeyboardEvent('keydown',{key:'Escape',bubbles:true}))"}}
```

---

## 保存页面为 PDF

`save_as_pdf` 可选参数：`paper_format`（letter/a4/legal/a3/tabloid）、`landscape`、`scale`（0.1-2.0）、`print_background`、自定义 `path`。上限 100 MB。

---

## 失败处理（必须遵守）

### 立即停止条件

出现以下任一情况，**必须立即停止执行，不得继续**：

1. `status` 无响应且启动守护进程后仍无响应
2. `extension_connected: false`
3. 连续 2 次 snapshot 找不到目标元素
4. `click`/`fill` 返回 `success: false`
5. 页面出现验证码、登录框、支付页面、安全确认
6. 操作结果与预期不符且重试 1 次后仍不符
7. 用户要求停止

### 失败报告格式

停止时必须按以下格式输出：

```
⚠️ 执行暂停
步骤：xxx
原因：xxx
已截图：C:/path/xxx.png
建议用户操作：xxx
```

并输出最终 JSON：

```json
{
  "status": "failure",
  "session": "my-task",
  "steps_completed": ["步骤1", "步骤2"],
  "failed_step": "步骤3",
  "error": "具体错误描述",
  "result": {
    "url": "当前 URL",
    "title": "当前标题",
    "summary": "失败摘要"
  },
  "screenshots": ["C:/path/xxx.png"],
  "files": [],
  "notes": "需要用户关注的事项"
}
```

### 错误速查表

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| `status` 无响应 | 守护进程未运行 | 按步骤 2 启动守护进程并重新健康检查 |
| `extension_connected: false` | 扩展未连接 | 重启浏览器/Kimi Desktop；重装扩展 |
| navigate 成功但 snapshot 为空 | 页面未加载完成 | 延长 sleep 至 5-10s |
| click/fill `success: false` | 元素不可交互 | 重新 snapshot，重新定位元素 |
| fill 返回 `"Uncaught"` | 页面拒绝 fill（如百度搜索框） | 用兜底方案：直接 `navigate` 到搜索 URL |
| 页面卡死/验证码 | 需要人工介入 | 停止并告知用户手动处理 |
| "请更新 Kimi WebBridge 扩展" | 扩展版本过旧 | https://www.kimi.com/zh-cn/features/webbridge |
| 端口 10086 被占用 | Desktop App 与其他 agent 冲突 | 关闭一个或共用守护进程 |
| 激活了错误的 Profile | 多个 Profile 装了扩展 | 重新确认 Profile，截图验证 |
| WebBridge 连到错误浏览器 | Edge 和 Chrome 同时运行 | 关闭非目标浏览器 |
| Cookie/登录态丢失 | 强制杀进程或等待不足 | 正常关闭 + 8-10s 等待；`--restore-last-session` |
| taskkill 输出乱码（GBK） | Git Bash 无法解码 GBK | 加 `> /dev/null 2>&1` |
| `MSYS_NO_PATHCONV` 错误 | Git Bash 路径转换冲突 | 前缀加 `MSYS_NO_PATHCONV=1` |
| 插件干扰 | 其他扩展冲突 | 禁用爬虫/录屏/AI 类扩展 |
| 切换浏览器后 `extension_connected: false` | 目标 Profile 未安装扩展 | 确认目标 Profile 有扩展 → 非目标浏览器完全关闭后等待 15s → 仍不行时让用户手动确认 |
| 扩展中途断开 | 浏览器切换或网络闪断 | 等待 5s 重试；`status` 确认后重新 navigate |

### 守护进程恢复

**连接被拒绝时，按步骤 2 处理**：先检查二进制是否存在，存在则启动守护进程，不存在则自动安装。不要问用户：

**检查二进制是否存在**：

```bash
test -f "$USERPROFILE/.kimi-webbridge/bin/kimi-webbridge.exe" && echo "BINARY_OK" || echo "BINARY_MISSING"
```

**若二进制已存在 → 直接启动**：

```powershell
& "$env:USERPROFILE\.kimi-webbridge\bin\kimi-webbridge.exe" start
```

**若二进制不存在 → 自动安装（见下方「守护进程自动安装」）**。

启动后重新执行 `curl.exe -s http://127.0.0.1:10086/status`。仍失败 → 指引用户访问 https://www.kimi.com/zh-cn/features/webbridge。

**禁止自动执行**：`stop`、`restart`、`uninstall`。

### 守护进程自动安装（二进制不存在时）

> **触发条件**：`status` 无响应且 `$USERPROFILE/.kimi-webbridge/bin/kimi-webbridge.exe` 不存在。
> **执行原则**：无需询问用户，直接安装。安装完成后自动启动守护进程并重试健康检查。

#### 方式 A：官方安装脚本（推荐，但 WorkBuddy PowerShell 工具可能拦截 `iex`）

用户在终端中可直接执行：

```powershell
irm https://cdn.kimi.com/webbridge/install.ps1 | iex
```

但在 WorkBuddy 的 PowerShell 工具中，`iex`（Invoke-Expression）会被安全策略拦截。因此需走方式 B。

#### 方式 B：手动下载二进制 + 启动（WorkBuddy 环境实测有效）

**第 1 步：下载二进制到 bin 目录**（用 PowerShell 工具执行）：

```powershell
$BinDir = Join-Path $env:USERPROFILE '.kimi-webbridge\bin'
New-Item -ItemType Directory -Path $BinDir -Force | Out-Null
$BinPath = Join-Path $BinDir 'kimi-webbridge.exe'
Invoke-WebRequest -Uri 'https://cdn.kimi.com/webbridge/latest/releases/kimi-webbridge-windows-amd64.exe' -OutFile $BinPath -UseBasicParsing -TimeoutSec 120
Write-Host "Installed to $BinPath, size: $((Get-Item $BinPath).Length) bytes"
```

**第 2 步：启动守护进程**（用 PowerShell 工具执行）：

```powershell
$BinPath = Join-Path $env:USERPROFILE '.kimi-webbridge\bin\kimi-webbridge.exe'
& $BinPath start
Start-Sleep -Seconds 3
# 验证
try {
    $resp = Invoke-WebRequest -Uri 'http://127.0.0.1:10086/status' -UseBasicParsing -TimeoutSec 5
    Write-Host "Status: $($resp.Content)"
} catch {
    Write-Host "Status check FAILED: $($_.Exception.Message)"
}
```

**第 3 步：验证 `running: true` + `extension_connected: true`**，通过后进入步骤 3。

#### 安装注意事项

| 事项 | 说明 |
|------|------|
| **跳过技能安装** | 官方脚本默认会 `install-skill`，可能覆盖本技能的定制内容。若用方式 A，加 `-NoSkill` 参数：`iex "& { $(irm https://cdn.kimi.com/webbridge/install.ps1) } -NoSkill"`。方式 B 不涉及技能安装，无此问题。 |
| **版本不匹配警告** | daemon 与 extension 版本可能不一致（如 daemon v1.11.2 vs extension v1.10.0），功能仍可用，无需处理。如出现「请更新 Kimi WebBridge 扩展」提示，则需用户手动更新扩展。 |
| **安装后扩展未连接** | 若 `extension_connected: false`，说明浏览器未装 WebBridge 扩展或浏览器未运行。指引用户访问 https://www.kimi.com/zh-cn/features/webbridge 安装扩展。 |
| **二进制大小参考** | 约 10 MB（v1.11.x），下载超时设 120s 足够。 |

---

## 已知限制

| 限制 | 详情 | 变通方案 |
|------|------|----------|
| `isTrusted` 检查 | 银行/验证码页面拒绝合成 `click`/`fill` | 告知用户手动交互 |
| 跨域 iframe | 工具仅操作顶层框架 | 直接导航到 iframe 的 URL |
| 扩展冲突 | 爬虫/录屏/AI 扩展会导致失败 | 禁用其他扩展，仅保留 WebBridge |
| 动态页面 | 复杂/加载中的页面容易操作失败 | 简化操作；先截图确认页面状态 |
| `@e` 引用点击不精准 | 复杂页面中 `@e` ref 可能点到嵌套子元素 | 优先用 `navigate` 直达目标 URL；必须点击时用 evaluate 定位并 `el.click()` |
| `evaluate` 中 `setTimeout` 无效 | evaluate 立即返回，异步回调可能未执行 | 改用同步操作或分两步 evaluate |
| Profile 扩展安装情况不可见 | 只有连上才知道哪个 Profile 有扩展 | 不得猜测。必须通过 status 验证；优先使用用户明确指定的 Profile |

---

## 版本不匹配

如果工具返回 **"请更新 Kimi WebBridge 扩展"**：不要重试，不要自行协调版本 — 告知用户更新：
- 英文：https://www.kimi.com/features/webbridge
- 中文：https://www.kimi.com/zh-cn/features/webbridge

---

## 安全与隐私

- 所有操作均在本地执行，登录会话和页面内容不会离开本机。
- 本技能仅用于授权范围内的浏览器自动化任务，不得越权操作。
- 每台计算机需单独配置 WebBridge 扩展和守护进程。
- 涉及购买、删除、发布、权限变更的操作，必须获得用户明确确认后方可执行。
