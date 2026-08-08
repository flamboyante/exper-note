# K230 SPI v3.4 博客笔记写作计划

## Summary

为 QEMU Camp 项目阶段产物——K230 SPI v3.4 patch 系列(11 个补丁,新增 K230 DesignWare SSI 控制器及机器集成,使 U-Boot 能从 SPI NOR 启动 K230)——撰写一篇 6–8 千字的中等技术深度博客。

**风格定位**:口语化、真实、不 AI 化,允许"我"视角叙述,要让老师看到真实的学习过程和工程经历,不弄虚作假。代码块、命令、寄存器表都要有,符合高标准技术博客。

**发布定位**:偏向公共平台(知乎/掘金类)+ Camp 结项展示双重定位,但**上游 review 部分克制**(用户上游交互本身不多),重点放在自己的学习过程和实现思路上。

**DW SSI 拆分议题**:只讲 v1 现状 + 点到上游 review 反馈"需要拆分"为止,**不写死 v2 目标结构**(QOM 继承链、属性表、目标文件布局都不写),给 v2 留自由度。

---

## Current State Analysis(已有素材盘点)

经过探索,写博客需要的素材在仓库里已经齐全:

### Patch 本体
- `k230-spiv3.4-patches/0000~0011.patch` — 12 个补丁文件,cover letter + 11 个实现补丁
- 总改动 14 文件、+3191 / -16 行,主实现 `hw/ssi/k230_dw_ssi.c` 1754 行,qtest `tests/qtest/k230-dw-ssi-test.c` 917 行

### 已有的写作素材(直接可用)
- `exper-note/k230/upstream-review/current/k230-spiv3.4-cover-letter.md` 和 `-cn.md` — 中英双语 cover letter,可摘录项目目标
- `exper-note/k230/upstream-review/current/k230-spiv3.4-commit-messages-bilingual.md` — 11 个 commit 的中英对照 + v3.4 相对 v3.3 的正文变化对比表
- `exper-note/k230/spi/k230-spi-qspi-final-report.md` — 交付范围、能力清单、qtest 收敛过程
- `exper-note/k230/spi/k230-spi-qspi-flash-window-study-plan.md` — 10-Patch 实施与 qtest Study,包含 RED 验证发现的 bug 历史
- `exper-note/k230/spi/k230-spi-qspi-register-audit.md` — 寄存器审计疑点(3 项待证据升级)
- `exper-note/k230/spi/k230-spi-flash-uboot-linux-quickstart.md` — U-Boot 启动 Linux 最小复现,含 Flash 布局、QEMU 命令、U-Boot 命令、实际运行日志(`c0000000: 56190527`)
- `exper-note/k230/spi/learning/k230-spi-patch3-learning-workbook.md` ~ `k230-spi-patch8-learning-workbook.md` — 各 patch 的学习手册,记录了不变量、踩坑、判断题
- `exper-note/k230/spi/learning/k230-spi-idma-patch-learning-workbook.md` — IDMA 四层资料对照(TRM/SDK/QEMU 模型)
- `exper-note/k230/upstream-review/k230-spi-qspi-dwssi-split-analysis.md` — DW SSI 拆分论证(三条证据链)
- `exper-note/k230/upstream-review/upstream-mail-log.md` — 上游邮件日志(Bin Meng、Alistair Francis 的反馈)

### 关键 patch 故事点(已从素材中提取)
- **commit 1**(寄存器模型):TRM 自承 Synopsys IP,VERSION 复位值 `0x3130332a` 对应字符串 `1.03*`
- **commit 3**(PIO/FIFO):TX FIFO 满丢弃、DFS 截断、禁用后 FIFO 保留、DR0..DR35 别名共享同一 FIFO
- **commit 6**(QSPI):Dual/Quad SDR 的指令/地址/mode/dummy/data 五阶段
- **commit 7**(挂 SPI NOR):num-cs 在 SDK 内部冲突(U-Boot DTS `1/5/5` vs Linux DTS `1/1/1`),按启动路径选 `1/5/5`
- **commit 8**(IDMA):同步实现的论证——SDK 驱动轮询 DONE,异步 DMA 没必要;只支持 8 位 SDR Dual/Quad 与驱动实际使用一致
- **commit 10**(XIP):128 MiB spi0 XIP MMIO region,受 bit 0 门控
- **commit 11**(trace):排除 DR 的论证——每帧都过 DR,日志会被淹没

