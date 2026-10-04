\---  
title: Auto.js / Auto.js Pro (Node.js 引擎) AI Agent 技能说明  
description: 面向 AI Agent 的 Auto.js Pro Node.js（第二代 API）自动化开发技能，包含引擎启用方式、异步心智模型、模块体系与迁移对照、控件定位、截图找图、OCR、悬浮窗、定时任务、Shell、MCP 工具接口，以及观察界面、生成脚本、语法检查、执行验证和失败修复的完整编排流程  
tags: autojs, nodejs, v8, javascript, android, 自动化, 无障碍, mcp, agent, 第二代api  
version: 1.4  
engines: Node.js (V8, 第二代 API)  
sources:  
\- Auto.js Pro 官方文档（<https://www.wuyunai.com/docs/llms-v9.txt> / llms-full-v9.txt）  
\- Auto.js Pro 9.3.11 APK 内置 Node.js 官方示例（assets/sample/中文/Node.js/）  
\- Auto.js-AIAgentSkills.md（Rhino 版技能，本文档为其 Node.js 姊妹篇）  
\---

# Auto.js Node.js 脚本编写 · AI Agent 技能说明

> **给 AI 的阅读须知**：本文档是你在编写 Auto.js **Node.js（第二代 API）** 脚本时的"大脑速查卡"。Node.js 模式的最大错误来源是：**把 Rhino/第一代 API 的全局函数思维（直接用 `sleep`/`toast`/`click`）和浏览器思维带进来**。第二代 API 中**一切模块皆需 `require()`，一切阻塞皆需 `await`**。请先完整读完「第 1 节 心智模型」和「第 7 节 常见低级错误」，再动手写代码。
>
> **⚠️ 语法检测规则（最高优先级）**：MCP 的 `checkSyntax` 工具**不支持 Node.js**（内部是 Rhino 解析器，`async/await` 必然误报 SYNTAX_ERROR"失踪 ;声明前"）。**Node.js 代码（`"nodejs";` 头 / `.node.js` / `.mjs`）一律不要调用 checkSyntax，直接 `runScript` 运行，用 `getMcpLog` 验证结果。** checkSyntax 仅用于 Rhino（第一代）代码。
>
> 与本文件配套的 `Auto.js-AIAgentSkills.md` 覆盖 Rhino 引擎。用户未明确指定引擎时，默认 Rhino；指定 Node.js / 第二代 API / npm / 高性能 / 加密需求时，使用本文档。

---

## 0. MCP 驱动的自动化开发协议

本节是 AI Agent 使用本技能时的执行协议，与 Rhino 版技能的协议一致，仅在引擎相关细节上有差异。编写脚本时必须优先遵守本节。

### 0.1 工作目标

将自动化任务拆成 **观察、规划、生成、检查、执行、验证、修复** 七个阶段。不要只生成一段看似合理的代码，也不要在没有真实界面证据时声称任务完成。

默认工作环境：

- 目标设备：运行 Auto.js Pro 的 Android 设备。
- 引擎：**Node.js（第二代 API）**——脚本文件头必须带 `"nodejs";` 标识，或以 `.node.js` / `.mjs` 结尾。**忘记引擎标识是最高频错误**：不带标识的代码会静默落入 Rhino 引擎，Node API（`require('accessibility')` 等）全部报错。
- MCP 地址：用户提供的本地或局域网地址；端点 `POST /mcp`，JSON-RPC 2.0 的 `tools/list` 与 `tools/call`，当前不要求 Token（若服务端增加认证，使用用户提供的凭据，不要猜测）。
- **⚠️ HTTP 请求体必须是纯 ASCII（最高优先级）**：请求 JSON 中含中文（多字节 UTF-8）会导致 MCP 服务器 HTTP 响应挂起直至超时（脚本最终可能仍被执行，但调用方收不到响应）。**规避：JSON 序列化时把所有非 ASCII 字符转成 `\uXXXX` 转义**（如 Python `json.dumps(..., ensure_ascii=True)`、JS 无需处理时手写转义）。脚本源码中的中文字符串/注释经 `\uXXXX` 转义后在设备上执行完全正常。实测：纯 ASCII 的 1.7KB payload 秒回；含中文的 payload 响应挂起。

### 0.2 当前 MCP 工具接口

