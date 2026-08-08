# K230 冷启动全 canmv 统一验证计划（方案一）v4

> v3 已完成 custom raw Flash 验证：host-assisted loader → SPL → U-Boot，随后由
> host 脚本向 U-Boot shell 注入 `sf read`/`bootm` 命令，最终进入 initramfs shell。
>
> v4 新目标：保持现有 QEMU 和 canmv_qemu 板级改动，切换为 SDK 官方 32 MiB
> SPI-NOR 偏移与 `linux_system.bin` 格式，加入有效 SPI environment，由
> `bootcmd=k230_boot auto auto_boot` 完成 **无人干预自动启动**，最终挂载 Flash
> rootfs。运行脚本只能启动 QEMU、限制超时和收集日志，不得再向串口注入命令。
>
> **三条修订约束（用户确认）：**
> 1. **QEMU 能不改就不改**——QEMU 侧零代码改动，依赖现有行为；
> 2. **U-Boot/SDK 改动拉新分支**管理，可回滚；
> 3. **canmv_qemu.dts 拿不准的设备，先对照 QEMU 已实现外设，未实现的一律 `disable`。**

## 0. v4 新增决策（相对已完成的 v3）

| 项 | v3 已完成基线 | v4 目标 |
|---|---|---|
| Flash 编排 | custom raw：Image/initramfs/fw_jump/DTB 分开放置 | 官方 offset：env `0x1e0000`、Linux `0xfc0000`、rootfs `0x16c0000` |
| OpenSBI/Linux | `fw_jump.uImage` 与 raw `Image` 分离 | SDK 官方 `fw_payload.bin → linux_system.bin` |
| U-Boot 后半段 | host 脚本注入 `sf read`/`bootm` | SPI env 自动执行 `k230_boot auto auto_boot` |
| 根文件系统 | `rootfs.cpio.gz` 加载到 RAM，重启丢失 | SPI NOR 上的 JFFS2 或 UBI/UBIFS，优先最小可验证 rootfs |
| 自动化性质 | 脚本化交互，不是 U-Boot 自主启动 | 从 QEMU 启动到 Linux 用户空间全程零串口输入 |
| QEMU 修改 | 无 | 无 |

**v4 最小改动原则：**

1. 不修改 SDK 原有 `configs/k230_canmv_defconfig`，新增派生顶层配置；
2. 不手改生成文件 `sdk_autoconf.h`，由 SDK 顶层配置生成；
3. 不修改 OpenSBI/Linux 源码，优先复用当前标准 PTE Image 与 SDK 打包工具；
4. 不修改 U-Boot 默认 `CONFIG_BOOTCOMMAND`，通过 SPI environment 覆盖 `bootcmd`；
5. 不删除 MMC environment 后端，NOR boot mode 会动态选择 SPI environment；
6. 保留 SDK 默认 `SPI_NOR_SUPPORT_CFG_PARAM` 时，产品配置分区由 `auto_boot` 读取，首轮必须放 SDK 生成的合法占位包；RT-Smart 分区不执行，但 offset 和分区边界仍保持官方布局；
7. 每项新增或保留改动都必须记录基线、所属层、原因、影响面和回滚方式。

## 0. v3 修订要点（相对 v2）

| # | 修订点 | v2 旧做法 | v3 新做法 |
|---|---|---|---|
| 1 | 工作区管理 | `git stash push -u` 暂存全部 | **不整体 stash**（SDK 有大量未跟踪编译产物 .o/.cmd/Image/DTB）；保留工作区直接随分支走，commit 时**只 add 明确文件** |
| 2 | dts 修改集中 | dtsi 有散落改动 | 所有 dts 改动集中在 `k230_canmv_qemu.dts`（uboot/linux 两侧）；`k230.dtsi`、`k230_evb.dtsi` **已还原** SDK 原始，spi0 使能由 `&spi0` override 承担 |
| 3 | 验证门禁 | GUEST_ERROR 作为门禁 | **GUEST_ERROR 不作严格门禁**（读 unimplemented boot 区返回 0 是方案依据，日志是否显示取决于 QEMU 日志配置）；门禁 = ① SPL 选择 NOR ② SPL 加载完整 U-Boot ③ 无阻断启动异常；boot 区日志仅观察项 |
| 4 | Flash 布局校验 | 按估计偏移直接打包 | **Flash 偏移与实际文件大小在打包阶段逐段校验**（SPL≤0x80000、U-Boot≤0x160000、Image/initramfs/fw_jump 不越界、DTB 按实际 75614B 预留 0x14000） |
| 5 | "同源"表述 | "U-Boot 与 Linux dtb 同源" | **表述修正**：U-Boot 与 Linux 是两个独立 dts 源文件，称"**同一 canmv_qemu 板级语义、设备范围一致**" |

**执行前新事实（本次会话已确认）：**
- linux `k230_canmv_qemu.dts` 已重写完成（QEMU 未建模设备 10 处 disable），dtb 编译成功 = **75614 字节（0x1275E）**，超出布局表原预留 0x1000 → Step 5 布局预留改为 **0x14000**
- `board.c` 已删除 `sysctl_boot_get_boot_mode()` 的 SDIO1 覆盖（替换为 QEMU cold boot 注释）
- 分支 `k230-coldboot-canmv-qemu` 已存在且当前在其上（HEAD = SDK v2.0 `7e302f733`）

---

## 1. 论证过程（讨论记录与结论）

### 1.1 现状 dtb 四源不一致

| dtb | 板级 compatible | 谁在用 |
|---|---|---|
| U-Boot 内嵌（上次 EVB 构建） | `kendryte,k230_evb` | 上次验证 |
| boot-assets 现成 U-Boot | `kendryte,k230_canmv_v2/canmv` | 未用 |
| Linux（上次验证） | `canaan,k230-canmv`（yocto 最小版） | 上次验证 |
| SDK linux 可编 | `k230_evb.dts` / `k230_canmv.dts` | 未编 |

