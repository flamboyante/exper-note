# K230 V2 第三步：解除 HI_SYS 反向依赖 — 实施细化

首次记录：2026-07-28
修订：2026-07-28（v2，应用 Review 意见 R1-R11）

> **修订说明**：本文档经 Review 后修订。修改点用 ~~删除线~~ 保留原文，紧随其后的 **加粗** 或引用块为新内容。修订编号 R1-R11 对应 Review 意见。

本文是 [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md) 第三步"解除 HI_SYS 反向依赖"的执行细化。所有文件路径与行号基于实际代码探索（基线 commit `1342f13a6e`，分支 `k230-spiv3.4`），代码仓库位于 `my-qemu-camp-2026-k230/`。

> **命名约定**：第二步重命名后 `k230_dw_ssi.*` → `dw_ssi.*`、`K230DwSsiState` → `DwSsiState`。为便于对照当前代码，本文在"当前状态分析"中仍使用旧名，在"提议改动"中使用新名。

---

## 摘要

将 SSI 控制器对 HI_SYS 的**反向依赖**（SSI 持有 HI_SYS 指针并调用 `k230_hi_sys_xip_enabled()`）改造为**通用 GPIO input** 模式。解耦后 SSI 不再 include 任何 K230 头文件，成为纯粹的 Synopsys DesignWare SSI 通用模型；HI_SYS 作为 K230 专有系统控制器，通过 GPIO output 驱动 XIP 使能信号。

---

## 当前状态分析

### 依赖方向

```
┌─────────────────────────────────────────────────────────┐
│  正向依赖（HI_SYS → SSI）—— 保留                         │
│  HI_SYS 持有 K230DwSsiState *ssi[3]                     │
│  读取 SSI_CTRL 时调用 getter 组合状态：                   │
│    • k230_dw_ssi_get_spi_mode(s->ssi[i])                │
│    • k230_dw_ssi_is_sleeping(s->ssi[i])                 │
│  建立: k230_hi_sys_set_ssi() × 3（全部实例）              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  反向依赖（SSI → HI_SYS）—— 消除，改为 GPIO              │
│  SSI 持有 K230HiSysState *hi_sys                        │
│  XIP read 时调用:                                        │
│    • k230_hi_sys_xip_enabled(s->hi_sys)  [k230_dw_ssi.c:751] │
│  建立: k230_dw_ssi_set_hi_sys() × 1（仅 SPI-OPI 实例）    │
└─────────────────────────────────────────────────────────┘
```

### 反向依赖的唯一运行时调用点

`hw/ssi/k230_dw_ssi.c:743-753`：

```c
static uint64_t k230_dw_ssi_xip_read(void *opaque, hwaddr address,
                                     unsigned int size)
{
    K230DwSsiState *s = K230_DW_SSI(opaque);
    ...
    if (!s->hi_sys || !k230_hi_sys_xip_enabled(s->hi_sys)) {
        return 0;                        // XIP 未使能，读返回 0
    }
    ...                                  // 执行 SPI 增强读事务
}
```

`xip_enabled` 状态实际存储在 HI_SYS 的 `ssi_ctrl` 寄存器 bit 0（`K230_SSI_CTRL_XIP_EN`），SSI 每次读 XIP 窗口时反向查询。该状态**只影响 XIP memory region 的读行为**，不写入 SSI 任何寄存器、不影响中断、不影响 PIO 事务。

### 耦合点清单（7 处）

| # | 文件 | 行号 | 当前代码 | 解耦动作 |
|---|------|------|----------|----------|
| C1 | `hw/ssi/k230_dw_ssi.c` | 22 | `#include "hw/misc/k230_hi_sys.h"` | 删除 |
| C2 | `include/hw/ssi/k230_dw_ssi.h` | 35 | `typedef struct K230HiSysState K230HiSysState;` | 删除 |
| C3 | `include/hw/ssi/k230_dw_ssi.h` | 86 | `K230HiSysState *hi_sys;` | 删除，新增 `bool xip_enabled;` |
| C4 | `include/hw/ssi/k230_dw_ssi.h` | 109 | `void k230_dw_ssi_set_hi_sys(...)` | 删除声明 |
| C5 | `hw/ssi/k230_dw_ssi.c` | 355-358 | `k230_dw_ssi_set_hi_sys()` 定义 | 删除 |
| C6 | `hw/ssi/k230_dw_ssi.c` | 751 | `if (!s->hi_sys \|\| !k230_hi_sys_xip_enabled(s->hi_sys))` | 改为 `if (!s->xip_enabled)` |
| C7 | `hw/riscv/k230.c` | 265-268 | `k230_dw_ssi_set_hi_sys(...)` 调用 | 删除，改为 GPIO 连接 |

