# HKForge

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey.svg)]()

HKForge 是一个面向 Windows 平台的、基于 AutoHotkey (AHK) 生态的现代化可视化图形脚本工具。

它的核心理念是 **“图形化编程”**：摒弃了传统图形化工具中复杂的积木形状（如 C 型循环包裹块），所有指令统一为“单栏竖向排列”，配合无限层级的表达式嵌套与拖拽。这种范式极大地降低了用户的认知负担，无需学习任何 AHK 语法，即可像搭建积木一样构建出结构清晰、可维护的自动化脚本。

为了打破传统脚本工具“单体架构”带来的性能瓶颈与稳定性灾难，HKForge 采用了 **C# (UI 表现层) + Go (控制与守护层) + AHK (执行层)** 的跨语言混合架构，实现了彻底的进程解耦。

## ✨ 核心特性

- **🧩 扁平化的一体化积木编辑**：单栏竖向排列，基于树状数据模型。支持按住拖拽排序、值槽递归嵌套。参数常驻输入框，并提供类似命令面板的本地极速检索。
- **🛡️ 内核级的进程隔离与安全**：借助 Go 编写的 Manager 守护进程，每个脚本都在独立的 `AutoHotkey64.exe` 子进程中执行。Manager 通过 Windows 内核对象 `Job Object` 与其强绑定，一旦宿主崩溃或被强杀，系统会自动清理所有脚本进程，杜绝“脱缰”的僵尸进程。
- **⚡ 绝对的急停热键 (Kill Switch)**：Manager 独占注册一个最高优先级的绝对热键。单击可强制中止当前脚本，长按 3 秒强制停止所有脚本并清理系统状态，彻底解决鼠标键盘卡死失控的痛点。
- **🖥️ 双 UI 协同模式**：
  - `HKForge Studio`：重型创作中心，专注于积木编辑、语法校验与代码生成。
  - `HKForge Player`：轻量级悬浮控制台，常驻桌面边缘，占用极低，用于全局热键状态的实时监控与快捷启停。
- **📁 原生 `.ahk` 文件级兼容**：积木的结构化数据被序列化为 Base64，以单行特殊注释（`;@HKFORGE`）嵌入在 `.ahk` 文件头部，下方则是纯净的 AHK v2 代码。这使得脚本既能在 HKForge 中完美还原为积木，也能在任意原生 AHK 环境中直接运行。
- **🔍 本地优先与动态扩展**：内置高频自动化指令集，支持无缝安装本地社区扩展包。所有扩展均支持中文、英文 ID 及拼音首字母的毫秒级模糊匹配。

## 🏛️ 运行架构

HKForge 采用了“数据面与控制面分离”的跨进程架构，四个独立部分各司其职：

