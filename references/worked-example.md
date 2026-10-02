# 参考实现导读：stm32cord-dev（STM32CordBoard / STM32F103RCT6）

参考实现是作者环境中的 `stm32cord-dev` skill（为一块自制 STM32F103RCT6 开发板生成，未随本仓库开源）——给新板造 skill 前，先按下表了解成品应有的样子；若你机器上已有按本流程生成的板卡 skill，直接以它为模板。

## 文件清单与角色（新板 skill 照此结构）

| 文件 | 角色 | 新板可否直接抄 |
|---|---|---|
| `SKILL.md` | 工作流入口：板子一句话/工具链表/黄金循环/板载资源/红线/排障/结构骨架/已验证状态 | **框架直接抄**，内容全换 |
| `references/pinmap.md` | 逐脚网络表 + 板载资源 + 排针逐脚表 + 常用外设→引脚对照 + 可信度分级 | 结构抄，内容按新板读图重写 |
| `references/hardware.md` | 电源树/时钟/复位启动/调试口/芯片资源清单/电气注意 | 结构抄 |
| `references/schematic.pdf` | 原理图原件（唯一硬件事实来源） | 换成新板的 |
| `scripts/build.sh` | 递归编译 + 自动 -I + size 上限检查 | 几乎直接抄（换编译器路径/架构参数） |
| `scripts/flash.sh` | OpenOCD program verify reset | 换 target cfg |
| `scripts/probe.sh` | halt 读 PC 排障 | 直接抄 |
| `scripts/monitor.ps1/.sh/.bat` | 串口监控（自动探测 COM、DTR/RTS 保护、时间戳） | **直接抄**，三件套含编码坑处理 |
| `assets/template/sys/` | 启动代码+寄存器头+链接脚本（自包含，不依赖 CMSIS） | **必须按新 MCU 重写** |
| `assets/template/bsp/` | 板载资源驱动（led/key/uart_log） | 按新板板载资源重写 |
| `assets/template/app/main.c` | 冒烟测试（LED+心跳+按键） | 照套路写 |
| `assets/template/PINMAP.md` | 引脚占用登记（板载固定占用预填） | 结构抄 |

## 这次实战的三次勘误（为什么流程里有那么多检查点）

1. **按键映射**：文本层把 KEY1/KEY2 配到 PC0/PC1（实际 PC1/PC2，PC0 无网络）→ 固件监听悬空脚，制造出"按键时好时坏、摸板翻转"的假象，一度误诊为"R6 虚焊"。用户实测推翻后修正。
2. **OLED SCK/RES**：PB13/PB14 配反，用户对原理图发现。
3. **排针与 H2**：文本层少算 J3 顶脚 GND、把 BOOT0 跳线座 H2 误读成电源座。渲染图逐脚读出后修正。

共同根源：只信 PDF 文本层提取。所以元流程把"渲染读图 + 用户核对 + 闭合检查 + 真机实测"设为硬性阶段。

## 真机验证记录（Definition of Done 的样子）

- 冒烟固件：1.6KB，心跳时间戳间隔精确 1s（时钟树正确），CFGR 读回确认 PLL 72MHz
- 串口双向：TX 心跳正常 + RX 回环测试 PASS
- 交互：K1/K2/K3 → LED 翻转用户实测通过
- 以上全部写进 SKILL.md"已验证状态"并带日期