| 工具               | 参数                              | 用途                                                       | 使用时机                               |
| ---------------- | ------------------------------- | -------------------------------------------------------- | ---------------------------------- |
| `getUiLayout`    | `{}`                            | 获取当前界面 UI 布局 XML、节点数量和控件结构                               | 每个新任务开始时优先调用                       |
| `runScript`      | `source` 或 `file`，可选 `config`   | 执行脚本源码或文件。**执行 Node.js 代码时 `source` 必须以 `"nodejs";` 开头** | **Node.js 代码直接调用（无需 checkSyntax）**；Rhino 代码在 checkSyntax 通过后调用 |
| `stopAllScripts` | `{}`                            | 停止所有正在运行的脚本                                              | 仅用户明确允许时调用                         |
| `checkSyntax`    | `{code}`                        | JavaScript 语法检查。**⚠️ 不支持 Node.js**：内部为 Rhino 解析器，对 `async/await` 必然误报"失踪 ;声明前"。**Node.js 代码一律不要调用本工具** | 仅 Rhino（第一代）代码在执行前调用 |
| `captureScreen`  | `format?`、`quality?`、`dataUri?` | 截图并返回图像信息                                                | UI 树不完整、Flutter/自绘控件、图色场景或执行后验证时调用 |
| `getMcpLog`      | `maxChars?`                     | 读取脚本执行日志                                                 | 执行失败、无响应或需确认后台状态时调用                |

工具参数规则：

- `runScript` 必须提供 `source` 或 `file` 之一；`config` 的 `path/delay/interval/loopTimes/arguments` 字段执行端默认忽略，循环、延迟与参数处理应写在脚本内部。
- **`checkSyntax` 不支持 Node.js，Node.js 代码一律直接运行、无需检测**：checkSyntax 内部是 Rhino（第一代）解析器，ES5 与部分 ES6 可通过，但 **`async function` / `await` 必然误报 SYNTAX_ERROR**（实测报"失踪 ;声明前"，而同一代码 `runScript` 在 Node.js 引擎（V8）真机执行完全正常）。这是解析器代差问题，不是代码问题。**判断规则：代码带 `"nodejs";` 头（或 `.node.js` / `.mjs` 后缀）→ 跳过 checkSyntax，直接 `runScript`，以 `getMcpLog` 为验证依据；不要因 checkSyntax 报错而反复"修复"正常代码。** checkSyntax 仅用于 Rhino（第一代）代码。
- `captureScreen` 默认 png/质量 100；需要小图时用 jpg 或降质。
- `getMcpLog.maxChars` 默认 `20000`，上限 `200000`，从较小值开始读取。
- `stopAllScripts` 影响整个运行环境，不得当普通重试按钮。

### 0.3 自动化任务编排

**阶段 1：明确任务** —— 提取目标 App、页面、输入数据、成功标准与禁止操作；模糊目标先澄清并改写为可验证结果。

**阶段 2：观察环境** —— `getUiLayout` 获取 UI 树，提取目标控件的 `text`/`desc`/`id`/`className`/可点击父节点；UI 树为空或 Flutter/Canvas/WebView 自绘界面时改用 `captureScreen`。不要凭空编造控件属性。

**阶段 3：建立状态机** —— 先写页面状态与转移条件（等待权限 → 定位控件 → 执行动作 → 等待页面变化 → 终态/异常态），再写代码。

**阶段 4：生成代码** —— 按 Node.js 模式规范生成：`"nodejs";` 头、`async function main()` 入口、`await` 所有异步 API、`main().catch(console.error)` 兜底。

**阶段 5：语法检查（仅 Rhino 代码）** —— Rhino 代码须 `checkSyntax` 通过后再执行；**Node.js 代码跳过此阶段、无需任何检测**（checkSyntax 的 Rhino 解析器不支持 Node.js 语法，async/await 必然误报，见 0.2），**直接进入阶段 6 用 `runScript` 运行**。

**阶段 6：执行与验证** —— `runScript` → `getMcpLog` 确认日志/验证结果 → 必要时 `captureScreen` 比对界面状态。

**阶段 7：失败修复** —— 日志定位 → 修改假设 → 回到阶段 4。禁止连续无修改重试。

---

## 1. 引擎与运行时心智模型（必读）

### 1.1 启用 Node.js 引擎的三种方式

