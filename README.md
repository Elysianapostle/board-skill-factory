# board-skill-factory · 板卡专属 AI 开发 Skill 生成器

> **一句话**：给任何一块新的 MCU 开发板，按七阶段流程快速生成一个"专属开发 skill"——让 AI 编码代理能够命令行自闭环（**改代码 → 编译 → 烧录 → 看串口日志**）地开发这块板子。
>
> **TL;DR (EN)**: A meta-skill for AI coding agents. Feed it a new MCU dev board + its schematic, and it produces a board-specific development skill (verified pin map, self-contained bare-metal firmware template, build/flash/probe/serial-monitor scripts) so the agent can develop the board fully from the command line.

这是一个 [Agent Skills](https://agentskills.io) 格式的 skill（`SKILL.md` + 附属资源），适用于 ZCode、Claude Code 等任何支持 skill 机制的 AI 编码工具。

## 为什么需要它（实战教训）

我们给一块自制 STM32F103RCT6 开发板搭建 AI 开发工作流时，踩过三个典型的坑：

1. **原理图文本层提取不可信**——直接解析 PDF 文本配对引脚网络，先后把按键映射（KEY1/KEY2 错位一行）、OLED 的 SCK/RES、扩展排针的顶脚（其实是 GND）全部配错。错误映射 + 固件监听悬空引脚，还制造出"按键时好时坏、摸一下板子灯就翻转"的**悬空脚闹鬼**假象，一度误诊为虚焊。
2. **GUI 工具链是人工瓶颈**——"先用 CubeMX 配好再给 AI"意味着每改一次引脚都要人工点一遍 GUI，AI 无法自闭环。
3. **环境差异**——机器上没有 make、python 是 Microsoft Store 空壳、PowerShell 5.1 按 GBK 解析无 BOM 的脚本……不预先审计环境，脚本全是坑。

本 skill 把这套方法论（渲染读图核实引脚 → 环境适配 → 自包含模板 → 脚本四件套 → 真机端到端验收）固化下来，**新板子直接复用整个流程**。

## 产出物长什么样

跑完七阶段，你会得到一个类似这样的板卡 skill：

```
<你的板子>-dev/
├── SKILL.md            # 工作流入口：黄金循环 / 板载资源表 / 红线 / 排障速查
├── references/
│   ├── pinmap.md       # ★逐脚核实的引脚表 + 排针逐脚定义 + 可信度分级
│   ├── hardware.md     # 电源树 / 时钟 / 复位启动 / 电气注意
│   └── schematic.pdf   # 原理图原件（唯一硬件事实来源）
├── scripts/            # build.sh / flash.sh / probe.sh / monitor(.ps1/.sh/.bat)
└── assets/template/    # 自包含裸机固件模板（启动代码+寄存器头+链接脚本+BSP+冒烟main）
```

从此开发这块板只需要：改代码 → `build.sh` → `flash.sh` → `monitor` 看日志。

## 七阶段流程

| 阶段 | 内容 | 产出 |
|---|---|---|
| 0 输入收集 | MCU 型号、原理图、调试器、串口方式 | 任务清单 |
| 1 环境审计 | 编译器/烧录工具/python 真伪/串口号逐项探测 | 工具链路径表 |
| 2 原理图消化 | **poppler 渲染成图逐区读图**（不要只信文本提取）+ 闭合检查 | pinmap.md ★用户核对 |
| 3 skill 骨架 | SKILL.md 八节式 + references | skill 目录 |
| 4 模板固件 | 自包含裸机模板 + BSP + 冒烟 main | assets/template ★用户选结构 |
| 5 脚本四件套 | build / flash / probe / monitor（真机逐个验证） | scripts/ |
| 6 端到端验收 | 心跳时间戳验时钟 + 串口回环 + 用户实测 | **Definition of Done** |
| 7 交付迭代 | 记忆沉淀 + 测试指令 + 接线咨询协议 | 可长期演进的 skill |

详细的红线、检查点与调试套路见 [SKILL.md](SKILL.md)；20 条通用坑（编码三坑、DTR/RTS 陷阱、向量表符号名、悬空脚闹鬼……）见 [references/gotchas.md](references/gotchas.md)。

## 安装

把整个目录复制到你的 AI 工具的 skill 目录：

```bash
# ZCode
git clone https://github.com/<you>/board-skill-factory.git ~/.agents/skills/board-skill-factory

# Claude Code
git clone https://github.com/<you>/board-skill-factory.git ~/.claude/skills/board-skill-factory

# 其它 agent：直接把 SKILL.md 喂给它即可
```

## 依赖

- **poppler**（`pdftoppm`/`pdftotext`，读原理图用）：Windows `scoop install poppler` 或 `choco install poppler`；Linux `apt install poppler-utils`；macOS `brew install poppler`
- **交叉工具链与烧录器**：按板子的 MCU 家族定（如 STM32/GD32: `arm-none-eabi-gcc` + OpenOCD/probe-rs + ST-Link；ESP32: idf.py/esptool；RP2040: picotool…）
- Windows 上无 python/make 也能跑（脚本用 bash + PowerShell 实现，监控器零依赖）

## 使用

skill 装好后，对 AI 说：

- 「我新画了一块 ESP32-S3 板子，原理图给你，帮我搭一个专属开发 skill」
- 「这块板子该用厂商 SDK 还是从头搭？先审计一下我机器的环境」
- 「给这块板子的 skill 补一个接线咨询协议」

## 已验证

方法论在 STM32F103RCT6 自制板上完整跑通：模板固件 1.6KB，心跳时间戳精确 1s（时钟树正确），串口双向回环 PASS，按键/LED 用户实测通过，三次引脚勘误全部通过渲染读图定位并修正。

## License

[MIT](LICENSE)
