# K230 SPI/QSPI V2 实施路线

首次记录：2026-07-28

修订：2026-07-30（同步 Step 4 Plan Final V1.3 最小消费者范围）

本文是 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 第 13 节的实施细化，不重复架构结论，只规定执行顺序、检查点和注意点。

## 当前检查点（2026-07-30）

| 项目 | 当前状态 |
|---|---|
| V2 开发分支 | `k230-V2-patch-spi` |
| 当前 HEAD | `c689ac865f` |
| Step 1 / Step 2 | 已完成并通过编译与 K230 SSI qtest |
| Step 3 | 已完成 HI_SYS 反向依赖解耦；通用 SSI 使用 `xip-enable` GPIO input |
| Step 4 Plan Final | [Standard PIO 第一批范围 V1.3](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.3.md) 已完成；Step 4.0 已实施并通过 12 项 K230 SSI qtest |

Step 4.1 至 Step 4.4 直接形成第一批上游投稿终态：只保留有当前消费者的 Standard PIO/FIFO、七路基础 IRQ、K230 三实例/PLIC 和 Standard 1-1-1 SPI NOR。第一批不保留内部 future bit、DMA layout、DONE/AXIE、XIP GPIO 或额外 MMIO 资源。

下文第一步至第三步保留最初路线和历史检查点；执行 Step 4 时以专门计划中的配置矩阵、TDD 顺序和命令为准。

## 1. 前提核实

开工前已核实以下前提：

| 前提 | 核实结果 |
|---|---|
| 基线 commit `1342f13a6e` | 在 `k230-spiv3.4` 分支顶端，与 origin 同步 |
| 文件命名 | `hw/ssi/k230_dw_ssi.c` + `include/hw/ssi/k230_dw_ssi.h` 仍在原命名 |
| HI_SYS 反向依赖 | `k230_dw_ssi_set_hi_sys` 定义在 `k230_dw_ssi.c:355`，唯一调用点 `hw/riscv/k230.c:266`，且只对 `logical_index==0`（SPI-OPI 实例）连接 |
| qtest | `tests/qtest/k230-dw-ssi-test.c` 存在 |
| build binary | `build/qemu-system-riscv64` 存在（7/24 构建，需确认是否对应当前 HEAD） |
| `.codegraph/` | 位于 `qemu-camp-2026-k230/.codegraph/`，含 311 MB `codegraph.db`，未跟踪，不清理 |

## 2. 五步实施路线

核心原则：分刀推进，每一刀只做一类改动，保持中间态可编译可回归。不在 v3.4 顶部追加单一重构 patch，最终重组为 11 个自洽提交。

### 第一步：冻结 v3.4 基线

对当前 `k230-spiv3.4` 做一次基线验证：

- 记录当前提交 `1342f13a6e`
- 重新 `ninja -C build qemu-system-riscv64`（确保 binary 对应当前 HEAD，而非旧构建）
- 运行现有 K230 SSI qtest
- 运行 `git diff --check`
- 保存测试结果，作为 v2 回归基线

`.codegraph/` 是未跟踪目录，不要删除或清理。

基线验证应记下确切命令和输出摘要，后续每一步都与此基线对比。

### 第二步：只做"行为不变"的通用化

第一刀不要同时引入 capability、XIP GPIO 和新属性，只做结构整理：

- `k230_dw_ssi.c` → `dw_ssi.c`
- `k230_dw_ssi.h` → `dw_ssi.h`
- `K230DwSsiState` → `DwSsiState`
- `TYPE_K230_DW_SSI` → `TYPE_DW_SSI`
- `k230_dw_ssi_*()` → `dw_ssi_*()`
- 更新 machine、HI_SYS、tests、trace 和 Meson/Kconfig
- 保持当前所有运行行为不变

这一刀的目标只有一个：证明当前控制器代码可以脱离 K230 命名空间编译和运行。

### 第三步：解除 HI_SYS 反向依赖

当前 CodeGraph 已确认：

- `k230_dw_ssi_set_hi_sys()` 只有 `k230_soc_realize()` 一个调用点；
- SSI 头文件直接保存 `K230HiSysState *`；
- XIP 路径因此由通用 SSI 反向依赖 K230 HI_SYS。

这一阶段改为：