### 当前 machine 连接代码（`hw/riscv/k230.c:259-273`）

```c
for (size_t logical_index = 0;
     logical_index < ARRAY_SIZE(k230_ssi_routes); logical_index++) {
    const K230SsiRoute *route = &k230_ssi_routes[logical_index];

    k230_hi_sys_set_ssi(&s->hi_sys, logical_index,
                        &s->dw_ssi[route->ssi_index]);   // 正向：全部 3 个
    if (logical_index == 0) {
        k230_dw_ssi_set_hi_sys(&s->dw_ssi[route->ssi_index],
                               &s->hi_sys);               // 反向：仅 SPI-OPI
    }
}

if (!sysbus_realize(SYS_BUS_DEVICE(&s->hi_sys), errp)) {
    return;
}
```

**关键事实**：反向依赖只对 `logical_index == 0`（SPI-OPI 实例，`dw_ssi[2]`）建立。QSPI0/QSPI1 从不调用 `k230_dw_ssi_set_hi_sys()`，其 `hi_sys` 指针恒为 NULL，XIP 读恒返回 0——这正好与"只有 SPI-OPI 才有 XIP aperture"的硬件事实一致。

---

## 提议改动

### 改动 1：SSI 侧 — 暴露通用 `xip-enable` GPIO input

**文件：`include/hw/ssi/dw_ssi.h`**

1. 删除 `typedef struct K230HiSysState K230HiSysState;`（第 35 行）
2. `DwSsiState` 结构体中：
   - 删除 `K230HiSysState *hi_sys;`（第 86 行）
   - 新增 `bool xip_enabled;`（放在 `sleep_status` 附近，同属运行时状态）
3. 删除 `void dw_ssi_set_hi_sys(...)` 声明（第 109 行）
4. **保留** `dw_ssi_get_spi_mode()` 和 `dw_ssi_is_sleeping()` 声明——它们是通用 getter，HI_SYS 仍需调用
5. VMState 描述中新增 `VMSTATE_BOOL(xip_enabled, DwSsiState)`

**文件：`hw/ssi/dw_ssi.c`**

1. 删除 `#include "hw/misc/k230_hi_sys.h"`（第 22 行）
2. 删除 `dw_ssi_set_hi_sys()` 函数定义（第 355-358 行）
3. 新增 GPIO input handler（放在文件静态函数区，`xip_read` 之前）：

```c
static void dw_ssi_xip_enable_handler(void *opaque, int n, int level)
{
    DwSsiState *s = DW_SSI(opaque);
    s->xip_enabled = (level != 0);
}
```

4. `dw_ssi_init()` 中注册 GPIO input（紧跟 `sysbus_init_irq` 之后）：

```c
qdev_init_gpio_in_named(dev, dw_ssi_xip_enable_handler, "xip-enable", 1);
```

5. `xip_read()` 第 751 行改为查本地字段：

```c
if (!s->xip_enabled) {
    return 0;
}
```

6. reset handler 中清 `s->xip_enabled = false;`（与 `s->sleep_status = false;` 同处）

### 改动 2：HI_SYS 侧 — 驱动 `xip-enable` GPIO output

**文件：`include/hw/misc/k230_hi_sys.h`**

`K230HiSysState` 结构体新增字段：

```c
qemu_irq xip_enable_out;
```

**文件：`hw/misc/k230_hi_sys.c`**

1. `k230_hi_sys_init()`（instance_init）中注册 GPIO output：

```c
qdev_init_gpio_out_named(DEVICE(obj), &s->xip_enable_out,
                         "xip-enable", 1);
```

