# 银河麒麟桌面操作系统 · 构建与测试镜像

与真实银河麒麟桌面系统尽可能一致的容器环境，用于软件构建、打包与兼容性测试。

**这不是服务器镜像，也不面向生产部署。** 镜像里没有内核，内核态组件（initramfs、TPM、KYSEC LSM 插件）由假包顶替或直接不装，systemd 只保留容器内有意义的部分。它们存在的理由只有一个：让你在 CI 里得到的构建结果，与在真机上得到的一致。

## 拉取

```
ghcr.io/distrotwin/kylin:<版本>-<档位>
```

| 版本 | 说明 | glibc | GCC |
|---|---|---|---|
| `v11` | 银河麒麟桌面 V11（2603） | 2.38 | 13.2 |
| `v10sp1` | 银河麒麟桌面 V10 SP1（2503） | 2.31 | 10.3 |
| `v4` | 银河麒麟桌面 V4（4.0.2） | 2.23 | 5.4 |

| 档位 | 内容 |
|---|---|
| `micro` | 最小可用集：CA 证书、时区、locale、libstdc++ |
| `base` | 加上常用命令行工具、python3、systemd |
| `devel` | 加上 `build-essential`、`pkg-config` 等构建链 |

不带档位后缀的 tag 指向 `devel`，与 `golang:1.21` 是完整版的惯例一致。`latest` 指向最高版本的 `devel`。

```
docker pull ghcr.io/distrotwin/kylin:v11-devel
docker pull ghcr.io/distrotwin/kylin:v11          # 同上
docker pull ghcr.io/distrotwin/kylin:v10sp1-base
```

## 请钉住带日期的 tag

每次发布同时打两个：滚动的 `v11-devel` 跟随最新构建，带日期的 `v11-devel-20260902` 约定不动。**CI 里请用后者。** 滚动 tag 会随重建而变化，`latest` 更会随大版本推进而跨越 ABI 边界——那正是这批镜像本该帮你避免的事故。

日期取的是**发布时仓库 HEAD 的 commit 日期**（UTC），不是构建时刻，所以同一个 commit 无论何时重建都是同一个 tag。它锁不住的是上游源：同一 commit 隔一周重建，若麒麟归档更新过包则镜像字节不同而 tag 相同。要判断手里两个镜像是否同一份，比 `cn.internal.source-date-epoch` 这个 label——它取自源 `Release` 的日期。

**真要钉死请用 digest。** GHCR 的 tag 本来就可变，日期 tag 是约定而非强制；一天内多次发布（调试时）会覆盖它。发布日志里会打印每个 tag 的 `@sha256:...`。

## 镜像自带的溯源信息

```
docker inspect --format '{{json .Config.Labels}}' ghcr.io/distrotwin/kylin:v11-devel
```

| label | 内容 |
|---|---|
| `cn.internal.kylin-commit` / `buildkit-commit` | 哪份配置与哪份脚本建的 |
| `cn.internal.suite` / `source-date-epoch` | 源的哪个快照 |
| `cn.internal.glibc` / `libstdcpp` / `glibcxx` / `arch` | **实测**的 ABI 值，不是 conf 里的期望值 |
| `cn.internal.build-run` | 跳回当时的 CI 日志与测试报告 |

跨架构时这几个 label 尤其有用：同一个 tag 在不同架构下的 ABI 并不一致（见下节），`docker inspect` 一看就知道拉到的是哪一档。

## 架构

amd64 与 arm64 由原生 runner 构建。LoongArch 尚未发布，原因见下。

**同一个版本号在不同架构下的 ABI 并不一致，这一点必须知道。** 银河麒麟的 LoongArch 移植是独立的产品线，不是同一套代码换个指令集：

| | amd64 / arm64 | LoongArch |
|---|---|---|
| V10 SP1 | glibc 2.31 · gcc 10.3 | glibc **2.28** · gcc **8.3**（Loongnix 血脉） |
| V11 | glibc 2.38 · gcc 13.2 | glibc 2.38 · gcc **14** |

