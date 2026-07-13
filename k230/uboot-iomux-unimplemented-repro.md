# K230 完整启动与 IOMUX 验证

## 目的

这篇文档现在作为 K230 QEMU 的启动主入口，记录四件事：

1. 如何直接使用预构建镜像启动 Linux。
2. 如何经过 SDK U-Boot 和 OpenSBI 启动 Linux。
3. 如何在 U-Boot 真实初始化路径里验证 IOMUX 模型。
4. 本地 `k230_sdk` 与预构建镜像之间有哪些相同点和差异。

当前推荐流程：

```text
先用 Yocto direct boot 确认 QEMU/Linux 基础环境
  -> 再用 U-Boot boot 验证固件链路
  -> 最后按需要跑 IOMUX/unimp 验证
```

原来的临时编译 U-Boot 和历史复测步骤保留在文档最后，作为遗留排障流程，不再作为日常启动的第一选择。

## 1. 当前结论

2026-07-10 本机实测：

| 路径 | 结果 | 最终状态 |
|------|------|----------|
| Yocto direct boot | 通过 | 进入 initramfs shell，出现 `~ #` |
| Buildroot direct boot | 通过 | Linux 5.10.4 进入用户空间并启动系统服务 |
| Yocto U-Boot boot | 通过 | `U-Boot 2022.10 -> OpenSBI 0.9 -> Linux 6.18.28 -> ~ #` |
| SDK U-Boot + IOMUX | 通过 | 进入 `K230#`，IOMUX 不再产生 unimplemented 日志 |

结论：

```text
k230-boot-assets 可以直接替代日常启动所需的手工镜像准备流程。
```

这里的“直接使用”有明确边界：

- Direct boot 由 QEMU 直接加载 OpenSBI、Linux、DTB 和 initramfs。
- U-Boot boot 由 QEMU 把 OpenSBI、Linux、DTB 和 initramfs 预放到 RAM，再由 U-Boot 执行 `bootm`。
- 这两条路径都不是完整的 BootROM、SPL、SD/eMMC/QSPI 存储启动链。

## 2. 工作区与环境

命令默认从工作区根目录执行：

```bash
WORKSPACE="/home/flamboy/qemu-camp-2026"
cd "$WORKSPACE"

QEMU="$WORKSPACE/qemu-camp-2026-k230/build/qemu-system-riscv64"
OPENSBI="$WORKSPACE/qemu-camp-2026-k230/pc-bios/opensbi-riscv64-generic-fw_dynamic.bin"
ASSETS="$WORKSPACE/k230-boot-assets"
SDK="$WORKSPACE/k230_sdk"
```

当前工作区使用的是本地编译 QEMU：

```text
QEMU emulator version 11.0.50
-machine k230
```

先确认二进制和 machine 存在：

```bash
"$QEMU" --version
"$QEMU" -machine help | rg '(^|\s)k230(\s|$)'
```

应看到：

```text
k230  RISC-V Board compatible with Kendryte K230 SDK
```

不要使用系统自带的 `/usr/bin/qemu-system-riscv64`。当前系统版本是 QEMU 8.2.2，不包含 `k230` machine。

## 3. 准备预构建镜像

镜像仓库：

```text
https://github.com/zevorn/k230-boot-assets
```

当前工作区已经拉取到：

```text
/home/flamboy/qemu-camp-2026/k230-boot-assets
```

新环境可以从工作区根目录执行：

```bash
git clone --depth 1 \
  "https://github.com/zevorn/k230-boot-assets.git" \
  "$WORKSPACE/k230-boot-assets"
```

如果当前容器因为缺少 CA 证书链出现：

```text
server certificate verification failed
```

可以只对本次公开仓库克隆临时关闭校验：

```bash
git -c http.sslVerify=false clone --depth 1 \
  "https://github.com/zevorn/k230-boot-assets.git" \
  "$WORKSPACE/k230-boot-assets"
```

这不会修改全局 Git 配置。

关键文件：

