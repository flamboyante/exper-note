# 用 B4 通过邮件向 QEMU/Linux 上游发送 patch

本文说明已经完成开发、测试和 commit 切分的 Git patch series，如何用 `b4` 生成并投递为公开邮件。它面向 QEMU 和 Linux kernel 上游，不替代各子系统的 `MAINTAINERS`、贡献文档或维护者在 review 中给出的要求。

**适用范围**：本文只覆盖 patch 邮件的准备、SMTP 投递、review 后 reroll 与跟踪；不覆盖创建邮箱账号、邮件服务商的企业策略，也不授权发送尚未通过本地测试的代码。地址、分支、目录均为占位示例。绝不把 SMTP 密码、应用专用密码、OAuth token 或原始认证日志写入仓库、shell history、cover letter 或补丁正文。

本机于 2026-08-07 检查到 `b4 0.13.0`。命令前先运行 `b4 --version` 和 `b4 <subcommand> --help`；不同版本的 `prep` 选项并不完全相同。

## 1. 先理解发送链路

`b4 prep` 管理一个待投递 series：建立或登记分支、保存 cover letter、记录发送历史并生成 patch 邮件。`b4 send` 则读取该 series 的收件人和 Git `sendemail.*` 配置，走 SMTP 或项目配置的 web endpoint 投递。

正常的收件人路径是：

```text
commit 与 cover letter
  -> b4 prep 生成 RFC 5322 patch 邮件
  -> b4 send 补齐 To/Cc 与线程头
  -> SMTP submission server
  -> QEMU/Linux 邮件列表、维护者和审阅者
  -> lore / 项目 CI / 人工 review
```

`user.name`、`user.email`、commit 作者身份、`Signed-off-by:` 和 SMTP 发件地址必须彼此可解释。`Signed-off-by:` 是贡献者对开发者来源证书的声明，不是 SMTP 登录名的替代品；不要代替他人添加这一行。

## 2. 邮箱与 SMTP：只配置非秘密项

`b4 send` 复用 Git 的 `sendemail.*` 配置。下面使用虚构服务商和地址；按实际邮箱服务商提供的 submission hostname、端口和 TLS 模式替换：

```sh
git config --global sendemail.from 'Your Name <you@example.example>'
git config --global sendemail.smtpServer 'smtp.example.example'
git config --global sendemail.smtpServerPort 587
git config --global sendemail.smtpEncryption tls
git config --global sendemail.smtpUser 'you@example.example'
```

常见的 submission 组合是端口 `587` 加 `tls`（STARTTLS），但这不是通用常量。只有服务商明确提供 SMTPS 时才使用其指定端口与 `ssl`。公司邮箱、学校邮箱、个人邮箱或自建域名的认证政策可能不同；先查该服务商的官方 SMTP 指南。

不要执行 `git config --global sendemail.smtpPass ...`。b4 在已设置 `smtpUser`、未设置 `smtpPass` 时会通过 Git credential helper 获取凭据。为 credential helper 配置系统的受保护存储，并将服务商要求的应用专用密码或 OAuth 凭据只交给该存储。SMTP 登录通常要求 TLS；无加密 SMTP 只能在你明确控制的本地 relay 场景使用。

如有多个发件身份，可选择 Git 的命名身份：

```ini
# ~/.gitconfig，示例，不含密码
[sendemail "upstream"]
    from = Your Name <you@example.example>
    smtpServer = smtp.example.example
    smtpServerPort = 587
    smtpEncryption = tls
    smtpUser = you@example.example
[b4]
    sendemail-identity = upstream
```

该配置仅选择邮件后端；不会自动替换 commit 中的作者、`Signed-off-by:` 或目标邮件列表。

## 3. QEMU series 的准备

QEMU 要求 patch 邮件投递到 `qemu-devel@nongnu.org`，同时抄送修改路径对应的维护者、reviewer 和子系统列表。QEMU 树根的 `.b4-config` 已配置默认的 `qemu-devel@nongnu.org`、`scripts/get_maintainer.pl` 自动 Cc，以及 `checkpatch.pl` 的逐补丁检查命令。

从已更新且干净的 QEMU `master` 开始：

```sh
cd /path/to/qemu
b4 prep -n k230-dw-ssi

# 在 b4/k230-dw-ssi 上完成独立、可构建和可测试的 commits。
git commit -s
b4 prep --edit-cover
b4 prep --auto-to-cc
```

