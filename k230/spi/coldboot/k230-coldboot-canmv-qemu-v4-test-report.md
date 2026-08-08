# K230 canmv QEMU v4 冷启动测试报告

## 1. 结论

v4 已按 SDK 官方 32 MiB SPI-NOR 布局完成无人干预冷启动验证：

```text
QEMU host-assisted SPL loader
  -> SPL
  -> SPI NOR U-Boot environment
  -> k230_boot auto auto_boot
  -> linux_system.bin
  -> OpenSBI v0.9
  -> Linux 5.10.4
  -> /dev/mtdblock11 JFFS2
  -> 最小用户空间
```

最终 runner 的 stdin 明确重定向为 `/dev/null`，并使用 QEMU `-snapshot` 隔离 guest
写入；未使用 expect、按键、`sf read` 或 `bootm` 注入。最终镜像严格为 32 MiB，
运行前后 SHA256 均为：

```text
e7728c24a7edef22c85d57dcd41c7c429de974a7337fde5d4eb02b0da689911a
```

最终证据目录：
`.trae/evidence/k230-coldboot-canmv-qemu-v4/`。

## 2. 测试基线与边界

| 项 | 本轮值 |
|---|---|
| QEMU 仓库 | `my-qemu-camp-2026-k230` |
| QEMU 分支 / HEAD | `k230-coldboot-validation` / `7730876ca51b303bd424d9ae71d9a932936ba208` |
| SDK 仓库 | `k230_sdk` |
| SDK 分支 / HEAD | `k230-coldboot-canmv-qemu` / `90f1f26b2ec079bcb4471964cf934b2fefaa34ad` |
| boot-assets HEAD | `c3c32fb46e8307c5063f13e8f367c98bf9273cd1` |
| QEMU 源码改动 | 无；`git status --short` 为空 |
| Linux 源码改动 | 本轮未新增；复用标准 PTE Image |
| rootfs 基线 | `k230-boot-assets/yocto/uboot-boot/rootfs.cpio.gz` |
| 最终串口输入 | `/dev/null` |
| runner 超时 | 60 秒；guest 到 shell 后由 host timeout 收集日志 |

SDK 工作树原有以下用户修改，本轮没有清理、覆盖或提交：

- `src/little/linux/arch/riscv/Makefile`
- `src/little/uboot/arch/riscv/cpu/k230/cache.c`
- `src/little/uboot/arch/riscv/cpu/start.S`

最终 SPL/U-Boot 从 SDK HEAD 的 U-Boot 子树归档到临时目录后重建，只注入顶层配置生成的
`sdk_autoconf.h`。`clean-uboot-build.log` 证明归档中的 `cache.c`、`start.S` 哈希与
SDK HEAD 一致，因此上述用户修改未进入最终 bootloader。

## 3. 本轮实现

SDK 新增：

- `configs/k230_canmv_qemu_spinor_defconfig`
- `board/common/env/k230_canmv_qemu_spinor.jffs2.env`

工作区根目录新增：

- `toolchains/build_canmv_qemu_v4_cfg_parts.sh`
- `toolchains/build_canmv_qemu_v4_rootfs.sh`
- `toolchains/k230-coldboot-v4-rootfs-init`
- `toolchains/k230-coldboot-v4-rootfs-marker`
- `toolchains/mkcanmv_qemu_v4_flash.py`
- `toolchains/run_canmv_qemu_v4.sh`

顶层派生配置相对 `k230_canmv_defconfig` 只改变以下能力：

```text
CONFIG_UBOOT_DEFCONFIG="k230_canmv_qemu"
CONFIG_LINUX_DTB="k230_canmv_qemu"
CONFIG_SPI_NOR=y
CONFIG_SPI_NOR_SUPPORT_CFG_PARAM=y
# CONFIG_SUPPORT_RTSMART is not set
CONFIG_SUPPORT_LINUX=y
```

生成配置门禁确认：

```text
CONFIG_SPI_NOR_LK_BASE  = 0xfc0000
CONFIG_SPI_NOR_LK_SIZE  = 0x700000
CONFIG_SPI_NOR_LR_BASE  = 0x16c0000
CONFIG_SPI_NOR_LR_SIZE  = 0x900000
```

SPI environment 的关键值为：

```text
bootcmd=k230_boot auto auto_boot;
bootargs=root=/dev/mtdblock11 rw rootwait rootfstype=jffs2 console=ttyS0,115200 earlycon=sbi;
quick_boot=false
```