结论：U-Boot 与 Linux dtb 不一类，本次统一到 canmv_qemu（两独立 dts 源文件，同一板级语义）。

### 1.2 三个必改点（均 U-Boot 源码侧）

1. **`CONFIG_K230_PUFS` 注释**：SDK 原始（v1.9/v2.0）`k230_board_common.h` 都开
   `#define CONFIG_K230_PUFS`；QEMU 无 PUFS 硬件模型。注释后落到 `k230_img.c`
   软件 `sha256_csum_wd()` 分支。evb/canmv 共用该头文件。**已改（工作区，待入分支 commit）。**
2. **canmv SPL boot mode**：`board/canaan/k230_canmv/board.c:43`
   `sysctl_boot_get_boot_mode()` 硬编码 `return SYSCTL_BOOT_SDIO1`（MMC）。
   SDK weak 实现（`arch/riscv/cpu/k230/sysctl.c:27`）读 `SOC_BOOT_CTL & 0x3`：
   0=NORFLASH、1=NAND、2=SDIO0、3=SDIO1。canmv 覆盖是真实板的产品决策（TF/SDIO 启动），
   本次验证应尊重 strap 语义——**删除覆盖**，让 weak 实现读 strap。**已改（工作区）。**
3. **dts 启用 spi0 + 1-1-1**：`k230_canmv_qemu.dts`（uboot 侧）已建：
   `spi@91584000` status=okay + `spi-flash@0` + bus-width=1 + `u-boot,dm-pre-reloc`。
   Linux 侧已建：bus-width 8→1 + 分区对齐官方 `genimage-spinor.cfg` + **QEMU 未建模外设 disable**（§3.2）。

### 1.3 方案选择与坑结论

**决策：方案一（全 canmv）**——改动量与 EVB 相当（差一个 boot mode 小改动），
且 Linux dtb 侧 canmv_qemu 已铺路，兑现参考 dts 价值。

- **坑 1（SOC_BOOT_CTL）：**`SOC_BOOT_CTL = SYSCTL_BOOT_BASE_ADDR + 0x40 = 0x91102040`。
  QEMU `K230_DEV_BOOT = 0x91102000` 是 `create_unimplemented_device()`，
  读返回 0 → weak 实现 `case 0: return SYSCTL_BOOT_NORFLASH` → SPI NOR。
  **结论（修订）：QEMU 零改动**，依赖 unimplemented 读 0 的标准行为。
  已知代价：SPL 读该寄存器时可能打一次 GUEST_ERROR 日志（**仅观察项，不入门禁**，不影响启动）。
- **坑 2（canmv_qemu dts 能否起来）：**U-Boot 侧风险中低（canmv 系 U-Boot 曾在 V1
  `-bios` 验证过）；Linux 侧**按 QEMU 外设对照表 disable 未建模设备**（§3.2），
  消除内核 probe 风险。
- **坑 3（复用 SDK 5.10.4）：**`k230-boot-assets/buildroot/uboot-boot/Image` =
  SDK T-HEAD 5.10.4 + Xuantie V2.6.0 编译 + **rebuilt for QEMU PTEs**（MAEE 位已清）。
  已实测跑通；SDK 内核在 `arch/riscv/kernel` 无严格 root compatible 校验（宽松）。
  **结论：复用（零内核编译），如实记录 provenance（无重编 recipe），不伪装成自建。**

### 1.4 与上次验证的区别

| | 上次（EVB 混搭） | 本次（方案一） |
|---|---|---|
| U-Boot/SPL defconfig | `k230_evb` | `k230_canmv_qemu`（新 dts） |
| canmv boot mode | 未处理（回避） | 修正（读 strap） |
| QEMU 改动 | 无 | **无（零改动，依赖 unimplemented 读 0）** |
| Linux dtb | yocto 最小版（不同源） | canmv_qemu（独立源文件，与 U-Boot 同一板级语义，未建模设备 disable） |
| Linux 内核 | 复用 SDK 5.10.4（未记录 provenance） | 复用同版 + **记录 provenance 与局限** |
| SDK 改动管理 | 工作区散放 | **独立分支 `k230-coldboot-canmv-qemu`** |
| Flash 编排 | raw 布局 | v3 仍为 raw 布局；v4 改用官方 offset 与 `linux_system.bin` |

---

## 2. v4 当前基线（截至 2026-08-07）

### 2.1 已验证能力

- v3 已跑通 `host-assisted loader -> SPL -> U-Boot -> OpenSBI -> Linux -> initramfs shell`；
- SPL 从 SPI NOR 的 `0x000000` 冷启动、加载 `0x080000` 完整 U-Boot 已验证；
- `fn_ug_u-boot.bin` 的 gzip 硬件解压路径已实际经过；
- Standard SPI、Linux 标准 PTE Image、canmv_qemu DTB 均有可复用基线；
- v3 后半段依赖 host 向 U-Boot shell 注入命令，因此只作为回归基线，不算 v4 的全自动验收结果。

### 2.2 本轮基线与派生关系