```text
k230-boot-assets/
  common/
    u-boot
    fw_jump.uImage
  yocto/direct-boot/
    Image
    k230-canmv.dtb
    rootfs.cpio.gz
  yocto/uboot-boot/
    Image
    k230-canmv.dtb
    rootfs.cpio.gz
  buildroot/direct-boot/
    Image
    k230-qemu.dtb
    rootfs.cpio.gz
  buildroot/uboot-boot/
    Image
    k230-qemu.dtb
    rootfs.cpio.gz
```

仓库里的 `buildroot/uboot-boot/rootfs.ext4` 使用 Git LFS。当前环境没有可用的 `git-lfs`，所以该文件是 LFS 指针，但本文所有已验证流程都使用 `rootfs.cpio.gz`，不受影响。

## 4. 主流程 A：Yocto Direct Boot

这是当前最推荐的环境验证命令，镜像小、启动快、依赖最少。

```bash
DST="$ASSETS/yocto/direct-boot"

"$QEMU" \
  -machine k230 \
  -bios "$OPENSBI" \
  -kernel "$DST/Image" \
  -dtb "$DST/k230-canmv.dtb" \
  -initrd "$DST/rootfs.cpio.gz" \
  -append "console=ttyS0,115200 earlycon=sbi" \
  -nographic \
  -no-reboot
```

启动链：

```text
QEMU OpenSBI 1.8.1
  -> Linux 6.18.28
  -> Yocto BusyBox initramfs
```

成功标志：

```text
meta-k230 initramfs starting...
Dropping to shell...
~ #
```

退出 QEMU：

```text
Ctrl-a x
```

## 5. 主流程 B：Buildroot Direct Boot

这条路径使用 K230 SDK 的 Linux 5.10.4 和 Buildroot rootfs，更接近 SDK 用户空间。

```bash
DST="$ASSETS/buildroot/direct-boot"

"$QEMU" \
  -machine k230 \
  -bios "$OPENSBI" \
  -kernel "$DST/Image" \
  -dtb "$DST/k230-qemu.dtb" \
  -initrd "$DST/rootfs.cpio.gz" \
  -append "console=ttyS0,115200 earlycon=sbi cma=0" \
  -nographic \
  -no-reboot
```

启动链：

```text
QEMU OpenSBI 1.8.1
  -> SDK Linux 5.10.4，已改成标准 RISC-V PTE
  -> SDK Buildroot 用户空间
```

预期最终进入 Buildroot 登录流程：

```text
Welcome to Buildroot
canaan login:
```

本机实测已经进入用户空间并启动 `syslogd`、`klogd`、`mdev`、`sshd` 等服务。

下面这些信息不代表启动失败：

```text
Unknown symbol kmem_cache_alloc_trace
ip: SIOCGIFFLAGS: No such device
Starting network: FAIL
```

原因是 SDK rootfs 带有额外厂商模块和网络配置，但当前 QEMU 没有对应设备，或者模块与重建后的 kernel 不完全匹配。

## 6. 主流程 C：U-Boot -> OpenSBI -> Linux

这条路径适合验证 U-Boot 早期初始化、IOMUX、CMU、RMU、WDT、UART 等板级行为。

优先使用 Yocto kernel 和小型 initramfs，减少等待时间：

```bash
COMMON="$ASSETS/common"
DST="$ASSETS/yocto/uboot-boot"
INITRD_END=$(printf "0x%x" \
  $((0x0a100000 + $(stat -c %s "$DST/rootfs.cpio.gz"))))

echo "Use this U-Boot initrd end: $INITRD_END"
```

当前仓库版本计算结果是：

```text
0xa2eec20
```

启动 QEMU：

```bash
"$QEMU" \
  -machine k230 \
  -bios "$COMMON/u-boot" \
  -device loader,file="$COMMON/fw_jump.uImage",addr=0xc100000,force-raw=on \
  -device loader,file="$DST/Image",addr=0x8200000,force-raw=on \
  -device loader,file="$DST/rootfs.cpio.gz",addr=0xa100000,force-raw=on \
  -device loader,file="$DST/k230-canmv.dtb",addr=0xa000000,force-raw=on \
  -nographic \
  -no-reboot
```