environment 大小为 65536 字节，计算 CRC 与存储 CRC 一致：

```text
stored_le=0x9ab3aef7 calculated=0x9ab3aef7
```

## 4. TDD 记录

本轮按已确认的四个 seam 做纵向 red-green：

| Seam | Red | Green |
|---|---|---|
| 顶层配置 -> `sdk_autoconf.h` | 派生配置尚不存在 | 生成头与 `include/generated/autoconf.h` 完全一致，offset 正确 |
| 构建产物 -> 32 MiB 镜像/manifest | 严格镜像尚未装配 | 所有段格式、大小、边界和整盘大小通过 |
| `/dev/null` 冷启动 -> JFFS2 用户空间 | rootfs 无 v4 marker/init | `/init` 与 `/sbin/init` 均存在并输出 marker |
| v3 raw/initramfs 回归 | 需证明历史链路未破坏 | 内核执行 `/init`，进入历史 initramfs shell |

实际失败与修复：

1. `mkfs.jffs2 -s 1` 产生约 252 MiB 镜像，远超 9 MiB 分区；改为 Linux 4 KiB
   page size，并按验收要求裁剪到 shell、init applet 和动态库后，最终 JFFS2 为
   2468820 字节。
2. 首次最小 rootfs 只有 `/init`，内核实际查找 `/sbin/init` 的路径覆盖不足；最终同时安装
   `/init` 与 `/sbin/init`。
3. 首版 bootloader 在含用户 `cache.c/start.S` 修改的工作树中构建；最终改用 SDK HEAD
   归档源码重建，并重新装配、复测。
4. 干净 bootloader 的 15 秒试跑停在配置包读取阶段；只延长门限到 60 秒后完整通过，确认
   是测试门限过紧，不是启动缺陷。
5. runner 最初用任意 `0xffffffff` 判旧 offset，误伤 Linux 正常位掩码日志；断言收窄为
   `off=0xffffffff` 后转绿。
6. SDK `mkimage -T multi` 拒绝真正的空 `rd`；空文件尝试保留在
   `linux-system-build.log`，最终使用不参与 rootfs 启动的 2 字节 `a\n` 占位段。
7. 首轮 runner 直接挂载可写 raw 镜像，JFFS2 写入改变了整盘哈希；最终增加
   `-snapshot`，重装配后证明运行前后哈希一致。
8. 双验收发现原 rootfs 仍含完整 Yocto initramfs 内容；最终仅保留 BusyBox shell、
   init 所需 applet、loader、`libc` 和 `libm`，重新完成端到端验证。

失败证据与最终证据均保留在 evidence 目录，没有用后一次 PASS 覆盖失败事实。

## 5. 最终产物 manifest

| 分区 | offset | 分区大小 | 实际大小 | SHA256 |
|---|---:|---:|---:|---|
| SPL | `0x000000` | `0x080000` | 226500 | `ab67c9e1c17c5755a85cd24bed3a3576eea80fd7d25f77955b54981f02eb75ab` |
| U-Boot | `0x080000` | `0x160000` | 318413 | `728a8f5ccaec2922af7a3e41b82b4b9c3524b94668f914efb5df4cfd877e1a84` |
| environment | `0x1e0000` | `0x020000` | 65536 | `9122bd5aba13184f3abf7b778dff5bebdd22e5b633eb8dc9ee67bb8017f62739` |
| quick_boot_cfg | `0x200000` | `0x080000` | 626 | `287d07c4b84f09d042073df1e5b7331ceba24d7439f814aefdcca4ab1f28321e` |
| face_db | `0x280000` | `0x080000` | 626 | `e2c7f3c6dda2d9c8a1f804e9ff94b77b61a3f4389a03c84d7384634932d6210e` |
| sensor_cfg | `0x300000` | `0x040000` | 626 | `1e44f3aa1fe6e5231a3a62bb2c01779c314814d0dad60790cee85aee03edf188` |
| ai_mode | `0x340000` | `0x300000` | 626 | `291ce41baa7e74c90cae43ef34c868738f73f9e6ea44a9a127867eda155efa04` |
| speckle_cfg | `0x640000` | `0x200000` | 626 | `c34037cc268aefdf9bf0ee2d56723163765b25ff303987833dba1fc0bf5320cc` |
| rtt（不执行） | `0x840000` | `0x1c0000` | 16 | `27bab8c62e230bb9ba2d4d760825a3b42fdf875a5d492e276e56459f420adf61` |
| rtt_app | `0xa00000` | `0x5c0000` | 626 | `9e660457072a43e3a31e632d3a5431614f2f90166b9c6107ce32148bd8949817` |
| linux | `0xfc0000` | `0x700000` | 7213154 | `49149cbc3627d1a3fe1bbd11f1c09de6690fd529ba6ff189f12e0483bfe21b3b` |
| rootfs_jffs2 | `0x16c0000` | `0x900000` | 2468820 | `0083509987dd00017e94dbefd10cf57a538e05790d488f47a4944b616cb4e469` |

