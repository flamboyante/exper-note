# K230 功能型冷启动验证：集成与执行计划（v3）

> 本计划替代 v1。v1 用 `-bios u-boot` 验证了 U-Boot 从 SPI Flash 读 Linux 的路径，
> 该路径已在 `exper-note/k230/spi/k230-spi-flash-uboot-linux-quickstart.md` 记录，
> 不是本计划目标。
>
> 本计划目标是 gap 文档 §6.1 路线 A：**CPU reset → functional BootROM → SPL →
> 完整 U-Boot → OpenSBI/Linux**，从同一张 SPI Flash 镜像逐级加载。
>
> v3 根据 2026-08-05 本地分支状态修订：外设模块和 GSDMA/decomp-gzip 已集成；
> 当前 HEAD 的冷启动入口是 host-assisted SPL loader，只是隔离 BootROM 的阶段性台阶，
> 尚未达到通过 Guest DW SSI 寄存器读取 Flash 的 functional BootROM 目标。

## 目标定义

### 功能型冷启动路径

```
CPU reset
  -> QEMU functional BootROM（从 Flash 0x0 读 SPL，经 SPI0 控制器）
  -> SPL（在 SRAM 中执行，初始化 DDR/PWR，从 Flash 加载完整 U-Boot）
  -> 完整 U-Boot（在 DDR 中执行，sf read 加载 OpenSBI/Linux）
  -> OpenSBI
  -> Linux + initramfs
```

### 不是本计划目标

- `-bios u-boot` 直接加载完整 U-Boot（已验证，见 quickstart 文档）
- 真实 BootROM binary 等价（无公开 binary，只做 functional BootROM）
- QSPI/Enhanced SPI/IDMA/XIP（Standard 1-1-1 足够冷启动）

## 当前状态

### 分支与基线

| 项 | 值 |
|---|---|
| 验证分支 | `k230-coldboot-validation`（基于 `T_v2-5patch`） |
| 当前 HEAD | `7730876ca5 feat: full-boot and ctrl0 reg` |
| 已集成模块 | 模块 2（DW SSI v2）、8（SRAM）、9（RMU）、10（DDR）、7（UART）、3（Timer）、11（IOMUX）、5（GSDMA + Decomp Gzip） |
| QEMU 二进制 | `my-qemu-camp-2026-k230/build/qemu-system-riscv64`（已编译） |
| 已验证路径 | `-bios u-boot` → sf read → Linux（v1 误做，非本计划目标） |
| 当前冷启动入口 | host 通过 `blk_pread()` 读取 Flash 0、还原 SPL 并写入 `0x80300000`，随后 reset stub 跳转 |

### v1 合并结果对真实冷启动的价值评估

| 模块 | v1 已合并 | 真实冷启动价值 | 处理 |
|---|---|---|---|
| 2 DW SSI v2 | 是 | **必需**：SPL 和 U-Boot 都通过 SPI0 读 Flash | 保留 |
| 8 SRAM | 是 | **必需**：SRAM 窗口从 `0x80200000` 开始，SPL 入口为 `0x80300000` | 保留 |
| 9 RMU | 是 | 可选：复位控制，SPL 阶段非硬阻塞 | 保留 |
| 10 DDR | 是 | **必需**：cover letter 已验证 SPL 能通过 DDR 初始化 | 保留，无需额外改动 |
| 5 GSDMA + Decomp Gzip | 是 | SDK 默认私有 gzip 的候选后端；已有 qtest，真实 SPL/U-Boot 组合待验证 | 保留并做 Guest 组合验证 |
| 7 UART | 是 | 非必需：QEMU 8250 通用模型已够 SPL/U-Boot 控制台（用户确认） | 可保留不删，但后续不依赖 |
| 3 Timer | 是 | 可选：SPL 阶段非硬阻塞，v2 未满足上游 review | 保留不删，低优先级 |
| 11 IOMUX | 是 | 可选：pinctrl 驱动可能需要，SPL 阶段非硬阻塞 | 保留不删，低优先级 |

### 已完成的 GSDMA 集成