```javascript
// 方式一：文件头标识（最常用）
"nodejs";
console.log(`Node.js版本: ${process.version}`);   // Node.js 16.x

// 方式二：文件后缀 .node.js
// 方式三：文件后缀 .mjs（同时启用 ES Module，可用 import/export）
```

### 1.2 第二代 API 与第一代（Rhino）的核心差异

| 维度    | Rhino / 第一代 API               | Node.js / 第二代 API                         |
| ----- | ----------------------------- | ----------------------------------------- |
| 语言标准  | ES5 + 部分 ES6                  | ES2021+                                   |
| 性能    | 基线                            | 100 倍以上                                   |
| 代码加密  | 可被还原                          | **强度高，目前不可还原**                            |
| 模块使用  | 全局变量（`app`、`sleep` 直接用）       | **全部 `require()` 导入**                     |
| 阻塞    | `sleep()` 同步阻塞                | `await delay()`（lang 模块）                  |
| 截图权限  | `requestScreenCapture()` 同步阻塞 | `await requestScreenCapture()` 返回 Promise |
| npm   | 不支持                           | 完整支持（需 `--no-bin-links`）                  |
| 无障碍选择 | `findOne()` 同步                | 选择器方法返回 Promise                           |

### 1.3 异步模型（Agent 生成代码的硬约束）

1. 第二代 API 中 **绝大多数 IO/交互/等待类函数返回 Promise**，必须 `await`；
2. 顶层不能直接 `await`，标准入口结构：

```javascript
"nodejs";
async function main() {
    // 业务逻辑，全部 await
}
main().catch(console.error);   // 必须兜底 catch，否则 Promise 拒绝被静默吞掉
```

1. 事件监听（`events`/`sh.on('line')`/`process.on('exit')`）用回调，不用 await。

### 1.4 线程与 UI 线程

- Node.js 引擎是单线程事件循环；CPU 并行用 Node 自带 `worker_threads`。
- **UI 线程**：文件头写 `"ui-thread";` —— 在 UI 线程运行但无 Activity 页面，用于悬浮窗。**同一时刻只能有一个引擎占用 UI 线程**，多个悬浮窗须由同一引擎管理。
- 带页面的界面（`ui` 模块 / 原生工程）运行在自身 Activity 中，控件事件回调运行在 UI 线程，回调内不得执行长阻塞任务。

### 1.5 与 Java/Android 交互

```javascript
"nodejs";
// 方式一：$autojs.java（原始方式）
const $java = $autojs.java;
const StringBuilder = $java.findClass('java.lang.StringBuilder');
const sb = new StringBuilder();
sb.append("Hello").append(2);
console.log(sb.toString());

// 方式二：rhino 模块（类 Pro 8 体验）
require('rhino').install();
const sb2 = new java.lang.StringBuilder();
sb2.append(android.util.Base64.decode("YXV0b2pz", 0));
```

> 注意：暂不支持 `importClass` / `importPackage` / `JavaAdapter`。`$autojs.androidContext` 可直接取得 Android Context。

---

## 2. 脚本结构与工程规范

### 2.1 单文件脚本

```text
hello.node.js          ← 任意目录，双击/运行即执行
```

### 2.2 Node.js 工程（需要 npm / 打包 / 多文件时必选）

```text
MyProject/
├── project.json        ← 包名必须含 "."，如 com.example.myproject
├── main.js             ← 入口（带 "nodejs"; 头或用 main.node.js）
├── package.json
├── node_modules/       ← npm 安装产物
├── lib/                ← 业务模块
└── .autojs.build.ignore / .autojs.source.ignore   ← 打包/上传忽略清单
```

- 新建项目：文件管理 → 右下角菜单 → 项目 → 模板选 **Node.js 项目**。
- npm 安装：项目内终端执行 `npm i --no-bin-links <模块名>`。**sdcard 文件系统不支持符号链接，`--no-bin-links` 必须带**；需要执行包内命令时改用 `node node_modules/<pkg>/bin/xxx.js`。
- 全局模块：`npm i -g typescript`（**不要带 --no-bin-links**）；**禁止升级内置 npm 版本**。
- 默认脚本目录可改为应用私有目录（`~`，支持符号链接），但卸载即清空。

### 2.3 打包

Node.js 工程可直接打包为独立 APK（内置完整 Node 运行时），支持全加密 Project Zip 与快照，产物同样支持开机自启与热更新。

