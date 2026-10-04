---
title: Auto.js / Auto.js Pro (Rhino 引擎) AI Agent 技能说明
description: 面向 AI Agent 的 Auto.js Rhino 自动化开发技能，包含 Auto.js 语法、控件定位、截图图色、线程权限、MCP 工具接口，以及观察界面、生成脚本、语法检查、执行验证和失败修复的完整编排流程
tags: autojs, rhino, javascript, android, 自动化, 无障碍, mcp, agent
version: 2.0
engines: Rhino (默认), Node.js (Pro 可选)
sources:
  - Auto.js Pro 官方文档
  - Auto.js 开源实战项目源码库
---

# Auto.js Rhino 脚本编写 · AI Agent 技能说明

> **给 AI 的阅读须知**：本文档是你在编写 Auto.js 脚本时的"大脑速查卡"。Auto.js 脚本运行在**安卓手机**上，不是浏览器、不是 Node.js 服务器。最大的错误来源是**把浏览器/Node 的思维带入 Auto.js**。请先完整读完「第 3 节 引擎心智模型」和「第 8 节 常见低级错误」，再动手写代码。

---

## 0. MCP 驱动的自动化开发协议

本节是 AI Agent 使用本技能时的执行协议。编写 Auto.js 脚本时，必须优先遵守本节，再参考后面的 API 和代码模式。

### 0.1 工作目标

将自动化任务拆成“观察、规划、生成、检查、执行、验证、修复”七个阶段。不要只生成一段看似合理的代码，也不要在没有真实界面证据时声称任务完成。

默认工作环境：

- 目标设备：运行 Auto.js / Auto.js Pro 的 Android 设备。
- 默认引擎：Rhino；除非用户明确指定 Node.js，否则不要使用 Node.js 专属语法。
- MCP 地址：用户提供的本地或局域网地址。
- MCP 端点：`POST /mcp`，使用 JSON-RPC 2.0 的 `tools/list` 和 `tools/call`。
- 当前 MCP 服务不要求 Token；如服务端未来增加认证，必须使用用户提供的凭据，不要猜测。

### 0.2 当前 MCP 工具接口

| 工具 | 参数 | 用途 | 使用时机 |
|---|---|---|---|
| `getUiLayout` | `{}` | 获取当前界面的 UI 布局 XML、节点数量和控件结构 | 每个新任务开始时优先调用 |
| `runScript` | `source` 或 `file`，可选 `config` | 执行 Auto.js 源码或脚本文件 | `checkSyntax` 通过后调用 |
| `stopAllScripts` | `{}` | 停止所有正在运行的 Auto.js 脚本 | 只有用户明确允许，或确认目标脚本占用资源时调用 |
| `checkSyntax` | `{code}` | 检查 Auto.js JavaScript 语法并返回错误 | 每次新生成或修改脚本后调用 |
| `captureScreen` | `format?`、`quality?`、`dataUri?` | 截图并返回图像信息 | UI 树不完整、Flutter/自绘控件、图色场景或执行后验证时调用 |
| `getMcpLog` | `maxChars?` | 读取脚本执行日志 | 执行失败、无响应或需要确认后台状态时调用 |

工具参数规则：

- `runScript` 必须提供 `source` 或 `file` 其中一个。
- `runScript.config` 当前接口虽然声明支持 `path`、`delay`、`interval`、`loopTimes`、`arguments`，但执行端默认使用固定执行配置；不要依赖这些字段实现关键控制逻辑，循环、延迟和参数处理应写在脚本内部。
- `checkSyntax` 只负责 JavaScript/Auto.js 语法检查，不代表控件一定存在、权限一定开启或业务一定成功。
- `captureScreen` 默认使用 `png` 和质量 `100`；需要传输较小图像时再使用 `jpg` 或降低质量。
- `getMcpLog.maxChars` 默认 `20000`，最大不超过 `200000`；读取日志时优先从较小值开始。
- `stopAllScripts` 影响整个 Auto.js 运行环境，不得把它当作普通重试按钮。

### 0.3 自动化任务编排

按以下顺序执行，每个阶段都必须有明确产物：

**阶段 1：明确任务**

- 提取目标 App、目标页面、输入数据、成功标准和禁止操作。
- 如果目标页面、包名、登录状态或成功条件不明确，先提出最少量澄清问题。
- 将模糊目标改写成可验证结果，例如“出现订单详情页”和“点击按钮”不是同一个成功标准。

**阶段 2：观察环境**

1. 调用 `getUiLayout` 获取当前 UI 树。
2. 找出目标控件的 `text`、`desc`、`id`、`className`、可点击父节点和可见状态。
3. 如果 UI 树为空、节点明显不完整、界面属于 Flutter 自绘、Canvas、图片按钮或 WebView，再调用 `captureScreen`。
4. 不要凭空编造控件文本、资源 id 或页面状态。

**阶段 3：建立状态机**

先写出页面状态和转移条件，再写代码：

```text
当前状态
  -> 等待权限/弹窗处理
  -> 定位目标控件
  -> 执行动作
  -> 等待页面变化
  -> 验证成功条件
  -> 成功结束 / 进入恢复分支
```

每个动作至少定义：

- 前置条件：执行前必须看到什么；
- 动作：点击、输入、滑动、返回、截图或等待；
- 成功信号：出现什么文本、节点、图像或日志；
- 失败处理：重试、返回、重新读取 UI，还是停止。

**阶段 4：生成最小脚本**

- 默认生成单文件、可直接通过 `runScript.source` 执行的最小版本。
- 第一行按运行模式写 `"auto";` 或 `"ui";`。
- 普通自动化脚本先检查无障碍服务，再执行控件操作。
- 先实现主路径，再补权限、弹窗、超时和恢复分支。
- 关键步骤使用 `toastLog()` 或 `log()` 输出短日志。
- 不要把账号、密码、验证码和 API Key 硬编码到脚本。

**阶段 5：语法门禁**

生成或修改脚本后，先调用：

```text
checkSyntax({ code: "完整脚本源码" })
```

只有返回无错误，才允许调用 `runScript`。语法失败时只修复语法和兼容性问题，不要跳过检查直接执行。

**阶段 6：实际执行**

调用 `runScript` 执行脚本源码或用户指定的文件。执行后记录返回的 `success`、`scriptId`、`result` 或 `error`，不要把“已提交执行”误认为“业务成功”。

**阶段 7：结果验证与修复**

执行后按优先级验证：

1. 用 `getUiLayout` 检查目标文本、控件状态和页面转移；
2. UI 树不足时用 `captureScreen` 检查真实画面；
3. 无响应、异常退出或状态不明时用 `getMcpLog`；
4. 根据真实结果修改脚本；
5. 重新执行 `checkSyntax -> runScript -> 验证`，最多按用户允许的次数迭代。

### 0.4 定位策略

按以下优先级定位控件：

1. `id`；
2. `text` / `textContains`；
3. `desc` / `descContains`；
4. `className` 与可见、可点击条件组合；
5. 从目标文本向上寻找可点击父节点；
6. `bounds()` 中心坐标；
7. 截图确认后的坐标、找图或找色。

每个查找都必须有超时和判空：

```javascript
var button = textContains("开始").visibleToUser(true).findOne(3000);
if (!button) {
    toastLog("未找到开始按钮");
    return false;
}
```

Flutter 页面注意事项：

- Flutter 控件只有在暴露 Semantics/无障碍信息时，`getUiLayout` 才可能看到有用文本。
- UI 树缺少目标节点时，不要反复猜测选择器；改用 `captureScreen`。
- 使用坐标前先确认当前屏幕尺寸、状态栏高度和目标区域。
- 坐标操作后必须通过页面变化、截图或日志确认动作生效。
- 在最终结果中说明哪些步骤依赖坐标、截图或图色，便于后续维护。

### 0.5 权限和安全边界