| 层 | v4 基线 | v4 处理 |
|---|---|---|
| QEMU | `k230-coldboot-validation`，现有 host-assisted loader、DW SSI、NOR、gzip 模型 | 零代码修改 |
| SDK 顶层 | `configs/k230_canmv_defconfig` | 新增 `k230_canmv_qemu_spinor_defconfig`，不改原文件 |
| U-Boot | `k230_canmv_qemu_defconfig` + `k230_canmv_qemu.dts` | 保留现有 QEMU 板级改动；由顶层配置选中 |
| Linux | boot-assets 的 SDK 5.10.4 标准 PTE `Image` + `k230_canmv_qemu.dts` | 不改内核源码；必要时只核对已启用的 MTD/JFFS2 驱动 |
| OpenSBI | SDK `src/common/opensbi` 与其 K230 platform | 不改源码；用现有 Image 重新链接 `fw_payload.bin` |
| environment | SDK `board/common/env/spinor.jffs2.env` | 复制为 QEMU 专用 env，修正 bootargs；用 SDK `mkenvimage` 生成 |
| Flash 布局 | SDK `genimage-spinor.cfg` | 复用官方 offset；产品分区首轮放 SDK 生成的占位镜像 |
| rootfs | SDK Buildroot/rootfs 产物体系 | 首轮放最小 JFFS2 rootfs；UBI/UBIFS作为后续兼容验证 |

### 2.3 三层配置边界

1. **SDK 顶层配置**决定构建哪些系统、选择哪个 U-Boot/Linux 配置、内存区间和 SPI 分区 offset；
2. **U-Boot defconfig/DTS**决定 SPL、SPI、environment 后端、板级设备和命令实现；
3. **Linux defconfig/DTS**决定 Linux 驱动、设备状态和 MTD 分区。

三者不通用。顶层配置只是选择并向各构建层传值，不能替代 U-Boot 或 Linux 自身配置。

## 3. v4 最终启动链路

```text
QEMU host-assisted loader 从 SPI Flash 0x000000 搬运 SPL
  -> SPL 读取 boot strap，选择 NOR
  -> SPL 通过 SPI 从 0x080000 加载并解压完整 U-Boot
  -> U-Boot 根据 NOR boot mode 选择 SPI environment
  -> 从 0x1e0000 读出有效 environment（CRC 正确）
  -> 自动执行 bootcmd=k230_boot auto auto_boot
  -> k230_boot 按 sdk_autoconf.h 的 0xfc0000 读取 linux_system.bin
  -> 解包并跳转 OpenSBI fw_payload（payload 为标准 PTE Linux Image）
  -> Linux 根据 DTS 注册 SPI NOR/MTD 分区
  -> 从 0x16c0000 的 JFFS2 根分区自动进入用户空间
```

这里的 host-assisted 只代替 BootROM 搬运 SPL，不向 U-Boot 输入任何命令。SPL 之后的镜像选择、environment、Linux 加载和 rootfs 挂载均由 guest 自主完成。

## 4. v4 实施步骤

### Step 1：建立 SDK 顶层派生配置

从 `configs/k230_canmv_defconfig` 复制为 `configs/k230_canmv_qemu_spinor_defconfig`，只做以下必要差异：

```text
CONFIG_UBOOT_DEFCONFIG="k230_canmv_qemu"
CONFIG_LINUX_DTB="k230_canmv_qemu"
CONFIG_SPI_NOR=y
CONFIG_SPI_NOR_SUPPORT_CFG_PARAM=y
CONFIG_SUPPORT_LINUX=y
# CONFIG_SUPPORT_RTSMART is not set
```

保留当前已验证的 Linux RAM 区间：

```text
CONFIG_MEM_LINUX_SYS_BASE=0x08000000
CONFIG_MEM_LINUX_SYS_SIZE=0x08000000
```

不基于 `k230_canmv_only_linux_defconfig`，因为其 Linux RAM 基址为 `0x0`，与当前验证链路不一致。执行顶层配置后，必须保留配置差异，并确认 SDK 自动生成：

```text
include/generated/autoconf.h
  -> src/little/uboot/board/canaan/common/sdk_autoconf.h
```

禁止直接手改 `sdk_autoconf.h`。验收点是 Linux SPI NOR offset 不再编译为 `0xffffffff`，而是官方 `0xfc0000`。

### Step 2：保留并收敛 U-Boot/SPL 改动

保留上一轮已经证明必要的改动：

1. `board/canaan/k230_canmv/board.c` 删除强制 `SYSCTL_BOOT_SDIO1` 覆盖，让 SPL 读取 boot strap；
2. `CONFIG_K230_PUFS` 关闭，QEMU 环境改走软件 SHA；
3. 使用 `k230_canmv_qemu_defconfig` 和 U-Boot `k230_canmv_qemu.dts`；
4. SPI0 使用 QEMU 可提供的 fixed-clock 50 MHz 和 Standard 1-1-1；
5. QEMU 未建模设备保持禁用，公共 `k230.dtsi`/`k230_evb.dtsi` 不改。

不改默认 `CONFIG_BOOTCOMMAND="run distro_bootcmd"`，也不删除 MMC environment 后端。K230 的 `arch_env_get_location()` 会在完整 U-Boot 阶段根据 `g_bootmod` 选择 SPI 或 MMC；而 SPL 加载 U-Boot 发生在读取 environment 之前，所以删除 SDIO1 强制覆盖仍然是必须的。

### Step 3：生成 QEMU 专用 SPI environment

从 `board/common/env/spinor.jffs2.env` 派生 QEMU 专用文件，复用 SDK 变量结构，仅调整启动所需字段：

```text
bootcmd=k230_boot auto auto_boot;
quick_boot=false
bootargs=root=/dev/mtdblock11 rw rootwait rootfstype=jffs2 console=ttyS0,115200 earlycon=sbi
```

说明：官方完整 DTS 有 12 个 MTD 分区，`rootfs_ubi` 实际为 `mtd11`。SDK `menuconfig_to_code.sh` 当前把 JFFS2 root 写成 `mtdblock9`，不能直接沿用。打包前必须用编译出的 DTB 和启动日志再次核对分区序号；若分区集合变化，环境同步更新，不能写死后不验证。

