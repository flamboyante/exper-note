# K230 U-Boot 真实路径 IOMUX unimplemented 复现

## 目的

验证在还没有实现 K230 IOMUX 模型、仍然使用 `create_unimplemented_device("iomux", ...)` 的情况下，真实 U-Boot 启动路径是否会访问 IOMUX。

结论：会。U-Boot 初始化阶段会通过 pinctrl 路径批量读写 `0x91105000` 附近的 IOMUX 寄存器，QEMU 会打印 `iomux: unimplemented device read/write`。

## 前置状态

QEMU 仓库已能编译 K230：

```bash
ninja -C "build" qemu-system-riscv64
```

K230 SDK 源码位于：

```bash
/home/flamboy/k230_sdk
```

当前 SDK 没有现成 `output/.../little/uboot/u-boot` 产物，所以这里用 `/tmp` 临时构建一个 U-Boot。临时构建不会修改 QEMU 仓库。

## 临时构建 U-Boot

先复制一份 SDK U-Boot 源码到 `/tmp`：

```bash
rm -rf "/tmp/k230-uboot-src" "/tmp/k230-uboot-build"
cp -a "/home/flamboy/k230_sdk/src/little/uboot" "/tmp/k230-uboot-src"
```

系统自带 `riscv64-linux-gnu-gcc` 对较新的 RISC-V ISA 拆分更严格，并且不认识玄铁私有 CSR 名称。为了临时验证启动路径，把两个 CSR 名称改成 SDK 自己 `csr.h` 中定义的数字编号：

```bash
sed -i 's/csrs[[:space:]]\\+mcor,\\(.*\\)$/csrs\t0x7c2,\\1/' \
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

这个构建后期可能会因为缺少 Python `Crypto` 模块，在打包 SPL 时停止：

```text
ModuleNotFoundError: No module named 'Crypto'
```

这不影响本次验证。我们需要的是主 U-Boot ELF：

```bash
ls -l "/tmp/k230-uboot-build/u-boot"
```

## 启动到 U-Boot 并打开 unimp 日志

运行：

```bash
timeout 20s "build/qemu-system-riscv64" \
  -machine k230 \
  -bios "/tmp/k230-uboot-build/u-boot" \
  -nographic \
  -d unimp \
  -D "/tmp/k230-uboot-unimp.log" \
  > "/tmp/k230-uboot-console.log" 2>&1
```

说明：

- `-bios /tmp/k230-uboot-build/u-boot`：从 M-mode U-Boot 启动。
- `-nographic`：串口输出走终端/日志。
- `-d unimp`：打开 unimplemented 设备访问日志。
- `-D /tmp/k230-uboot-unimp.log`：把 unimp 日志写到文件。
- `timeout 20s`：到达 U-Boot prompt 后自动结束 QEMU。退出码 `124` 是 timeout 的正常结果。

查看串口是否到达 U-Boot：

```bash
sed -n '1,120p' "/tmp/k230-uboot-console.log"
```

期望能看到类似：

```text
U-Boot 2022.10 (...)
Model: kendryte k230 canmv
...
K230#
```

查看 IOMUX 访问：

```bash
rg -n "^iomux:" "/tmp/k230-uboot-unimp.log" | sed -n '1,40p'
```

统计 IOMUX 访问次数：

```bash
rg -c "^iomux:" "/tmp/k230-uboot-unimp.log"
```

本次实测结果是 `256` 行，开头类似：

```text
iomux: unimplemented device read  (size 4, offset 0x000)
iomux: unimplemented device write (size 4, offset 0x000, value 0x000002c4)
iomux: unimplemented device read  (size 4, offset 0x004)
iomux: unimplemented device write (size 4, offset 0x004, value 0x000002c4)
```

## 说明

这说明 IOMUX 不是“没人调用”。至少 SDK U-Boot 的真实启动路径会访问 IOMUX，并且访问点很早，在进入 `K230#` 前就发生。

当前 IOMUX 还是 `unimplemented-device`，所以：

- 读会返回 `0`，并打印 `unimplemented device read`。
- 写不会保存值，只打印 `unimplemented device write`。
- 实现 IOMUX 后，这些日志应消失，寄存器应能读回写入值。

这条路径可以作为 IOMUX 第一阶段模型的真实验证：跑同样的 U-Boot 启动命令，确认不再出现 `iomux: unimplemented device ...`，并且 U-Boot 仍能到达 `K230#`。

## 2026-07-06：实现 K230 IOMUX 后复测

当前 QEMU 已把 K230 IOMUX 从 `create_unimplemented_device("iomux", ...)` 替换为真实 MMIO 寄存器模型。

复测目标：

```text
1. U-Boot 仍能启动到 K230#。
2. -d unimp 日志里不再出现 iomux: unimplemented device read/write。
3. 如果还有 unimplemented 日志，确认它们来自其他暂未实现设备。
```

### 编译 U-Boot

当前 `/home/flamboy/k230_sdk` 仍没有 `output/k230_canmv_defconfig` 产物，所以继续临时构建 U-Boot：

