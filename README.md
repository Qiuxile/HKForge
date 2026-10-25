# HKForge

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey.svg)]()

HKForge 是一个面向 Windows 平台的高性能、基于 AutoHotkey (AHK v2) 生态的现代化可视化图形脚本工具。

**核心理念**：摒弃传统图形化编程中复杂的嵌套形状，采用**“单一块（Single-Block）”**范式。所有指令统一为单栏竖向排列，支持无限层级的表达式递归与值槽嵌套。用户无需掌握任何 AHK 语法，即可通过拖拽拼装生成原生自动化脚本。

**工程架构**：为了解决传统单体架构的执行阻塞、热键冲突与内存泄漏问题，HKForge 采用了 **C# (UI 表现层) + Go (底层守护层) + AHK (执行层)** 的微服务化混合架构。组件间通过本地 IPC 通信，实现了彻底的进程解耦与异常隔离。

---

## ✨ 核心特性

- **🧩 扁平化的一体化积木编辑**：基于树状数据模型，支持单栏拖拽排序。参数常驻输入框，提供类似命令面板（Command Palette）的拼音/中英混合极速模糊检索。
- **🎬 智能录制与意图转换**：内置录制引擎，支持全局键鼠事件捕获。不仅能记录操作，更能通过轨迹精简与窗口探测，尝试将“死坐标”转换为目标窗口/控件，大幅降低脚本的脆弱性。
- **🤖 专属 AI Skill 辅助开发**：内置 AI 扩展模块，允许用户通过自带 API Key 接入大模型。配合专属的 System Prompt（Skill），AI 可直接生成结构化的积木 JSON，或将晦涩的野生 AHK 代码逆向翻译成可编辑的图形化积木。
- **📦 原生代码逃生舱 (Escape Hatch)**：内置专属的“多行代码块”积木。遇到 AI 无法拆解、或外部复制的复杂底层逻辑，可一键装入“黑盒”中与积木混排，实现图形化与纯文本的混合双打。
- **🛡️ 内核级的进程隔离与双模式调度**：底层由 Go 编写的 `Runtime` 托管。业务热键全权交由 AHK 原生处理；日常运行采用主进程直接派发，调试模式则拉起独立的 Go Worker 子进程作为 DBGp 代理，确保调试崩溃绝不会影响主进程。配合 Windows `Job Object` 强绑定，系统自动清理所有残留子进程。
- **⚡ 绝对的急停热键 (Kill Switch)**：`Runtime` 独占注册**唯一**一个最高优先级的系统级急停热键。触发后中止当前脚本并强制清理所有实例，彻底杜绝鼠标/键盘卡死失控。
- **🤖 深度 UIA 自动化支持**：底层默认搭载带有 UI Automation 扩展的 AHK 引擎。可无视窗口遮挡与分辨率变化，精准对 Chrome、Electron (如 VS Code) 等现代应用进行语义级控件操作。
- **🔄 统一的生命周期与版本管理**：`Manager` 统一负责 `Studio`、`Player`、`Runtime` 以及 AHK 编译器 (`Ahk2Exe`) 的下载、校验与无缝热更新。

---

## 🏛️ 四角协同架构

HKForge 采用“控制面与执行面分离”的设计，由四个核心进程构成，通过本地命名管道（Named Pipe / IPC）进行高吞吐通信。