使用 SDK 已编译的 `mkenvimage` 生成带 CRC 的 `0x10000` environment：

```bash
src/little/uboot/tools/mkenvimage -s 0x10000 \
  -o jffs2.env <qemu-spinor.env>
```

将 `jffs2.env` 放到 `0x1e0000`。该命名与 SDK `sysimage-spinor32m_jffs2.img` 配置一致；U-Boot 实际只关心 offset 和内容。官方分区为 `0x20000`，其中 environment 有效数据仍是 U-Boot `CONFIG_ENV_SIZE=0x10000`；剩余空间保持擦除值即可。

### Step 4：复用 Image，生成 `fw_payload.bin` 和 `linux_system.bin`

不修改 OpenSBI 源码。复用 SDK OpenSBI platform，仅把现有标准 PTE Linux `Image` 作为 `FW_PAYLOAD_PATH` 重新链接，得到包含 OpenSBI + Linux 的 `fw_payload.bin`。

然后完整复用 SDK `gen_linux_bin()` 打包链：

```text
fw_payload.bin
  -> k230_gzip
  -> mkimage -T multi -C gzip（fw_payload.bin.gz + 空 rd + k230_canmv_qemu.dtb）
  -> firmware header
  -> linux_system.bin
```

这里不是重新编译 Linux，也不是改 OpenSBI 功能；只是把过去分开的 `fw_jump.uImage + Image + DTB` 变成 `k230_boot` 能识别的官方容器格式。

硬门禁：`linux_system.bin <= 0x700000`。历史官方文件仅比 7 MiB 上限小约 800 B，而 QEMU DTB 更大，因此必须以本轮实际产物为准。若超限，按以下顺序处理：

1. 精简 QEMU DTB 中无关且已禁用的节点/属性；
2. 检查是否误带 initramfs、调试信息或错误 payload；
3. 在不改启动 ABI 的前提下调整构建尺寸；
4. 不优先改官方分区边界，因为会扩大 DTS、environment 和工具链联动范围。

### Step 5：生成最小 Flash rootfs

首轮选择 JFFS2，原因是 SDK 已提供 `spinor.jffs2.env`、`rootfs.jffs2` 和 `sysimage-spinor32m_jffs2.img` 完整路径，且不需要先验证 UBI attach/volume 两层行为。

Linux `k230_canmv_defconfig` 已经包含 `MTD_BLOCK`、`MTD_SPI_NOR`、`JFFS2_FS`、`SPI_DW_MMIO` 和 UBI/UBIFS 支持；本轮先复用这些配置，不新增 Linux defconfig 改动。rootfs 只保留进入 shell 的必要内容：init、BusyBox、基础设备节点/挂载脚本和验证标记。必须满足：

```text
rootfs.jffs2 <= 0x900000
```

不要把 SDK 完整产品 rootfs 硬塞进 32 MiB Flash。若完整 rootfs 超过 9 MiB，那是产品内容与固定布局不匹配，不是冷启动链路问题；本轮用最小 rootfs 验证 Flash 根文件系统，完整产品系统应改用更大 Flash、网络根、SD/eMMC，或另行裁剪。

JFFS2 验证通过后，再把 UBI/UBIFS作为第二阶段兼容项，使用：

```text
ubi.mtd=rootfs_ubi root=ubi0_0 rootfstype=ubifs rw
```

UBI 路径不进入 v4 首轮通过门禁。

### Step 6：按官方 offset 装配 32 MiB Flash

优先调用 SDK `gen_image_spinor`/genimage 流程；只有 SDK 整体构建被无关 RT-Smart 或产品资源阻塞时，才写一个薄装配脚本复用同一批 SDK 产物和 offset。

由于保留 `CONFIG_SPI_NOR_SUPPORT_CFG_PARAM=y`，`auto_boot` 会读取 quick_boot、face_db、sensor_cfg、ai_mode、speckle_cfg 和 rtt_app。它们必须复用 SDK `gen_cfg_part_bin()` 生成带合法 header 的占位包，不能留全 `0xff`。`CONFIG_SUPPORT_RTSMART` 关闭后不会加载 `rtt`，该分区只需给 genimage 提供一个已在 manifest 标记为“不执行”的最小占位文件；这比为 QEMU 构建完整 RT-Smart 更符合 YAGNI。

| 偏移 | 大小 | 内容 | 首轮策略 |
|---:|---:|---|---|
| `0x000000` | `0x080000` | `swap_fn_u-boot-spl.bin` | 必须生成 |
| `0x080000` | `0x160000` | `fn_ug_u-boot.bin` | 必须生成 |
| `0x1e0000` | `0x020000` | 有效 `jffs2.env` | 必须生成，后半保持擦除值 |
| `0x200000` | `0x080000` | quick_boot_cfg | SDK 合法占位包，`auto_boot` 会读 |
| `0x280000` | `0x080000` | face_db | SDK 合法占位包，`auto_boot` 会读 |
| `0x300000` | `0x040000` | sensor_cfg | SDK 合法占位包，`auto_boot` 会读 |
| `0x340000` | `0x300000` | ai_mode | SDK 合法占位包，`auto_boot` 会读 |
| `0x640000` | `0x200000` | speckle_cfg | SDK 合法占位包，`auto_boot` 会读 |
| `0x840000` | `0x1c0000` | rtt | genimage 最小占位，不执行 |
| `0xa00000` | `0x5c0000` | rtt_app | SDK 合法占位包，`auto_boot` 会读 |
| `0xfc0000` | `0x700000` | `linux_system.bin` | 必须生成并校验尺寸 |
| `0x16c0000` | `0x900000` | `rootfs.jffs2` | 必须生成并校验尺寸 |

