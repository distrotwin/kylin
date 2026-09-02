# 银河麒麟桌面操作系统 · 构建与测试镜像

与真实银河麒麟桌面系统尽可能一致的容器环境，用于**软件构建、打包与兼容性测试**。

```bash
# 在麒麟 V11 的构建环境里编一个 C 程序
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v11-devel \
  gcc -O2 -o hello hello.c

# 看看这个环境的 ABI 底座
docker run --rm ghcr.io/distrotwin/kylin:v11-devel \
  bash -c 'ldd --version | head -1; gcc --version | head -1'
```

## 这不是部署用的镜像

请先读这一段，它决定你该不该用这些镜像。

**它们是替身，不是本体。** 镜像里没有内核；内核态组件（initramfs、TPM、KYSEC LSM 插件）由假包顶替或直接不装；systemd 只保留容器内有意义的部分。真机上依赖内核特性、安全模块或硬件的行为，在这里不成立。

**该用它的场景**：在 CI 里编出能在麒麟上跑的二进制、验证 `.deb` 打包与依赖、检查你的产物需要的 glibc/libstdc++ 符号版本是否被目标系统满足、复现只在麒麟上出现的编译问题。

**不该用它的场景**：把它当生产运行时基础镜像、指望它复现内核相关的行为、当作麒麟系统的完整替代品做验收测试。

判断标准很简单：**你关心的是"编出来的东西对不对"，还是"跑起来的系统对不对"。** 前者用它，后者请用真机或虚拟机。

## 有哪些镜像

```
ghcr.io/distrotwin/kylin:<版本>-<档位>
```

| 版本 | 系统 | glibc | GCC | 上游血脉 |
|---|---|---|---|---|
| `v11` | 银河麒麟桌面 V11（2603） | 2.38 | 12.2 | openKylin 2.0 |
| `v10sp1` | 银河麒麟桌面 V10 SP1（2503） | 2.31 | 9.3 | Ubuntu 20.04 |
| `v4` | 银河麒麟桌面 V4（4.0.2） | 2.23 | 5.3 | Ubuntu 16.04 |

| 档位 | 装了什么 | 典型用途 |
|---|---|---|
| `micro` | CA 证书、时区、locale、libstdc++ | 跑已编好的二进制 |
| `base` | 加常用命令行工具、python3、systemd | 平台脚本、集成测试 |
| `devel` | 加 `build-essential`、`pkg-config` 等 | **编译与打包** |

不带档位后缀的 tag 指向 `devel`，与 `golang:1.21` 是完整版的惯例一致。

```bash
docker pull ghcr.io/distrotwin/kylin:v11-devel
docker pull ghcr.io/distrotwin/kylin:v11          # 同上
docker pull ghcr.io/distrotwin/kylin:v10sp1-base
docker pull ghcr.io/distrotwin/kylin:v4-micro
docker pull ghcr.io/distrotwin/kylin:latest       # = v11-devel
```

## 快速上手

**编一个 C 程序并确认它的 ABI 需求**

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v10sp1-devel bash -c '
  gcc -O2 -o app app.c
  objdump -T app | grep -oE "GLIBC_[0-9.]+" | sort -uV | tail -1
'
```

最后那行告诉你产物需要的最高 glibc 符号版本。它不高于目标系统的 glibc，产物才能在那台机器上跑起来。

**在三个版本上同时验一遍**

```bash
for v in v4 v10sp1 v11; do
  echo "== $v"
  docker run --rm -v "$PWD:/w" -w /w "ghcr.io/distrotwin/kylin:$v-devel" \
    bash -c 'gcc -O2 -o /tmp/app app.c && echo 编译通过' || echo 编译失败
done
```

这是这批镜像最实际的用法：**一次改动，三个 ABI 世代同时过一遍**。V4 那档尤其能暴露对新 glibc 特性的隐式依赖。

**在 GitHub Actions 里用**

```yaml
jobs:
  build-for-kylin:
    runs-on: ubuntu-latest
    container: ghcr.io/distrotwin/kylin:v11-devel-20260902   # 钉住带日期的 tag
    steps:
      - uses: actions/checkout@v4
      - run: make
```

**打一个 .deb 并检查依赖**

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v11-devel bash -c '
  dpkg-buildpackage -us -uc -b
  dpkg-deb -I ../*.deb | grep -E "Depends|Architecture"
'
```