2. ~~`k230_hi_sys_write()` 处理 SSI_CTRL 写入时，比较新旧 `XIP_EN` 位，若变化则驱动 GPIO（第 79-83 行改写）：~~

   ```c
   ~~if (addr == K230_HI_SYS_SSI_CTRL_OFFSET) {
       bool old_xip = s->ssi_ctrl & K230_SSI_CTRL_XIP_EN;
       s->ssi_ctrl = (s->ssi_ctrl & ~K230_SSI_CTRL_WRITABLE_MASK) |
                     ((uint32_t)value & K230_SSI_CTRL_WRITABLE_MASK);
       s->ssi_ctrl &= K230_SSI_CTRL_IMPLEMENTED_MASK;
       if ((s->ssi_ctrl & K230_SSI_CTRL_XIP_EN) != old_xip) {
           qemu_set_irq(s->xip_enable_out,
                        s->ssi_ctrl & K230_SSI_CTRL_XIP_EN ? 1 : 0);
       }
       return;
   }~~
   ```

   **R1+R2 修订**：改为无条件刷新，去掉 `old_xip` 比较逻辑。统一通过辅助函数 `k230_hi_sys_update_xip_enable()` 驱动 GPIO，在 write / reset / post_load 三处复用，保证行为一致：

   ```c
   /* 统一的 GPIO 刷新函数，复用在 write/reset/post_load */
   static void k230_hi_sys_update_xip_enable(K230HiSysState *s)
   {
       qemu_set_irq(s->xip_enable_out,
                    !!(s->ssi_ctrl & K230_SSI_CTRL_XIP_EN));
   }
   ```

   write 处理器改为：
   ```c
   if (addr == K230_HI_SYS_SSI_CTRL_OFFSET) {
       s->ssi_ctrl = (s->ssi_ctrl & ~K230_SSI_CTRL_WRITABLE_MASK) |
                     ((uint32_t)value & K230_SSI_CTRL_WRITABLE_MASK);
       s->ssi_ctrl &= K230_SSI_CTRL_IMPLEMENTED_MASK;
       k230_hi_sys_update_xip_enable(s);   /* 无条件刷新 */
       return;
   }
   ```

3. ~~`k230_hi_sys_reset()` 中驱动 GPIO 为初始电平（`K230_SSI_CTRL_RESET = 0x00004000`，bit 0 = 0）：~~

   ```c
   ~~qemu_set_irq(s->xip_enable_out, 0);~~
   ```

   **R2 修订**：reset 中同样调用统一刷新函数（`ssi_ctrl` 已被 reset 到 `K230_SSI_CTRL_RESET`，函数内部会算出 bit 0 = 0）：

   ```c
   k230_hi_sys_update_xip_enable(s);
   ```

   **R3 新增**：增加 `post_load` 钩子，迁移加载后重新驱动 GPIO，确保 HI_SYS 的 `ssi_ctrl` 与 SSI 的 `xip_enabled` 一致：

   ```c
   static int k230_hi_sys_post_load(void *opaque, int version_id)
   {
       K230HiSysState *s = opaque;
       k230_hi_sys_update_xip_enable(s);
       return 0;
   }
   ```

   在 VMStateDescription 的 `.post_load = k230_hi_sys_post_load`。

4. ~~**保留** `k230_hi_sys_xip_enabled()` 函数——它仍可用于测试或其他内省用途，只是不再被 SSI 调用~

   **R7 修订**：删除 `k230_hi_sys_xip_enabled()`（声明 + 定义）。当前无调用点（SSI 已不调用，qtest 不调用 C 内部函数），保留违背 YAGNI，且会留下两个"XIP 是否使能"的查询入口。由 SSI 自己维护 `xip_enabled` 输入电平；如需调试，用 trace 或 qtest 观察行为。

### 改动 3：Machine 侧 — GPIO 连接替代函数调用

**文件：`hw/riscv/k230.c`**

删除第 265-268 行的 `dw_ssi_set_hi_sys()` 调用块，替换为 GPIO 连接：

```c
for (size_t logical_index = 0;
     logical_index < ARRAY_SIZE(k230_ssi_routes); logical_index++) {
    const K230SsiRoute *route = &k230_ssi_routes[logical_index];

    k230_hi_sys_set_ssi(&s->hi_sys, logical_index,
                        &s->dw_ssi[route->ssi_index]);   // 正向依赖保留

    if (logical_index == 0) {
        /* 只有 SPI-OPI 实例有 XIP aperture，连接 xip-enable 信号 */
        qdev_connect_gpio_out_named(DEVICE(&s->hi_sys), "xip-enable", 0,
            qdev_get_gpio_in_named(DEVICE(&s->dw_ssi[route->ssi_index]),
                                   "xip-enable", 0));
    }
}
```