每段在写入前校验 `offset + actual_size <= partition_end`，最终镜像严格为 32 MiB。产品分区使用占位内容，但 offset 不得挪动；日志中这些配置包的加载失败同样视为 v4 失败，避免用“反正最后 Linux 能起”掩盖错误布局。

### Step 7：无人干预启动验证

运行脚本仅允许：启动 QEMU、设置超时、保存 stdout/stderr、根据日志判定结果。禁止使用管道、expect、按键或延时写入向串口发送 U-Boot/Linux 命令。

启动形式保持现有 host-assisted loader 接口，Flash 参数指向新镜像。当前 loader 自己从 Flash `0x0` 读取并还原 SPL，因此本方案不需要再用 `-bios` 把 SPL 或完整 U-Boot 预置到 RAM；`-bios` 仅在单独的 SPL 隔离实验中使用，不属于最终自动链路。无论是否使用 `-bios`，均不得向 U-Boot 注入 `sf read`/`bootm`。

若失败，按阶段定位：

1. 未到 SPL：查 firmware header/host-assisted loader；
2. SPL 未到 U-Boot：查 boot mode、SPI、U-Boot header/gzip；
3. U-Boot 停在 shell：查 environment CRC、位置和 `bootcmd`；
4. `k230_boot` 报 offset/镜像错误：查 `sdk_autoconf.h` 和 `linux_system.bin`；
5. 内核起但未挂根：查 MTD 分区序号、JFFS2驱动和 rootfs 内容。

### Step 8：生成本轮独立测试与复现文档

测试结束后必须新建以下两份文档，不能用链接旧文档代替正文，也不能因为 v3 已记录过相同步骤而省略：

1. `exper-note/k230/spi/coldboot/k230-coldboot-canmv-qemu-v4-test-report.md`
   - 记录测试环境、commit、配置差异、产物 manifest、实际命令和门禁逐项结果；
   - 摘录从 SPL、environment、`k230_boot`、OpenSBI、Linux 到 Flash rootfs 的关键日志；
   - 明确失败过的步骤、根因、修复和最终遗留风险；
   - 明确结论是“v4 官方布局无人干预启动”，不得混用 v3 raw/initramfs 证据。
2. `exper-note/k230/spi/coldboot/k230-coldboot-canmv-qemu-v4-reproduction.md`
   - 从干净 SDK 基线开始，完整写出依赖、派生配置、所有必要修改和构建命令；
   - 写出 environment、OpenSBI payload、`linux_system.bin`、JFFS2、32 MiB Flash 的生成方法；
   - 写出 QEMU 启动命令、预期日志、成功判据和按阶段排错方法；
   - 即使与旧文档重复，也要完整重写本轮所需内容，保证读者不打开旧文档也能独立复现。

两份文档只能在真实测试后按实际结果填写。计划中的预期命令不能直接冒充实测证据。

## 5. 配置与改动影响矩阵

| 层 | 文件/产物 | 动作 | 原因 | 影响面 | 回滚 |
|---|---|---|---|---|---|
| QEMU | K230 machine/设备模型 | 不改 | 现有模型已覆盖链路 | 无新增影响 | 无 |
| SDK 顶层 | 新 `k230_canmv_qemu_spinor_defconfig` | 新增派生配置 | 选择 QEMU U-Boot/DTB、SPI NOR、Linux-only | 仅新配置使用者 | 删除派生配置 |
| SDK 生成层 | `sdk_autoconf.h` | 由顶层配置再生 | 给 `k230_boot` 正确 offset | 影响该次 U-Boot 构建 | 重选原配置并再生 |
| SPL 板级 | canmv `board.c` | 删除 SDIO1 强制覆盖 | environment 读取前必须先从 NOR 加载 U-Boot | QEMU canmv 构建；真实板上游化需评估 | 恢复覆盖或仅限 QEMU 板配置 |
| U-Boot 安全 | `CONFIG_K230_PUFS` | QEMU 构建关闭 | QEMU 无 PUFS | 仅镜像校验实现改为软件 SHA | 恢复宏 |
| U-Boot 配置/DTS | `k230_canmv_qemu_defconfig/.dts` | 保留现有派生 | SPI、env、QEMU 设备范围 | 仅 QEMU 板配置 | 切回 `k230_canmv` |
| environment | QEMU 专用 env + `jffs2.env` | 新增 | 覆盖默认 distro boot，实现自动 SPI 启动 | 只影响该 Flash 镜像 | 擦除 `0x1e0000` 或换回官方 env |
| OpenSBI | `fw_payload.bin` | 重新链接 payload，不改源码 | 适配官方 `linux_system.bin` | 仅生成产物 | 继续使用旧 `fw_jump.uImage` 回归 v3 |
| Linux | 标准 PTE Image | 复用 | 已验证可在 QEMU 运行 | 无源码影响 | 换回原资产 |
| Linux DTS | `k230_canmv_qemu.dts` | 保留 MTD/QEMU 设备调整 | 分区发现、避免 probe 未建模设备 | 仅 QEMU DTB | 换回原 canmv DTB |
| rootfs | 最小 JFFS2 | 新生成 | 适配 9 MiB 分区并验证 Flash root | 仅镜像内容 | 换 rootfs 镜像 |
| host 脚本 | 构建/装配/运行脚本 | 薄封装、禁止串口注入 | 可复现与自动判定 | 仅本地验证流程 | 使用 v3 脚本回归 |
| 测试/复现文档 | 两份 v4 新文档 | 新建并完整记录 | 固化本轮证据和独立复现流程 | 文档层 | 删除新文档，不影响代码 |

### 改动记录要求

每次执行都要落一份 manifest，至少记录：