1. **HKForge.Studio (C# / WPF)**：表现层。负责 UI 渲染、AST（抽象语法树）解析、积木状态管理及 AHK 代码生成。
2. **HKForge.Player (C# / WPF)**：轻量控制台。作为独立窗口或托盘程序运行，提供实时的状态流展示。
3. **HKForge.Manager (Go)**：控制面（守护进程）。无 UI 界面，以极低内存占用常驻后台。负责底层全局热键 Hook、子进程生命周期管理、安全点（快照）记录，以及 IPC 消息路由。
4. **AutoHotkey64.exe**：执行面。官方的 AHK 引擎，作为纯执行沙箱，只负责运行 Manager 派发的代码。

**通信机制**：两个 UI 进程与 Manager 之间，全部通过本地命名管道 (Named Pipe) 进行高吞吐、低延迟的 IPC 通信。Studio 和 Player 之间不直接交互，所有状态同步均通过 Manager 作为中枢进行分发，避免了多客户端的状态撕裂。

## 📁 仓库结构

本项目采用 Monorepo（单仓库）结构进行规范化管理：

```text
HKForge/
├── HKForge.slnx              # Visual Studio 解决方案 (仅管理 C# 项目)
├── global.json               # 锁定 .NET SDK 版本，确保构建一致性
├── src/
│   ├── CSharp/
│   │   ├── HKForge.Studio/   # 积木编辑器 WPF 主项目
│   │   ├── HKForge.Player/   # 轻量播放器 WPF 主项目
│   │   └── HKForge.Shared/   # C# 共享类库 (IPC 客户端, 传输协议模型)
│   └── Go/
│       └── HKForge.Manager/  # 后台守护进程 (Go 源码，独立通过 go build 编译)
├── assets/
│   └── Runtime/              # AutoHotkey 官方解释器及必要的运行环境
├── docs/                     # 开发者文档 (积木元数据规范、IPC 协议说明)
└── build/                    # 统一构建、持续集成与打包脚本 (如 Inno Setup)
```

## 🔐 脚本文件格式与冲突解决

HKForge 生成的文件完全保留 `.ahk` 后缀。文件头部包含一段单行 Base64 的 JSON 注释，用于持久化积木状态：

```autohotkey
;@HKFORGE v1 eyJibG9ja3MiOlt7ImtpbmQiOiJtc2dib3giLC...
; -----------------------------------------
; 以上为 HKForge 积木元数据，请勿手动修改。
; 以下为生成的 AutoHotkey v2 代码。
; -----------------------------------------

MsgBox "Hello, HKForge!"
```

**安全与降级策略**：
- **数据唯一真相源**：编辑器以积木数据为准。每次保存时，会根据积木重新生成下方的 AHK 代码。
- **手写代码保留**：用户可以在标记下方直接手写额外的 AHK 逻辑，编辑器保存时不会覆盖该部分。
- **智能降级**：如果在外部编辑器中修改了代码区，导致代码与积木标记不匹配，Studio 打开该文件时会自动将积木视图**降级为只读**，并提示用户选择“以代码为准”或“以积木为准”，防止无意间的数据覆盖丢失。如果误删了标记，文件将被视作普通 AHK 脚本。

## 🚀 快速开始 (开发环境)

### 环境依赖

- Windows 10 / 11
- [.NET SDK 8.0+](https://dotnet.microsoft.com/)
- [Go 1.21+](https://go.dev/)
- [AutoHotkey v2](https://www.autohotkey.com/) (开发阶段请手动将 `AutoHotkey64.exe` 放入 `assets/Runtime/` 目录)

### 编译运行

1. 克隆项目并进入目录：

```bash
git clone https://github.com/yourname/HKForge.git
cd HKForge
```

2. 编译 Go 底层守护进程，并输出到运行目录：

```bash
cd src/Go/HKForge.Manager
go build -o ../../../assets/Runtime/HKForge.Manager.exe
```

3. 使用 Visual Studio 打开 `HKForge.slnx`，在顶部菜单设置 **多启动项目 (Multiple Startup Projects)**，将 `HKForge.Studio` 和 `HKForge.Player` 的动作设为 `Start`。

4. 按下 `F5` 即可同时拉起两个 UI 并进行联调。

## 🗺️ 开发路线 (Roadmap)

- [x] 确立 C# + Go + AHK 跨语言架构，及 `.ahk` 注释元数据格式标准。
- [ ] 实现 Manager 核心功能：全局热键监听 Hook、Named Pipe IPC 服务端、Job Object 进程强杀机制。
- [ ] 实现 Player：轻量级悬浮窗与系统托盘，接收 Manager 的状态推送。
- [ ] 实现 Studio：基于 MVVM 的单栏积木列表、值槽递归拖拽交互、本地拼音模糊检索。
- [ ] 建立本地积木扩展规范，支持动态加载社区开发的新积木。

## 📄 开源协议

本项目基于 **[GNU General Public License v3.0 (GPL-3.0)](LICENSE)** 开源。

你可以自由地修改和分发本项目，但衍生作品必须同样开源。请注意，底层的 `AutoHotkey64.exe` 遵循其自身的开源协议，本项目仅通过进程通信与其交互。由于本工具涉及底层按键模拟与进程控制，请勿将其用于制作违反游戏协议或破坏系统安全的软件。