### 踩坑痕迹(PLAN.md / study-plan / mail-log)
- RED 验证发现的:补丁 2 enabled-write 测试失败、补丁 3 PIO 数据错误、补丁 6 构建失败(`EEPROM_READ` 多余大括号)、补丁 7 破坏 `send_frame()`
- TRM URL 返回 404,需要改成 `/K230/en-us/...` 链接
- num-cs SDK 内部冲突
- 版本号谜题:`snps,dwc-ssi-1.01a` 兼容串 vs `0x3130332a`(版本字符串 `1.03*`)对应不上

---

## Proposed Changes

### 一、新建博客文件

**路径**:`exper-note/k230/spi/k230-spi-v3.4-blog.md`

**命名理由**:遵循 `exper-note/k230/README.md` 的 `k230-主题-用途.md` 规范,与同目录的 `k230-spi-qspi-final-report.md`、`k230-spi-flash-uboot-linux-quickstart.md` 并列,定位为"对外可发布的博客稿"。

**不修改其他文件**,只在博客里以相对链接引用现有笔记和源码。

### 二、博客结构与字数分配(目标 6800–7500 字)

```
┌─ 1. 引子:为什么有这篇博客(500字)
├─ 2. K230 和它的 SPI 控制器(700字)
├─ 3. v3.4 patch 系列总览(500字)
├─ 4. 关键 patch 深入(3500字)
│   ├─ 4.1 commit 1:寄存器模型怎么搭(700字)
│   ├─ 4.2 commit 3:PIO 数据路径和 FIFO(700字)
│   ├─ 4.3 commit 6:QSPI 增强传输(700字)
│   ├─ 4.4 commit 8:IDMA 同步实现的故事(700字)
│   └─ 4.5 commit 10:XIP 读窗口(700字)
├─ 5. 踩过的坑和怎么爬出来的(900字)
├─ 6. 上游 review 和 DW SSI 拆分议题(600字)
├─ 7. 实机验证:U-Boot 真的能启动 Linux(500字)
└─ 8. 结尾:这次 Camp 我学到了什么(500字)
```

### 三、各章节写作要点

#### 1. 引子:为什么有这篇博客(500字)
- 一句话交代 QEMU Camp 是什么(开源贡献训练营性质,有 mentor 带)
- 我选 K230 SPI 这个题目的原因:K230 已经有人建了 C908 内核、CLINT、PLIC、UART,但缺真实启动路径,我补上 SPI 这块就能让 U-Boot 真的从 SPI NOR 把 Linux 拉起来
- 这篇博客要讲什么:11 个 patch 怎么从零搭起来、踩了什么坑、上游怎么 review、最后怎么收尾
- **风格示范**:口语,允许"我当时想……""结果一跑就炸"这种表达,不堆砌形容词

#### 2. K230 和它的 SPI 控制器(700字)
- K230 是 RISC-V SoC,C908 内核,本系列前已有 C908/CLINT/PLIC/WDG/UART
- 三个 SSI 实例(按 SDK 地址映射编号):
  - spi0 @ `0x91584000`,SPI-OPI(支持 8 线),num-cs=1,有 XIP
  - spi1 @ `0x91582000`,QSPI0,num-cs=5
  - spi2 @ `0x91583000`,QSPI1,num-cs=5
- 关键寄存器块地址表(用 markdown 表格)
- 必须配的命令块:QEMU 启动 K230 机器的命令

```bash
qemu-system-riscv64 -M k230 -nographic \
  -bios <opensbi> -kernel <uboot> \
  -drive if=none,file=flash.bin,format=raw,id=flash0 \
  -device loader,file=flash.bin,addr=0xc0000000
```

(具体命令以 `k230-spi-flash-uboot-linux-quickstart.md` 里的真实命令为准,不编造)