- SDK commit/分支、QEMU commit/分支、boot-assets 来源与 SHA256；
- 顶层 defconfig 相对 `k230_canmv_defconfig` 的完整 diff；
- U-Boot defconfig/DTS、Linux DTS 和 board.c/PUFS 的完整 diff；
- 自动生成的关键宏：boot mode、Linux offset/size、environment offset/size；
- SPL、U-Boot、env、linux_system、rootfs 和整盘镜像的大小、SHA256、offset；
- 实际 QEMU 命令、超时、完整串口日志和门禁结果；
- 每项临时调试改动是否进入最终产物，默认不得进入。

## 6. v4 验收门禁

- [ ] 新顶层派生配置可复现生成 `sdk_autoconf.h`，Linux offset=`0xfc0000`；
- [ ] SPL 自动选择 NOR，并从 `0x080000` 加载、解压完整 U-Boot；
- [ ] U-Boot 显示从 SPI Flash 加载 environment，且无 `bad CRC`；
- [ ] 实际执行 `bootcmd=k230_boot auto auto_boot`，无需 `run distro_bootcmd` 兜底；
- [ ] `k230_boot` 从 `0xfc0000` 加载 `linux_system.bin`，不再出现 `off=0xffffffff`；
- [ ] 所有 `SPI_NOR_SUPPORT_CFG_PARAM` 配置分区均能读取合法占位包，无 header/load error；
- [ ] `linux_system.bin <= 0x700000`，`rootfs.jffs2 <= 0x900000`；
- [ ] Linux 日志中的 MTD 分区与官方 offset 一致，rootfs 确认为 `/dev/mtdblock11`；
- [ ] Linux 自动挂载 JFFS2 并进入用户空间，可输出预置验证标记；
- [ ] 从启动 QEMU 到用户空间全程无串口输入、无 `sf read`/`bootm` 注入；
- [ ] manifest、完整日志、尺寸和 SHA256 均已保存；
- [ ] 新建 v4 测试报告，内容基于本轮真实日志，不复用 v3 结果冒充；
- [ ] 新建 v4 从零复现文档，单篇即可完成构建、装配、启动和判定；
- [ ] v3 raw/initramfs 回归链路未被破坏。

## 7. 实施顺序与停止条件

1. 先新增顶层派生配置并只生成/审查配置，不立即全量改源码；
2. 重建 U-Boot/SPL，先回归到 U-Boot shell；
3. 生成有效 environment，单独证明 U-Boot 能自动进入 `k230_boot`；
4. 生成并做离线格式/尺寸检查的 `linux_system.bin`；
5. 生成最小 JFFS2 并装配官方 32 MiB 镜像；
6. 执行无人干预端到端验证；
7. 根据真实测试结果新写测试报告和从零复现文档；
8. 只有证据指向具体缺口时才增加 SDK 修改，且同步更新影响矩阵和 manifest。

遇到以下情况立即停在当前步骤，不扩大改动：镜像越过固定分区、SDK 生成配置与预期不符、需要修改 QEMU 公共设备模型、或需要改动公共 K230 DTSI。先记录证据，再单独评估。

---

## 附录 A：v3 执行时状态快照（截至 2026-08-06）

| 项 | 状态 |
|---|---|
| QEMU 分支 | `k230-coldboot-validation`（`f7730876ca5`），**本次不改** |
| 官方工具链 | `toolchains/` + `/opt/toolchain`，Xuantie V2.6.0 可用 |
| SDK 分支 | **`k230-coldboot-canmv-qemu` 已建且当前在其上**（v2.0 `7e302f733`） |
| SDK 工作区改动 | PUFS 注释、`board.c` boot mode 覆盖已删、`k230_img.c`/`designware_spi.c` 调试打印、`cache.c`/`start.S`/linux Makefile hack（与 canmv 验证无关，不 commit） |
| dts 集中化 | **`k230.dtsi` / `k230_evb.dtsi` 已还原 SDK 原始**；新 dts 仅 uboot/linux 两侧 `k230_canmv_qemu.dts` |
| 新 dts/defconfig | uboot `k230_canmv_qemu.dts`、linux `k230_canmv_qemu.dts`（已重写+disable）、uboot `k230_canmv_qemu_defconfig` 已建 |
| linux dtb | 编译成功，**75614 字节（0x1275E）**，超布局表原预留 0x1000 → Step 5 调整 |
| resource | `resource/dts/{original,new,dtb}` + README 已建 |
| 内核资产 | `k230-boot-assets/buildroot/uboot-boot/Image`（SDK 5.10.4 标准 PTE 版） |

---

## 附录 B：v3 原实施步骤（历史记录，状态以文首 v3 结论为准）

### Step 1：SDK 分支与工作区管理（修订点 1、2）

分支**已建且当前在其上**，无需再 checkout。工作区保留现状，**不整体 stash**：

```bash
cd /home/flamboy/qemu-camp/k230_sdk
git status          # 确认：PUFS 注释、board.c、两个 canmv_qemu.dts、defconfig 等
```

**提交规划**（按逻辑切 commit，便于 review/回滚；验证通过后再 commit）：
- `c1` dts：uboot/linux `k230_canmv_qemu.dts` + uboot `k230_canmv_qemu_defconfig`（新文件）
- `c2` canmv boot mode：删除 `board.c` 的 `sysctl_boot_get_boot_mode()` 覆盖
- `c3` PUFS：`k230_board_common.h` 注释 `CONFIG_K230_PUFS`（本地验证专用，注释说明理由）
- `c4`（**不入本分支**）：linux Makefile `_zicsr_zifencei` hack、`cache.c`/`start.S` 等
  与 canmv 验证无关的改动，保留在工作区或单独说明；commit 时**只 add 明确文件**
  （绝不 `git add -A`，避免带入编译产物）

### Step 2：QEMU 侧零改动（修订点 1）

