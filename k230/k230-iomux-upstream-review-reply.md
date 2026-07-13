# K230 IOMUX 首轮上游评审合并回复

## 发送方式

回复 Alistair Francis 的评审邮件，保留原主题，并将 Chao Liu 加入直接
Cc。邮件挂在 Alistair 的评论下面，同时让 Chao 收到并看到对其意见的回应。

## 英文发送稿

```text
Subject: Re: [PATCH RFC 1/1] hw/riscv/k230: add IOMUX register block model

Hi Alistair and Chao,

Thank you both very much for the reviews and references.

I will split v2 into three patches: one adding the IOMUX device model,
one wiring it into the K230 SoC, and one adding the qtest coverage.

I will also remove the explicit offset, size, and alignment checks from
the read and write callbacks, since the MemoryRegion size and
MemoryRegionOps access constraints already enforce the valid MMIO range
and aligned 32-bit accesses.

For the currently modeled and tested use cases, the device only needs
register-backed storage. The SDK U-Boot pinctrl-single driver and the
SDK Linux IOMUX-related code paths I checked use 32-bit
read-modify-write accesses and require the configuration values to be
retained.

On real hardware, writing these registers changes the pin function and
electrical configuration. However, the current K230 machine does not
model physical pin routing or electrical characteristics, and I have
not found any additional register side effects required by the U-Boot
or Linux code paths I tested.

I will update v2 to follow the documented bit-level access permissions:
bit 31 (DI) is read-only, bits 30 through 14 are read-only reserved bits,
and bits 13 through 0 are read-write configuration fields. Guest writes
will use a 0x00003fff writable mask, so they cannot modify bit 31 or the
reserved bits. I will also add qtests covering writable fields,
read-only fields, reserved bits, and normal read-modify-write accesses.

The SoC memory map reserves a 0x800-byte window for the Main IOMUX, but
the documentation defines only 64 pin registers at offsets 0x00 through
0xfc. I will model only these documented registers in v2 and leave
offsets 0x100 through 0x7ff reserved/unimplemented.

I will include these changes in v2.

Thanks,
Kangjie


```

## 中文对照

```text
主题：Re: [PATCH RFC 1/1] hw/riscv/k230: add IOMUX register block model

Alistair、Chao，你们好：

十分感谢两位的评审和参考资料。

我会将 v2 拆分为三个补丁：第一个添加 IOMUX 设备模型，第二个将其接入
K230 SoC，第三个添加 qtest 测试覆盖。

我也会删除读写回调中对偏移、访问大小和对齐的显式检查，因为
MemoryRegion 的大小以及 MemoryRegionOps 的访问约束已经保证访问位于有效
MMIO 范围内，并且是对齐的 32 位访问。

对于当前已经建模和测试的使用场景，该设备只需要基于寄存器的存储。我所
检查的 SDK U-Boot pinctrl-single 驱动和 SDK Linux IOMUX 相关代码路径使用
32 位读改写访问，并且需要保留写入的配置值。

在真实硬件上，写入这些寄存器会改变引脚功能和电气配置。但是，当前 K230
machine 没有建模物理引脚路由或电气特性，而且我没有在已经测试的 U-Boot
或 Linux 代码路径中发现必须实现的其他寄存器副作用。

我会更新 v2，使其遵循文档规定的位级访问权限：bit 31（DI）为只读，
bits 30–14 为只读保留位，bits 13–0 为可读写配置字段。guest 写入将使用
`0x00003fff` 可写掩码，因此无法修改 bit 31 或保留位。我也会增加 qtest，
覆盖可写字段、只读字段、保留位以及正常的读改写访问。

SoC 内存图为 Main IOMUX 保留了 `0x800` 字节的地址窗口，但文档只定义了
64 个 pin 寄存器，偏移范围为 `0x00..0xfc`。我会在 v2 中只建模这些已有
文档定义的寄存器，并将 `0x100..0x7ff` 保留为 reserved/unimplemented。

我会在 v2 中包含以上修改。

谢谢，
Kangjie
```
