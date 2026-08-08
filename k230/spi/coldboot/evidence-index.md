# K230 SPI-NOR 冷启动证据索引

本页索引对应本机 `.trae/evidence/k230-coldboot-canmv-qemu-v4/` 中的验收产物。
本次只同步 Markdown，不把镜像、二进制和原始日志纳入 `exper-note`。

## v4 关键证据

| 证据 | 用途 |
|---|---|
| `manifest.txt` | 32 MiB 镜像分区、offset、大小和 SHA256 清单 |
| `provenance-hashes.log` | 构建输入来源与哈希 |
| `final-independent-audit.log` | 独立静态审计结果 |
| `v4-qemu-no-serial-final-clean.log` | `/dev/null` 输入下的无人干预启动日志 |
| `rootfs-build-final-clean.log` | 最小 JFFS2 rootfs 构建结果 |
| `artifact-format-final-clean.log` | SPL、U-Boot、Linux 和 rootfs 格式检查 |

## 复现入口

完整构建和运行命令见
[v4 JFFS2 复现文档](./k230-coldboot-canmv-qemu-v4-reproduction.md)。

最终镜像 SHA256：

```text
e7728c24a7edef22c85d57dcd41c7c429de974a7337fde5d4eb02b0da689911a
```