`b4 prep -n` 创建 `b4/k230-dw-ssi`；已有 topic branch 时，先确认基线，再用 `b4 prep --enroll <base>` 登记它。不要在不知道 fork point 的分支上 enrol，也不要把工作区中未提交的实验改动混入投递范围。

cover letter 应说明目的、每个 patch 的职责、依赖、测试、已知限制和收件人选择。多 patch series 必须有 cover letter。若依赖尚未进入 QEMU master 的公开 series，在 cover 中写 `Based-on: <message-id>`。每个 commit 仍须独立编译和运行；不要让后续 patch 才补齐前一 patch 所需的文件或测试。

QEMU 当前文档列出 `b4 prep --check`。本机 `b4 0.13.0` 的 `prep --help` **没有**该选项，因此不要盲目执行。使用支持该选项的新版 b4 时可运行：

```sh
b4 prep --check
```

否则保留同等的显式检查，并按项目实际构建和 qtest 补全验证：

```sh
git diff --check <base>..HEAD
git format-patch --stdout --no-cover-letter <base>..HEAD \
  | scripts/checkpatch.pl -q --terse --no-summary --mailback -
```

`checkpatch.pl` 只是风格与常见错误筛查，不能代替目标构建、qtest、功能验证或人工审阅。

## 4. Linux kernel 的差异

Linux 不存在可以替代子系统收件人的单一投递地址。不要把 QEMU 的 `qemu-devel@nongnu.org` 照搬到 Linux，也不要因为补丁属于 Linux 就无差别抄送 `linux-kernel@vger.kernel.org`。必须从目标 Linux tree 的 `MAINTAINERS`、`scripts/get_maintainer.pl` 和子系统文档确认 `M:`、`R:`、`L:`、维护树与特别说明。

```sh
git format-patch -1 --stdout HEAD > /tmp/0001-local.patch
scripts/get_maintainer.pl /tmp/0001-local.patch

# 将上一步确认的列表和人员填入；先只生成邮件。
b4 send --to 'subsystem-list@example.org' \
  --cc 'maintainer@example.org' --dry-run
```

Linux 的每个列表可能有独立的订阅、审核、大小限制和不接收策略。SMTP 成功只说明 submission server 接收了邮件，不能证明列表接收、archive 收录或维护者同意审阅。使用不会折行、改码、替换空白符或将 patch 变为附件的客户端；`b4 send` 和 `git send-email` 的目的正是避免这类破坏。

## 5. 真发送前的安全梯

先确认所在仓库、基线、分支和 Git 身份，再按以下顺序推进。第 4 步及以后会产生真实邮件；不要在还没检查收件人时跳过前面步骤。

```sh
# 1. 只看 series 的邮件文本、Subject、To/Cc、threading 和 diff。
b4 send --dry-run > /tmp/k230-dw-ssi-dry-run.txt

# 2. 将每封最终邮件保存成 .eml，供邮件客户端或 git am 检查。
b4 send -o /tmp/k230-dw-ssi-eml

# 3. 真实投递，但 SMTP envelope 仅指向自己。
b4 send --reflect

# 4. 可选：向协作者预审；实际投递给该地址，Subject 会加 PREVIEW。
b4 send --preview-to 'reviewer@example.org'

# 5. 最终投递到 series 的 To/Cc。发送前 b4 会显示收件人并要求确认。
b4 send
```

`--dry-run` 不连接 SMTP，因此无法验证认证、TLS 或服务商拒绝。`-o` 强制 dry-run。`--reflect` 是真实发送，只是 envelope recipient 为自己，原本的 `To:`/`Cc:` 头仍会保留；它适合检查邮件是否到达、是否为 inline patch、是否被邮件服务商改写。`--preview-to` 也是真实投递，但只适合事先约定的预审地址。

在 `--reflect` 邮件中检查：发件人是否正确、cover 与所有 N 个 patch 是否齐全、`[PATCH n/m]` 编号和 `In-Reply-To` 是否合理、收件人是否没有遗漏，以及导出的 `.eml` 是否可由干净 worktree 的 `git am --check` 接受。不要把 reflect 当作邮件列表送达测试。

如果 b4 报告没有收件人，先检查 cover letter 中的 `To:`/`Cc:` trailer，重新运行 `b4 prep --auto-to-cc`，或明确添加 `b4 send --to ... --cc ...`；不要为了绕过错误而向无关的大列表群发。

## 6. 发送后、review 与 reroll

记录发送时的 cover Message-ID、lore 链接、测试结果和版本号。正常 `b4 send` 成功后会为已发送版本建本地 tag、在 cover tracking 中记录 Message-ID，并把下一次 revision 递增。`--dry-run`、`--reflect`、`--preview-to` 和 `--resend` 不会推进这个状态。