- SSI 提供通用 `xip-enable` GPIO input；
- HI_SYS 负责驱动这个信号；
- `dw_ssi_get_spi_mode()` 和 `dw_ssi_is_sleeping()` 作为通用 getter；
- 通用 SSI 不再 include 任何 K230 头文件。

### 第四步：再引入实例配置

行为边界稳定后，按 [Step 4 Plan Final V1.3](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.3.md) 分五个小目标实施：

1. Step 4.0 已完成：修正 ordinary enhanced/IDMA 对 XIP mode fields 的错误消费，普通事务固定为 instruction → address → dummy → data；
2. 建立最小 `DwSsiConfig`、三项 properties、动态 FIFO/CS、通用测试机、Standard PIO 四种 TMOD、写保护/状态契约和三项配置迁移 equality；
3. 注册并实现 TXE/TXO/RXF/RXO/TXU/RXU/MST 七路基础 IRQ；
4. 在 K230 machine 中应用三实例最小 profile、映射 region 0、连接七路 PLIC，并删除第一批 XIP 接口；
5. 挂接 Standard 1-1-1 Flash，完成构建、通用/K230 qtest、公共头文件、迁移边界和 patch 归属检查。

第一批 property 集合为：

- `fifo-depth`
- `num-cs`
- `imr-reset`

`DwSsiConfig` 第一批只包含 `num_cs`、`fifo_depth`、`imr_reset`。`max-lines`、internal IDMA、enhanced、XIP properties 和内部状态分别随对应后续功能 series 引入；第一批不创建 XIP GPIO 或第二个 sysbus MMIO region。

K230 的 `0x04c/0x050/0x054` 第一批只按 internal-AXI `DMACR/AXIAWLEN/AXIARLEN` 版图识别并固定 RAZ/WI。external `DMATDLR/DMARDLR` 布局不进入当前 V2 路线，未来出现真实消费者后再单独评估。

XIP 寄存器边界按 TRM 最终裁决：`XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 均为 FMC XIP 专用寄存器，第一批统一 RAZ/WI。TXU 属于基础 TX FIFO underflow IRQ；DONE/AXIE output 和状态全部随 IDMA series 引入。

### 第五步：最后重组第一批 5 个提交

最终不能只在 3.4 顶部追加一个“大重构 patch”。第一批按通用 Standard PIO、基础 IRQ、K230 实例、PLIC、Standard Flash 重组为 5 个提交；enhanced、internal IDMA、XIP 使用独立 follow-up series。external DMA 不在当前 V2 路线。

## 3. 执行注意点

### 3.1 基线 binary 时效性

`build/qemu-system-riscv64` 是 7/24 构建的，但 HEAD 是 `1342f13a6e`（trace events patch）。如果 trace patch 之后没重建，binary 是旧的。基线验证时必须重新 `ninja` 一次再跑 qtest，否则基线不准。

### 3.2 第二步重命名的隐藏点

`git mv` + 全局 `sed` 之外，容易漏的几处：

- `hw/ssi/meson.build` 和 `hw/ssi/Kconfig` 里的 `k230_dw_ssi` → `dw_ssi`
- `hw/ssi/trace-events` 里 `k230_dw_ssi_*` → `dw_ssi_*`
- `tests/qtest/meson.build` 里 `k230-dw-ssi-test` 的依赖（测试文件名本身可保留 K230 前缀，因为它测的是 K230 集成）
- `include/hw/riscv/k230.h` 里的 `K230DwSsiState` 字段类型
- VMState 名称字符串（`vmstate_k230_dw_ssi` 之类的字符串 ID 要改成 `dw_ssi`，否则 migration 会断——K230 当前大概率不在意 migration 兼容，但上游会看）

### 3.3 第三步 HI_SYS 解耦的具体形态

当前 `hw/riscv/k230.c:266` 的调用上下文：

```c
k230_hi_sys_set_ssi(&s->hi_sys, logical_index,
                    &s->dw_ssi[route->ssi_index]);
