# K230 官方 32 MiB SPI-NOR 路线 1/2/4 实际难度评估

日期：2026-08-07

## 1. 结论

当前最合理的推进顺序是：

1. 保留路线 4 作为已经跑通的回归基线；
2. 优先实施路线 1 的“官方布局 + 最小根文件系统”版本；
3. 只有路线 1 遇到无法接受的容量或 UBI/JFFS2 阻塞时，才退到路线 2；
4. “官方布局 + 完整 SDK 产品根文件系统”单独视为高难度目标，不与最小启动验证绑定。

难度总览：

| 路线 | 当前完成度 | 新增难度 | 主要价值 | 建议 |
|---|---:|---:|---|---|
| 1A 官方布局 + 最小 JFFS2/UBI | 约 65% | 中等，3/5 | 验证官方 32 MiB 镜像格式和自动启动路径 | **下一步首选** |
| 1B 官方布局 + 完整 SDK rootfs | 约 35% | 高，4.5/5 | 接近产品镜像 | 暂不与冷启动验证绑定 |
| 2 回收中间分区、自定义布局 | 约 85% | 低到中，2/5 | 快速容纳更大 payload/rootfs | 路线 1 的退路 |
| 4 raw + initramfs | 100% | 很低，1/5 | 稳定验证启动硬件链 | 保留为回归基线 |

这里的“完成度”是工程评估，不是测试覆盖率。

## 2. 已确认的官方布局

SDK 官方 32 MiB SPI-NOR 配置来自：

- `k230_sdk/board/common/gen_image_cfg/genimage-spinor.cfg:27`
- `k230_sdk/board/common/gen_image_cfg/genimage-spinor.cfg:177`

主要分区如下：

| 偏移 | 大小 | 内容 |
|---:|---:|---|
| `0x000000` | `0x080000` | SPL |
| `0x080000` | `0x160000` | U-Boot |
| `0x1e0000` | `0x020000` | U-Boot environment |
| `0x200000` | `0x0dc0000` | quick boot、AI、RTT 等产品分区 |
| `0xfc0000` | `0x700000` | `linux_system.bin` |
| `0x16c0000` | `0x900000` | `rootfs.ubi` 或 `rootfs.jffs2` |

官方 `linux_system.bin` 生成链来自
`k230_sdk/board/common/gen_image_script/gen_image_comm_func.sh:241`：

```text
OpenSBI fw_payload.bin
→ k230_gzip
→ fw_payload.bin.gz + 1 字节 rd + k230.dtb
→ mkimage -T multi -C gzip
→ K230 firmware header
→ linux_system.bin
```

因此官方 7 MiB 分区中并不包含外置的 UBI/JFFS2 rootfs；rootfs 在后面的独立 9 MiB 分区。

## 3. 当前实际资产与容量

### 3.1 当前已跑通资产

| 文件 | 实际大小 | 说明 |
|---|---:|---|
| boot-assets 标准 PTE `Image` | 26,783,744 B | 当前 QEMU 全链路使用 |
| 上述 Image 的 `gzip -9` 结果 | 7,092,508 B | 距 7 MiB 上限约 247,524 B |
| SDK 工作区 `Image` | 22,789,632 B | 不是当前启动使用的内核 |
| 上述 SDK Image 的 `gzip -9` 结果 | 8,312,279 B | 已超过 7 MiB，不能直接使用 |
| Yocto `rootfs.cpio.gz` | 2,026,528 B | 解压后 3,553,792 B |
| Buildroot `rootfs.cpio.gz` | 26,400,254 B | 解压后 76,188,160 B |

结论：

- 当前标准 PTE 内核有机会装入官方 `linux_system.bin`，但必须制作完整包后判断，不能只看 Image gzip 大小；
- Yocto 最小 rootfs 很适合转换为 9 MiB 以内的 JFFS2/UBI 验证镜像；
- 当前 Buildroot initramfs 不适合直接转成 9 MiB Flash rootfs。

### 3.2 已发现的历史官方构建实例

临时 SDK 构建目录中存在一套基于 SDK `7e302f733` 的官方 CanMV SPI-NOR 产物：