执行前检查任务所需权限：

- 无障碍：控件读取、点击、输入和滑动；
- 截图权限：`captureScreen`、找图、找色和视觉确认；
- 悬浮窗权限：`floaty` 或跨 App 悬浮控制；
- 存储权限：读写脚本、配置和截图；
- 通知使用权：读取通知或监听通知；
- 电池优化豁免：长时间后台任务。

权限不足时：

1. 说明缺少的权限；
2. 只执行用户明确允许的授权引导；
3. 不要绕过 Android 权限；
4. 不要因为权限失败就调用 `stopAllScripts`；
5. 继续执行前重新检查权限状态。

### 0.6 交付格式

任务完成时输出：

```text
【任务目标】完成了什么
【脚本】完整 Auto.js 源码或文件路径
【检查】checkSyntax 结果
【执行】runScript 结果和 scriptId
【验证】UI、截图或日志中确认到的成功信号
【限制】坐标依赖、权限要求、Flutter 节点缺失或其他已知风险
```

如果任务未完成，必须明确说明失败步骤、真实错误和下一步建议，不要输出“已完成”。

### 0.7 Agent 快速指令模板

接收具体任务时，按下面的最短模板工作：

```text
请读取并遵守本技能说明，使用 MCP 完成 Auto.js 自动化任务：
先 getUiLayout，必要时 captureScreen；根据真实界面规划状态机并生成 Rhino 脚本；
先 checkSyntax，只有通过后才能 runScript；执行后用 getUiLayout、captureScreen 或 getMcpLog 验证，
失败就根据真实结果修复并重新检查。不要凭空猜控件，不要未经允许调用 stopAllScripts。

任务：<填写目标 App、操作步骤、输入数据和成功标准>
```

---

## 目录