**交互式排查**

```bash
docker run --rm -it -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v11-devel bash
```

## tag 与可复现

每次发布对每个档位打这些 tag：

| tag | 性质 |
|---|---|
| `v11-devel` | 滚动，跟随最新构建 |
| `v11` | 滚动，= `v11-devel` |
| `latest` | 滚动，= 最高版本的 `devel` |
| `v11-devel-20260902` | 按 commit 日期，约定不动 |
| `v11-devel-20260902-amd64` | 单架构，manifest list 的成员 |

**CI 里请用带日期的那个。** 滚动 tag 会随重建变化，`latest` 更会随大版本推进跨越 ABI 边界——那正是这批镜像本该帮你避免的事故。

日期取的是**发布时仓库 HEAD 的 commit 日期**（UTC）而非构建时刻，所以同一个 commit 无论何时重建都是同一个 tag。它锁不住的是上游源：同一 commit 隔一周重建，若麒麟归档更新过包则镜像字节不同而 tag 相同。**要确认手里两个镜像是不是同一份，只能比 digest。** 镜像里没有记录源快照的 label——那个值只存在于构建机上，发布阶段取不到。

**真要钉死请用 digest。** GHCR 的 tag 本来就可变，日期 tag 是约定而非强制；一天内多次发布（调试时）会覆盖它，发布日志里会打印每个 tag 的 `@sha256:...`。

```bash
docker pull ghcr.io/distrotwin/kylin@sha256:<digest>
```

## 镜像自带的溯源信息

```bash
docker inspect --format '{{json .Config.Labels}}' \
  ghcr.io/distrotwin/kylin:v11-devel | python3 -m json.tool
```

| label | 内容 |
|---|---|
| `cn.internal.kylin-commit` / `buildkit-commit` | 哪份配置与哪份脚本建的 |
| `cn.internal.suite` | 建自源的哪个 suite |
| `cn.internal.glibc` / `libstdcpp` / `glibcxx` / `arch` | **实测**的 ABI 值，不是配置里的期望值 |
| `cn.internal.build-run` | 跳回当时的 CI 日志与测试报告 |

跨架构时这几个 label 尤其有用，原因见下节。

## 架构

amd64 与 arm64 由原生 runner 构建；V11 另有 loong64（QEMU 模拟构建，实测 micro 约 3 分钟、devel 约 30 分钟）。

**同一个版本号在不同架构下的 ABI 并不一致，这一点必须知道。** 银河麒麟的 LoongArch 移植是独立的产品线，不是同一套代码换个指令集：

| | amd64 / arm64 | loong64 |
|---|---|---|
| V11 | glibc 2.38-1ok6.9k0.5 · libstdc++ 6.0.32（`GLIBCXX_3.4.32`） | glibc **2.38-1ok6.12** · libstdc++ **6.0.33**（**`GLIBCXX_3.4.33`**） |
| V10 SP1 | glibc 2.31 · gcc 9.3 | glibc **2.28** · gcc **8.3**（Loongnix 血脉） |

两支的差距不是一个量级。V10 SP1 差着一整个工具链世代；V11 的两支编译器同为 GCC 12.2，分叉在运行时——loong64 那支的 libstdc++ 反而更新一档，多出 `GLIBCXX_3.4.33` 这一级符号。

也就是说，在 amd64 上验过的构建结论，不能直接套用到 loong64 镜像上。这是厂商产品线的事实，不是本仓库的取舍——所以每个镜像都把**实测**的 ABI 值写进了 label，`docker inspect` 一看就知道拉到的是哪一档。

**V10 SP1 没有 LoongArch 镜像，V4 也没有。** V10 SP1 的 `loongarch64` 是旧世界 ABI：它的 glibc 2.28 仍在调用 `syscall 79/80`（`newfstatat` / `fstat`），而 LoongArch 新世界把这两个调用去掉了，上游 QEMU 的 loongarch64 target 因此没有实现。最小 rootfs 实测的表现是打开 `libtinfo.so.6`、读出 ELF 头后报 `Unknown syscall 80` 退出 127。没有龙芯补丁版 QEMU 或真机就造不出来。V4 则是源里根本没有任何 LoongArch 架构。

## 这些镜像是怎么来的