if (logical_index == 0) {
    k230_dw_ssi_set_hi_sys(&s->dw_ssi[route->ssi_index],
                           &s->hi_sys);
}
```

当前逻辑：

- `k230_hi_sys_set_ssi()` 对所有三实例建立 HI_SYS → SSI 的反向引用；
- `k230_dw_ssi_set_hi_sys()` 只对 `logical_index==0`（SPI-OPI）建立 SSI → HI_SYS 的正向引用。

解耦后：

- HI_SYS → SSI：保留 `k230_hi_sys_set_ssi()`（HI_SYS 本来就是 K230 设备，引用通用 SSI 通过 getter，方向正确）；
- SSI → HI_SYS：删掉 `k230_dw_ssi_set_hi_sys()`，改成 SSI 暴露 `xip-enable` GPIO input，machine 里 `qdev_connect_gpio_in()` 把 HI_SYS 的 output 接到 SPI-OPI 实例的 input。

注意：当前只对 `logical_index==0` 连 HI_SYS，意味着只有 SPI-OPI 实例需要 `xip-enable` input。QSPI0/1 不连这个 GPIO，`xip_enabled` 恒为 false——这正好和"只有 SPI-OPI 才有 XIP aperture"吻合，验证了决策文档第 5 节"XIP region 大小由 SoC 配置、映射由 machine 决定"的边界。

### 3.4 第四步属性化的顺序

执行顺序已经在 Step 4 最终计划中收敛，不再按单个 property 零散提交：

1. Step 4.0 已完成：解除普通 enhanced/IDMA 与 XIP 的错误耦合，并增加 `0xeb` 四阶段回归；
2. Step 4.1 建立最小配置、Standard PIO/FIFO、四种 TMOD、基础 VMState 和通用 qtest；
3. Step 4.2 增加七路基础 IRQ 和通用 IRQ qtest；
4. Step 4.3 应用 K230 三实例最小 profile、region 0 和七路 PLIC 路由，并删除 XIP 接口；
5. Step 4.4 挂接 Standard 1-1-1 Flash，执行完整构建、qtest、公共头文件、依赖残留和 patch 归属检查。

enhanced、internal IDMA、XIP property、IRQ、GPIO 和额外 MMIO 资源必须留到对应后续 series；Step 4.1 至 Step 4.3 删除当前中间态的未来接口，不提前预埋内部位图或 helper。external DMA 不在当前 V2 路线。

### 3.5 分支命名

`k230-spiv3.4` 仍是 V1 基线；V2 当前工作位于已存在的 `k230-V2-patch-spi` 分支。未经用户明确要求，不创建新分支、不提交、不推送。Step 4 计划只描述未来源码改动和验证，不改变当前分支状态。

## 4. 每步检查点

| 步骤 | 检查项 |
|---|---|
| 第一步 | `ninja` 成功；qtest 全过；`git diff --check` 无报错；记录 commit hash 和测试输出 |
| 第二步 | `ninja` 成功；qtest 结果与基线一致；`grep -r k230_dw_ssi hw/ssi/ include/hw/ssi/` 无残留（测试文件名除外）；`grep -r K230DwSsiState` 无残留 |
| 第三步 | `ninja` 成功；qtest 结果与基线一致；`grep 'k230_hi_sys.h' hw/ssi/` 无结果；`grep 'K230HiSysState' include/hw/ssi/` 无结果；XIP 相关 qtest 场景仍通过 |
| 第四步 | Step 4.0 纠错已完成；Step 4.1 Standard PIO/FIFO；Step 4.2 七路 IRQ；Step 4.3 K230 三实例/PLIC；Step 4.4 Standard Flash 与收敛；每步有定向 qtest |
| 第五步 | 第一批 5 个 patch 各自 checkout 后 `ninja` 成功；`git diff --check` 和 `checkpatch` 无报错；qtest 全过；cover letter 更新 |

## 6. 验证命令合集

本节积累 V2 重构过程中用到的所有验证命令，按用途分组。所有命令默认在 QEMU 源码根目录 qemu-camp-2026-k230/ 下执行，build 目录为 build/。

### 6.1 环境信息

```bash
git log --oneline -1
git branch --show-current
head -5 build/config-host.mak
build/qemu-system-riscv64 --version | head -1
```

### 6.2 编译

```bash
# 增量编译 QEMU（最常用）
ninja -C build qemu-system-riscv64

# 编译 qtest 二进制
ninja -C build tests/qtest/dw-ssi-test
ninja -C build tests/qtest/k230-dw-ssi-test
ninja -C build tests/qtest/k230-wdt-test