| 文件 | 实际大小 | 官方上限 | 余量 |
|---|---:|---:|---:|
| `linux_system.bin` | 7,339,232 B | 7,340,032 B | **800 B** |
| `rootfs.ubi` | 4,456,448 B | 9,437,184 B | 4,980,736 B |
| `rootfs.ubifs` | 4,316,928 B | — | — |
| `sysimage-spinor32m.img` | 33,292,288 B | 32 MiB 地址空间 | 镜像止于 `0x1fc0000` |

该实例证明官方布局在容量上可以成立，但不能直接当作 QEMU 已验证资产：

- 它使用原始 `k230_canmv` DTS，不含 QEMU fixed-clock 和未建模外设裁剪；
- CanMV SPL 原始代码固定选择 SDIO1；
- 该内核是否采用 QEMU 可用的标准 PTE 尚未证明；
- `linux_system.bin` 仅剩 800 B，任何 DTS 或内核变化都可能超限。

历史官方 DTB 为 74,352 B，当前 QEMU DTB 约 75,778 B，多 1,426 B。若其他组成完全不变，直接替换会超过 7 MiB 约 626 B。通过精简无关 DTS 节点或略微缩减 payload 即可解决，属于小规模容量修整。

## 4. 路线 1：保持官方 32 MiB 布局

### 4.1 路线 1A：官方布局 + 最小根文件系统

难度：**中等，3/5**。

推荐第一轮使用 JFFS2，原因是链路更短：

```text
SPI NOR MTD partition → /dev/mtdblockN → JFFS2
```

当前标准 PTE 内核的内嵌配置已确认包括：

- `CONFIG_MTD_SPI_NOR=y`
- `CONFIG_JFFS2_FS=y`
- `CONFIG_MTD_UBI=y`
- `CONFIG_UBIFS_FS=y`

因此不需要为了文件系统驱动重新构建内核。

需要完成：

1. 用当前标准 PTE Image 重建 OpenSBI `fw_payload.bin`；
2. 按 SDK `gen_linux_bin()` 生成真正的 `linux_system.bin`；
3. 检查完整产物是否不超过 `0x700000`；
4. 若只超少量，优先精简 QEMU DTB，而不是先裁剪内核；
5. 将 Yocto 最小根目录制作成 JFFS2，控制在 `0x900000` 内；
6. 生成官方 offset 的 32 MiB Flash 镜像；
7. 让 U-Boot 执行官方 `k230_boot auto auto_boot`，不再手工 `sf read`；
8. 验证 Linux 从 Flash JFFS2 根文件系统进入 shell。

当前确定的配置问题：

- `k230_img.c` 的 NOR 自动加载依赖 `CONFIG_SPI_NOR_LK_BASE=0xfc0000` 等顶层 SDK 配置，`k230_canmv_qemu` 的独立 U-Boot defconfig 尚未证明包含这些值；
- 当前 DTS 有 12 个 MTD 分区，`rootfs_ubi` 通常是 `mtd11`，但 SDK 默认 NOR bootargs 写死为 `ubi.mtd=9`；应优先改为按名称 `ubi.mtd=rootfs_ubi`，避免编号漂移；
- JFFS2 路线同样应避免硬编码旧的 `/dev/mtdblock9`，需要按当前分区顺序校准。

主要风险：

- 完整 `linux_system.bin` 容量余量可能很窄；
- 官方自动加载和 firmware header 校验尚未在这套 Linux 包上验证；
- JFFS2 首次扫描和 SPI NOR 写路径尚未完成端到端验证。

### 4.2 路线 1B：官方布局 + 完整 SDK 产品 rootfs

难度：**高，4.5/5**。

当前 Buildroot initramfs 解压后约 72.7 MiB，明显不能直接放进 9 MiB。需要使用 SDK 的 `shrink_rootfs_common()` 逻辑，系统性删除：

- 内核模块；
- 测试程序和示例；
- SSH、调试和性能工具；
- 多媒体、AI 与产品无关依赖；
- 不需要的共享库和资源。

即使最终能生成 4.5 MiB 左右的历史 `rootfs.ubi`，也必须重新验证被裁剪后的用户空间是否满足“完整 SDK rootfs”的定义。因此这条路线已经从启动验证变成产品裁剪项目，不建议作为当前冷启动工作的门禁。