U-Boot 会先尝试默认 MMC 启动。当前没有 SD/MMC 后端时会看到：

```text
MMC: no card present
load error
```

这不是 U-Boot 本身启动失败。等待回到 `K230#`，执行：

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi
fdt addr 0xa000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0xa100000>
fdt set /chosen linux,initrd-end <0x0 0xa2eec20>
bootm 0xc100000 - 0xa000000
```

实际使用时，把 `0xa2eec20` 替换成主机命令刚计算出的 `$INITRD_END`。

成功标志：

```text
U-Boot 2022.10
OpenSBI v0.9
Linux version 6.18.28
Dropping to shell...
~ #
```

OpenSBI 0.9 阶段可能打印：

```text
Malformed early option 'earlycon'
```

当前实测串口仍能切换到 `ttyS0`，Linux 和 initramfs 可以继续启动，这条信息不是阻塞错误。

### 6.1 切换为 Buildroot

只需要切换目录、重新计算 initrd 结束地址，并增加 `cma=0`：

```bash
DST="$ASSETS/buildroot/uboot-boot"
INITRD_END=$(printf "0x%x" \
  $((0x0a100000 + $(stat -c %s "$DST/rootfs.cpio.gz"))))

echo "$INITRD_END"
```

QEMU loader 中的 DTB 改为：

```text
$DST/k230-qemu.dtb
```

U-Boot bootargs 改为：

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi cma=0
```

其他地址和 `bootm` 命令不变。

## 7. IOMUX 验证

### 7.1 只启动到 U-Boot

现在不需要先临时编译 U-Boot，可以直接使用镜像仓库里的 SDK U-Boot ELF：

```bash
"$QEMU" \
  -machine k230 \
  -bios "$ASSETS/common/u-boot" \
  -nographic
```

期望看到：

```text
U-Boot 2022.10 (...)
Model: kendryte k230 canmv
DRAM:  512 MiB
...
K230#
```

### 7.2 手动读写 IOMUX

当前 IOMUX 模型覆盖：

```text
0x91105000 - 0x911057ff
```

查看前四个寄存器 reset 值：

```text
md.l 0x91105000 4
```

前几个值应类似：

```text
00000944 00000944 00000929 00000908
```

读写测试：

```text
mw.l 0x91105000 0x12345678 1
md.l 0x91105000 1

mw.l 0x91105004 0xa5a5a5a5 1
md.l 0x91105000 2
```

边界测试：

```text
mw.l 0x911057fc 0x00000abc 1
md.l 0x911057fc 1
```

不要访问 `0x91105800` 验证 IOMUX。该地址已经属于 K230 timer 区间。

### 7.3 检查 unimplemented 日志

非交互式启动：

```bash
timeout 20s "$QEMU" \
  -machine k230 \
  -bios "$ASSETS/common/u-boot" \
  -nographic \
  -d unimp \
  -D "/tmp/k230-uboot-iomux-unimp.log" \
  > "/tmp/k230-uboot-iomux-console.log" 2>&1
```

`timeout` 返回 `124` 是正常的，表示 QEMU 到达 U-Boot 后被主机超时结束。

查看串口：

```bash
sed -n '1,160p' "/tmp/k230-uboot-iomux-console.log"
```

检查 IOMUX：

```bash
rg -n '^iomux:' "/tmp/k230-uboot-iomux-unimp.log"
```

判断：

- IOMUX 实现前：会出现 `iomux: unimplemented device read/write`。
- IOMUX 实现后：不应再出现 `iomux:`。
- `stc`、`cmu`、`pwr`、`sd0`、`sd1`、`gpio0` 等日志属于其他设备。

## 8. 本地 SDK 与预构建镜像差异

### 8.1 SDK 基线相同

本地 SDK：

```text
路径：/home/flamboy/qemu-camp-2026/k230_sdk
Git commit：7e302f733311d284be255f0d81d3463b6ae6ee6d
Git tag：v2.0
```