三个版本都不经过 ISO，直接从麒麟的 apt 归档 `archive.kylinos.cn/kylin/KYLIN-ALL/` bootstrap。V11 走 `mmdebstrap`；V10 SP1 与 V4 走两段式自举，因为宿主 dpkg 写出的 `status` 这两个系统自带的旧 dpkg 读不了。

有两处容易踩：V10 SP1 的 suite 是 `10.1` 而不是 `10.0`（后者的 `Release` 自述 `Label: v10-离在线更新推送`，是给已装机器推更新的差异源，`universe` 有四万多个包却没有 `libc6`）；V4 要用 `4.0.2-desktop` 而不是 `4.0.2`（`debootstrap` 会核对请求的 suite 名与 `Release` 自述是否一致，而 apt 不核对）。

每个镜像发布前都在**干净机器上**装载、真正启动、跑完整检查集——构建阶段的机器状态（builder 容器、本地源、宿主装的包）会掩盖镜像自身的缺陷。最近一轮：21 个镜像、899 项检查、零异常。测试报告与完整日志按系统打包在每次 run 的 artifact 里。

构建机器码在 [`distrotwin/buildkit`](https://github.com/distrotwin/buildkit)，本仓库以 submodule 引用并钉住 commit。本仓库自己只有三个 `distros/*.conf` 和一份调用 workflow。

## 已知的期望失败

V4 比其余版本老一个时代，有些检查在它身上注定红，而且红得有道理——这类项在 `distros/v4.conf` 的 `XFAIL` 里声明，报告中标为 🟡 而不计为失败：

- `elf_broken`：V4 的 gnupg 会装一个链接 `libldap` 的 `gpgkeys_ldap`，而本项目不装 Recommends，`ldd` 必报缺库；ircd 的可选模块 `m_xt.so` 同理。**真实的 V4 装机同样如此**，镜像是忠实的。这条不是「允许有坏 ELF」的通行证，白名单机制仍在，只是这两个文件的缺库状态与真机一致。

判据是「这个现象在真机上也一样吗」。一样就进 `XFAIL`，不一样就是真缺陷。

## 本地构建

```bash
git clone --recurse-submodules https://github.com/distrotwin/kylin.git
cd kylin
```

两条构建路径都要一个 Debian 13 的 builder 容器（selfhost 用它跑阶段一，mmdebstrap 整个跑在里面）：

```bash
docker build -f buildkit/Dockerfile.builder -t dosbuild-cache buildkit/
docker run -d --name dosb --privileged --init -v "$PWD:/w" dosbuild-cache sleep infinity
```

V10 SP1 与 V4 在宿主起，脚本自己进容器完成阶段一，并自己把产物导成镜像：

```bash
sudo ARCH=amd64 ROOT=$PWD DID=v10sp1 buildkit/build/build-selfhost.sh micro base devel
sudo ARCH=amd64 ROOT=$PWD buildkit/test/verify.sh v10sp1
```

V11 走 `mmdebstrap`，**必须在容器里跑**：宿主 Ubuntu 的 apt 与 mmdebstrap 差了几代，`signed-by` 的解释不同，在宿主上会以 `NO_PUBKEY` 告终，尽管 keyring 里就有那把钥匙。而且 `build.sh` 只落 tarball 不建镜像，导入是单独一步：

```bash
sudo ARCH=amd64 ROOT=$PWD buildkit/tools/mk-localrepo.sh v11
docker exec dosb bash -c 'umask 022; ROOT=/w BK=/w/buildkit ARCH=amd64 \
  /w/buildkit/build/build.sh v11 micro base devel'
sudo ARCH=amd64 ROOT=$PWD buildkit/build/import.sh v11 micro base devel
sudo ARCH=amd64 ROOT=$PWD buildkit/test/verify.sh v11
```

跨架构构建需要宿主装 `qemu-user-static` 与 `binfmt-support`，且 **QEMU 要够新**：LoongArch 的用户态模拟是 QEMU 7.1 才加的，Ubuntu 22.04 只带 6.2。不要引入 `tonistiigi/binfmt` 容器，实测它会破坏本来可用的 binfmt 注册。

## CI

构建只允许手动触发（`workflow_dispatch`），因为一次要跑二十来个 job、拉几个 GB。两个开关：

- `include-loongarch`：是否一并构建 V11 的 loong64（默认关，需 QEMU 模拟）
- `publish`：测试全绿后是否发布到 GHCR（默认关）

发布是独立阶段，要求 `test` 与 `report` 都成功才执行。
