---
name: board-skill-factory
description: 为新 MCU 开发板快速生成"专属开发 skill"的元流程（方法论提炼自 STM32CordBoard/F103 实战全记录）。凡用户说"新画了一块板子/新板子要搭开发环境/给这块板做一个专属 skill/像 stm32cord-dev 那样复刻一套"，或拿来一块陌生 MCU 板要求建立 AI 开发工作流时使用；也适用于评估"新板子该用 CubeMX 还是 AI 从头搭"。
---

# 板卡专属开发 skill 生成器（board-skill-factory）

目标：给一块新板子造出类似 `stm32cord-dev` 的专属 skill——AI 能命令行自闭环（**改代码→编译→烧录→看串口日志**）地开发这块板。
动手前先读 `references/gotchas.md`（通用坑清单）和 `references/worked-example.md`（参考实现导读）。

## 五条核心理念

1. **闭环高于一切**：skill 的价值 90% 在"已核实的引脚表 + 四件套脚本 + 自包含模板"，不在知识堆砌——模型本来就懂 MCU，缺的是能跑通的循环。
2. **证据链：实测 > 原理图渲染图 > 文本层提取 > 既有文档**。原理图 PDF 用 poppler 渲染成 PNG 视觉读图，**永远不要只信文本层提取**（实战中它配错过按键、OLED、排针三处）。
3. **环境适配**：脚本只依赖机器上真实存在的东西。没有 python 就用 PowerShell，没有 make 就写构建脚本，缺什么绕什么。
4. **串口日志是 AI 的眼睛**：模板必须自带日志 UART + 心跳输出；固件每个关键动作都打日志。
5. **用户是硬件的手**：接线/上电/按键由用户执行，AI 通过串口与调试口读状态；拿不准的引脚/电压，先问再接。

## 七个阶段（★ = 用户检查点）

### 阶段 0：输入收集
- 向用户要：MCU 型号与封装、原理图（PDF/高清图）、调试器型号、串口方式（板载 USB 串口还是外接 TTL）、板载资源清单。
- STM32/GD32/CH32 系 → 直接参照 worked-example 复用模板；其它家族（ESP32/AVR/RP2040…）→ 流程不变，阶段 4/5 换工具链（idf.py/esptool、avrdude、picotool…）。

### 阶段 1：环境审计（产出 → SKILL.md 的工具链表）
bash 逐项探测并记录绝对路径：交叉编译器、烧录调试工具（openocd/probe-rs/esptool…）及现成 target 配置、python 真身与否（Windows Store 空壳无输出）、make/cmake 有无、串口号（注册表 `HARDWARE\DEVICEMAP\SERIALCOMM`）、用户级可写目录（`~/.agents/skills/`）。

### 阶段 2：原理图消化（产出 → references/pinmap.md）
- 先 96DPI 渲整页定位，再 300/600DPI 裁剪逐区**读图**（命令模板见 gotchas.md）。
- 产出：逐脚网络表（含电气特性：高/低有效、上拉、5V 容忍）、板载资源表、排针**逐脚**表、电源/时钟/复位/启动说明。
- 排针表逐脚核实（防"顶脚其实是 GND"这类不对称）；用闭合检查验证完整性（所有应引出的引脚都出现且只出现一次）。
- ★检查点①：引脚表交用户核对——重点排针顺序、电源脚位置、易错行的标签配对。

### 阶段 3：skill 骨架
目录：`SKILL.md + references/{pinmap.md, hardware.md, schematic.pdf 原件} + scripts/ + assets/template/`。
SKILL.md 必含八节：板子一句话 / 工具链绝对路径表 / 黄金循环 / 板载资源表 / **红线** / 排障速查 / 工程结构骨架 / 已验证状态（带日期）。description 要"抢触发"：板名 + MCU 型号 + 常见动词（烧录、点灯、串口、接线、接哪儿）。

### 阶段 4：模板固件
- 首选**自包含裸机模板**（自有寄存器头 + 启动代码 + 链接脚本），不依赖 CMSIS/HAL 联网下载；vendor 库仅在确实需要时再引。
- 工程结构问用户选（分层 app/bsp/drivers/middleware/sys vs 扁平）——★检查点②。
- BSP 模块覆盖全部板载资源；冒烟 main = LED + 串口心跳 + 按键回显；项目根放 `PINMAP.md` 占用登记（板载固定占用预填）。

### 阶段 5：脚本四件套
`build.sh`（递归编译 + size 上限检查）/ `flash.sh`（program verify reset）/ `probe.sh`（halt 读状态）/ `monitor`（PowerShell ps1 + **纯 ASCII** bat 双击版，自动探测串口号）。
每个脚本写完必须真机过一遍；Windows 编码三坑见 gotchas.md。

### 阶段 6：真机端到端验收（Definition of Done）
1. build 零警告、size 合理；2. flash verify OK；3. monitor 心跳时间戳间隔精确（证明时钟树对）；4. 串口回环测试 PASS（PC 发→MCU 收，验证 RX 方向）；5. 有交互资源的让用户实测一圈；6. "已验证状态 + 日期"写进 SKILL.md。
**没跑通本阶段不许宣布 skill 完成。** 失败时调试套路：probe.sh halt 读 PC 与外设寄存器（寄存器地址先查手册，别读错偏移）；查时钟切换状态位；查向量表中断项是否用了符号名+weak alias。

### 阶段 7：交付与迭代
- 更新用户记忆文件（板子、skill 路径、坑、勘误史）。
- 给用户 2-3 个测试指令验证 skill 触发与效果，按反馈迭代 SKILL.md。
- 此后每次外设接线按协议走：AI 先报器件典型引脚请用户核对 → 给接线表（含 5V/3.3V 判定）→ 占用登记进 PINMAP.md → 进黄金循环写驱动。

## 红线
- 用户每处更正必须回写 pinmap/SKILL/记忆，并记录"曾错成什么"——勘误史就是 skill 的可信度。
- 诊断结论（如"虚焊""器件坏了"）在被实测证实前不写死进文档；与实测矛盾时先怀疑文档。
- 悬空脚会制造"闹鬼"现象（恒低、随触摸漂移）——遇到"时好时坏"先查引脚映射，再怀疑焊接。