0. [MCP 驱动的自动化开发协议](#0-mcp-驱动的自动化开发协议)
1. [什么是 Auto.js](#1-什么是-autojs)
2. [引擎与运行时](#2-引擎与运行时)
3. [引擎心智模型（必读）](#3-引擎心智模型必读)
4. [脚本结构与工程规范](#4-脚本结构与工程规范)
5. [核心 API 速查](#5-核心-api-速查)
   - 5.1 无障碍与自动操作
   - 5.2 控件选择器
   - 5.3 触摸与手势
   - 5.4 界面 (UI)
   - 5.5 悬浮窗 (floaty)
   - 5.6 图片与图色
   - 5.7 HTTP 与 WebSocket
   - 5.8 文件与存储
   - 5.9 线程、协程与定时
   - 5.10 对话框与通知
   - 5.11 设备与系统
   - 5.12 调用 Java API
   - 5.13 事件与监听
   - 5.14 Pro 版专属 ($ 前缀)
6. [实战代码模式（从真实项目提炼）](#6-实战代码模式从真实项目提炼)
7. [典型项目骨架示例](#7-典型项目骨架示例)
8. [常见低级错误清单（AI 必查）](#8-常见低级错误清单ai-必查)
9. [编写规范与代码风格](#9-编写规范与代码风格)

---

## 1. 什么是 Auto.js

Auto.js 是一个 Android 平台上的 JavaScript 自动化框架。它通过**无障碍服务 (AccessibilityService)** 读取屏幕上的控件树，从而定位、点击、输入、滑动任意 App 的界面元素；同时提供截图、找图找色、悬浮窗、HTTP、文件、数据库等能力，用来编写"手机上的自动化脚本"。

**关键事实**：

| 事实 | 说明 |
|---|---|
| 运行环境 | Android 手机（非浏览器、非 Node 服务器） |
| 语言 | JavaScript，默认 **Rhino 引擎**（ES5.1+ 部分 ES6）；Auto.js Pro 可选 Node.js 引擎 |
| 界面读取 | 依赖无障碍服务，须先开启 |
| 截图 | 依赖 MediaProjection 权限（`requestScreenCapture`） |
| 悬浮窗 | 依赖"显示在其他应用上层"权限 |
| 定时/后台 | 依赖电池优化白名单、前台服务等 |
| 分发形态 | .js 脚本文件、打包为 .apk、或配合 VSCode 远程调试 |

> **免责声明**：Auto.js 的使用需遵守目标 App 的条款与当地法律法规。本说明仅面向技术学习与合法自动化场景。

---

## 2. 引擎与运行时

### 2.1 Rhino 引擎（默认，重点）

Auto.js 默认使用 Mozilla Rhino 引擎执行脚本。编写时必须注意：

- **不支持浏览器 API**：`document`、`window`、`alert`（全局弹窗是 `alert()` 但来自 Auto.js 自身，非浏览器）、`fetch`、`XMLHttpRequest` 等**全部不存在**。
- **模块系统**：Rhino 模式下用 `require('./xxx.js')` 引入本地文件模块，`module.exports` 导出；不支持 `import/export` 语法。
- **ES 语法支持**：Rhino 引擎支持 `var/let/const`、箭头函数、模板字符串、`class`（部分版本）、`Promise`（Pro 8.0+ 支持）、`async/await`（Pro 8.0+ 支持协程 + 异步）。**但不要依赖最新 ES 特性**（如可选链 `?.`、空值合并 `??`、`Array.prototype.flat` 等在部分版本缺失），最稳妥写法是 ES5 风格函数 + `var`/`let`。
- **XML 字面量 (E4X)**：Rhino 支持 `new XML("<...>")` 和 `<card>...</card>` 字面量，常用于悬浮窗与 UI 布局。实战中常见 `floaty.window(new XML("<card ...>..."))`。
- **Java 互操作**：可无缝调用 Java 类，见 [5.12](#512-调用-java-api)。

### 2.2 Node.js 引擎（Auto.js Pro 可选）

文件头加 `"nodejs";` 启用。此时：

- 支持完整 ES6+、CommonJS、Node 风格 API；
- **没有** Rhino 的 `importClass`/`Packages`/`new XML` 等语法；
- 自动操作等 Auto.js 核心 API 仍可用，但部分 UI/悬浮窗写法有差异。

**编写建议**：除非明确要求 Node.js 引擎，一律按 **Rhino** 语法编写，兼容性最好。

### 2.3 版本差异注意

- 官方文档对应 **Auto.js Pro 8/9.x**，大量示例使用 `$` 前缀 API（`$ui`、`$events`、`$settings`、`$plugins`）与 `"rhino";` 头部声明。
- 开源版 Auto.js 4.x 无 `$` 前缀 API，用 `ui`、`events`、`settings`、`plugins`（部分无）。
- 写脚本前先确认目标环境是 Pro 还是开源版。

---

## 3. 引擎心智模型（必读）

写 Auto.js 之前，先把下面这张图刻进脑子：

```
┌─────────────────────────────────────────────────────┐
│                 你的 .js 脚本 (Rhino)                 │
│   auto.waitFor() → 无障碍服务就绪                     │
│   requestScreenCapture() → 截图权限                  │
│                                                      │
│   ┌────────────┐    ┌────────────┐                   │
│   │ 主线程      │    │ threads.start│  ← 工作线程     │
│   │ 自动操作    │    │ (每脚本一个)  │                  │
│   └─────┬──────┘    └─────┬──────┘                   │
│         │                 │                          │
│         ▼                 ▼                          │
│   无障碍服务(控件树)    截图/找色/找图/OCR/HTTP        │
└─────────────────────────────────────────────────────┘
        │
        ▼
   安卓系统（目标 App、系统 UI、网络、文件系统）
```

### 三条铁律

1. **先等无障碍，再谈自动化**：脚本第一行（非 UI 模式）应 `auto.waitFor();`，否则 `text()`/`click()` 全部无效。
2. **控件优先于坐标**：优先用控件选择器（text/id/desc）定位，坐标（x,y）作为兜底。坐标依赖屏幕分辨率，换机即失效。
3. **UI 线程不可阻塞**：`"ui";` 模式下主线程就是 UI 线程，直接 `sleep()` 会让界面卡死；长任务放 `threads.start()` 或 `setTimeout` 里。

---

## 4. 脚本结构与工程规范

### 4.1 文件头声明（引擎/模式开关）

脚本**第一行**必须是以下之一（字符串 + 分号）：

```javascript
"auto";      // 自动模式：运行前自动申请无障碍服务，等待用户确认
"ui";        // UI 模式：脚本构建一个 Android 界面，主线程为 UI 线程
"rhino";     // 强制 Rhino 引擎（Pro）
"nodejs";    // 强制 Node.js 引擎（Pro）
```

常用组合：

```javascript
"auto";                    // 纯自动化，无界面
"ui";                      // 有界面，运行时自动申请权限需另行处理
"ui"; "rhino";             // Pro 强制 Rhino + UI
```

> **实战注意**：`"auto";` 会弹出无障碍授权引导。更稳妥的写法是在代码里 `auto.waitFor();` 并配合权限检查 UI（见第 6 节权限引导模式）。

### 4.2 project.json（工程清单）

Auto.js Pro 工程目录下必有 `project.json`，打包成 apk 的依据：

```json
{
  "assets": [],
  "build": { "build_id": "A9F1BB41-1", "build_number": 1, "build_time": 1738337378224 },
  "useFeatures": [],
  "icon": "res/icon.png",
  "launchConfig": {
    "displaySplash": true,
    "hideLogs": false,
    "splashIcon": "res/splashIcon.png",
    "splashText": "Powered by xxx",
    "stableMode": true
  },
  "main": "main.js",
  "name": "脚本名",
  "optimization": { "removeOpenCv": false, "unusedResources": false },
  "packageName": "com.script.xxx",
  "scripts": {},
  "versionCode": 1,
  "versionName": "1.0.0"
}
```

要点：
- `main` 指定入口文件；`name`/`packageName` 决定 apk 名称与包名；`icon` 指向 res 下图标。
- `launchConfig.stableMode: true` 表示启用无障碍稳定模式（后台更稳）。
- `useFeatures` 用于声明插件特性；`encryptLevel` 控制加密级别（0 不加密）。

### 4.3 模块化

Rhino 用 CommonJS 风格模块化：

```javascript
// config.js —— 配置文件
var config = {
    commonScriptKey: "my_app",
    SDK_API_VERSION: 29
};
module.exports = config;

// utils.js —— 工具模块
var utils = {};
utils.toast = function (msg) { toast(msg); };
utils.getDeviceUUID = function () {
    var storage = storages.create("my_app");
    var uuid = storage.get("deviceUUID");
    if (!uuid) {
        uuid = (device.sdkInt > 28 ? device.getAndroidId() : device.getIMEI());
        storage.put("deviceUUID", uuid);
    }
    return uuid;
};
module.exports = utils;

// main.js —— 入口
"ui";
var config = require("./config.js");
var utils = require("./utils.js");
```

> **陷阱**：`require` 的路径是相对当前脚本文件的，用 `./` 开头；不要用浏览器 `import` 语法。

---

## 5. 核心 API 速查

> 本节按"最常用 → 次常用"排列。所有 API 均为全局函数/对象，除非注明 `$` 前缀（Pro）。

### 5.1 无障碍与自动操作

| API | 说明 |
|---|---|
| `auto.waitFor()` | 等待无障碍服务开启（阻塞）。**自动化脚本几乎必调** |
| `auto.service` | 无障碍服务对象，`null` 表示未开启 |
| `auto.rootInActiveWindow()` | 获取当前窗口根控件（需要无障碍已开启） |
| `auto.setWindowFilter()` | 设置窗口过滤（Pro） |
| `auto.click(x, y)` / `click(x, y)` | 坐标点击 |
| `auto.longClick(x, y)` / `longClick(x, y)` | 长按 |
| `setText(控件, 文本)` / `setText(x, y, 文本)` | 输入文本到输入框 |
| `paste(x, y)` / `paste(控件)` | 粘贴 |
| `input(text)` | 模拟输入（需焦点在输入框） |
| `back()` / `home()` / `recents()` | 返回 / 主页 / 最近任务 |
| `notification()` | 下拉通知栏 |
| `KeyCode` | 按键常量（`KeyCode.BACK`、`KeyCode.HOME`、`KeyCode.ENTER` 等） |
| `press(KeyCode.BACK, 200)` | 按下物理键 |
| `swipe(x1, y1, x2, y2, duration)` | 滑动 |
| `gesture(duration, [x1,y1], [x2,y2], ...)` | 手势（多指） |
| `pinch(x1,y1,x2,y2,duration,dir)` | 捏合 |

**实战标准开头**：

```javascript
"auto";
auto.waitFor();
// 设备分辨率声明（用于坐标换算，非强制）
setScreenMetrics(device.width, device.height);
```

### 5.2 控件选择器（UiSelector）

控件查找是 Auto.js 的核心。选择器可以链式组合：

```javascript
text("登录").findOnce();
textContains("登录").findOne(3000);       // 包含匹配
textStartsWith("登").findOnce();
textEndsWith("录").findOnce();
textMatches(/登录.*/).findOnce();          // 正则匹配（注意是完整正则，不是子串！）
desc("描述内容").findOnce();               // content-desc 属性
descContains("描述").findOnce();
id("btn_login").findOnce();                // 资源 id（不含包名前缀）
idContains("btn").findOnce();
idMatches(/btn.*/).findOnce();
className("android.widget.Button").findOnce();
classNameContains("Button").findOnce();
packageName("com.xxx").findOnce();
bounds(10, 10, 200, 100).findOnce();       // 精确 bounds
```

**链式与查找方法**：

```javascript
// 链式条件
text("确定").clickable(true).visibleToUser(true).findOne(3000);

// 查找方法
selector.findOne(timeout)  // 查找第一个，可等待最多 timeout ms，超时返回 null
selector.findOnce()        // 立即查找，找不到返回 null（不等待）
selector.find()             // 返回所有匹配集合
selector.untilFind()        // 阻塞直到找到
selector.waitFor()          // 等待出现
selector.exists()           // 是否存在
```

**控件对象常用方法/属性**：

```javascript
var btn = text("确定").findOnce();
btn.click();                 // 控件点击（等价于自动点击该控件）
btn.clickable();             // 是否可点击
btn.text();                  // 获取文本
btn.desc();                  // 获取描述
btn.id();                    // 获取 id
btn.className();             // 获取类名
btn.bounds();                // 获取边界 Rect：left/top/right/bottom/width()/height()
btn.bounds().centerX();      // 中心 x
btn.bounds().centerY();      // 中心 y
btn.parent();                // 父控件
btn.child(i);                // 第 i 个子控件
btn.children();              // 所有子控件
btn.visibleToUser();         // 是否对用户可见
btn.enabled();               // 是否可用
btn.depth();                 // 深度
btn.packageName();           // 所在包名
```

> **重要**：`findOne(ms)` 超时返回 `null`，**必须先判空再使用**，否则 `null.click()` 直接报错。

### 5.3 触摸与手势

```javascript
// 坐标点击
click(540, 1200);
// 滑动
swipe(500, 1800, 500, 300, 800);   // 800ms 从下往上滑
// 长按
longClick(540, 1200, 1000);        // 长按 1 秒
// 手势（画心形、多点）
gesture(1000, [100, 100], [200, 200], [300, 100]);
// 捏合
pinch(540, 1200, 200, 200, 500, 1);  // 参数：中心点、半径、时长、方向(1内收 -1外扩)
```

**坐标分辨率问题**：手机分辨率不同，坐标会偏移。可用 `setScreenMetrics(1080, 1920)` 声明设计分辨率，但**最佳实践是运行时用 `device.width/device.height` 动态计算**，或用控件 bounds 取中心：

```javascript
var obj = text("立即领取").findOnce();
if (obj) click(obj.bounds().centerX(), obj.bounds().centerY());
```

### 5.4 界面 (UI)

`"ui";` 模式下用 XML 布局构建界面：

```javascript
"ui";

ui.layout(
    <vertical padding="16">
        <text text="欢迎使用" textSize="18sp" textColor="#000000" />
        <input id="username" hint="请输入账号" />
        <input id="password" hint="请输入密码" inputType="textPassword" />
        <button id="btn_start" text="开始运行" w="*" />
        <switch id="sw_auto" text="自动模式" checked="true" />
        <checkbox id="cb_agree" text="同意协议" />
    </vertical>
);

// 控件访问：ui.控件id
ui.btn_start.on("click", function () {
    var name = ui.username.text();
    var pwd = ui.password.text();
    toast("账号:" + name + " 密码:" + pwd);
    threads.start(function () {
        // 长时间任务放线程，避免卡 UI
    });
});

// UI 线程操作（子线程更新界面必须 ui.run）
threads.start(function () {
    ui.run(function () {
        ui.btn_start.setText("运行中...");
    });
});
```

**UI 属性速查**：`w/h`（宽高，`"*"` 填充、`"auto"` 自适应、数字 dp）、`layout_weight`（权重）、`gravity`（对齐）、`bg`（背景色）、`textColor`、`textSize`、`padding`、`margin`、`id`、`visibility`（visible/invisible/gone）、`alpha`。

**控件类型**：`text`、`input`、`button`、`switch`、`checkbox`、`radio`、`seekbar`、`progress`、`card`、`frame`、`linear`、`vertical`、`horizontal`、`scroll`、`tabs`、`viewpager`、`webview`、`img`、`spinner`、`list` 等，还可直接写完整 Java 类名（如 `<com.google.android.material.floatingactionbutton.FloatingActionButton .../>`）。

**动态设置属性**：

```javascript
ui.btn.attr("text", "新文本");
ui.btn.attr("bg", "#FF0000");
```

**布局动态创建与正则/JS 表达式绑定**：

```javascript
var color = "#ff0000";
ui.layout(<text text="{{color}}" />);   // {{}} 绑定变量
```

> **陷阱**：`ui.layout()` 一个脚本只能调用一次；UI 模式下不能直接 `sleep()` 主线程。

### 5.5 悬浮窗 (floaty)

悬浮窗用于在任意界面显示悬浮信息、控制按钮、日志面板：

```javascript
// 简单悬浮文本
var w = floaty.window(
    <frame>
        <text id="txt" text="悬浮文本" textColor="#ffffff" bg="#80000000" padding="8" />
    </frame>
);
w.setPosition(100, 200);   // 位置
w.setSize(200, 60);        // 尺寸
w.txt.setText("更新内容");
// 点击
w.txt.on("click", function () { toast("点了"); });
// 关闭
w.close();

// rawWindow 悬浮窗（不可触摸或自定义触摸）
var rw = floaty.rawWindow(<frame><text id="t2" text="raw"/></frame>);
rw.setTouchable(false);  // 不拦截触摸
```

**实战：可拖动悬浮窗 + 控制台（来自真实项目 `日志悬浮窗带按钮控制.js` / `duola/mode/悬浮窗.js`）**：

```javascript
"ui";
var window = floaty.rawWindow(
    <card id="main" h="150" cardBackgroundColor="#b3000000" cardCornerRadius="8dp">
        <vertical paddingLeft="5" paddingRight="5" w="*">
            <horizontal>
                <text id="title" text="" textSize="13dp" textColor="#FFD700" textStyle="bold" />
                <Chronometer id="runTime" textSize="13dp" textColor="#03D96A" w="auto" />
            </horizontal>
            <com.stardust.autojs.core.console.ConsoleView id="console" w="*" h="*" />
        </vertical>
    </card>
);
window.setSize(device.width - 100, device.height / 2.5);
window.console.setConsole(runtime.console);
window.console.setInputEnabled(false);

// 拖动实现
var x = 0, y = 0, windowX = 0, windowY = 0;
window.main.setOnTouchListener(function (view, event) {
    switch (event.getAction()) {
        case event.ACTION_DOWN:
            x = event.getRawX(); y = event.getRawY();
            windowX = window.getX(); windowY = window.getY();
            return true;
        case event.ACTION_MOVE:
            var tx = windowX + (event.getRawX() - x);
            var ty = windowY + (event.getRawY() - y);
            tx = Math.max(0, Math.min(tx, device.width - window.getWidth()));
            ty = Math.max(0, Math.min(ty, device.height - window.getHeight()));
            window.setPosition(tx, ty);
            return true;
    }
    return true;
});
```

> **陷阱**：悬浮窗一般要在**子线程**中创建（`threads.start` 里），避免阻塞；`floaty.window` 与 `floaty.rawWindow` 区别在于 raw 需要自己处理触摸事件。

### 5.6 图片与图色

```javascript
// 申请截图权限（脚本中只需一次）
if (!requestScreenCapture()) {
    toast("请求截图失败");
    exit();
}
sleep(1000);  // 等 1 秒让权限生效

// 截图（截全屏）
var img = captureScreen();
// 保存
images.save(img, "/sdcard/screen.png");
// 回收
img.recycle();
```

**找色**：

```javascript
var img = captureScreen();
// 找单色：在区域(100,220,400,400)内找 #75438a
var point = findColorInRegion(img, "#75438a", 100, 220, 400, 400);
// 找色（全图，可配阈值）
var point2 = findColor(img, "#ff0000", { region: [0, 0, 1080, 2400], threshold: 4 });
// 多点找色（更精准，防误匹配）
var point3 = findMultiColors(img, "#ff0000", [[10, 0, "#00ff00"], [0, 10, "#0000ff"]], { region: [0,0,1080,2400] });
if (point) { click(point.x, point.y); }
img.recycle();
```

**找图（模板匹配）**：

```javascript
var img = captureScreen();
var template = images.read("/sdcard/template.png");
var p = images.findImage(img, template, { threshold: 0.8 });  // 阈值 0~1，越高越严格
if (p) { click(p.x + template.getWidth()/2, p.y + template.getHeight()/2); }
template.recycle(); img.recycle();
```

**图像处理**：

```javascript
var img = images.read("/sdcard/a.png");
var crop = images.clip(img, 0, 0, 100, 100);       // 裁剪
var scaled = images.scale(img, 0.5, 0.5);          // 缩放
var c = images.pixel(img, 50, 50);                 // 取像素颜色（int）
var hex = colors.toString(c);                      // "#ff0000"
images.save(scaled, "/sdcard/b.png");
```

**OCR 文字识别（Pro 插件）**：

```javascript
// MLKit OCR 插件
var MLKitOCR = $plugins.load("org.autojs.autojspro.plugin.mlkit.ocr");
var result = MLKitOCR.recognizeText(images.read("/sdcard/a.png"));
log(result.text);
// PaddleOCR 插件
var PaddleOCR = $plugins.load("org.autojs.autojspro.plugin.paddleocr");
```

> **陷阱**：截图权限申请后**必须 sleep 一下**再截图；大图用完 `recycle()` 防内存泄漏；`captureScreen()` 不可在 `requestScreenCapture()` 之前调用。

### 5.7 HTTP 与 WebSocket

```javascript
// GET
var res = http.get("https://api.example.com/data", {
    headers: { "User-Agent": "Mozilla/5.0" },
    timeout: 10000
});
if (res.statusCode == 200) {
    var body = res.body.string();   // 文本
    var json = res.body.json();     // 解析 JSON
}

// POST（表单）
var res2 = http.post("https://api.example.com/login", {
    username: "admin", password: "123456"
});
// POST（JSON）
var res3 = http.post("https://api.example.com/api", {
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ a: 1 })
});

// 文件上传
var res4 = http.postMultipart("https://api.example.com/upload", {
    file: open("/sdcard/a.png")
});

// 下载
http.get("https://example.com/a.png", { path: "/sdcard/a.png" });
```

**WebSocket（Pro）**：

```javascript
var ws = web.newWebSocket("wss://echo.websocket.events", { eventThread: 'this' });
ws.on("open", function (res, ws) { log("已连接"); })
  .on("text", function (text, ws) { log("收到: " + text); })
  .on("failure", function (err, res, ws) { log("失败: " + err); });
ws.send("hello");
```

### 5.8 文件与存储

```javascript
// 文件
files.exists(path);               // 是否存在
files.create(path);               // 创建
files.read(path);                 // 读文本
files.write(path, "内容");        // 写文本（覆盖）
files.append(path, "追加");       // 追加
files.remove(path);               // 删除
files.listDir(path);              // 列目录
files.path(relative);             // 相对当前脚本路径转绝对路径
files.cwd();                      // 脚本所在目录
files.join(dir, "sub");           // 拼接路径
files.getSdcardPath();            // /sdcard

// 本地存储（键值对，可存对象）
var storage = storages.create("app_name");   // 按名字建一个命名空间
storage.put("key", value);
var v = storage.get("key", defaultVal);
storage.remove("key");
storage.clear();

// 全局存储
var g = storages.global();
g.put("k", 1);

// 保存/读取复杂对象
storage.put("user", { name: "张三", age: 18 });
var user = storage.get("user");
```

> **陷阱**：`files.read` 的路径可用 `/sdcard/...` 或相对路径（相对当前脚本）；写到 `/sdcard` 需要存储权限（Android 11+ 用 SAF 或指定目录）。

### 5.9 线程、协程与定时

```javascript
// 线程
var thread = threads.start(function () {
    while (true) {
        // 长时间任务
        sleep(1000);
    }
});
threads.shutDownAll();   // 停止所有线程
// thread.interrupt();

// 定时器
setTimeout(function () { toast("3秒后"); }, 3000);
var id = setInterval(function () { log("每秒"); }, 1000);
clearInterval(id);

// 原子计数器（多线程安全）
var count = threads.atomic(0);
count.incrementAndGet();
```

**UI 模式下的线程注意事项**：子线程中更新控件必须 `ui.run(...)`。

### 5.10 对话框与通知

```javascript
// 简单对话框
alert("标题", "内容");
var ok = confirm("确认执行？");
var input = rawInput("请输入", "默认值");
var sel = dialogs.select("选择一个", ["A", "B", "C"]);
var pwd = dialogs.rawInput("密码", "", { inputType: "textPassword" });

// 复杂对话框
dialogs.build({
    title: "配置",
    content: "内容",
    positive: "确定",
    negative: "取消",
    neutral: "稍后"
}).on("positive", function () { toast("确定"); }).show();

// 通知（Pro）
notification.autoNotify("标题", "内容", "脚本名");
// 或
var n = notification.build({ title: "t", content: "c" });
n.show();
```

### 5.11 设备与系统

```javascript
device.width; device.height;           // 屏幕宽高
device.sdkInt;                          // Android API 级别
device.brand; device.model;             // 品牌/型号
device.getAndroidId();                  // Android ID
device.getIMEI();                       // IMEI（需权限，SDK>28 可能不可用）
device.getMacAddress();
device.vibrate(300);                    // 震动
device.setBrightness(128);              // 亮度 0-255
device.setMusicVolume(5);
device.getBattery();                    // 电量
device.isScreenOn();                    // 屏幕是否亮
device.wakeUp(); device.wakeUpIfNeeded();
// shell
var r = shell("ls /sdcard", true);      // 需要 root 或 adb shell 权限
log(r.result);
```

### 5.12 调用 Java API

Rhino 的最大优势之一：

```javascript
// 导入类
importClass(android.widget.Toast);
importClass(android.content.Intent);
importClass(java.text.SimpleDateFormat);

// 直接用完整类名（等价 importClass）
var date = new java.util.Date();
var fmt = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(date);
log(fmt);

// 包导入
importPackage(android.os);
var handler = new Handler(Looper.getMainLooper());

// 实现接口（JavaAdapter 是关键）
var receiver = new JavaAdapter(android.content.BroadcastReceiver, {
    onReceive: function (context, intent) {
        log(intent.getAction());
    }
});

// 访问静态字段
var Build = android.os.Build;
log(Build.MODEL);
```

**JavaAdapter 用法**（实现接口/继承类，第二个参数是方法映射对象）：

```javascript
new JavaAdapter(java.lang.Runnable, {
    run: function () { log("run"); }
});
```

### 5.13 事件与监听

```javascript
// 按键监听
events.observeKey();
events.onKeyDown("volume_up", function (event) {
    toast("音量+按下");
});

// 通知监听（需通知使用权）
events.observeNotification();
events.onNotification(function (n) {
    log(n.getText());
});

// Toast 监听（无障碍）
events.observeToast();
events.onToast(function (toast) {
    log("Toast: " + toast.getText());
});

// 触摸监听（需无障碍）
events.observeTouch();
events.onTouch(function (event) {
    // event.getRawX(), event.getRawY(), event.getAction()
});
```

### 5.14 Pro 版专属（$ 前缀）

| API | 说明 |
|---|---|
| `$ui.layout(xml)` / `$ui.post(fn)` / `$ui.run(fn)` | UI 操作（等价格式 `"ui"` 下的 `ui.*`） |
| `$events.on('exit', fn)` | 脚本退出时回调（清理资源） |
| `$settings.isEnabled('foreground_service')` | 查询前台服务开关 |
| `$settings.setEnabled('stable_mode', true)` | 设置无障碍稳定模式 |
| `$plugins.load("插件包名")` | 加载 Pro 插件（OCR/ffmpeg/7zip 等） |
| `$threads` | 线程模块的 `$` 形式 |
| `runtime.loadDex("loader.dex")` | 加载 dex（加密/防破解场景） |
| `runtime.loadJar("xxx.jar")` | 加载 jar |

```javascript
// 插件加载示例（真实项目）：浩然 OCR / tomato OCR
var hrOcr = $plugins.load("com.hraps.ocr");
var tomatoOcrClass = $plugins.load("com.tomato.ocr");
var ocr = new tomatoOcrClass();
ocr.start();
var result = ocr.recognizeText(images.read("/sdcard/a.png"));
```

---

## 6. 实战代码模式（从真实项目提炼）

> 以下模式均来自 Auto.js 社区的真实项目（代码集锦、巨推客、今日头条、PPMT 助手等），是经过大量实战验证的写法。

### 6.1 模式一：通用点击工具函数 UtilClick（核心中的核心）

这是社区最常用的控件点击封装，解决"控件可能不可点击""文本可能为空""需要等待出现"三大问题：

```javascript
/**
 * 通用控件查找+点击
 * @param {string|RegExp} textContent 查找内容（正则用于 textMatches）
 * @param {number} times 等待毫秒（findOne 超时）
 * @param {string} type 查找类型：
 *   "text"(精确) | "textC"(包含) | "textM"(正则) |
 *   "id"(精确) | "idC"(包含) | "idM"(正则) |
 *   "className" | "desc"(精确) | "descC"(包含) | "descM"(正则)
 * @param {number} ifclick 0=找到即坐标点击并返回true；1=控件点击(向上找可点击父节点)；2=只返回控件不点击
 * @param {number} hold_time 点击前停顿（随机化）
 * @param {number} before_time 点击后停顿
 * @returns {boolean|UiObject|null}
 */
function UtilClick(textContent, times, type, ifclick, hold_time, before_time) {
    if (typeof times != 'number') times = 1000;
    if (typeof hold_time != 'number') hold_time = 0;
    if (typeof ifclick != 'number') ifclick = 0;
    if (typeof before_time != 'number') before_time = 0;

    var thisEle;
    switch (type) {
        case "text":
            thisEle = text(textContent).visibleToUser(true).findOne(times) || desc(textContent).visibleToUser(true).findOnce();
            break;
        case "textC":
            thisEle = textContains(textContent).visibleToUser(true).findOne(times) || descContains(textContent).visibleToUser(true).findOnce();
            break;
        case "textM":
            thisEle = textMatches(textContent).visibleToUser(true).findOne(times) || descMatches(textContent).visibleToUser(true).findOnce();
            break;
        case "id":
            thisEle = id(textContent).visibleToUser(true).findOne(times);
            break;
        case "idC":
            thisEle = idContains(textContent).visibleToUser(true).findOne(times);
            break;
        case "idM":
            thisEle = idMatches(textContent).visibleToUser(true).findOne(times);
            break;
        case "className":
            thisEle = className(textContent).visibleToUser(true).findOne(times);
            break;
        case "desc":
            thisEle = desc(textContent).visibleToUser(true).findOne(times) || text(textContent).visibleToUser(true).findOnce();
            break;
        case "descC":
            thisEle = descContains(textContent).visibleToUser(true).findOne(times) || textContains(textContent).visibleToUser(true).findOnce();
            break;
        default:
            thisEle = text(textContent).visibleToUser(true).findOne(times) || desc(textContent).visibleToUser(true).findOnce();
    }
    if (thisEle) {
        log(textContent);
        if (ifclick == 2) return thisEle;      // 只返回控件
        sleepS(hold_time);
        if (ifclick == 0) {                     // 坐标点击
            clickS(thisEle);
            sleepS(before_time);
            return true;
        }
        // ifclick == 1：向上找可点击的父节点，再点击
        var parentEle = thisEle;
        var p = 0;
        while (!parentEle.clickable()) {
            parentEle = parentEle.parent();
            p++;
            if (!parentEle || p > 8) {          // 找不到可点击父节点就坐标点击
                clickS(thisEle);
                sleepS(before_time);
                return true;
            }
        }
        parentEle.click();
        sleepS(before_time);
        return true;
    }
    return false;
}

/** 坐标点击封装（带随机偏移与日志） */
function clickS(ele) {
    var b = ele.bounds();
    var x = b.centerX() + random(-10, 10);
    var y = b.centerY() + random(-10, 10);
    click(x, y);
}

/** 随机延迟，模拟人类操作，规避检测 */
function sleepS(_time) {
    if (typeof _time != 'number') _time = 1000;
    var time = random(_time * 0.8, _time * 1.2);
    sleep(parseInt(time));
}
```

**使用示例（真实场景）**：

```javascript
// 1. 处理各种弹窗（始终允许/同意/跳过）
UtilClick(/始终允许|暂不开启|同意|跳过|允许|继续登录|我知道了/, 1, "textM", 1);
// 2. 关闭广告角标（id 正则）
UtilClick(/.*main_iv_close|.*live_iv_bottom_close/, 1, "idM", 1);
// 3. 点击"完成安全验证"并处理滑块
if (UtilClick("完成安全验证", 1, "textC", 2)) {   // 只找不点
    // 拖动滑块
    var slider = UtilClick(/拖动滑块.*/, 5000, "textM", 2);
    var block = UtilClick("滑块id", 1, "idC", 2);
    if (slider && block) {
        swipe(block.bounds().centerX(), block.bounds().centerY(),
              slider.bounds().left + 0.7 * slider.bounds().width(), slider.bounds().centerY(), 500);
    }
}
```

### 6.2 模式二：弹窗清扫循环（守护线程）

主循环里持续清扫所有可能的弹窗，保证主流程不被干扰（真实项目 `UtilClick.js`）：

```javascript
threads.start(function () {
    while (true) {
        try {
            UtilClick(/始终允许|暂不开启|同意|跳过|允许|我知道了/, 1, "textM", 1);
            UtilClick(/.*main_iv_close.*/, 1, "idM", 1);
            if (UtilClick(/立即升级|更新到最新版本/, 1, "textM", 1)) continue;
            if (UtilClick(/提醒我休息/, 1, "textM", 1) && UtilClick(/取消/, 1, "textM", 1)) continue;
            if (UtilClick(/最小化/, 1, "textM", 2) && UtilClick(/退出/, 1, "textM", 1)) continue;
        } catch (e) {
            log("弹窗清扫异常: " + e);
        }
        sleepS(800);
    }
});
```

### 6.3 模式三：权限引导 UI（提升用户体验）

真实项目里常见的权限状态检查与跳转：

```javascript
"ui";
// 无障碍服务开关
ui.autoService.on("check", function (checked) {
    if (checked && auto.service == null) {
        app.startActivity({
            action: "android.settings.ACCESSIBILITY_SETTINGS"
        });
    }
    if (!checked && auto.service != null) {
        auto.service.disableSelf();
    }
});
// 悬浮窗权限
ui.overlayService.on("check", function (checked) {
    if (checked && !floaty.checkPermission()) {
        app.startActivity({
            action: "android.settings.action.MANAGE_OVERLAY_PERMISSION",
            data: "package:" + context.getPackageName()
        });
    }
});
```

### 6.4 模式四：网络验证 / 卡密登录 SDK 封装

真实项目（`main.js` + 瑞科网络验证）的请求封装套路：

```javascript
var sdk = new SDK("appKey", "appSecret");
sdk.request = function (method, path, params) {
    this.Ping();
    params["app_key"] = this._app_key;
    method = method.toUpperCase();
    var max_retries = this._retry_count;
    var retries_count = 0;
    var data = { "code": -1, "message": "连接服务器失败" };
    do {
        retries_count++;
        var sec = this.retry_fib(retries_count);   // 斐波那契退避
        params["nonce"] = this.randomNonce();
        params["timestamp"] = String(new Date().getTime());
        var ps = this.joinParams(params);
        var sign = this.MD5(method + this._host + path + ps + this._app_secret);
        params["sign"] = sign;
        var resp, body;
        try {
            if (method === "GET") resp = http.get(this._protocol + "://" + this._host + path + "?" + ps + "&sign=" + sign);
            else resp = http.post(this._protocol + "://" + this._host + path, params);
            body = resp.body.string();
            data = JSON.parse(body);
            return data;
        } catch (error) {
            log("[*] request error: ", error, sec + "s后重试");
            this.switchHost();           // 多服务器容灾
            sleep(sec * 1000);
        }
    } while (retries_count < max_retries);
    return data;
};
```

**要点**：参数签名（MD5）、时间戳防重放、重试退避、服务器容灾切换。

### 6.5 模式五：OCR 初始化（多 OCR 切换）

```javascript
var curOcrName = "浩然";
var hrOcr = null, tomatoOcr = null;

function initOcr(ocrName) {
    curOcrName = ocrName;
    if (tomatoOcr) tomatoOcr.end();
    if ("浩然" === ocrName) {
        hrOcr = hrOcr || $plugins.load("com.hraps.ocr");
        console.log("初始化浩然OCR");
    } else if ("tomato" === ocrName) {
        var cls = $plugins.load("com.tomato.ocr");
        tomatoOcr = new cls();
    }
}
```

### 6.6 模式六：WebView 桥接（Hybrid UI）

真实项目 `project加载网页webview/main.js` 的思路：UI 内嵌 WebView 加载远程/本地 HTML，通过 `evaluateJavascript` 与 jsbridge 协议双向通信：

```javascript
"ui";
ui.layout(<webview id="web" w="*" h="*" />);

function callJs(code, callback) {
    ui.web.evaluateJavascript("javascript:" + code, new JavaAdapter(android.webkit.ValueCallback, {
        onReceiveValue: function (val) { if (callback) callback(val); }
    }));
}

// 网页里 console.log("jsbridge://cmd/data") 触发
ui.web.webChromeClient = new JavaAdapter(android.webkit.WebChromeClient, {
    onConsoleMessage: function (msg) {
        if (msg.message().indexOf("jsbridge://") == 0) {
            var parts = msg.message().split("/");
            var data = JSON.parse(java.net.URLDecoder.decode(parts[2], "UTF-8"));
            var result = handleCmd(data.cmd, data.args);
            callJs("auto.callback(" + JSON.stringify({ callId: data.callId, args: result }) + ")");
        }
        return true;
    }
});
ui.web.loadUrl("http://xxx.com/index.html?t=" + new Date().getTime());
```

### 6.7 模式七：多账号/队列任务

```javascript
var storage = storages.create("task_app");
var nowIndex = storage.get("nowAccountIndex") || 0;
var accounts = ["user1|pass1", "user2|pass2"];

function runTask(account) {
    var parts = account.split("|");
    var user = parts[0], pwd = parts[1];
    // ...执行登录+任务...
}

// 主流程
for (var i = nowIndex; i < accounts.length; i++) {
    storage.put("nowAccountIndex", i);      // 进度持久化，崩溃可续跑
    runTask(accounts[i]);
    sleepS(2000);
}
```

---

## 7. 典型项目骨架示例

一个"UI + 悬浮窗 + 后台任务"的完整项目结构：

```
project/
├── project.json
├── main.js            # 入口（UI 模式）
├── config.js          # 配置
├── utils.js           # 工具函数
├── task.js            # 核心任务逻辑
└── res/
    ├── icon.png
    └── splashIcon.png
```

```javascript
// main.js —— 完整骨架
"ui";
var config = require("./config.js");
var utils = require("./utils.js");
var task = require("./task.js");

ui.statusBarColor("#000000");
ui.layout(
    <vertical padding="16">
        <text text="自动化助手" textSize="20sp" textStyle="bold" gravity="center" />
        <input id="input_count" hint="任务次数" inputType="number" />
        <button id="btn_start" text="开始任务" w="*" bg="#4CAF50" textColor="#ffffff" />
        <button id="btn_stop" text="停止任务" w="*" bg="#F44336" textColor="#ffffff" />
    </vertical>
);

var running = false;
var worker = null;

ui.btn_start.on("click", function () {
    if (running) return toast("已在运行");
    if (auto.service == null) return toast("请先开启无障碍服务");
    running = true;
    worker = threads.start(function () {
        task.run(ui.input_count.text());
    });
});

ui.btn_stop.on("click", function () {
    running = false;
    if (worker) worker.interrupt();
});

$events.on('exit', function () {
    // 清理资源（Pro）
});
```

---

## 8. 常见低级错误清单（AI 必查）

> 这一节是给 AI 的**自查清单**。每次写完代码，逐条核对。

### 8.1 致命错误（编译/运行直接失败）

| # | 错误 | 正确做法 |
|---|---|---|
| 1 | 使用 `document`/`window`/`fetch` 等浏览器 API | Auto.js 无 DOM，改用 `http.*`、`ui.*` |
| 2 | 用 `import`/`export` ES 模块语法 | Rhino 用 `require`/`module.exports` |
| 3 | 忘记文件头 `"auto";`/`"ui";` | 第一行必须声明模式 |
| 4 | 直接调用 `exit()` 后的代码 | `exit()` 立即终止，其后代码不执行 |
| 5 | `text("x").findOne()` 结果不判空就 `.click()` | `var el = text("x").findOne(2000); if (el) el.click();` |
| 6 | 变量名与全局冲突（如 `window`、`date`、`name`） | 避免覆盖全局；用 `var` 声明 |
| 7 | 在 `"ui";` 模式用 `importClass` 但没写 `"rhino";` | Pro 下 UI 模式默认 Rhino，但混用时确认引擎 |
| 8 | 在子线程里直接操作 UI 控件 | 必须 `ui.run(function(){...})` 包裹 |

### 8.2 逻辑错误（能运行但行为不对）

| # | 错误 | 正确做法 |
|---|---|---|
| 9 | `textMatches("登录")` 当子串匹配 | `textMatches` 是**正则**匹配，子串用 `textContains` |
| 10 | `id("com.xxx:id/btn")` 带包名 | `id()` 只匹配资源 id 名，如 `id("btn")`；包名不匹配 |
| 11 | 截图后不 `sleep` 直接识别 | `requestScreenCapture()` 后 `sleep(1000)` 再截 |
| 12 | 坐标写死（1080x1920）换机失效 | 用 `device.width/height` 或控件 `bounds()` |
| 13 | `findOne` 无超时参数（默认无限等待）卡死 | 必须传超时：`findOne(3000)` |
| 14 | 控件不可点击仍调用 `.click()` | 向上找可点击父节点或用 bounds 中心坐标（见 6.1） |
| 15 | `swipe` 目标点超出屏幕 | 目标点范围 `0 <= x <= device.width` |
| 16 | 图片对象不 `recycle()` 内存泄漏 | 用完 `img.recycle()` |
| 17 | `requestScreenCapture()` 重复调用抛异常 | 脚本内只调用一次 |
| 18 | `storages.create` 用相同名字在不同脚本间冲突 | 用唯一命名空间名 |
| 19 | `http.get` 不设 timeout，网络异常卡死 | `timeout: 10000` 或 try/catch |
| 20 | UI 模式主线程 `sleep(10000)` 卡死界面 | 长任务放 `threads.start` |
| 21 | `exit()` 用在 UI 回调里直接闪退 | 用 `engines.myEngine().stop()` 或标记停止 |
| 22 | 正则里中文未转义或编码问题 | 用 `/[\u4e00-\u9fa5]/` 或直接中文（文件保存为 UTF-8） |
| 23 | `floaty.window` 在主线程创建卡死 | 放 `threads.start` 中创建 |
| 24 | 判断 `text("确定")` 但按钮是 ImageView 无 text | 改用 `desc()` 或 `id()` 或找父节点 |

### 8.3 细节坑（资深开发者也可能踩）

- **`sleep()` 与 `sleepS()`**：真实项目几乎都用随机延迟 `sleepS`（`random(0.8x, 1.2x)`）模拟人类，降低风控。AI 写自动化任务时**默认使用随机延迟**。
- **`visibleToUser(true)`**：真实项目普遍在查找时加 `.visibleToUser(true)`，只找屏幕上可见的控件，避免点到"存在但不可见"的僵尸节点。
- **`clickable` 层级**：很多 App 的可点击节点是文本的**父容器**，所以社区封装会向上最多找 8 层。
- **`toast` vs `log` vs `console`**：`toast` 短暂气泡；`log`/`toastLog` 进日志；`console.log` 也可用但样式不同。UI 中显示日志用 `console` 控件（`ConsoleView`）。
- **`runtime.console`**：可把日志输出到悬浮窗/UI 的 ConsoleView，是调试利器。
- **字符串模板**：Rhino 支持 `` `模板` `` 但也支持 ES5 拼接 `+`；老版本避免嵌套模板。
- **`new Date().getTime()`**：时间戳用这个；不要用 `Date.now()`（部分旧版 Rhino 缺）。
- **`random(a, b)`**：Auto.js 全局 `random` 是**整数**区间 [a,b)；需要浮点用 `Math.random()`。
- **JSON 序列化**：`JSON.stringify`/`JSON.parse` 可用；含中文正常。
- **`open()` 文件**：`open(path, "r"/"w"/"a")` 返回文件对象，用完 `.close()`。
- **shell 权限**：`shell()` 需 root；无 root 用 `$shell`（Pro）或 adb。
- **Android 11+ 存储**：直接读写 `/sdcard` 受限，用 `files` API 或申请 `MANAGE_EXTERNAL_STORAGE`。
- **通知监听**：`events.observeNotification()` 需要"通知使用权"单独授权。
- **`auto.waitFor()` 在 UI 模式**：`"ui";` 模式下不要无条件 `auto.waitFor()`（会阻塞 UI），改用开关引导（见 6.3）。
- **打包加密**：Pro 支持 Rhino 普通加密 / Dex 加密 / Snapshot 加密；加密后 `.js` 不可读，调试困难，保留源码。

---

## 9. 编写规范与代码风格

结合官方文档与真实项目，推荐以下规范（**AI 生成 Auto.js 代码时默认遵守**）：

1. **文件头**：第一行 `"auto";` 或 `"ui";`（+ 必要时 `"rhino";`），空一行后写代码。
2. **命名**：项目里大量使用中文变量名（`账号库`、`运行记录路径`、`主线程`），JavaScript 完全支持。推荐：**中文命名用于业务概念，英文命名用于通用工具**，一致性优先。
3. **入口结构**：
   ```javascript
   "auto";
   auto.waitFor();
   setScreenMetrics(device.width, device.height);
   // 声明配置
   // 定义函数
   // 主流程
   ```
4. **防崩溃**：所有 `findOne` 结果判空；所有可能抛错的网络/文件操作包 try/catch。
5. **日志**：关键节点 `toastLog()`（既提示又留痕）；调试用 `console.show()` 浮窗看日志。
6. **随机延迟**：交互之间加 `sleepS(500~2000)` 随机间隔。
7. **进度持久化**：多账号、长任务用 `storages` 记录断点，支持续跑。
8. **退出清理**：`$events.on('exit', ...)` 关闭悬浮窗/回收图片/注销监听。
9. **模块化**：config / utils / task 分离；`require` 相对路径。
10. **兼容**：优先 ES5 语法 + `let/const`；避免 `?.`、`??`、`flat` 等新特性；用 `function` 而非纯箭头（回调里 `this` 语义）。

---

## 附录 A：高频 API 一句话速查

| 场景 | API |
|---|---|
| 等待无障碍 | `auto.waitFor()` |
| 点击 | `click(x,y)`、`el.click()` |
| 输入 | `setText(el, "文本")`、`setText(x,y,"文本")` |
| 查找控件 | `text()/desc()/id()/className()/bounds()` + `findOne(ms)/findOnce()/find()` |
| 滑动 | `swipe(x1,y1,x2,y2,ms)`、`gesture(ms, [x,y], ...)` |
| 返回 | `back()` |
| 截图 | `requestScreenCapture()` → `captureScreen()` |
| 找色 | `findColorInRegion(img, color, x, y, w, h)`、`findColor`、`findMultiColors` |
| 找图 | `images.findImage(img, template, {threshold})` |
| 图片 | `images.read/save/clip/scale/pixel/recycle` |
| OCR | `$plugins.load("...mlkit.ocr")`、`$plugins.load("...paddleocr")` |
| HTTP | `http.get/post/postMultipart(url, options)`，`res.body.string()/json()` |
| WebSocket | `web.newWebSocket(url, {eventThread:'this'})` + `.on("text",...)` |
| 存储 | `storages.create(name)` → `put/get/remove` |
| 文件 | `files.read/write/append/exists/create/remove/listDir` |
| 线程 | `threads.start(fn)`、`threads.shutDownAll()`、`threads.atomic(n)` |
| 定时 | `setTimeout/setInterval/clearInterval` |
| 对话框 | `alert/confirm/rawInput/dialogs.select/dialogs.build` |
| 悬浮窗 | `floaty.window(xml)`、`floaty.rawWindow(xml)`、`w.setPosition/setSize/close` |
| UI 布局 | `"ui"; ui.layout(<xml>); ui.控件id.on("click", fn)` |
| 设备 | `device.width/height/sdkInt/getAndroidId()` |
| Java | `importClass(android.xxx)`、`new JavaAdapter(接口, {方法})` |
| 事件 | `events.observeKey/onKeyDown/observeNotification/onNotification` |
| Pro 插件 | `$plugins.load("包名")` |
| Pro 设置 | `$settings.isEnabled/setEnabled('stable_mode'/'foreground_service')` |
| 退出回调 | `$events.on('exit', fn)` |
| 加载 dex | `runtime.loadDex("loader.dex")` |

---

## 附录 B：官方文档结构索引

官方完整文档包含以下板块，编写时可参考：

- **博客区**：7Zip 插件、FFMpeg 插件、MLKitOCR 插件、加密说明（Rhino/Node/Dex/Snapshot）、UI 与权限互动、VSCode 调试
- **自定义教程**：监听电话状态（BroadcastReceiver + JavaAdapter）、拨打电话、发送短信、渐变背景 Drawable
- **示例代码区**（按分类）：应用、画布、控制台、协程、加密摘要、数据库、设备、对话框、脚本引擎、事件监听、文件读写、悬浮窗、HTTP、图片图色、调用 Java、JavaScript、Node.js、多线程、OCR、本地存储、项目打包、Root 自动操作、传感器、设置、Shell、任务定时、WebSocket、Zip
- **常用 Javascript 函数接口示例**

文档中大量示例为 `"auto";`/`"ui";` + `auto.waitFor()` 结构，与本文档第 7 节骨架一致。

---

*本文档由 Auto.js Pro 官方文档与大量开源实战项目源码综合提炼而成，供 AI Agent 编写 Auto.js Rhino 脚本时作为技能参考。*

---

<!-- ==================== 第二部分：Node.js 引擎（第二代 API） ==================== -->
<!-- 以下内容由 Auto.js-AIAgentSkills-NodeJS.md v1.4 并入（checkSyntax Node 规则 + 请求体纯 ASCII 规则） -->

# 第二部分 · Node.js 引擎（第二代 API）AI Agent 技能

> **给 AI 的阅读须知**：Rhino（第一部分）与 Node.js（本部分）是两套不同的 API 体系。用户未指定引擎时默认 Rhino（第一部分）；指定 Node.js / 第二代 API / npm / 高性能 / 高加密需求时使用本部分。两部分的心智模型不可混用。

> title: Auto.js / Auto.js Pro (Node.js 引擎) AI Agent 技能说明  
> description: 面向 AI Agent 的 Auto.js Pro Node.js（第二代 API）自动化开发技能，包含引擎启用方式、异步心智模型、模块体系与迁移对照、控件定位、截图找图、OCR、悬浮窗、定时任务、Shell、MCP 工具接口，以及观察界面、生成脚本、语法检查、执行验证和失败修复的完整编排流程  
> tags: autojs, nodejs, v8, javascript, android, 自动化, 无障碍, mcp, agent, 第二代api  
> version: 1.4  
> engines: Node.js (V8, 第二代 API)  
> sources:  
> \- Auto.js Pro 官方文档（llms-v9 系列公开知识库）  
> \- Auto.js Pro 9.3.11 APK 内置 Node.js 官方示例（assets/sample/中文/Node.js/）  
> \- Auto.js-AIAgentSkills.md（Rhino 版技能，本文档为其 Node.js 姊妹篇）  

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

- 文档总览：Auto.js Pro 官方 llms-v9 知识库
- 完整文档：Auto.js Pro 官方 llms-full-v9 知识库（含全部类/接口/模块参考）
- 重点章节：第二代API和第一代API的区别 / 快速开始（node.js 引擎、npm、Java 调用）/ 线程与线程模型 / 模块参考（accessibility、media_projection、image、ocr、ui_selector、floating_window、shell、work_manager）/ 如何从第一代API迁移过来
- 官方 Node.js 示例：APK 内 `assets/sample/中文/Node.js/`（任务、shell、剪贴板、对话框、传感器、OCR、找图找色、悬浮窗、原生界面、Java 交互、npm 模块等 18 组）