#### 3. v3.4 patch 系列总览(500字)
- 11 个 patch 一张表搞定,每行:commit hash、标题、一句话职责、改动行数
- 强调分 patch 的逻辑:从寄存器壳 → 实例化 → PIO → IRQ → PLIC 路由 → QSPI → 挂 Flash → IDMA → HI_SYS → XIP → trace,每一步都能独立编译独立测试
- 这套拆分思路本身是上游要求的(Alistair 在 IOMUX 上提过 split 设备/接线/测试)

#### 4. 关键 patch 深入(3500字,每子节 700字)

每个子节固定结构:**这一步要做什么 → 我怎么实现的 → 关键代码片段(伪代码或真实 C 片段)→ 有什么有意思的细节**

##### 4.1 commit 1:寄存器模型怎么搭
- 目标:把 K230 DW SSI 的寄存器空间(CTRLR0/1、SSIENR、SER、BAUDR、FIFO 阈值、SR、IMR/ISR/RISISR、ICR、IDR、VERSION、DR0..DR35)映射到 QEMU MemoryRegion
- 实现:QOM 类型 `TYPE_K230_DW_SSI`,sysbus 设备,`ssi-bus` 子对象,MemoryRegion ops 实现 read/write
- 伪代码片段:QOM 类型声明 + realize 函数骨架
- 有意思的细节:IDR 复位值 `0xa1b2c3d5`、VERSION `0x3130332a` 对应字符串 `1.03*`,TRM 自己写"Contains the hex representation of the Synopsys component version"——这是后面 DW SSI 拆分论证的伏笔

##### 4.2 commit 3:PIO 数据路径和 FIFO
- 目标:DR 写入 → TX FIFO → TMOD transfer pump → RX FIFO → DR 读取
- 实现:Fifo32 256 项,DR0..DR35 别名共享同一个 TX FIFO,四种 TMOD 模式
- 关键不变量:`0 <= TXFLR <= 256`、FIFO 存帧不存字节、SSIENR=0 时 DR 写入被拒
- 伪代码:`k230_dw_ssi_push_tx()` 和 `k230_dw_ssi_pop_rx()` 的核心逻辑
- 命令块:运行 pio 测试
```bash
tests/qtest/run-k230-dw-ssi-tests.sh "pio/dr-aliases"
```

##### 4.3 commit 6:QSPI 增强传输
- 目标:Dual/Quad SDR 的指令/地址/mode/dummy/data 五阶段事务
- 实现:`SPI_CTRLR0` 字段解码,按字段分阶段喂给 SSI 总线
- 寄存器字段表:`SPI_CTRLR0` 的 INST_L/ADDR_L/MODE_BITS/DATA_WIDTH 等
- 伪代码:五阶段事务的状态机骨架
- 细节:spi0 虽然在 TRM 12.3 描述里支持 8 线 OPI,但当前模型保留 8 线 profile 却**拒绝 Octal 数据事务**(因为不在启动路径上,且 SDK 启动软件没用)——这是一个"做减法"的决策

##### 4.4 commit 8:IDMA 同步实现的故事
- 目标:K230 的 internal DMA(SPIDR/SPIAR/AXIAR0/AXIAR1/DONE/AXIE 一组寄存器)
- **核心故事**:为什么不实现异步 DMA?
  - 看 SDK:U-Boot `designware_spi.c` 和 RT-Smart `drv_spi.c` 都是**轮询 DONE**,等 DMA 完成才继续
  - 看主线 QEMU 两种实现方式:Versal OSPI+CSU DMA 的 stream 连接 vs Aspeed SMC 的内部 AddressSpace
  - 既然软件是轮询的,QEMU 同步模型完全够用,异步反而增加复杂度
- 只支持 8 位 SDR Dual/Quad,与驱动实际使用一致
- 伪代码:`k230_dw_ssi_idma_run()` 的同步搬运骨架
- 这是个很好的"设计权衡"案例,可以独立成段

##### 4.5 commit 10:XIP 读窗口
- 目标:spi0 上挂一个 128 MiB 的 MMIO region @ `0xc0000000`,让 CPU 直接当内存读 SPI NOR
- 实现:MemoryRegion subregion,根据 `SPI_CTRLR0` 当前的 DATA_WIDTH 选 Standard/Dual/Quad SDR 读命令,翻译成 SPI 事务发给 m25p80
- 命令块:U-Boot 下验证 XIP
```
=> md 0xc0000000 4
c0000000: 56190527 ...
```
- 细节:XIP 受 bit 0 门控,关闭时读返回 0