```bash
rm -rf /tmp/k230-uboot-src /tmp/k230-uboot-build
cp -a /home/flamboy/k230_sdk/src/little/uboot /tmp/k230-uboot-src

sed -i 's/csrs[[:space:]]\+mcor,\(.*\)$/csrs\t0x7c2,\1/' \
  /tmp/k230-uboot-src/arch/riscv/cpu/start.S

sed -i 's/csrr %0, mhcr/csrr %0, 0x7c1/' \
  /tmp/k230-uboot-src/arch/riscv/cpu/k230/cache.c

sed -i 's/csrw mhcr, %0/csrw 0x7c1, %0/' \
  /tmp/k230-uboot-src/arch/riscv/cpu/k230/cache.c

make -C /tmp/k230-uboot-src \
  O=/tmp/k230-uboot-build \
  ARCH=riscv \
  CROSS_COMPILE=riscv64-linux-gnu- \
  k230_canmv_defconfig

make -C /tmp/k230-uboot-src \
  O=/tmp/k230-uboot-build \
  ARCH=riscv \
  CROSS_COMPILE=riscv64-linux-gnu- \
  ARCH_C=c_zicsr_zifencei \
  -j"$(nproc)"
```

本次编译仍在 SPL 后处理阶段因为缺少 Python `Crypto` 模块停止：

```text
ModuleNotFoundError: No module named 'Crypto'
```

但是主 U-Boot ELF 已经生成：

```text
/tmp/k230-uboot-build/u-boot
```

### 运行 QEMU

```bash
timeout 20s build/qemu-system-riscv64 \
  -machine k230 \
  -bios /tmp/k230-uboot-build/u-boot \
  -nographic \
  -d unimp \
  -D /tmp/k230-uboot-iomux-unimp.log \
  > /tmp/k230-uboot-iomux-console.log 2>&1
```

查看串口：

```bash
sed -n '1,180p' /tmp/k230-uboot-iomux-console.log
```

实测能到 U-Boot prompt：

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

检查 IOMUX unimplemented 访问：

```bash
rg -n "iomux|unimplemented device|K230#|U-Boot|Model|pinctrl" \
  /tmp/k230-uboot-iomux-unimp.log \
  /tmp/k230-uboot-iomux-console.log
```

结果：

```text
没有 iomux: unimplemented device read/write
```

这说明 U-Boot 真实启动路径中的 IOMUX 访问已经命中 `k230_iomux` 真实模型。

### 剩余 unimplemented 访问

本次 unimp 日志仍有 41 行：

```bash
wc -l /tmp/k230-uboot-iomux-unimp.log /tmp/k230-uboot-iomux-console.log
```

结果：

```text
  41 /tmp/k230-uboot-iomux-unimp.log
  25 /tmp/k230-uboot-iomux-console.log
  66 total
```

剩余访问来自其他暂未实现设备：

```text
stc
cmu
pwr
hi_sys_cfg
sd0
sd1
gpio0
```

示例：

```text
stc: unimplemented device write (size 4, offset 0x020, value 0x00000001)
cmu: unimplemented device write (size 4, offset 0x004, value 0x80199805)
hi_sys_cfg: unimplemented device read  (size 4, offset 0x07c)
sd0: unimplemented device read  (size 4, offset 0x040)
gpio0: unimplemented device read  (size 4, offset 0x004)
```

结论：

```text
U-Boot IOMUX 复测通过。

实现前：
  -d unimp 中有 iomux: unimplemented device read/write。

实现后：
  U-Boot 仍能到 K230#。
  -d unimp 中没有 iomux:。
  剩余 unimp 来自其他设备，不是 IOMUX。
```

## Linux 路径当前状态

`docs/system/riscv/k230.rst` 里的 Linux direct boot 需要 SDK 输出产物：

```text
/home/flamboy/k230_sdk/output/k230_canmv_defconfig/images/little-core/Image
/home/flamboy/k230_sdk/output/k230_canmv_defconfig/images/little-core/k230.dtb
/home/flamboy/k230_sdk/output/k230_canmv_defconfig/images/little-core/rootfs.cpio.gz
```

U-Boot boot Linux 还需要：

```text
/home/flamboy/k230_sdk/output/k230_canmv_defconfig/images/little-core/fw_jump.bin
/home/flamboy/k230_sdk/output/k230_canmv_defconfig/little/uboot/u-boot
```

当前本机 `/home/flamboy/k230_sdk` 没有 `output/k230_canmv_defconfig`，也没有找到：

```text
Image
k230.dtb
rootfs.cpio.gz
fw_jump.bin
```

所以这次不能按 rst 实测 Linux 启动。

这不是 IOMUX 模型本身的问题，而是缺少 SDK 构建产物。等 SDK 生成 `output/k230_canmv_defconfig` 后，再按 `docs/system/riscv/k230.rst` 的 direct Linux boot 或 U-Boot boot Linux 命令继续验证。