**连接时机分析**：
- SSI 实例的 `init`（instance_init）在 SoC 对象创建时就已运行，GPIO input 已注册
- HI_SYS 实例的 `init` 同样在创建时运行，GPIO output 已注册
- 连接发生在 SSI realize 之后、HI_SYS realize 之前——QEMU 的 `qdev_connect_gpio_out_named` 允许在 realize 前连接，延迟到双方 realize 后生效
- ~~HI_SYS realize 触发 reset → `qemu_set_irq(xip_enable_out, 0)` → SSI handler 设 `xip_enabled = false`，时序正确~~

  **R5 修订**：上条表述不准确。QEMU 的 realize 和 reset 不是简单的同步关系。正确时序为：
  1. `instance_init` 阶段：SSI 和 HI_SYS 分别注册 GPIO input/output
  2. machine 完成连接（`qdev_connect_gpio_out_named`）
  3. 系统 reset 阶段：HI_SYS reset handler 被调用 → `k230_hi_sys_update_xip_enable()` 驱动 `xip-enable` 信号
  4. SSI input handler 接收该电平 → 设置 `xip_enabled`

  reset 由 QOM 生命周期统一调度，不是 realize 的直接副作用。

### 改动 4：测试侧 — ~~无需改动~~ **需增加 GPIO 解耦验证**

**文件：`tests/qtest/k230-dw-ssi-test.c`**

- `enable_xip()`（第 474-478 行）：仍写 `K230_SSI_CTRL_ADDR`，GPIO 连接对 qtest 透明
- `test_hi_sys()`（第 801-847 行）：验证正向依赖（mode/sleep getter），不受影响
- `test_xip_read_window()`（第 849-899 行）：验证 XIP 使能前后行为差异，解耦后应继续通过——HI_SYS 写 SSI_CTRL 触发 GPIO → SSI 更新 `xip_enabled` → XIP 读行为变化

**R6 修订**：现有测试会继续通过，但第三步改变了信号传播机制（直接函数调用 → GPIO 回调），应增加显式断言验证 GPIO 解耦真的生效。在 `test_xip_read_window()` 中增加 enable/disable 往返断言：

```c
/* XIP disable 后读返回 0 */
qtest_writel(qts, K230_SSI_CTRL_ADDR, K230_SSI_CTRL_RESET);
g_assert_cmphex(qtest_readl(qts, low_addr), ==, 0);

/* XIP enable 后读到 Flash 数据 */
qtest_writel(qts, K230_SSI_CTRL_ADDR,
             K230_SSI_CTRL_RESET | K230_SSI_CTRL_XIP_EN);
g_assert_cmphex(qtest_readl(qts, low_addr), ==, expected);
```

在 `test_hi_sys()` 中增加 reset 后的 XIP enable 状态检查（reset 后 XIP 应处于禁用态）：

```c
/* reset 后 XIP_EN 应为 0 */
g_assert_cmphex(qtest_readl(qts, K230_SSI_CTRL_ADDR) &
                K230_SSI_CTRL_XIP_EN, ==, 0);
```

测试不需要拆文件，但不能完全"不改"。

---

## "通用化"设计原则

第三步的核心目标是让 SSI 成为**通用 DesignWare SSI 模型**，"通用"体现在四个层面：

### 1. 编译期无 K230 依赖

解耦后 `dw_ssi.c` / `dw_ssi.h` 不 include 任何 `k230_*.h`。SSI 是一个自包含的 IP 模型，可以被任何使用 DesignWare SSI 的 SoC 复用，不限于 K230。

### 2. GPIO 语义是功能名而非来源名

信号命名为 `"xip-enable"`（描述功能：XIP 使能），而非 `"hi-sys-xip"` 或 `"k230-xip"`（描述来源）。任何 SoC 的系统控制器——无论是 K230 的 HI_SYS、还是其他芯片的 syscon——只要能产生一个"XIP 使能"电平信号，都可以连接到这个 GPIO input。

### 3. 未连接时安全降级

QSPI0/QSPI1 实例不连接 `xip-enable` GPIO。QEMU 中未连接的 GPIO input 电平恒为 0（deasserted），`xip_enabled` 恒为 `false`，XIP 读返回 0。~~这与硬件事实一致：只有 SPI-OPI 实例才有 XIP aperture。**不需要为 QSPI 实例写任何特殊处理代码**。~~

**R8 修订**：上述表述混淆了三层概念，应分开描述：