#### 5. 踩过的坑和怎么爬出来的(900字)
这一章是最能体现"真实学习经历"的,从 PLAN.md 和 study-plan 里挑 4–5 个具体案例:

1. **补丁 6 构建失败**:`EEPROM_READ` 多余大括号,LLVM 报错,gcc 不报——发现 Clang/gcc 严格性差异
2. **补丁 7 破坏 `send_frame()`**:挂 SPI NOR 时改动影响到了原本的 PIO 路径,RED 测试发现回退
3. **num-cs 的 SDK 内部冲突**:U-Boot DTS `1/5/5` vs Linux DTS `1/1/1`,怎么决策——按当前启动软件路径选 `1/5/5`,并在上游说明里保留证据差异,不写成 TRM 唯一结论
4. **TRM URL 404**:文档链接访问不通,需要改成 `/K230/en-us/...` 路径
5. **版本号谜题**:`snps,dwc-ssi-1.01a` 兼容串 vs `0x3130332a`(对应字符串 `1.03*`)对不上,作为悬而未决的疑点保留

每条都按"现象 → 我怎么定位 → 怎么修/怎么决策"三段写,允许"调了一晚上"这种真实表达。

#### 6. 上游 review 和 DW SSI 拆分议题(600字)
**克制原则**:用户上游交互不多,这一章只讲两件事:

1. **Bin Meng 在 v3.2 上的反馈**:要求把模型拆成"通用 Synopsys DW SSI"+"可选 K230 wrapper"两层
2. **我为什么没在 v3.4 拆**:三条证据链说明确实该拆(TRM 自承 Synopsys IP、U-Boot driver 是通用 driver、QEMU 已有 `designware_i2c.c` 先例),但 v3.4 重点是收敛启动路径,拆分留给 v2

**关键**:不写 v2 的目标文件布局、不画 QOM 继承链、不列属性表。只说"v2 计划把通用层摘出来",给读者一个"作者知道下一步该干什么"的印象,但不把未来结构写死。

#### 7. 实机验证:U-Boot 真的能启动 Linux(500字)
- 三条启动路径的最小复现命令(从 `k230-spi-flash-uboot-linux-quickstart.md` 摘):
  - 标准 SPI:`sf read` 加载 OpenSBI/Linux/initrd/DTB → Linux initramfs shell
  - Quad SPI:spi0 配 4 位,U-Boot 擦/写/读 256 字节 + 加载启动载荷 → Linux initramfs shell
  - XIP:从 `0xc0000000` 读 OpenSBI uImage 头、验校验和、bootm → Linux initramfs shell
- 配一段真实的 U-Boot 启动日志(从 quickstart 里摘)
- `git diff --check` 无空白错误,`checkpatch` 无错误

#### 8. 结尾:这次 Camp 我学到了什么(500字)
**真实的学习收获**,不堆砌:
- QEMU 的 QOM 设备建模、MemoryRegion、SSI 总线抽象
- qtest 怎么写、TAP 输出怎么看、RED/GREEN 怎么用
- 上游协作:cover letter 怎么写、commit message 怎么拆、review 反馈怎么回应
- RISC-V SoC 的启动路径全貌:从 SPI NOR → U-Boot → OpenSBI → Linux
- 一个反思:第一版把所有东西塞进 `k230_dw_ssi.c` 是为了快速跑通,但代价是 v2 要重构——下次会一开始就分层

### 四、风格守则(贯穿全文)

