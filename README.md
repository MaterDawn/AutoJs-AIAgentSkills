# AutoJs-AIAgentSkills

> **Auto.js 高级版（MCP 版）** AI Agent 技能文档 —— 供 Claude / Cursor / 任意支持 MCP（Model Context Protocol）的 AI 客户端加载学习，用于在 Auto.js 高级版上安全、规范地编写与调试自动化脚本。

---

## 一、这是什么

本文档是一份**写给 AI Agent 看的技能说明**（Skills），**同时覆盖两套引擎体系**：

| 引擎 | 说明 |
| --- | --- |
| **Node.js（第二代 API）** | V8 运行时，ES2021+，`async/await`、完整 `require()` 模块体系、npm、高性能与高强度代码加密 |
| **Rhino（第一代 API）** | 经典 JavaScript 引擎，兼容存量脚本生态 |

内容涵盖：

- **MCP 工具协议**：`learnAutoJsSkill` / `getUiLayout` / `runScript` / `stopAllScripts` / `checkSyntax` / `captureScreen` / `getMcpLog` 七个工具的参数、使用时机与七阶段任务编排流程；
- **引擎心智模型**：两代 API 的核心差异、异步模型硬约束、模块体系与迁移对照；
- **API 速查与实战模式**：控件定位、截图找图、OCR、悬浮窗、定时任务、Shell、npm 模块等；
- **常见低级错误清单**：AI 必查规则（含 MCP 协议层注意事项）。

AI 客户端连接设备的 MCP 服务后，通过 `learnAutoJsSkill` 工具一次性读取本文档即可完成"学习"，无需人工投喂。本文档同时内置在安装包 `assets/skills/` 内，随应用分发。

## 二、适用版本与安装

| 项目 | 说明 |
| --- | --- |
| 适用范围 | **Auto.js 高级版（MCP 版）**，内置 Node.js 引擎、Rhino 引擎与 MCP 服务 |
| 下载 | 官方下载页：[点击前往下载](https://api.intcl.top/down/) ；或直接下载最新 APK：[AutoJs高级版_MCP全新版本S.apk](../../releases/latest/download/AutoJs高级版_MCP全新版本S.apk) |
| 安装步骤 | 下载安装包 → 安装 → 启动应用 → **点击「登录」** 完成账号登录 |
| MCP 服务 | 在应用内的 AI Agent / MCP 页面开启服务开关（默认端口 `8765`，端口占用时自动向后递增，最大 `8795`） |

> MCP 服务开启后，同一局域网内的 AI 客户端即可通过 `http://<设备IP>:8765/mcp` 连接（端点 `POST /mcp`，JSON-RPC 2.0，无需 Token）。

## 三、快速开始（AI 客户端侧）

1. **连接**：`POST http://<设备IP>:8765/mcp`，调用 `tools/list` 确认七个工具可用；
2. **学习**：调用 `learnAutoJsSkill`（`maxChars` 建议 `200000`），AI 完整阅读技能文档；
3. **观察**：`getUiLayout` 获取界面控件树（Flutter/自绘界面改用 `captureScreen`）；
4. **编写**：按文档规范生成脚本 —— Node.js 模式（`"nodejs";` 头、`async function main()` 入口、全部 `await`、`require()` 导入模块）或 Rhino 模式（第一代全局 API）；
5. **运行**：**Node.js 代码直接 `runScript`，无需 `checkSyntax`**（其内部为 Rhino 解析器，不支持 ES2021+ 语法，会对 `async/await` 误报）；Rhino 代码先 `checkSyntax` 再运行；
6. **验证**：`getMcpLog` 读取执行日志确认结果，必要时 `captureScreen` 比对界面。

## 四、两条最高优先级规则（协议层注意事项）

1. **`checkSyntax` 不支持 Node.js**：Node.js 代码（`"nodejs";` 头 / `.node.js` / `.mjs`）一律跳过语法检测，直接 `runScript`，以 `getMcpLog` 为验证依据。`checkSyntax` 仅用于 Rhino（第一代）代码。
2. **MCP 请求体必须是纯 ASCII**：请求 JSON 中含中文（多字节 UTF-8）会导致 HTTP 响应挂起。规避：JSON 序列化时将所有非 ASCII 字符转为 `\uXXXX` 转义（如 Python `json.dumps(..., ensure_ascii=True)`）。脚本执行时中文正常还原。

## 五、文档结构

| 文件 | 说明 |
| --- | --- |
| [`AutoJs-AIAgentSkills.md`](./AutoJs-AIAgentSkills.md) | 技能文档本体（v1.4）—— 第一部分：Rhino 引擎技能；第二部分：Node.js 引擎技能 |

## 六、声明

- 本项目仅用于在**用户自有设备**上进行自动化效率工具的开发与学习；
- 请遵守目标应用的服务条款与当地法律法规，不得用于任何违法违规用途；
- 使用 MCP 服务时请注意网络安全：仅在可信局域网内开启，用完及时关闭服务开关。

## 许可

仅供学习交流使用。