**不改 `hw/riscv/k230.c`。** 依赖现有行为：
- `create_unimplemented_device("boot", 0x91102000, 0x1000)` 读返回 0
- weak 实现读 `SOC_BOOT_CTL & 0x3 == 0` → `SYSCTL_BOOT_NORFLASH`

**验证预期**：SPL 日志从 SPI NOR 选介；GUEST_ERROR **仅观察项**，不影响启动。

### Step 3：dts 与 QEMU 外设对照（修订点 2、3；**已完成**）

#### 3.1 QEMU k230 machine 已实现外设清单（自 `hw/riscv/k230.c` 提取）

| 已实现 | 地址 |
|---|---|
| 1× c908 CPU / CLINT / PLIC | 0xF00000000 / 0xF04000000 |
| 5× UART（8250） | 0x91400000~0x91404000 |
| 3× DW SSI（spi0@91584000 等） | 0x91582000~0x91584000 |
| 2× WDT / Timer / RMU / IOMUX | 0x91105800 / 0x91106000 / 0x91101000 / 0x91105000 |
| SRAM / DDRC CFG / DDR PHY | 0x80200000 / 0x98000000 / 0x9A000000 |
| GSDMA / Decomp gzip / NOC stub | 0x80800000 / 0x80808000 / 0x91302000 |
| SPI NOR Flash（m25p80，spi0 CS0） | — |

**unimplemented**（读 0，无模型）：boot/pwr/ipcm/i2c0-4/gpio0-1/i2s/codec/adc/pwm/
usb0-1/sd0-1/qspi0-1/vo/isp/dma/kpu 等。

#### 3.2 linux `k230_canmv_qemu.dts` 设备对照 → disable 清单（已执行）

| dts 节点 | QEMU 实现 | 处理 |
|---|---|---|
| `&uart0` (0x91400000) | ✔ 8250 | 保留 |
| `&spi0` (0x91584000) + flash | ✔ DW SSI + m25p80 | **保留（核心）** |
| `&ddr` | ✔ DDRC/PHY 模型 | 保留 |
| `&mmc_sd0` / `&mmc_sd1` | ✘ sd0/sd1 unimplemented | **disable**（QEMU 无 SDHCI 模型） |
| `&usbotg0` / `&usbotg1` | ✘ usb0/usb1 unimplemented | **disable** |
| `&gpio52` / `&gpio53` / `&gpio_key` / `&gpio1` | ✘ gpio0/1 unimplemented | **disable** |
| `&dsi` + panel | ✘ vo unimplemented | **disable**（显示无模型） |
| `sound`（root 节点） | ✘ i2s/codec unimplemented | canmv dts 已 `status="disable"`，保持 |

**U-Boot 侧** `k230_canmv_qemu.dts` 维持现状（mmc0/mmc1/usbotg1 okay）：
U-Boot dm 框架对 probe 失败容错好（上次 evb 验证 mmc 无卡报错仍到 `K230#`）；
实测若 usb/mmc probe 卡住，再按同样 disable。

#### 3.3 Linux dtb 编译（已完成）

```bash
cpp -nostdinc -I linux/include -I linux/arch -undef -x assembler-with-cpp \
  linux/arch/riscv/boot/dts/kendryte/k230_canmv_qemu.dts | \
  linux/scripts/dtc/dtc -I dts -O dtb -o /tmp/k230_canmv_qemu.dtb
```

**结果**：编译成功，`75614 字节（0x1275E）`。⚠ 超布局表原预留 0x1000，
Step 5 将 DTB 预留改为 **0x14000**。

### Step 4：U-Boot 重编与打包（canmv_qemu，待实施）

```bash
cd k230_sdk/src/little/uboot
export CROSS_COMPILE=/opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin/riscv64-unknown-linux-gnu-
make k230_canmv_qemu_defconfig
make -j$(nproc)
```
打包（复用 `toolchains/pkg_uboot.sh`）：
`u-boot.bin → k230_gzip → mkimage(-T firmware gzip) → firmware_gen.py → fn_ug_u-boot.bin`
`u-boot-spl.bin → firmware_gen.py → endian-swap.py → swap_fn_u-boot-spl.bin`

预期：`u-boot` 内嵌 canmv_qemu dtb，`Model: kendryte k230 canmv (qemu cold boot)`。

### Step 5：Flash 装配（raw 布局，修订点 4：逐段大小校验）

| 内容 | Flash 偏移 | RAM 地址 | 预留长度 |
|---|---:|---:|---:|
| SPL（swap 格式） | 0x000000 | —（BootROM 直读） | 0x80000 |
| 完整 U-Boot（fn_ug 格式） | 0x080000 | 0x08000000 | 0x160000 |
| SDK 5.10.4 Image | 0x200000 | 0x08200000 | 0x198c800 |
| busybox initramfs（yocto） | 0x1b8c800 | 0x0a100000 | 0x1eec20 |
| OpenSBI fw_jump.uImage | 0x1e0ec20 | 0x0c100000 | 0x14000 |
| **canmv_qemu.dtb** | 0x1f00000 | 0x0a000000 | **0x14000（修订：原 0x1000）** |

**打包阶段必做校验（修订点 4）**——逐段核对实际文件大小 ≤ 预留：
- SPL 实际大小 ≤ 0x80000
- fn_ug_u-boot.bin 实际大小 ≤ 0x160000
- Image ≤ 0x198c800；initramfs ≤ 0x1eec20；fw_jump ≤ 0x14000
- **DTB 实际 75614B = 0x1275E ≤ 预留 0x14000 ✓**
- 相邻段不重叠、不越界（脚本断言，失败即停）