1. **HKForge.Manager (C# / WPF) — 运维管家**
   - 系统的首要启动入口。负责初始化环境、校验依赖。
   - 统一管理 `Studio`、`Player`、`Runtime` 以及 AHK 编译器 (`Ahk2Exe`) 的版本更新与下载。
   - 负责自身的独立热更新（通过释放临时 Updater 进程替换自身文件）。

2. **HKForge.Runtime (Go / 无 UI) — 守护内核**
   - 以极低内存占用常驻后台。系统的**纯粹宿主与保镖**，不干涉任何业务热键逻辑。
   - 独占注册全局唯一的 ESC 急停热键，与 C# 客户端进行 IPC 通信。
   - **进程生命周期管理**：支持两种调度模式（日常直接管理 AHK 进程；调试时拉起隔离的 Go Worker 子进程接管调试通信）。
   - 通过 Windows `Job Object` 对 AHK 进程进行强绑定与回收。

3. **HKForge.Studio (C# / WPF) — 创作中心**
   - 重型可视化编辑器。负责 AST 解析、UI 渲染、积木状态管理、代码生成以及**全局动作录制**。
   - 内置 AI 转换引擎，负责处理自然语言到积木 JSON，以及 AHK 代码向积木的智能转换。
   - 通过 IPC 向 `Runtime` 发送执行/调试指令。

4. **HKForge.Player (C# / WPF) — 轻量仪表盘**
   - 极小占用的桌面悬浮窗或托盘程序，用于实时监看脚本运行队列及手动启停。

---

## 📁 仓库结构 (Monorepo)

项目采用单仓库管理，代码按语言与技术栈物理隔离：

```text
HKForge/
├── HKForge.slnx              # Visual Studio 解决方案 (统一管理 C# 项目)
├── global.json               # 锁定 .NET SDK 版本
├── src/
│   ├── CSharp/
│   │   ├── HKForge.Manager/  # 环境与版本管理器 (WPF)
│   │   ├── HKForge.Studio/   # 积木编辑器 (WPF)
│   │   ├── HKForge.Player/   # 轻量播放器 (WPF)
│   │   └── HKForge.Shared/   # C# 共享类库 (IPC 客户端, 协议模型)
│   └── Go/
│       └── HKForge.Runtime/  # 底层守护进程 (Go 源码)
├── assets/
│   ├── Runtime/              # AutoHotkey 解释器 (由 Manager 维护)
│   ├── Compiler/             # Ahk2Exe 编译器及相关依赖 (由 Manager 维护)
│   └── Skills/               # 供 AI 转换使用的提示词与 JSON Schema
├── docs/                     # 开发者文档 (元数据规范、IPC 协议说明)
└── build/                    # CI/CD 构建脚本与发布工具
```

---

## 🔐 文件格式与数据流向规范

HKForge 不再发明晦涩的自定义语法，而是彻底拥抱双文件流/双格式，通过 JSON 与 AHK 原生代码的极致配合来运作：

1. **工程文件 (`.hkf`)**：纯 JSON 格式。仅存储积木的树状结构、参数和层级关系，是 Studio 的唯一真相源。
2. **执行脚本 (`.ahk`)**：纯文本格式。由 Studio（或 Go Runtime）根据 `.hkf` 模板生成，交由 AHK 底层执行。

**核心处理策略**：
- **正向生成**：用户拖拽积木（保存为 JSON），点击运行时，程序根据 `catalog.json` 模板将其渲染为 100% 纯正的 AHK 代码。
- **原生代码逃生舱**：在 JSON 中专门有一种 `raw_code` 积木类型，用户在其中手写的任何代码，在生成时都会原封不动地插入到 `.ahk` 中。
- **代码逆向与 AI 救援**：如果用户拿了一段手写的纯 `.ahk` 代码想图形化，直接交给 Studio 内置的 AI。AI 会将其“翻译”为标准 JSON。如果遇到无法翻译的高级语法，AI 会自动将其打包进 `raw_code` 积木中，确保逻辑不丢失。

---

## 🚀 快速开始 (开发环境)

### 环境依赖

- Windows 10 / 11
- [.NET SDK 8.0+](https://dotnet.microsoft.com/)
- [Go 1.21+](https://go.dev/)
- [AutoHotkey v2](https://www.autohotkey.com/) (请将 `AutoHotkey64_UIA.exe` 放入 `assets/Runtime/` 目录)

### 编译运行

1. 克隆项目并进入目录：

   ```bash
   git clone https://github.com/yourname/HKForge.git
   cd HKForge
   ```

2. 编译 Go 底层守护进程：

   ```bash
   cd src/Go/HKForge.Runtime
   go build -o ../../../assets/Runtime/HKForge.Runtime.exe
   ```

3. 使用 Visual Studio 打开 `HKForge.slnx`，在顶部菜单设置 **多启动项目 (Multiple Startup Projects)**，将 `HKForge.Manager` 设为 `Start`（其余按需设为 Start without debugging）。

4. 按下 `F5`，环境管理器将初始化并拉起底层运行时。

---

## 🗺️ 开发路线 (Roadmap)

- [x] 确立四角协同架构与 JSON/AHK 双模态数据规范。
- [ ] 实现 Runtime 核心：ESC 急停热键监听、Named Pipe IPC 服务端、Job Object 进程强制回收。
- [ ] 实现 Runtime 双模式：支持主进程直接拉起 AHK，以及 Worker 子进程进行 DBGp 调试代理。
- [ ] 实现 Manager 基础：环境校验、组件版本管理、静默拉起与自更新机制。
- [ ] 实现 Studio 编辑器：基于 MVVM 的单栏积木列表、值槽递归拖拽、拼音模糊检索。
- [ ] 实现 Studio 录制器：全局事件抓取、轨迹精简与目标控件元素解析。
- [ ] 实现 AI Skill 管道：支持解析自然语言生成积木 JSON，以及代码向积木的智能转换。
- [ ] 建立本地积木扩展规范，支持动态加载社区开发的新积木。

---

## 📄 开源协议

本项目基于 **[GNU General Public License v3.0 (GPL-3.0)](https://www.gnu.org/licenses/gpl-3.0.html)** 开源。你可以自由地修改和分发本项目，但衍生作品必须同样开源。

**第三方组件特别说明**：
底层的 `AutoHotkey64_UIA.exe` 遵循其自身的 GPL v2 协议。HKForge 通过独立的进程通信（Mere Aggregation）与其交互，未发生代码级链接，因此不影响本项目 GPL v3 协议的独立适用性。请勿将本工具用于制作违反游戏协议或破坏系统安全的软件。