---

## 3. 模块体系与迁移对照

### 3.1 三层模块来源

| 来源                | 导入方式                         | 代表                                                               |
| ----------------- | ---------------------------- | ---------------------------------------------------------------- |
| Node.js 内置        | `require('fs')`              | fs、path、http/https、crypto、worker_threads、buffer、os、child_process |
| Pro 9 内置（第二代 API） | `require('accessibility')` 等 | 见 3.3                                                            |
| npm 第三方           | `require('axios')`           | axios、ws、mysql、uuid、webpack、typescript                           |

### 3.2 第一代 → 第二代 迁移对照（AI 改写代码时必查）

| 第一代（Rhino 全局）                                   | 第二代（Node.js）                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------ |
| `sleep(ms)`                                     | `const {delay} = require('lang'); await delay(ms)`                 |
| `toast()` / `toastLog()`                        | `const {showToast} = require('toast'); showToast('x', {log:true})` |
| `exit()`                                        | `process.exit()`                                                   |
| `log()` / `print()`                             | `console.log()`（不可简写）                                              |
| `random(a,b)`                                   | `Math.random` 自行封装                                                 |
| `files.*`                                       | Node `fs` + `path`                                                 |
| `floaty.*`                                      | `require('floating_window')`                                       |
| `colors.*`                                      | `require('color')`                                                 |
| base64 / crypto                                 | `Buffer.from(x,'utf8').toString('base64')` / Node `crypto`         |
| `events` 的 `exit` 事件                            | `process.on('exit', () => {})`                                     |
| `device.width`                                  | `device` 模块对应属性（以 device 模块文档为准，注意属性名差异）                           |
| `requestScreenCapture()`（同步）                    | `await requestScreenCapture()`（Promise）                            |
| `setClip()/getClip()`                           | `require('clip_manager')` 解构导入                                     |
| WebSocket（内置）                                   | npm `ws` 模块                                                        |
| `app`/`images`/`dialogs`/`engines`/`device` 等全局 | 同名 `require()` 导入，但 API 设计有差异（多为异步）                                |

### 3.3 Pro 9 内置模块清单（按能力域）

| 能力域    | 模块                                                                                                 |
| ------ | -------------------------------------------------------------------------------------------------- |
| 无障碍自动化 | `accessibility`、`ui_object`、`ui_selector`、`root_automator`                                         |
| 界面     | `ui`（activity/layout/view/web/res/util）、`floating_window`、`dialogs`、`toast`                        |
| 截图与图色  | `media_projection`、`image`、`color`、`@autojs/opencv`                                                |
| 文字识别   | `ocr`、`plugins`（MLKitOCR / PaddleOCR 插件）                                                           |
| 设备     | `device`、`sensors`、`power_manager`、`settings`                                                      |
| 应用与系统  | `app`、`shell`、`zip`、`notification`、`clip_manager`                                                  |
| 多媒体    | `media`（MediaPlayer）                                                                               |
| 运行时    | `engines`、`work_manager`（定时/意图任务）、`lang`（delay/Deferred）、`plugins`、`rhino`、`$autojs`（全局）、`console` |
| 存储     | `datastore` + Node `fs`/`path`                                                                     |


## 5. 实战代码模式（官方示例提炼）

### 5.1 截图找图循环（轮询直到出现目标）

```javascript
"nodejs";
const { requestScreenCapture } = require('media_projection');
const { readImage, findImage } = require('image');
const { showToast } = require('toast');

async function main() {
    const template = await readImage('./template.png');
    const capturer = await requestScreenCapture();
    while (true) {
        const capture = await capturer.nextImage();
        const result = await findImage(capture, template);
        if (result) { console.log('found:', result); break; }
    }
    showToast('找图结束');
    capturer.stop();
}
main();
```

### 5.2 OCR 周期检测

```javascript
"nodejs";
const plugins = require('plugins');
const { requestScreenCapture } = require('media_projection');
const { delay } = require('lang');

async function main() {
    const MLKitOCR = await plugins.load('org.autojs.autojspro.plugin.mlkit.ocr');
    const ocr = new MLKitOCR();
    const capturer = await requestScreenCapture();
    for (let i = 0; i < 5; i++) {
        const capture = await capturer.nextImage();
        const result = await ocr.detect(capture);   // 首次较慢
        console.log(result);
        await delay(3000);
    }
}
main().catch(console.error);
```