## 5. 路线 2：回收中间分区，采用 QEMU 定制布局

难度：**低到中，2/5**。

从 `0x200000` 到 `0xfc0000` 的产品分区合计约 13.75 MiB。当前 QEMU 验证不使用 quick boot、face database、sensor、AI、speckle、RTT 和 RTT app，可将这段空间重新分配给 Linux 或 rootfs。

当前 raw 方案已经完成了这条路线的大部分工作：

- SPL/U-Boot 使用官方头部偏移；
- Linux、initramfs、OpenSBI 和 DTB 使用自定义绝对偏移；
- 完整启动链已到 shell；
- 装配脚本已有大小和重叠断言。

若希望比当前 raw 方案更接近官方，可保留 `linux_system.bin` 的官方 multi-uImage 格式，只改变它和 rootfs 的 Flash 偏移及分区大小。

需要修改：

- DTS fixed-partitions；
- SDK 顶层 `CONFIG_SPI_NOR_*_BASE/SIZE`；
- U-Boot 自动加载 offset；
- rootfs bootargs；
- Flash 装配脚本。

优点是容量宽松、实现直接；缺点是测试结果不能宣称“官方 32 MiB Flash 布局原样兼容”。

## 6. 路线 4：继续 raw + initramfs

难度：**很低，1/5；当前已经完成**。

已验证路径：

```text
host-assisted SPL loader
→ SPL 从 SPI NOR 加载并通过 GSDMA/decomp-gzip 解压 U-Boot
→ U-Boot 使用 DW SSI `sf read`
→ OpenSBI
→ Linux 5.10.4
→ gzip initramfs
→ shell
```

它适合作为后续所有实验的回归基线：只要路线 1 或 2 失败，先用路线 4 判断是 SPI/启动链回归，还是官方包解析/rootfs 挂载问题。

它不能证明：

- 官方 `linux_system.bin` 解析；
- 官方 `k230_boot auto auto_boot`；
- JFFS2/UBI Flash rootfs；
- 官方 32 MiB 分区布局兼容。

## 7. 推荐执行顺序与门禁

### Phase A：容量预检，不启动

1. 复用当前标准 PTE Image 重建 OpenSBI `fw_payload.bin`；
2. 生成 QEMU CanMV DTB；
3. 按官方脚本生成 `linux_system.bin`；
4. 门禁：`stat` 大小必须 `<= 0x700000`。

若只超数 KiB，先精简 DTS；若超数百 KiB，再考虑内核裁剪。

### Phase B：官方包自动加载，暂不要求根文件系统成功

1. 将 `linux_system.bin` 放到 `0xfc0000`；
2. 补齐官方 offset 配置；
3. 运行 `k230_boot auto auto_boot`；
4. 门禁：U-Boot 能解析 firmware header/multi-uImage，并进入 Linux；此阶段允许最终因尚无可用 Flash rootfs 而 mount/panic。

这一步只隔离“官方包加载”。官方 `gen_linux_bin()` 默认 `rd` 只是 1 字节占位，不能把当前 2 MiB initramfs 直接加入 7 MiB 包，否则很可能超限；Flash rootfs 留到 Phase C。

### Phase C：最小 JFFS2 rootfs

1. 由 Yocto 最小根目录生成不超过 9 MiB 的 JFFS2；
2. 写入 `0x16c0000`；
3. 修正 rootfs 分区名/编号和 bootargs；
4. 门禁：从 Flash rootfs 启动到 shell。

### Phase D：UBI/完整 SDK rootfs

仅在 Phase A-C 全部通过后进行。失败时可以明确归因到 UBI 或用户空间裁剪，不会反向污染启动链判断。

## 8. 最终建议

当前不需要先走路线 2，也不应该把完整 SDK rootfs 作为路线 1 的首个目标。

最佳选择是：

> 以路线 4 为稳定基线，先做路线 1A：保持官方 32 MiB 偏移和 `linux_system.bin` 格式，使用最小 JFFS2 rootfs 完成自动启动；通过后再选择是否升级到 UBI 和完整 SDK 用户空间。

这条路线能以中等工作量获得最高验证价值，同时保留清晰的故障隔离边界。