`k230-boot-assets` 声明的 SDK 来源也是：

```text
k230_sdk v2.0
7e302f733
```

所以本地 SDK 仓库与镜像使用的 SDK 基线没有版本偏差。

### 8.2 产物并非原样复制

虽然 SDK 基线相同，但镜像作者做了 QEMU 适配：

| 组件 | 本地 SDK 源码 | 镜像仓库产物 | 差异 |
|------|--------------|--------------|------|
| Linux | 5.10.4 | Buildroot `Image` 也是 5.10.4 | 镜像内核改成标准 RISC-V PTE，去掉对 T-HEAD MAEE 的依赖 |
| U-Boot | 2022.10 | `common/u-boot` 也是 2022.10 | 镜像仓库直接提供可用于 QEMU `-bios` 的 ELF |
| OpenSBI | 0.9 | `common/fw_jump.uImage` 内是 0.9 | 已封装成 U-Boot `bootm` 可识别的 legacy uImage |
| SDK DTB | 描述真实 K230 外设 | `k230-qemu.dtb` 只有 1602 字节 | 只保留当前 QEMU 已建模的 CPU、内存、CLINT、PLIC、UART |
| rootfs | 需要本地完整构建 | 已提供 `rootfs.cpio.gz` | 可以直接作为 initramfs 使用 |

最关键的两项差异：

```text
原始 SDK Linux Image 不能直接假定可在通用 QEMU RISC-V MMU 下运行。
原始 SDK k230.dtb 不能直接用于当前裁剪过的 k230 machine。
```

因此 Buildroot 流程必须优先使用：

```text
k230-boot-assets/buildroot/*/Image
k230-boot-assets/buildroot/*/k230-qemu.dtb
```

不要把镜像命令中的 `k230-qemu.dtb` 随意换回原始 `k230.dtb`。

### 8.3 Yocto 镜像不是 SDK 产物

Yocto 路径来自独立的 BSP 构建：

```text
Yocto scarthgap 5.0.17
Linux 6.18.28
BusyBox 1.36.1
```

它适合快速验证 QEMU machine、OpenSBI、PLIC、CLINT、UART 和 Linux 基础启动，不代表 K230 SDK 用户空间。

### 8.4 本地 SDK 当前没有输出产物

本机 `k230_sdk` 当前没有：

```text
output/k230_canmv_defconfig/images/little-core/Image
output/k230_canmv_defconfig/images/little-core/k230.dtb
output/k230_canmv_defconfig/images/little-core/rootfs.cpio.gz
output/k230_canmv_defconfig/images/little-core/fw_jump.bin
output/k230_canmv_defconfig/little/uboot/u-boot
```

这不再阻塞日常 QEMU 启动，因为对应预构建产物已经由 `k230-boot-assets` 提供。

本地 SDK 仍然有价值，主要用于：

- 查 Linux、U-Boot、OpenSBI、驱动和 DTS 源码。
- 修改内核配置或驱动后重新生成定制镜像。
- 对照 QEMU unimplemented 日志定位真实寄存器访问。
- 研究真实 SD/QSPI/BootROM/SPL 启动链。

## 9. 当前能力边界

### 9.1 已经能验证

- QEMU direct Linux boot。
- SDK U-Boot M-mode 初始化。
- U-Boot `bootm` 启动 OpenSBI/Linux。
- Linux little-core 最小启动。
- IOMUX、UART、WDT、PLIC、CLINT 等当前模型行为。
- SPI/QSPI 控制器 probe、SPI NOR 和只读 XIP window 的当前实现。

### 9.2 仍然不是完整真实启动

当前 QEMU 已支持通过下面的方式给 SPI NOR 提供后端：

```bash
"$QEMU" \
  -machine k230 \
  -drive file="flash.bin",format=raw,if=mtd \
  -nographic
```

但这只表示 SPI NOR 和 `0xC0000000` 只读 flash/XIP window 有数据来源，不表示 SDK 已经能自动走通：