### 5.3 Root/ADB Shell 编排

```javascript
"nodejs";
const { createShell, checkAccess, isRootAvailable, exec, Tap } = require('shell');
async function main() {
    console.log(await isRootAvailable(), await checkAccess('adb'));
    const sh = createShell({ root: true });
    sh.on('line', l => console.log('line:', l));
    console.log(await sh.exec('ls /'));
    await sh.exit();
    await Tap(100, 100);      // adb 权限点击（需 shizuku 授权）
}
main();
```

### 5.4 悬浮日志窗（ui-thread + binding）

```javascript
"ui-thread";
const { createWindow, canDrawOverlays, manageDrawOverlays } = require('floating_window');
const { showToast } = require('toast');
async function main() {
    if (!canDrawOverlays()) { showToast('没有悬浮窗权限'); manageDrawOverlays(); return; }
    const win = createWindow({ initialPosition: { x: 0, y: 200 } });
    win.setViewFromXml(`<column bg="#ffffff">
        <globalconsole id="console" h="400"/>
        <row><input id="input" layout_weight="1"/><button id="ok" text="确定"/></row>
    </column>`);
    const { input, ok, close } = win.view.binding;
    ok.on('click', () => console.log('输入:', input.getText()));
}
main();
```

### 5.5 定时任务管理

见 4.7；注意任务脚本路径必须真实存在，意图任务的 action 与系统广播一致。

### 5.6 npm 模块（axios / http-server）

```javascript
"nodejs";
const axios = require('axios');               // 项目内 npm i --no-bin-links axios
const res = await axios.get('https://httpbin.org/get');
console.log(res.status, res.data);
```

Node 内置 http 亦可直接起服务（参考官方示例 `http-server-helloworld.js`）。

### 5.7 Java 互操作（SQLite / 自定义 View）

继承 Java 类用 `rhino` 模块 + `$java`（参考官方示例 `继承Java类-SQLite数据库.js`、`继承Java类-自定义View.js`）；注意不支持 JavaAdapter。

---

## 6. 典型项目骨架（Agent 生成工程时使用）

```text
TaskApp/
├── project.json           {"name":"TaskApp","packageName":"com.example.taskapp","main":"main.node.js"}
├── main.node.js           入口：解析参数 → 状态机 → 调 lib/
├── package.json
├── lib/
│   ├── ui.js              界面（floating_window 或 ui 模块）
│   ├── vision.js          截图/找图/OCR 封装（capturer 单例）
│   └── actions.js         业务动作（accessibility 封装）
└── .autojs.build.ignore   node_modules 等打包忽略
```

---

## 7. 常见低级错误清单（AI 必查）

1. **忘写 `"nodejs";` 头** → 代码落入 Rhino 引擎，`require('accessibility')` 等全部报错。
2. **用 Rhino 全局函数**：`sleep()` / `toast()` / `log()` / `click()` 直接调用 → ReferenceError。一律改 `delay` / `showToast` / `console.log` / `accessibility.click`。
3. **顶层 await** → 语法错误。必须包在 `async function main()` 里并 `main().catch(console.error)`。
4. **Promise 未 await / 未 catch** → 逻辑静默失败或 unhandled rejection。
5. **npm 装包没带 `--no-bin-links`** → sdcard 不支持符号链接，安装失败。
6. **升级内置 npm** → 可能破坏内置工具链。
7. **全局模块用了 --no-bin-links** → 找不到 tsc/webpack 等命令（应执行具体 js 路径或改私有目录）。
8. **截图权限未请求就取帧**、`capturer` 用完未 `stop()`、`Image` 未 `recycle()` → 权限挂起/资源泄漏。
9. **悬浮窗没写 `"ui-thread";` 头**，或多个引擎同时占 UI 线程 → 悬浮窗创建失败。
10. **OCR 插件未在插件商店下载** 就 `plugins.load()` → 加载失败。
11. **work_manager 任务路径不存在** / 广播 action 拼写错误 → 任务静默不执行。
12. **打包白屏/资源找不到** → 检查 `project.json` 的 `androidResources` 配置与 res 目录结构（资源包名由打包管线统一管理，脚本侧不要硬编码）。
13. **混淆控件（如新版微信）**：UI 树拿不到属性时，改用截图 + 找图/OCR 路线，不要死磕选择器。
14. **`device` 属性名凭记忆乱写** → 以 device 模块文档为准（如宽高、屏幕参数属性名与第一代不同）。
15. **`process.exit()` 前未清理**：capturer/shell/wakelock 尽量在 `process.on('exit')` 或 finally 中收尾。
16. **信了 checkSyntax 的 async 误报**：checkSyntax 内部是 Rhino 解析器，不支持 Node.js 语法，`async function`/`await` 必报 SYNTAX_ERROR（"失踪 ;声明前"），但同一代码 runScript 在 Node.js 引擎（V8）真机执行完全正常。**Node.js 代码一律跳过 checkSyntax 直接 runScript**，以 getMcpLog 为验证依据；不要因误报反复"修复"正常代码。checkSyntax 仅用于 Rhino（第一代）代码。
17. **HTTP 请求体里放了中文**：MCP 请求 JSON 含中文（多字节 UTF-8）时，服务器 HTTP 响应会挂起直到调用方超时（脚本可能仍被执行但收不到结果）。**所有请求 payload 必须纯 ASCII**：中文一律转 `\uXXXX`（如 `json.dumps(ensure_ascii=True)`），脚本执行时中文正常还原。

