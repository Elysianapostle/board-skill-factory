# 通用坑清单（跨板卡适用，来自实战）

## Windows 环境三坑
1. **python 可能是 Microsoft Store 空壳**：`python --version` 无任何输出 = 空壳，别用它，也别 pip。串口监控等用 PowerShell（`System.IO.Ports.SerialPort`，零依赖）实现。
2. **编码**：`.ps1` 必须 UTF-8 **带 BOM**（否则 PowerShell 5.1 按 GBK 解析，中文注释直接破坏语法）；`.bat` 必须**纯 ASCII**（cmd 按 GBK 读，中文注释变乱码命令）。中文说明放 md 里，脚本内注释用英文或确保编码正确。
3. **Git Bash 路径**：`C:\Users\...` 反斜杠会被转义吃掉 → "No such file or directory"。给用户的命令一律正斜杠 `/c/Users/...`，或提供双击版 .bat。

## 原理图阅读
4. **PDF 读取**：ZCode 的 Read 工具读某些 PDF 会误报"password-protected"（pdfinfo 显示 Encrypted: no）。用 poppler（本机已装）：
   ```bash
   pdfinfo board.pdf                                   # 先确认加密状态与页数
   pdftoppm -png -r 96 board.pdf page                  # 整页定位（约 2203x1560 px @A2横向）
   pdftoppm -png -r 300 -x X -y Y -W W -H H board.pdf crop   # 高倍裁剪，坐标=96DPI像素×300/96
   pdftotext -layout board.pdf - 2>/dev/null           # 文本层：ASCII 标号可用、中文丢失
   ```
   渲染出的 PNG 用视觉读图，中文也正常。**文本层提取会丢行/错位配对**——只用作辅助，引脚映射以读图为准。
5. **文本层配对陷阱**：标签列与引脚列在提取文本里容易错位一行（实战：KEY1/KEY2 实际挂 PC1/PC2，PC0 无网络；OLED SCK/RES 配反；排针少算一个顶脚 GND）。读图时逐行核对标签与引脚的视觉对齐关系。
6. **闭合检查**：把所有排针/接插件的信号集合与 MCU 应引出引脚集合对账，数目和成员都闭合才算读完。

## 串口
7. **DTR/RTS 陷阱**：不少板子把 DTR/RTS 接到复位或一键下载电路（Arduino 自动复位、CH340 一键 ISP）。打开串口的库常默认拉高 DTR/RTS → 一开监控板子就复位/进 bootloader。监控脚本必须显式 `DtrEnable=$false; RtsEnable=$false`，并写进 skill 红线。
8. **监控与烧录可并行**：monitor 常开 + 另一终端 build/flash 是推荐工作法。固件启动 banner 只在复位瞬间打一次，想看 banner 先开 monitor 再烧录或按复位键。
9. **串口占用**："拒绝访问" = 端口被别的程序占用。

## 调试与烧录（OpenOCD + SWD/JTAG 系）
10. 标准组合：`interface/stlink.cfg + target/<chip>.cfg + reset_config none`；烧录用 `program xx.elf verify reset exit` 一条龙。
11. **读寄存器先查手册地址**（实战：RCC_CFGR 在 base+0x04，+0x08 是 CIR——读错地址会得出"时钟没配好"的假结论）。
12. **向量表中断项必须写符号名 + weak alias**，链接器才会解析到用户的强定义；直接填 `Default_Handler` 会让中断一触发就死循环，表现为"程序没反应/串口只有半个字节"。
13. 程序跑飞定位：`probe.sh`（halt + reg pc）→ `arm-none-eabi-addr2line -e firmware.elf <PC>`。
14. g_ticks 类计数器放 .bss，可用 OpenOCD `mdw` 直读验证主循环是否活着；读两次间隔对比。

## 模板与构建
15. 编译参数：`-mcpu=… -mthumb -O2 -Wall -Wextra -ffreestanding -ffunction-sections -fdata-sections` + 链接 `--gc-sections`；build.sh 递归扫子目录、自动收集 `-I`、obj 名用相对路径防重名、size 超 Flash 上限时报错。
16. 时钟初始化要带**超时回退**（HSE 不振回退 HSI），并在心跳日志里体现时间戳准确性来验证时钟。
17. 悬空脚会"闹鬼"：恒低/恒高漂移、随触摸翻转——遇到"时好时坏、摸板子就变"先怀疑引脚映射错误（固件监听了悬空脚），再怀疑焊接。诊断结论未经实测证实不要写死进文档。

## 工作流
18. 每个功能一次"改→编译→烧→看日志"循环，不攒批；串口日志 + 小步快跑是嵌入式 AI 开发的生命线。
19. 外设接线咨询协议：先报器件典型引脚请用户核对丝印 → 确认后才给接线表（含供电电压判定与电平容忍检查）→ 占用登记进项目 PINMAP.md → 再写驱动。
20. 用户的每处更正回写全部相关文档与记忆；给用户 2-3 个测试指令验证 skill 触发。