**论证（为什么不用官方 linux_system.bin 编排）：**本次验证对象是"启动路径正确性"——
BootROM/SPL 从 SPI NOR 读镜像、U-Boot `sf read` + `bootm`。linux_system.bin 是 SDK
镜像打包格式（multi uImage 压缩），与当时 v3 启动路径验证正交，故留作后续。该阶段
并未完成当前标准 PTE Image 的 `linux_system.bin` 实际尺寸判定；v4 必须以完整打包产物
是否 ≤ 7 MiB 为准。dts 分区表已对齐官方布局，后续可平滑切换。

### Step 6：端到端验证（修订点 3：门禁调整）

```bash
QEMU=./my-qemu-camp-2026-k230/build/qemu-system-riscv64   # 不改，用现有二进制
"$QEMU" -M k230,spi-flash=w25q256 -drive if=mtd,format=raw,file=<新镜像> \
  -nographic -monitor none -display none -no-reboot
```
U-Boot 命令（同 `toolchains/run_510.sh`，偏移按上表）：
```
setenv bootargs console=ttyS0,115200 earlycon=sbi cma=0
sf probe 0:0
sf read 0x0c100000 0x1e0ec20 0x14000
sf read 0x08200000 0x200000 0x198c800
sf read 0x0a100000 0x1b8c800 0x1eec20
sf read 0x0a000000 0x1f00000 0x14000
fdt addr 0x0a000000; fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0x0a100000>
fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>
bootm 0x0c100000 - 0x0a000000
```

**最终门禁（修订点 3，GUEST_ERROR 不入门禁）：**
- ✔ SPL 最终选择 SPI NOR（读 SOC_BOOT_CTL → NORFLASH）
- ✔ SPL 能加载完整 U-Boot
- ✔ 无阻断启动的异常
- boot 区 GUEST_ERROR 日志：**仅观察项**，允许存在，不影响通过

### Step 7：上游合规与文档收尾

- **清理调试打印**（挑错越少越好）：`k230_img.c` `sha256 error got=`、
  `designware_spi.c` `k230dbg:`、`k230.c` L649/699。独立 commit 收尾。
- **文档**：
  - `resource/README.md`：记录 SDK 5.10.4 内核 provenance 与局限（无重编 recipe）
  - `exper-note/k230/spi/coldboot/k230-cold-boot-from-zero.md`：新增"方案一（全 canmv）"结果与日志
  - `exper-note/workspace-state.md`：里程碑记录
- **提交**：按 Step 1 规划 commit 到 `k230-coldboot-canmv-qemu` 分支。

---

## 附录 C：v3 假设与决策

1. **决策**：方案一（全 canmv），dtb 三端一致（canmv_qemu 板级语义）。
2. **决策（修订）**：**QEMU 零改动**，依赖 unimplemented boot 区读 0 → NORFLASH。
3. **决策（修订）**：SDK 改动拉分支 `k230-coldboot-canmv-qemu`，可回滚、可 review。
4. **决策（修订）**：linux canmv_qemu.dts 按 QEMU 外设对照表 disable 未建模设备；
   U-Boot 侧维持现状，实测卡住再禁。
5. **决策**：Flash 用 raw 布局（sf read + bootm），不用官方 linux_system.bin 编排。
6. **决策**：Linux 内核复用 boot-assets SDK 5.10.4（标准 PTE 版），如实记录 provenance。
7. **决策**：canmv boot mode 用"删除覆盖、读 strap"修正，而非硬编码 NORFLASH。
8. **假设（修订）**：unimplemented boot 区读返回 0（QEMU 标准行为）；GUEST_ERROR 仅观察项不影响启动。
9. **决策**：本次不动 linux Makefile `_zicsr_zifencei` hack（未重编内核则无需触碰）。
10. **决策（修订点 5）**："同源"表述 = 同一板级语义、设备范围一致，非同一 dts 源文件。

## 附录 D：v3 本地验证 vs 上游提交改动边界

| 改动 | 归属 |
|---|---|
| uboot dts 使能 spi0 + 1-1-1（canmv_qemu dts） | **可上游化**（配合 K230 machine 文档） |
| canmv board.c 删除 SDIO1 覆盖（读 strap） | **上游化需评估**（改变真实板行为；cover letter 说明"尊重 strap"；否则留本地） |
| `CONFIG_K230_PUFS` 注释（软件 SHA） | **本地验证专用**（QEMU 无 PUFS 模型；上游需另附 PUFS 最小模型，超出本次范围） |
| linux dts 设备 disable | 视情况（配合 QEMU 模型范围；是 QEMU 验证常见做法，可接受） |
| Linux 内核"标准 PTE 重编" | **本地资产**（boot-assets 产物，无 recipe；上游 QEMU 侧不应依赖） |
| QEMU | **零改动** |

## 附录 E：v3 原验证清单（历史记录）

- [x] 分支 `k230-coldboot-canmv-qemu` 已建立且工作区改动完整（未 stash）
- [x] linux canmv_qemu.dts 按对照表 disable 完成，dtb 编译成功（75614B）
- [x] board.c 删除 SDIO1 覆盖（boot mode 修正）
- [ ] U-Boot canmv_qemu 重编成功，内嵌 dtb `Model: kendryte k230 canmv (qemu cold boot)`
- [ ] SPL 最终选 NOR、从 SPI NOR 正常读 U-Boot
- [ ] 端到端：BootROM → SPL → U-Boot → OpenSBI → Linux 5.10.4 → initramfs shell `~ #`
- [ ] GUEST_ERROR 仅限 boot 区、无阻断启动异常（观察项，非门禁）
- [ ] Flash 各段实际大小 ≤ 预留（Step 5 校验清单全过）
- [ ] 调试打印清理完成，`git diff --check` 干净
- [ ] 文档（from-zero / workspace-state / resource README）更新完成