---

## 8. 编写规范与代码风格

- 统一入口：`async function main() { ... } main().catch(console.error);`
- 统一 `require` 置顶，解构只导入用到的函数；
- 所有等待类调用必须 `await`，等待控件用选择器 `timeout`/`maxretries` 而非裸循环 sleep；
- 图片资源（readImage 产物）用完 `recycle()`，capturer/shell/wakelock 生命周期与脚本一致；
- 日志用 `console.log/error`，关键节点必打日志（配合 `getMcpLog` 排障）；
- 业务状态机显式化（状态常量 + 转移函数），禁止面条式嵌套 while。


---

## 附录 A：高频 API 一句话速查

| 需求      | 代码                                                                                             |
| ------- | ---------------------------------------------------------------------------------------------- |
| 等 1 秒   | `await delay(1000)`                                                                            |
| 点坐标     | `await accessibility.click(x, y)`                                                              |
| 点文本     | `await accessibility.clickText('登录')`                                                          |
| 输入文本    | `await accessibility.inputText('追加')` / `setText('替换', index?)`                                |
| 找控件     | `await new UiSelector().where(cond).firstOrNull()`                                             |
| 回主页     | `await accessibility.home()`                                                                   |
| 截屏      | `const c = await requestScreenCapture(); const f = await c.nextImage();`                       |
| 找图      | `await findImage(frame, template)`                                                             |
| OCR     | `(await plugins.load('org.autojs.autojspro.plugin.mlkit.ocr'))` → `new MLKitOCR().detect(img)` |
| Root 执行 | `await createShell({root:true}).exec('cmd')`                                                   |
| adb 点击  | `await Tap(x, y)`                                                                              |
| 剪贴板     | `await setClip('x')` / `await getClip()`                                                       |
| Toast   | `showToast('x', {log:true})`                                                                   |
| 悬浮窗     | `"ui-thread";` + `createWindow()` + `setViewFromXml()`                                         |
| 定时任务    | `workManager.addDailyTask({path, time})`                                                       |
| 子引擎     | `engines.execScript({source})`                                                                 |
| 调 Java  | `$autojs.java.findClass(...)` 或 `require('rhino').install()`                                   |
| Context | `$autojs.androidContext`                                                                       |

## 附录 B：官方文档索引

- 文档总览（llms-v9.txt）：<https://www.wuyunai.com/docs/llms-v9.txt>
- 完整文档（llms-full-v9.txt，含全部类/接口/模块参考）：<https://www.wuyunai.com/docs/llms-full-v9.txt>
- 重点章节：第二代API和第一代API的区别 / 快速开始（node.js 引擎、npm、Java 调用）/ 线程与线程模型 / 模块参考（accessibility、media_projection、image、ocr、ui_selector、floating_window、shell、work_manager）/ 如何从第一代API迁移过来
- 官方 Node.js 示例：安装包内 `assets/sample/中文/Node.js/`（任务、shell、剪贴板、对话框、传感器、OCR、找图找色、悬浮窗、原生界面、Java 交互、npm 模块等 18 组）