- **`xip-enable` GPIO**：运行时访问开关。未连接只表示 XIP 默认不可用，不代表"没有 XIP aperture"
- **`has-xip` / `xip-window-size` 属性**：实例能力配置（第四步引入），决定 SSI 是否创建 XIP memory region 及其大小
- **machine mapping**：SoC 地址集成，决定 XIP region 是否映射到地址空间（当前 machine 只映射了 `dw_ssi[2]` 的第二个 region，即 SPI-OPI 的 XIP 窗口）

当前第三步只处理第一层（运行时开关）。QSPI0/QSPI1 不连接 GPIO 仅意味着它们的 XIP 读恒返回 0；是否存在 XIP aperture 是第四步 `has-xip` 的问题。

### 4. Getter 是通用内省接口

`dw_ssi_get_spi_mode()` 和 `dw_ssi_is_sleeping()` 是 SSI 对外暴露的状态查询接口，任何调用方（不限于 HI_SYS）都可以使用。这两个函数读取 SSI 自身状态（`regs[R_CTRLR0]` 的 SPI_FRF 字段、`sleep_status` 布尔值），不依赖任何外部设备，本身就是通用的。

### 设计对比

| 维度 | 解耦前（V1） | 解耦后（V2 第三步） |
|------|-------------|-------------------|
| SSI 知道 HI_SYS 存在 | 是（持有指针 + include 头文件） | 否 |
| XIP 使能查询方式 | 函数调用 `k230_hi_sys_xip_enabled()` | 本地布尔字段 `s->xip_enabled` |
| 信号来源 | 硬编码为 HI_SYS | 任意 SoC 控制器（通过 GPIO） |
| 可复用性 | 仅限 K230 | 任何 DesignWare SSI SoC |
| 未连接实例行为 | `hi_sys == NULL` 特判返回 0 | GPIO 默认低电平，`xip_enabled = false` |

---

## 假设与决策

1. **GPIO 名称**：`"xip-enable"`（与实施路线文档一致，功能名）
2. **`xip_enabled` 初始值**：`false`（对应 `K230_SSI_CTRL_RESET = 0x00004000`，bit 0 = 0）
3. ~~**VMState**：将 `xip_enabled` 加入 VMState（`VMSTATE_BOOL`）。虽然理论上可从 HI_SYS 的 `ssi_ctrl` 迁移后重新驱动 GPIO 恢复，但加入 SSI 自身 VMState 更安全，避免迁移时序依赖~~

   **R4 修订**：当前有两个状态源——HI_SYS 的 `ssi_ctrl` bit 0 和 SSI 的 `xip_enabled`。迁移加载时若只恢复 `ssi_ctrl`，HI_SYS 不一定会重新驱动 GPIO；若只恢复 `xip_enabled`，两者可能不一致。正确策略：

   - SSI 保存 `xip_enabled` 到 VMState，作为迁移兜底（通用 SSI 的 GPIO 可能来自任意外部设备，不能假设来源一定有 post_load）
   - HI_SYS 在 `post_load` 阶段调用 `k230_hi_sys_update_xip_enable()` 重新驱动 GPIO，确保外部寄存器状态和 SSI 本地状态最终一致

   即：**SSI 保存输入电平用于迁移兜底；HI_SYS 在 post-load 阶段重新驱动 GPIO，确保外部寄存器状态和 SSI 本地状态一致**。
4. **`k230_hi_sys_xip_enabled()` 保留**：函数仍留在 HI_SYS 中，不再被 SSI 调用，但可用于测试断言或未来内省
5. **getter 命名**：本步假设第二步已完成重命名（`k230_dw_ssi_get_spi_mode` → `dw_ssi_get_spi_mode`），本步只做 GPIO 解耦。**R9 补充**：HI_SYS 持有 `DwSsiState *` 指针并调用 getter 是当前的正向依赖关系，本步保留合理（避免扩大改动范围）。长期看可改用 QOM link/property，但不建议在第三步同时做，否则会把 GPIO 解耦、QOM 关系重构和属性化混在一个 patch 中。
6. **GPIO 连接时机**：在 SSI realize 之后、HI_SYS realize 之前连接（当前代码结构天然支持）
7. **reset 协调**：HI_SYS reset 驱动 GPIO 为低电平 → SSI handler 设 `xip_enabled = false`；SSI 自身 reset 也清 `xip_enabled = false`。两者一致，顺序无关

---

## 验证步骤