也就是说，在 amd64 上验过的构建结论，不能直接套用到 LoongArch 镜像上。这是厂商产品线的事实，不是本仓库的取舍。

LoongArch 还有一层：麒麟两个版本用的是两套互不兼容的 ABI。V10 SP1 的架构名是 `loongarch64`，属于旧世界，动态链接器为 `/lib64/ld.so.1`；V11 是 `loong64`，属于新世界，动态链接器为 `/lib64/ld-linux-loongarch-lp64d.so.1`。两者的 multiarch 三元组同名，产物却不可互换。

这个差别已经实测过，结论是**旧世界那一支在上游 QEMU 下跑不起来**。把两版的 `libc6` 与 `bash` 各拼成一个最小 rootfs，用 `qemu-loongarch64-static -L` 直接执行：V11 的 bash 正常退出，内建变量 `MACHTYPE=loongarch64-unknown-linux-gnu`（编译期烧进二进制，可以证明执行的确实是目标架构的产物）；V10 SP1 的 bash 停在动态链接阶段，`-strace` 显示它打开了 `libtinfo.so.6`、读出了 ELF 头，然后撞上 `Unknown syscall 80`。

79 与 80 是通用 Linux ABI 的 `newfstatat` 与 `fstat`。LoongArch 新世界把这两个系统调用去掉了，只保留 `statx`，因此上游 QEMU 的 loongarch64 target 没有实现它们；而旧世界的 glibc 2.28 仍在调用。这不是缺库或路径问题，是系统调用 ABI 本身不同——没有龙芯补丁版 QEMU 或真机，`v10sp1` 的 LoongArch 镜像造不出来。

V11 的 `loong64` 通过了这项 smoke test，但**跑通一个 bash 不等于跑得完一次 bootstrap**：完整构建要在模拟下执行几百个包的维护者脚本，是另一个量级。它目前仍在 `workflow_dispatch` 开关后面，等首次实跑确认。

V4 没有任何 LoongArch 架构的软件源，不会有对应镜像。

## 镜像怎么造出来的

三个版本都不经过 ISO，直接从麒麟的 apt 归档 `archive.kylinos.cn/kylin/KYLIN-ALL/` bootstrap。V11 走 `mmdebstrap`；V10 SP1 与 V4 走两段式自举，因为宿主 dpkg 写出的 `status` 这两个系统自带的旧 dpkg 读不了。

有一处容易踩：V10 SP1 的 suite 是 `10.1` 而不是 `10.0`。`10.0` 的 `Release` 自述 Label 是「v10-离在线更新推送」，是给已装机器推更新的差异源，`universe` 有四万多个包却没有 `libc6`，从零 bootstrap 不出来。V4 同理，要用 `4.0.2` 而不是 `4.0.2sp4`。

构建机器码在 [`distrotwin/buildkit`](https://github.com/distrotwin/buildkit)，本仓库以 submodule 引用并钉住 commit。本仓库自己只有三个 `distros/*.conf` 和一份调用 workflow。

## 本地构建

```
git clone --recurse-submodules https://github.com/distrotwin/kylin.git
cd kylin
sudo ARCH=amd64 ROOT=$PWD buildkit/build/build.sh v11 micro base devel
sudo ARCH=amd64 ROOT=$PWD buildkit/test/verify.sh v11
```

跨架构构建需要宿主装 `qemu-user-static` 与 `binfmt-support`。不要引入 `tonistiigi/binfmt` 容器，实测它会破坏本来可用的 binfmt 注册。

## 状态

V4 尚未跑通一次完整构建。它自带 dpkg 1.18 / apt 1.2，比 V10 SP1 的 1.19.7 更旧，自举第二段可能还需适配。conf 里的 `EXPECT_LIBSTDCPP` 与 `EXPECT_GLIBCXX` 是由源索引里的 gcc 版本推导的，尚未经实测确认；写推导值而不留空是刻意的——留空会让门禁因「期望空等于实际空」而判通过。