# 全量编译（改动较大时）
ninja -C build
```

编译成功标准：无错误退出，build/qemu-system-riscv64 时间戳更新。
### 6.3 qtest 运行

```bash
# 运行通用 DW SSI qtest
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/dw-ssi-test -v

# 直接运行 K230 SSI qtest（带详细输出）
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v

# 运行 K230 WDT qtest
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-wdt-test -v

# 列出 SSI qtest 的测试路径；只运行一个用例时，使用 -p TEST_PATH -v
build/tests/qtest/k230-dw-ssi-test -l
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -p "/TEST_PATH" -v

# 通过 meson 运行 Step 4 相关 qtest
meson test -C build dw-ssi-test k230-dw-ssi-test k230-wdt-test -v

# 运行全部 riscv64 qtest
meson test -C build --suite qtest-riscv64-softmmu -v
```

参数说明：`-v` 显示详细输出；`-l` 列出测试路径；`-p TEST_PATH` 按路径过滤单个或一组用例，不能单独使用。`QTEST_QEMU_BINARY` 指定被测 QEMU 二进制路径。

成功标准：所有测试用例 PASS，无 FAIL/ERROR。

### 6.4 代码规范检查

```bash
# git 行尾空白检查
git diff --check

# checkpatch 单 patch 检查
scripts/checkpatch.pl -f <(git format-patch -1 HEAD)

# checkpatch 检查整个文件
scripts/checkpatch.pl -f hw/ssi/dw_ssi.c
scripts/checkpatch.pl -f include/hw/ssi/dw_ssi.h
```
### 6.5 残留检查（第二步重命名后）

```bash
# 确认旧命名无残留（测试文件名除外）
grep -rn k230_dw_ssi hw/ssi/ include/hw/ssi/ hw/riscv/ hw/misc/ 2>/dev/null
grep -rn K230DwSsiState hw/ include/ 2>/dev/null
grep -rn TYPE_K230_DW_SSI hw/ include/ 2>/dev/null
grep -rn k230_dw_ssi_ hw/ include/ 2>/dev/null
# 预期：以上均无输出（或仅 tests/qtest/k230-dw-ssi-test.c 文件名命中）
```

### 6.6 反向依赖检查（第三步解耦后）

```bash
# 通用 SSI 不再 include K230 头文件
grep -rn k230_hi_sys.h hw/ssi/ include/hw/ssi/ 2>/dev/null
grep -rn K230HiSysState hw/ssi/ include/hw/ssi/ 2>/dev/null
grep -rn k230_hi_sys hw/ssi/ include/hw/ssi/ 2>/dev/null
# 预期：以上均无输出
```
### 6.7 属性化验证（第四步每组属性后）

```bash
ninja -C build qemu-system-riscv64
ninja -C build tests/qtest/dw-ssi-test
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v

# 确认第一批属性在 machine 中正确设置
rg -n 'fifo-depth|num-cs|imr-reset' hw/riscv/k230.c

# 确认未来接口未进入第一批
rg -n 'max-lines|dma-register-layout|has-enhanced-spi|has-idma|has-xip|xip-window-size' \
  hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h hw/riscv/k230.c
# 预期：无第一批接口或状态命中
```

### 6.8 单 commit 验证（第五步重组时）

```bash
# 针对某个中间 commit 验证可编译性
git checkout COMMIT_HASH
ninja -C build qemu-system-riscv64
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
git diff --check
scripts/checkpatch.pl -f <(git format-patch -1 HEAD)
git checkout k230-spiv3.4  # 回到开发分支
```

### 6.9 基线对比

```bash
# 记录基线 qtest 输出（第一步执行）
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v > /tmp/baseline_qtest.log 2>&1

# 后续每步与基线对比
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v > /tmp/current_qtest.log 2>&1
diff /tmp/baseline_qtest.log /tmp/current_qtest.log
```

成功标准：diff 无输出（行为完全一致）。

## 7. 与决策文档的关系

本文档是 V2 决策记录第 13 节的执行细化。架构边界、证据标准和 patch 归属以决策文档为准，本文档只规定执行顺序和检查点。如果实施中发现边界问题，先更新决策文档，再回头调整本文档。