1. **第一人称"我"**,允许口语:"我当时想……""结果一跑就炸""调了一晚上才反应过来"
2. **禁止 AI 化表达**:不写"综上所述""总而言之""值得注意的是""让我们深入探讨""本文将……"这种套话
3. **代码块必须有语言标签**:C 代码用 ```c,bash 用 ```bash,寄存器表用 markdown 表格
4. **真实命令**:所有 QEMU 命令、qtest 命令、U-Boot 命令必须从 `k230-spi-flash-uboot-linux-quickstart.md` 等真实笔记里摘,不编造
5. **伪代码标注**:用 ```c 但顶部注释标 `/* 伪代码:展示核心逻辑,省略错误处理 */`
6. **引用源码用相对链接**:按 `exper-note/k230/README.md` 规范,从博客文件出发的相对链接,保证 Markdown 预览可跳转
7. **DW SSI 拆分章节**不写 v2 目标结构,只说"v2 计划拆分"
8. **心路历程**写真实的工程决策过程(RED→定位→修复),允许情绪化表达("那天晚上调到3点")但不夸大

### 五、不写的内容(明确排除)

- 不写 v2 的目标文件布局(`hw/ssi/dw_ssi.c` / `k230_dw_ssi.c` 拆分结构)
- 不画 QOM 继承链图(`TYPE_K230_DW_SSI` → `TYPE_DW_SSI` → `SYS_BUS_DEVICE`)
- 不列通用层 QOM 属性表(num-cs/max-lines/fifo-depth/internal-dma/xip-enabled 等)
- 不列寄存器归属表(哪些归通用层、哪些归 K230 wrapper)
- 不写完整的上游邮件往来(用户上游互动不多,克制处理)
- 不写 IOMUX 相关内容(本系列只讲 SPI)

---

## Assumptions & Decisions

### 假设
1. 博客文件放在 `exper-note/k230/spi/k230-spi-v3.4-blog.md`,与同目录其他笔记并列
2. 字数目标 6800–7500 字(中文,代码块不计入字数)
3. 关键 patch 选择:commit 1/3/6/8/10 五个深入展开,其余 patch 在总览表里一笔带过
4. 实机验证章节包含三条启动路径,但每条只给最小命令+一行结果,不展开日志
5. 上游 review 章节只讲 Bin Meng 的拆分反馈一处,不铺开历史

### 决策
- **DW SSI 拆分章节**:讲 v1 现状 + 点到"需要拆分"为止,不写 v2 目标结构(用户明确要求)
- **风格**:口语化、真实、允许"我"视角,不 AI 化(用户明确要求)
- **篇幅**:6–8 千字中等长文,只展开关键 patch(用户明确要求)
- **平台定位**:公共平台 + Camp 结项双重,上游部分克制(用户明确要求)
- **commit 11(trace 排除 DR)**:不单独成章,作为 commit 1 或踩坑章节里的细节穿插
- **commit 7(挂 SPI NOR + num-cs 冲突)**:不单独成章,把 num-cs 冲突放进第 5 章"踩过的坑"

### 风险与处理
- **风险**:博客写完可能超字数。**处理**:优先压缩第 2 章背景和第 7 章实机验证,关键 patch 章节不压缩
- **风险**:口语化和"高标准博客"的平衡。**处理**:口语只用在过渡和心路历程,技术段落保持准确
- **风险**:DW SSI 拆分写得太浅显得没思考。**处理**:把三条证据链(TRM 自承 Synopsys IP / U-Boot driver 是通用 driver / QEMU 已有 designware_i2c 先例)简短列出,证明"我知道为什么该拆",只是选择不在 v1 拆

---

## Verification Steps

博客写完后,按以下清单验证:

1. **字数检查**:用 `wc -m` 或编辑器统计,确认 6800–7500 字
2. **代码块语言标签**:grep 所有 ``` 围栏,确认都有语言标签(c/bash/text)
3. **相对链接可跳转**:在 Markdown 预览里点几个源码链接,确认能跳到对应文件
4. **命令真实性**:对照 `k230-spi-flash-uboot-linux-quickstart.md` 检查所有 QEMU/qtest/U-Boot 命令,确认没有编造
5. **AI 化表达扫描**:搜索"综上所述""总而言之""值得注意的是""让我们""本文将""首先……其次……最后",确认没有
6. **DW SSI 拆分边界**:确认第 6 章没有出现 `hw/ssi/dw_ssi.c`、`TYPE_DW_SSI`、QOM 继承链、属性表等 v2 目标结构
7. **commit 覆盖度**:确认 11 个 patch 在总览表里都有,关键 patch(1/3/6/8/10)有独立小节
8. **口语化样本**:随机读 3 段过渡文字,确认读起来像真人写的,不像 AI 生成
