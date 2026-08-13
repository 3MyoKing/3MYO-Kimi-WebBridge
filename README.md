# 3MYO-Kimi-WebBridge

> 通过本地守护进程控制用户真实浏览器（含登录态），支持导航、点击、输入、截图、PDF 保存等操作。

这是 **3MYO-Kimi-WebBridge** 技能的官方文档仓库，面向 WorkBuddy、Kimi 等 Agent 平台，提供一套可复用、带强制检查清单的浏览器自动化规范。

---

## 功能特性

- **真实浏览器控制**：基于本地守护进程 `http://127.0.0.1:10086`，保留用户登录态与 Cookie
- **完整操作指令**：导航、点击、填写、截图、保存 PDF、网络抓包、JS 执行、文件上传
- **稳定元素定位**：优先使用可访问性树 `@e` 引用，降低 CSS 类名变化带来的脆弱性
- **强制规范输出**：每次任务必须按标准 JSON 模板汇报状态、结果、截图与文件
- **Windows 深度适配**：针对 Git Bash、PowerShell、中文编码、Profile 切换等场景给出明确规则

---

## 仓库结构

```
3MYO-Kimi-WebBridge/
├── README.md                          # 本文件：项目首页与快速入口
├── SKILL.md                           # 技能主文档：完整规范、工作流、工具速查、失败处理
├── references/
│   └── operations.md                  # 守护进程生命周期、浏览器管理、恢复与排错
└── Kimi WebBridge 操作指南.md          # 原始中文操作指南（历史参考）
```

| 文档 | 用途 |
|------|------|
| `SKILL.md` | Agent 执行浏览器任务时的**主规范**，包含检查清单、调用示例、错误速查 |
| `references/operations.md` | 守护进程安装/启动/恢复、浏览器 Profile 管理、多浏览器冲突处理 |
| `README.md` | 项目简介、快速开始、文档索引 |

---

## 快速开始

### 1. 环境要求

- Windows（主要支持平台）/ macOS / Linux
- Edge 或 Chrome 浏览器
- 已安装 Kimi WebBridge 扩展
- `curl.exe`（Windows 必须带 `.exe`）

### 2. 启动守护进程

```bash
# Windows PowerShell
& "$env:USERPROFILE\.kimi-webbridge\bin\kimi-webbridge.exe" start

# macOS / Linux
~/.kimi-webbridge/bin/kimi-webbridge start
```

### 3. 健康检查

```bash
curl.exe -s http://127.0.0.1:10086/status
```

期望返回：

```json
{
  "running": true,
  "extension_connected": true,
  "version": "1.x.x",
  "extension_version": "1.x.x"
}
```

### 4. 打开一个页面

```bash
curl.exe -s -X POST http://127.0.0.1:10086/command \
  -H "Content-Type: application/json" \
  -d '{"action":"navigate","args":{"url":"https://example.com","newTab":true,"group_title":"Task.1"},"session":"demo"}'
```

详细调用格式、中文参数处理、失败恢复等内容请阅读 [`SKILL.md`](./SKILL.md)。

---

## 标准工作流

```
收到任务 → 系统自检 → 健康检查 → 导航+等待 → 操作+验证（循环） → 清理 → 输出最终 JSON
```

每个步骤都有明确的 `[✓]` 标记要求与失败停止规则，详见 `SKILL.md`。

---

## 核心规则摘要

| 规则 | 说明 |
|------|------|
| 一个任务 = 一个 session | 全程不变，跨网站也不换 |
| 每步必须验证 | click/fill/evaluate 后必须 snapshot 或 screenshot 确认 |
| 优先用 `@e` 引用 | snapshot 返回的可访问性树引用比 CSS 选择器更稳定 |
| Windows 必须用 `curl.exe` | PowerShell 中 `curl` 是 `Invoke-WebRequest` 别名 |
| Windows 中文必须文件传参 | 中文 JSON 写入临时文件后用 `--data-binary @路径` 提交 |
| 任务结束必须关 session | 调用 `close_session`，除非用户明确说保留 |

---

## 版本

- 当前技能版本：**1.0.4**
- 适用守护进程：`kimi-webbridge`（本地 HTTP 服务，端口 `10086`）

---

## 安全与隐私

- 所有浏览器操作均在本地执行，登录会话和页面内容不会离开本机。
- 涉及购买、删除、发布、权限变更的操作，必须获得用户明确确认后方可执行。
- 每台计算机需单独配置 WebBridge 扩展和守护进程。

---

## 相关链接

- [SKILL.md](./SKILL.md) — 完整技能规范
- [references/operations.md](./references/operations.md) — 守护进程与浏览器运维指南
- [Kimi WebBridge 官方页面](https://www.kimi.com/zh-cn/features/webbridge)

---

*由 3MyoKing 维护。欢迎通过 Issue 提交问题或改进建议。*