| 模块 | 补丁数 | lore 标识 | 冷启动用途 |
|---|---|---|---|
| 5 GSDMA + Decomp Gzip | 7 | `20260727155702.36484-1-dingtao0430@163.com` | 已集成；独立/协同 qtest 不等于默认镜像端到端已通过 |

## Gap 矩阵修正（基于新证据）

### 原始 gap 矩阵（来自 gap 文档 §5）

| Gap | 原阻塞级别 | 修正后 | 依据 |
|---|---|---|---|
| BootROM 不从 Flash 加载 SPL | 硬阻塞 | **仍硬阻塞** | QEMU 需实现 functional BootROM stub |
| DDR training 越界 `0x9a04017c` | 硬阻塞 | **候选模型已有解决证据** | DDR patch 作者验证 SPL 能通过；本地组合仍需复验 |
| DDR-ready bit `0x980001bc` | 硬阻塞 | **正常路线不需要** | 优先验证 DDRC/PHY 完整握手；ready bit 仅作故障隔离备选 |
| CanMV 固定 SDIO1 | 硬阻塞 | **仍硬阻塞** | 需改用 EVB 或最小修改 CanMV boot mode |
| CanMV SPL DT 禁用 SPI0 | 硬阻塞 | **仍硬阻塞** | 需改用 EVB 或启用 SPI0 |
| CanMV SPL DT 缺 spi-flash@0 | 硬阻塞 | **仍硬阻塞** | 需增加 CS0 SPI NOR child |
| PWR 状态位恒为 0 | 性能阻塞 | 非阻塞，低优先级 | 有限忙等 1M 次循环后自动退出，不阻塞 SPL |
| PUFS/OTP 非安全镜像仍读 | 后续硬阻塞 | **仍硬阻塞** | 需最小 PUFS 模型或软件 SHA |
| 私有 gzip（k230_gzip） | 后续硬阻塞 | **可规避；候选模型已集成** | 先用 IH_COMP_NONE 隔离，再验证 GSDMA/decomp-gzip 真实 Guest 路径 |

### 新证据详情（2026-08-04）

DDR patch（模块 10）的 cover letter 明确写道：

> Booted the K230 SDK U-Boot SPL through DDR initialization to its SPI boot path

**含义**：
- DDR patch 作者已用 SDK SPL 做过阶段性验证：SPL → DDRC/PHY 初始化和训练握手完成 → 到达 SPI boot path
- gap 文档 §4.1 的“DDR training 越界 `0x9a04017c` 硬阻塞”是 **DDR patch 之前** 的判断，现已过时
- gap 文档 §5.2 的“DDR-ready bit `0x980001bc bit0=1`”不应作为正常路线——优先使用 DDR patch 的完整 DDRC/PHY 握手；仅在定位问题时把 ready bit 当作临时隔离手段
- 之前检查 `k230_ddrc_reset_values` 时看到 DFISTAT 默认为 0，但 DDR 模型的运行时行为（`k230_ddr_cfg_update_state()` + PHY `dfi_init_complete` 握手）能让 DFI_INIT_COMPLETE 正确置位

**结论**：DDR 模块已经集成，当前不预设额外 DDR hack；必须先在本地组合分支复验作者
声称的 SPL 路径，再决定是否需要修正模型。

## 实施路线（基于 gap 文档 §7）

### Phase 0：集成 GSDMA 模块（已完成）

模块 5（GSDMA + Decomp Gzip）已经合入 `k230-coldboot-validation` 分支。以下命令仅保留
为来源记录，不应在当前分支重复执行：

```bash
cd /home/flamboy/qemu-camp/my-qemu-camp-2026-k230
git checkout k230-coldboot-validation
curl -sL "https://lore.kernel.org/qemu-devel/20260727155702.36484-1-dingtao0430@163.com/t.mbox.gz" \
  | gunzip | git am --3way
```

集成时已处理 `hw/riscv/k230.c` 中 GSDMA/decomp-gzip/noc-stub 实例化及 qtest 列表冲突。