`linux_system.bin` 比 7 MiB 上限小 127178 字节；rootfs 比 9 MiB 上限小
6968364 字节。

## 6. 实测命令

最终执行入口：

```bash
toolchains/run_canmv_qemu_v4.sh \
  .trae/evidence/k230-coldboot-canmv-qemu-v4/k230_canmv_qemu_v4_spinor32m.img \
  .trae/evidence/k230-coldboot-canmv-qemu-v4/v4-qemu-no-serial-final-clean.log
```

runner 展开的 QEMU 命令为：

```bash
timeout 60s my-qemu-camp-2026-k230/build/qemu-system-riscv64 \
  -M k230,spi-flash=w25q256 \
  -drive if=mtd,format=raw,file=.trae/evidence/k230-coldboot-canmv-qemu-v4/k230_canmv_qemu_v4_spinor32m.img \
  -nographic -monitor none -display none -no-reboot -snapshot </dev/null
```

QEMU 最终状态必须为 124，即到达 shell 后的 host 采集超时；其他退出状态直接失败。
runner 在完整检查所有门禁后返回 0。

## 7. 关键日志与门禁

最终日志 `v4-qemu-no-serial-final-clean.log` 证明：

```text
stdin=/dev/null
Loading Environment from SPIFlash... OK
imge: quick_boot_cfg load ...
imge: face_db load ...
imge: sensor_cfg load ...
imge: ai_mode load ...
imge: speckle load ...
imge: rtapp load ...
imge: linux load to 8000000 ...
OpenSBI v0.9
Kernel command line: root=/dev/mtdblock11 ... rootfstype=jffs2 ...
0x000000fc0000-0x0000016c0000 : "linux"
0x0000016c0000-0x000001fc0000 : "rootfs_ubi"
VFS: Mounted root (jffs2 filesystem) on device 31:11.
K230_V4_JFFS2_ROOTFS_OK
K230 v4 JFFS2 userspace ready
```

门禁结果：

- [x] 派生配置可生成正确 `sdk_autoconf.h`
- [x] SPL/U-Boot 从干净 SDK HEAD 源码构建
- [x] SPI environment 有效且无 bad CRC
- [x] environment 包含 `bootcmd=k230_boot auto auto_boot`
- [x] 六个配置包均实际加载，无 header/load error
- [x] Linux 使用官方 `0xfc0000` 分区，不存在 `off=0xffffffff`
- [x] Linux MTD 分区与官方布局一致
- [x] `/dev/mtdblock11` 自动挂载 JFFS2 并进入用户空间
- [x] 全程无串口输入
- [x] QEMU 源码零改动
- [x] v3 raw/initramfs 回归进入 `/init` 和 shell
- [x] manifest 包含 commit、配置/源码证据、关键宏、命令、日志和临时改动边界
- [x] `-snapshot` 运行前后最终镜像 SHA256 不变

## 8. 局限与遗留风险

1. rootfs 是从已验证的 Yocto initramfs 派生的最小 JFFS2 验证系统，不是完整 CanMV 产品 rootfs。
2. 六个产品配置包是 SDK 合法容器中的一字节占位 payload，只验证布局、header、解包和加载路径。
3. `rtt_system.bin` 是不执行的 16 字节占位；本配置关闭 RT-Smart。
4. JFFS2、gzip/uImage 包含构建时间信息，重新构建时哈希可能变化；应以每次新 manifest 为准。
5. U-Boot SPI NOR software reset 仍报告 `-524`，但驱动随后识别 w25q256、加载 environment
   并完成全链路，属于已知非阻断日志。
6. 根目录 `toolchains/` 不属于 SDK Git 仓库；若要提交，需由工作区所属仓库明确接管这些文件。

本轮未执行 `git commit`、`git push`、stash、clean 或 reset。