QEMU patch 可在 [Patchew](https://patchew.org/QEMU/) 与 [QEMU lore archive](https://lore.kernel.org/qemu-devel/) 跟踪收录和 CI。Linux 则查相应 lore archive、子系统的 CI/patch tracker 和维护者回复；不要假定所有 Linux 子系统都使用相同的 Patchwork 页面。

收到 review 后，先逐条理解并回复必要的澄清，再修改提交。只对地址和 commit 身份确认匹配的已获 `Reviewed-by:`、`Acked-by:`、`Tested-by:` 等 trailer 执行 `b4 trailers` 或手工纳入；显著修改某 patch 时移除不再适用的 review trailer。更新 cover 的 `---` 以下版本说明，保留 commit message 上方的历史语义不被版本噪声污染。

QEMU 的规则是：任何 patch 改动后，重发**完整** series，标为 `v2`/`v3`，并开新的顶层线程，而不是把新版本深埋在 v1 reply 中。b4 会在正常发送后准备下一 revision：

```sh
# 修改、rebase、重新测试并更新 cover 的 version history 后。
b4 prep --compare-to v1
b4 send --dry-run
b4 send
```

仅在需要重新发送完全相同的已发版本时使用 `b4 send --resend v1`。它不是“修改后的 v2”命令，也不会更新 b4 的 revision tracking。

QEMU series 若一到两周没有回复，可对原邮件 reply-all 发送简短 `ping`，附 Patchew 或 lore 链接；继续等待并避免高频催促。一般贡献者不应直接向 QEMU 主线发送 pull request，维护者会在审阅后将 series 纳入维护树。

## 7. 故障定位

| 现象 | 先检查 | 处理方向 |
| --- | --- | --- |
| `b4 send` 没有收件人 | cover `To:`/`Cc:`、`.b4-config`、`b4 prep --auto-to-cc` 输出 | 用 `get_maintainer.pl` 和 `MAINTAINERS` 重新确认，而不是猜地址。 |
| TLS 或认证失败 | hostname、port、`smtpEncryption`、服务商账号策略、credential helper | 先用 `b4 send --reflect` 复现；只在本机查看诊断日志，不提交或转发凭据。 |
| `no key configured` | patatt/GPG 配置 | 正式使用 web endpoint 时配置签名；SMTP 路径如确有必要才显式 `--no-sign`，并记录为什么。 |
| 生成邮件仍含 `EDITME` | cover letter | `b4 prep --edit-cover` 后重新 dry-run；不要发送占位 cover。 |
| `prep --check` 不存在 | `b4 prep --help` 与版本 | 升级 b4 或使用本节给出的 `checkpatch.pl` 回退命令。 |
| 邮件未出现在列表/archive | SMTP 是否接收、列表审核/订阅策略、收件人、垃圾邮件 | 等待合理审核时间，查目标列表规则和 lore；不要立即重复发送。 |
| v2 没有被视为完整新系列 | Subject、版本号、线程关系 | QEMU 重发全部 patch，使用 `[PATCH v2 ...]`，建立新顶层线程。 |

## 8. 官方资料

- [b4 文档](https://b4.docs.kernel.org/)：`prep`、`send`、web submission 与配置总入口。
- [b4 源码](https://git.kernel.org/pub/scm/utils/b4/b4.git/tree/)：`sendemail.*` 读取、SMTP TLS/认证、dry-run、reflect、reroll 的实现依据。
- [Git `send-email` 手册](https://git-scm.com/docs/git-send-email)：Git 邮件后端、`sendemail.*` 与确认行为。
- [QEMU Submitting a Patch](https://www.qemu.org/docs/master/devel/submitting-a-patch.html)：QEMU 收件人、b4 工作流、cover、reroll、review 与 ping。
- [QEMU `.b4-config`](https://gitlab.com/qemu-project/qemu/-/blob/master/.b4-config)：QEMU 的默认列表、自动 Cc 和检查命令。
- [Linux Submitting Patches](https://docs.kernel.org/process/submitting-patches.html)：Linux 的提交和收件人准则。
- [Linux Email Clients](https://docs.kernel.org/process/email-clients.html)：避免邮件客户端破坏 patch 的要求。
- [Linux `MAINTAINERS`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/MAINTAINERS) 与 [`scripts/get_maintainer.pl`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/get_maintainer.pl)：按路径确定 Linux 收件人。