**剩余验证**：不能只以编译和 qtest 为完成标准；Phase 4 还要证明真实 Guest
`k230_priv_unzip()` 能完成默认 U-Boot/Linux payload 解压。

### Phase 0.5：host-assisted SPL loader（已实现，仅作隔离台阶）

当前 HEAD `7730876ca5` 在 QEMU machine 侧执行：

```text
blk_pread(Flash offset 0)
  -> host 侧 32-bit byte swap
  -> 检查 K230 magic/length
  -> address_space_write(SPL, 0x80300000)
  -> reset stub 跳转 SPL
```

该实现可用于验证 SPL 后半段，但绕过了 Guest BootROM、DW SSI 寄存器、SSI bus 和 NOR
read command，因此不得标记为 functional BootROM 完成。最终 Phase 5 必须替换这一入口。

### Phase 1：定板型（SDK 侧准备）

gap 文档 §4.6 提到 EVB 配置原生启用 SPI0 + spi-flash@0 + 读 BOOT strap，但 EVB SPL
构建未完成（工具链 `-march` 问题）。CanMV 需改 boot mode 和 DT。

**策略**：两者都试，先看哪个能构建出 SPL。

#### 1a. 尝试 EVB defconfig

```bash
cd /home/flamboy/qemu-camp/k230_sdk
# SDK 根目录配置必须通过 CONF 传入
make CONF=k230_evb_defconfig
# 解决工具链 -march=rv64imac_zicsr_zifencei 问题后构建
make CONF=k230_evb_defconfig -j$(nproc)
# 检查产物：spl/u-boot-spl
```

**成功条件**：生成可运行的 EVB SPL 二进制，并确认其内嵌 SPL DT 启用 SPI0/CS0。

#### 1b. 尝试 CanMV 最小修改（备选）

若 EVB 构建失败，对 CanMV 做最小修改：

1. `board/canaan/k230_canmv/board.c`：移除 `sysctl_boot_get_boot_mode()` 覆盖，
   使用 weak 实现读 `SOC_BOOT_CTL[1:0]`
2. CanMV DTS：启用 `spi0@91584000` + 添加 `spi-flash@0` 子节点
   （参考 gap 文档 §4.4）

**成功条件**：生成能选 NOR 启动的 CanMV SPL 二进制。

### Phase 2：-bios SPL 打通至完整 U-Boot header 校验入口（QEMU + SDK）

暂不实现 BootROM，用 `-bios SPL` 验证 SPL 能完成 DDR、选择 NOR、枚举 Flash 并读取
`0x80000` 的完整 U-Boot。若停在 PUFS/SHA，属于 Phase 2 的预期终点。

#### 2a. QEMU 侧：先复验现有 DDR 模型，PWR 暂不作为硬阻塞

**重大新证据（2026-08-04）**：DDR patch（模块 10）的 cover letter 明确写道：

> Booted the K230 SDK U-Boot SPL through DDR initialization to its SPI boot path

即 DDR patch 作者已验证 SPL 能通过 DDR 初始化并到达 SPI boot path。

**对 gap 文档的修正**：
- gap 文档 §4.1 的“DDR training 越界 `0x9a04017c` 硬阻塞”是**DDR patch 之前**的判断，现已过时
- gap 文档 §5.2 的“DDR-ready bit `0x980001bc bit0=1`”不作为正常实现；只有在隔离 DDR 问题时临时使用
- 之前检查 `k230_ddrc_reset_values` 时看到 DFISTAT 默认为 0，但 DDR 模型的运行时行为（`k230_ddr_cfg_update_state()` + PHY `dfi_init_complete` 握手）能让 DFI_INIT_COMPLETE 正确置位

**结论**：先用本地 SPL 复验 DDRC/PHY 握手。作者测试是强证据，但不能替代当前模块组合、
板型和 SPL 产物的本地验证。

**PWR 状态位（低优先级，非阻塞）**：

SPL 的 device_disable()（k230_spl.c:96）会检查 4 个 PWR 状态寄存器的 bit0，循环 1,000,000 次。