```text
BootROM
  -> SPL
  -> QSPI/SD/eMMC 读镜像
  -> 初始化 DDR
  -> 解包并搬运固件
  -> OpenSBI
  -> Linux
```

当前 U-Boot Linux 主流程仍然用 `-device loader` 绕过存储读取和解包阶段。

## 10. 遗留流程：临时编译 U-Boot

只有以下情况才需要继续使用这套流程：

- 修改了本地 SDK U-Boot 源码。
- 需要验证不同 U-Boot 配置。
- 需要追踪预构建 U-Boot 与本地源码的差异。

复制一份 SDK U-Boot 源码到 `/tmp`：

```bash
rm -rf "/tmp/k230-uboot-src" "/tmp/k230-uboot-build"
cp -a "$SDK/src/little/uboot" "/tmp/k230-uboot-src"
```

系统自带 `riscv64-linux-gnu-gcc` 不认识玄铁私有 CSR 名称。临时验证时，把 CSR 名称替换成 SDK 定义的数字编号：

```bash
sed -i 's/csrs[[:space:]]\+mcor,\(.*\)$/csrs\t0x7c2,\1/' \
  "/tmp/k230-uboot-src/arch/riscv/cpu/start.S"

sed -i 's/csrr %0, mhcr/csrr %0, 0x7c1/' \
  "/tmp/k230-uboot-src/arch/riscv/cpu/k230/cache.c"

sed -i 's/csrw mhcr, %0/csrw 0x7c1, %0/' \
  "/tmp/k230-uboot-src/arch/riscv/cpu/k230/cache.c"
```

配置并编译：

```bash
make -C "/tmp/k230-uboot-src" \
  O="/tmp/k230-uboot-build" \
  ARCH=riscv \
  CROSS_COMPILE=riscv64-linux-gnu- \
  k230_canmv_defconfig

make -C "/tmp/k230-uboot-src" \
  O="/tmp/k230-uboot-build" \
  ARCH=riscv \
  CROSS_COMPILE=riscv64-linux-gnu- \
  ARCH_C=c_zicsr_zifencei \
  -j"$(nproc)"
```

后期 SPL 打包可能因为缺少 Python `Crypto` 模块停止：

```text
ModuleNotFoundError: No module named 'Crypto'
```

主 U-Boot ELF 通常已经生成：

```bash
ls -l "/tmp/k230-uboot-build/u-boot"
```

启动：

```bash
"$QEMU" \
  -machine k230 \
  -bios "/tmp/k230-uboot-build/u-boot" \
  -nographic
```

## 11. 遗留记录：IOMUX 实现前后

IOMUX 还是 `unimplemented-device` 时，SDK U-Boot 初始化阶段会批量访问 `0x91105000` 附近寄存器，曾产生约 256 行日志：

```text
iomux: unimplemented device read  (size 4, offset 0x000)
iomux: unimplemented device write (size 4, offset 0x000, value 0x000002c4)
iomux: unimplemented device read  (size 4, offset 0x004)
iomux: unimplemented device write (size 4, offset 0x004, value 0x000002c4)
```

实现 `k230_iomux` 后，2026-07-06 复测仍能进入：

```text
U-Boot 2022.10 (Jul 06 2026 - 17:49:44 +0800)

CPU:   rv64imafdcvsu
Model: kendryte k230 canmv
DRAM:  512 MiB
Core:  25 devices, 13 uclasses, devicetree: embed
MMC:   mmc0@91580000: 0, mmc1@91581000: 1
...
K230#
```

`-d unimp` 中不再出现 `iomux:`。剩余日志来自其他尚未完全实现的设备。

## 12. 与 k230.rst 的关系

QEMU 当前权威启动参数仍以：

```text
qemu-camp-2026-k230/docs/system/riscv/k230.rst
```

为准。

本文在它的基础上增加了：

- 可直接使用的预构建镜像路径。
- 当前工作区的完整环境变量。
- 三条实际验证过的启动命令。
- IOMUX 验证步骤。
- 本地 SDK 与镜像产物的差异说明。