### 1. 编译

```bash
ninja -C build qemu-system-riscv64
ninja -C build tests/qtest/k230-dw-ssi-test
ninja -C build tests/qtest/k230-wdt-test
```

### 2. qtest 回归（结果应与第二步基线一致）

```bash
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
meson test -C build k230-dw-ssi-test k230-wdt-test -v
```

重点关注：
- `test_hi_sys`：正向依赖（getter）仍工作
- `test_xip_read_window`：XIP 使能前返回 0、使能后返回数据、XIP 与 PIO 并存

### 3. 反向依赖残留检查（全部应无结果）

**R11 修订**：使用 `rg` 替代 `grep`，符合仓库工作规范。

```bash
rg -n 'k230_hi_sys\\.h|K230HiSysState|k230_hi_sys' hw/ssi include/hw/ssi
rg -n 'k230_dw_ssi_set_hi_sys|s->hi_sys' hw/ssi
```

### 4. 代码规范

```bash
git diff --check
scripts/checkpatch.pl -f hw/ssi/dw_ssi.c
scripts/checkpatch.pl -f include/hw/ssi/dw_ssi.h
scripts/checkpatch.pl -f hw/misc/k230_hi_sys.c
scripts/checkpatch.pl -f include/hw/misc/k230_hi_sys.h
```

**R11 修订**：第五步重组提交时，验证单个 commit 不要用裸 `git checkout COMMIT_HASH`（可能覆盖未提交修改）。推荐：

```bash
# 方式 A：独立 worktree（推荐，不干扰主工作区）
git worktree add ../k230-step3-verify COMMIT_HASH
cd ../k230-step3-verify
ninja -C build qemu-system-riscv64
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
cd -
git worktree remove ../k230-step3-verify

# 方式 B：detach 切换（先确认工作区干净）
git status --short                    # 必须为空
git switch --detach COMMIT_HASH
ninja -C build qemu-system-riscv64
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
git switch -                          # 回到原分支
```

### 5. 基线对比（辅助检查）

**R11 修订**：qtest 输出 diff 过于严格（含时间戳、进度条等噪声），降级为辅助检查。主成功标准：

- 测试进程退出码为 0
- 所有用例 PASS（`-v` 输出中无 FAIL/SKIP）
- 重点场景（`test_hi_sys`、`test_xip_read_window`）单独运行通过

```bash
# 主检查：退出码 + PASS
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
echo "exit: $?"

# 辅助：diff（仅参考，时间戳/进度条差异可忽略）
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v > /tmp/step3_qtest.log 2>&1
diff /tmp/baseline_qtest.log /tmp/step3_qtest.log | head -40
```

---

## 实施清单（Review 后收敛）

第三步最终收敛为以下 6 个动作，按顺序执行：

1. **删除 SSI 对 HI_SYS 的依赖**：删除 `K230HiSysState *hi_sys` 字段、`typedef` 前向声明、`#include "hw/misc/k230_hi_sys.h"`、`dw_ssi_set_hi_sys()` 声明与定义
2. **SSI 增加 `xip-enable` GPIO input**：新增 `bool xip_enabled` 字段、`dw_ssi_xip_enable_handler()` 回调、`qdev_init_gpio_in_named()` 注册、`xip_read()` 改查本地字段、reset 清 `xip_enabled`
3. **HI_SYS 增加 `xip-enable` output + 统一刷新函数**：新增 `qemu_irq xip_enable_out` 字段、`qdev_init_gpio_out_named()` 注册、`k230_hi_sys_update_xip_enable()` 辅助函数；**删除** `k230_hi_sys_xip_enabled()`（R7）
4. **在 reset / write / post_load 中统一调用 `k230_hi_sys_update_xip_enable()`**（R1+R2+R3）：保证三个时机行为一致，无需 `old_xip` 比较逻辑
5. **machine 只连接 SPI-OPI 实例的 GPIO**：`qdev_connect_gpio_out_named()` 替代 `dw_ssi_set_hi_sys()` 调用
6. **qtest 增加 GPIO 解耦验证**（R6）：XIP enable/disable 往返断言 + reset 后状态检查

---

## 与决策文档的关系

本文档是 V2 实施路线第三步的执行细化。架构边界（SSI 通用层 vs K230 层）、HI_SYS 解耦方案的架构决策以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准。如果实施中发现边界问题，先更新决策文档，再回头调整本文档。