**为什么不阻塞**：
1. 前置 if (readl(0x9110302c) & 0x2) 检查 bit1（电源是否开启），QEMU 返回 0，if 条件不成立，不发起关电源操作
2. 进入 while 循环等 bit0（power-off complete），由于没发起关电源，且 QEMU 始终返回 0，bit0 永远不置位
3. 循环执行 1,000,000 次后 value 减到 0，自动退出循环
4. 继续执行后续 CMU 时钟关闭和 DDR 初始化

**实际影响**：循环次数有限，所以不是永久阻塞；实际耗时和日志量取决于 TCG、trace/
`-d unimp` 配置，不能固定写成几毫秒。

**处理策略**：第一轮不处理；若它显著拖慢测试或淹没日志，再补最小 PWR complete 状态。

#### 2b. SDK 侧：打包未压缩 U-Boot

将完整 U-Boot legacy uImage 打包为 `IH_COMP_NONE`：
```bash
# SDK 构建时使用 -C none，规避 k230_priv_gzip 路径
# GSDMA/decomp-gzip 已集成，但默认 gzip 必须在 Phase 4 单独做 Guest 端到端验证
```

#### 2c. 运行验证

```bash
QEMU=./build/qemu-system-riscv64
SPL=<SDK 构建的 SPL>
FLASH=<含 SPL + 完整 U-Boot + OpenSBI + Linux 的 Flash 镜像>

$QEMU -M k230,spi-flash=w25q256 \
  -bios "$SPL" \
  -drive if=mtd,format=raw,file="$FLASH" \
  -nographic -no-reboot
```

**验证检查点**：
1. SPL 启动，DDRC/PHY training 和 DFI 握手完成
2. SPL 出现 1M 次 PWR 忙等（非阻塞，知道是正常现象即可）
3. SPL 选择 SPI NOR 启动模式
4. SPL 从 Flash 读完整 U-Boot package
5. 到达 firmware header/PUFS 校验入口；本阶段不要求出现 `K230#`

### Phase 3：处理 firmware header 校验

SPL 加载完整 U-Boot 时会校验 K230 firmware header（gap 文档 §5.4）：

1. `cb_pufs_read_otp()`：判断是否允许非安全启动
2. `cb_pufs_hash()`：SHA-256 payload，与 header hash 比较

**最小方案**（二选一）：
- **QEMU 最小 PUFS 模型**：实现 OTP 读取 + SHA-256 兼容
- **SDK 软件 SHA**：取消 `CONFIG_K230_PUFS`，用 `sha256_csum_wd()` 软件分支

**验证**：SPL 能校验、解包并跳转完整 U-Boot，最终出现 `K230#`。

### Phase 4：打通 SPL → 完整 U-Boot → QEMU 适配 Linux

在 Phase 2/3 基础上，让完整 U-Boot 从 Flash 读 OpenSBI/Linux 并 bootm：

```bash
# 这是阶段验证用的 custom raw payload 布局，不是 SDK linux_system.bin 量产布局：
# 0x00000000  SPL
# 0x00080000  完整 U-Boot（IH_COMP_NONE）
# 0x00fc0000  OpenSBI uImage
# 0x01000000  Linux Image
# 0x01c00000  rootfs
# 0x01f00000  DTB
```

**验证**：`-bios SPL` → SPL → U-Boot → sf read → bootm → Linux initramfs shell。
成功后再决定是否把这些 payload 按 SDK `linux_system.bin` 内部格式重新封装。

### Phase 5：实现 functional BootROM

最后实现 reset-to-Linux 功能闭环，替换 Phase 0.5 的 host-assisted loader。在
`0x91200000` 放置 functional BootROM stub：

1. 从 BOOT 配置确定 SPI NOR
2. 通过 SPI0/DW SSI 寄存器从 Flash 偏移 `0x0` 读取 SPL
3. 按当前已知布局识别 K230 528 字节 firmware header
4. 处理 `swap_fn_u-boot-spl.bin` 字节交换格式（具体真实 ROM 算法仍是功能型假设）
5. 非安全 SHA-256 校验，或明确把校验责任留给 SPL（需记录边界）
6. 把 SPL payload 放到 `0x80300000` 并跳转

**关键**：最终 BootROM stub 必须通过 Guest 可见的 SPI0/DW SSI 寄存器访问 Flash，不能
继续使用 host-side `blk_pread()`。这样才验证 BootROM → SPI controller → Flash 完整路径。

**验证**：`-drive if=mtd` → CPU reset → BootROM → SPL → U-Boot → Linux。
不再需要 `-bios`。

### Phase 6（可选）：SDK 默认镜像原样启动

若需要 SDK v2.0 默认压缩镜像原样启动，需完整建模：
- PUFS OTP + 内部 DMA + HMAC/SHA-256
- GSDMA + decomp_gzip（Phase 0 已合并模块 5）
- RMU 完整状态语义
- PWR 独立模块（如需补 power-off complete 状态位）
- DDRC/PHY training 完整模型

**决策点**：若 Phase 0-5 已满足"功能型完整冷启动"目标，Phase 6 可不做。

## 阶段验证镜像布局

```
偏移          内容                      备注
0x00000000    SPL                       swap_fn_u-boot-spl.bin（字节交换格式）
0x00080000    完整 U-Boot               fn_ug_u-boot.bin（IH_COMP_NONE 或默认 gzip）
0x00fc0000    OpenSBI uImage            fw_jump.uImage（custom raw payload）
0x01000000    Linux Image               IH_COMP_NONE
0x01c00000    rootfs.cpio.gz
0x01f00000    DTB                       启用 SPI0 + spi-flash@0
```

SDK 官方 SPI-NOR 镜像则是：

```text
0x000000  swap_fn_u-boot-spl.bin
0x080000  fn_ug_u-boot.bin
0x1e0000  environment
0xfc0000  linux_system.bin
```

两种布局不能混称；Phase 4 使用 custom raw 布局是为了隔离加载问题，Phase 6 才处理
SDK 默认 package 的原样兼容。

## 假设与决策

1. **假设**：Standard 1-1-1 SPI NOR 访问足够 SPL 和 U-Boot 冷启动（V1 已验证）
2. **决策**：不合入模块 6（SPI v1 完整版），与模块 2 冲突
3. **决策**：UART 已随旧验证分支集成，但冷启动不新增依赖，通用控制台能力已足够
4. **决策**：模块 5 已集成；是否支持 SDK 默认 gzip 以 Guest 端到端结果为准
5. **决策**：DDR 优先采用 patch 完整握手模型；DDR-ready bit 只作故障隔离，不进入正常路线
6. **决策**：BootROM 为 functional stub，非真实 ROM binary 等价（gap 文档 §5.1）
7. **决策**：优先尝试 EVB defconfig，失败则最小修改 CanMV（用户确认）
8. **决策**：验证分支 `k230-coldboot-validation` 仅用于本地组合验证

## 回滚策略

本计划不预置 `git reset --hard` 或强制删分支。需要对照旧阶段时，保留当前验证分支并使用
新 worktree：

```bash
# 先保存当前验证头，再切回 V2 终态
git branch coldboot-validation-backup k230-coldboot-validation
git switch T_v2-5patch

# 对照 GSDMA 前状态，不修改现有分支
git worktree add /tmp/k230-coldboot-pre-gsdma <Phase-0-before-commit>
```

任何强制回退或删除操作都应在确认没有未提交修改、且备份引用存在后单独执行，不作为普通
验证步骤。

## 参考文档

- 稳定知识手册：`exper-note/k230/spi/coldboot/k230-cold-boot-handbook.md`
- gap 分析：`exper-note/k230/spi/coldboot/k230-cold-boot-gap-investigation.md`
- v1 验证记录：`exper-note/k230/spi/k230-spi-flash-uboot-linux-quickstart.md`
- 待 review 模块清单：`exper-note/k230-new-modules-guide.md`
- V2 决策入口：`exper-note/k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md`
- DDR patch cover letter：`https://lore.kernel.org/qemu-devel/20260716132423.931427-1-caojunze424@gmail.com/`
