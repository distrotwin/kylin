# 银河麒麟桌面操作系统 · 构建与测试镜像

与真实银河麒麟桌面系统尽可能一致的容器环境，用于**软件构建、打包与兼容性测试**。三个大版本、三个档位、跨三种指令集，全部从麒麟官方 apt 归档自举，公开在 GHCR。

```bash
docker run --rm ghcr.io/distrotwin/kylin:v11-devel \
  bash -c 'cat /etc/os-release | grep PRETTY; ldd --version | head -1; gcc -dumpfullversion'
```

```
PRETTY_NAME="Kylin V11"
ldd (Ubuntu GLIBC 2.38-1ok6.9k0.5) 2.38
12.3.0
```

## 这是什么，不是什么

镜像里没有内核。内核态组件（initramfs、TPM、KYSEC LSM 插件）由假包顶替或干脆不装，systemd 只保留容器内有意义的部分。真机上依赖内核特性、安全模块或硬件的行为，在这里不成立。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在麒麟上跑的二进制；验证 `.deb` 打包与依赖；检查产物需要的 glibc / libstdc++ 符号版本目标系统能否满足；复现只在麒麟上出现的编译问题。

**别用它**：当生产运行时基础镜像；复现内核相关的行为；当作麒麟系统的完整替代品做验收测试。后面这几件事得用真机或虚拟机。

## 先跑一遍

进容器，写个 A+B，编了跑。三个版本都做一遍，你就知道该拿它干什么了。

```bash
docker run -it --rm ghcr.io/distrotwin/kylin:v11-devel /bin/bash
```

进去以后：

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

```
7
GLIBC_2.34
```

（V11 的 `gcc` 会额外吐一行 `grep: /CurrentlyBuilding: No such file or directory`，那是厂商包装脚本的动静，不影响编译，见下面「怪癖」一节。）

换成另外两个版本，同一份代码、同样输出 `7`，**符号底线却不一样**：

| 镜像 | 运行结果 | 产物要求的最高 glibc 符号 |
|---|---|---|
| `v11-devel` | `7` | `GLIBC_2.34` |
| `v10sp1-devel` | `7` | `GLIBC_2.7` |
| `v4-devel` | `7` | `GLIBC_2.7` |

这个差别决定了产物能往哪儿送。把 V11 编的 `ab` 拿到 V4 里跑：

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v4-micro \
  bash -c 'echo "3 4" | ./ab'
```

```
./ab: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./ab)
```

反过来，V4 编的产物在 V11 上跑得好好的，输出 `7`。**兼容性是单向的：在老版本上编，往新版本上送。** 一个连库都没链的 A+B 就能撞上这堵墙——glibc 2.34 把 `libpthread` 与 `libdl` 并进了 libc，符号版本随之抬高。真实项目只会撞得更早。

## 选哪一个

```
ghcr.io/distrotwin/kylin:<版本>-<档位>
```

**先选版本**，按你要交付的目标机器：

| 版本 | 系统 | glibc | GCC | 上游血脉 | 架构 |
|---|---|---|---|---|---|
| `v11` | 银河麒麟桌面 V11（2603） | 2.38 | 12.3 | openKylin 2.0 | amd64 · arm64 · **loong64** |
| `v10sp1` | 银河麒麟桌面 V10 SP1（2503） | 2.31 | 9.3 | Ubuntu 20.04 | amd64 · arm64 |
| `v4` | 银河麒麟桌面 V4（4.0.2） | 2.23 | 5.4 | Ubuntu 16.04 | amd64 · arm64 |

不确定目标机器是哪一版，就三个都过一遍，见下面「三个 ABI 世代同时验」。

**再选档位**，按你要在里面做什么：

| 档位 | 装了什么 | 什么时候用 |
|---|---|---|
| `micro` | CA 证书、时区、locale、libstdc++ | 跑已经编好的二进制，验证它在目标系统上能不能起来 |
| `base` | 加常用命令行工具、python3、systemd | 平台脚本、集成测试 |
| `devel` | 加 `build-essential`、`pkg-config`、`dpkg-dev` | **编译与打包** |

不带档位后缀的 tag 指向 `devel`，与 `golang:1.21` 是完整版的惯例一致；`latest` 是最高版本的 `devel`，也就是 `v11-devel`。

```bash
docker pull ghcr.io/distrotwin/kylin:v11-devel
docker pull ghcr.io/distrotwin/kylin:v11          # 同上
docker pull ghcr.io/distrotwin/kylin:v10sp1-base
docker pull ghcr.io/distrotwin/kylin:v4-micro
docker pull ghcr.io/distrotwin/kylin:latest       # = v11-devel
```

## 怎么用

**编一个程序，并确认它的 ABI 需求**

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v10sp1-devel bash -c '
  gcc -O2 -o app app.c
  objdump -T app | grep -oE "GLIBC_[0-9.]+" | sort -uV | tail -1
'
```

最后那行是产物需要的最高 glibc 符号版本。它不高于目标机器的 glibc，产物才跑得起来。C++ 产物同理，把 `GLIBC_` 换成 `GLIBCXX_`。

**三个 ABI 世代同时验**

```bash
for v in v4 v10sp1 v11; do
  printf '%-8s ' "$v"
  docker run --rm -v "$PWD:/w" -w /w "ghcr.io/distrotwin/kylin:$v-devel" \
    bash -c 'gcc -O2 -o /tmp/app app.c 2>/dev/null && echo 通过' || echo 失败
done
```

一次改动，三代 ABI 一起过。`v4` 那档尤其能暴露对新 glibc / 新 C++ 标准的隐式依赖——它是 Ubuntu 16.04 血脉，GCC 5.4 默认还不是 C++14。

**在 GitHub Actions 里当构建环境**

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

**验证一个已有产物能不能在目标系统上起来**（用 `micro`，最接近「装完系统什么都没多装」的状态）

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v4-micro \
  bash -c 'ldd ./app; ./app --version'
```

**跑非本机架构**（需要宿主注册 binfmt，见下方「本地构建」）

```bash
docker run --rm --platform linux/loong64 ghcr.io/distrotwin/kylin:v11-devel \
  bash -c 'echo $MACHTYPE; gcc -dumpmachine'
```

**交互式排查**

```bash
docker run --rm -it -v "$PWD:/w" -w /w ghcr.io/distrotwin/kylin:v11-devel bash
```

## 认出自己在哪个系统上

脚本里要分支、CI 里要判断目标平台，靠的是 `/etc/os-release`。它是 systemd 定的标准格式，可以直接 `source` 进 shell 变量。

```bash
docker run --rm ghcr.io/distrotwin/kylin:v11-base bash -c '
  . /etc/os-release
  echo "$PRETTY_NAME | ID=$ID | ID_LIKE=${ID_LIKE:-（无）} | VERSION_ID=$VERSION_ID"
  . /etc/lsb-release; echo "DISTRIB_RELEASE=$DISTRIB_RELEASE"
  echo "dpkg vendor: $(sed -n "s/^Vendor: //p" /etc/dpkg/origins/default)"
  echo "libc6: $(dpkg-query -W -f="\${Version}" libc6)"
'
```

三个版本的输出：

```
# v11
Kylin V11 | ID=kylin | ID_LIKE=（无） | VERSION_ID=v11
DISTRIB_RELEASE=V11
dpkg vendor: openKylin
libc6: 2.38-1ok6.9k0.5

# v10sp1
Kylin V10 SP1 | ID=kylin | ID_LIKE=debian | VERSION_ID=v10
DISTRIB_RELEASE=V10
dpkg vendor: Ubuntu
libc6: 2.31-0kylin9.1k20.3

# v4
Kylin 4.0-2 | ID=kylin | ID_LIKE=debian | VERSION_ID=4.0-2
DISTRIB_RELEASE=4.0-2
dpkg vendor: Ubuntu
libc6: 2.23-0kord10
```

有四处需要留意：

**`VERSION_ID` 分不出 SP。** V10 SP1 给的是 `v10`，SP 号不在里面。要区分得看 `PRETTY_NAME`（`Kylin V10 SP1`）或 `/etc/os-release` 里的 `PROJECT_CODENAME=v10sp1`。

**`ID_LIKE` 在 V11 上消失了。** V4 与 V10 SP1 都声明 `debian`，V11 不声明。`case "$ID_LIKE" in *debian*)` 这类写法会在 V11 上掉进 default 分支。判族系请用 `ID=kylin` 加包管理器探测（`command -v dpkg`），别指望 `ID_LIKE`。

**`dpkg vendor` 记着血脉换代。** V4 与 V10 SP1 写 `Ubuntu`，V11 写 `openKylin`——这是 V11 换上游的直接痕迹。

**包版本后缀是很硬的指纹。** `0kord10`（V4）、`0kylin9.1k20.3`（V10 SP1）、`1ok6.9k0.5`（V11，`ok` 即 openKylin）。只拿到一份包列表也能认出是哪一版。

另外两点值得单说。`lsb_release` 这个**命令**三个镜像都没装（它是独立的包），但 `/etc/lsb-release` 这个**文件**在，直接读文件就好。而在容器里 `uname` 报的是**宿主内核**，不是麒麟的：

```
容器内: Linux 73e0cdf8ec6f 6.8.0-137-generic #137-Ubuntu SMP ... x86_64
宿主  : Linux pjnl104220238 6.8.0-137-generic #137-Ubuntu SMP ... x86_64
```

一字不差。userland 是麒麟而 `uname` 说 Ubuntu，所以容器里判断身份不能用 `uname`。

## tag 与钉版

每个档位同时打这些 tag：

| tag | 性质 |
|---|---|
| `v11-devel` | 滚动，跟随最新构建 |
| `v11` | 滚动，= `v11-devel` |
| `latest` | 滚动，= 最高版本的 `devel` |
| `v11-devel-20260902` | 按 commit 日期，约定不动 |
| `v11-devel-20260902-amd64` | 单架构，manifest list 的成员 |

**CI 里请用带日期的那个。** 滚动 tag 会随重建变化，`latest` 还会随大版本推进跨越 ABI 边界，把你的构建环境从 glibc 2.31 换成 2.38。

日期取的是**发布时仓库 HEAD 的 commit 日期**（UTC）而非构建时刻，所以同一个 commit 无论何时重建都是同一个 tag。它锁不住的是上游源：同一 commit 隔一周重建，若麒麟归档更新过包则镜像字节不同而 tag 相同。

**要钉死请用 digest。** GHCR 的 tag 本来就可变，日期 tag 是约定而非强制；一天内多次发布会覆盖它，发布日志里会打印每个 tag 的 `@sha256:...`。

```bash
docker buildx imagetools inspect ghcr.io/distrotwin/kylin:v11-devel --format '{{.Manifest.Digest}}'
docker pull ghcr.io/distrotwin/kylin@sha256:<digest>
```

## 镜像自带的溯源信息

```bash
docker inspect --format '{{json .Config.Labels}}' \
  ghcr.io/distrotwin/kylin:v11-devel | python3 -m json.tool
```

| label | 内容 |
|---|---|
| `cn.internal.glibc` / `libstdcpp` / `glibcxx` | **实测**的 ABI 值，来自测试阶段在干净机器上跑出来的结果，不是配置里的期望值 |
| `cn.internal.arch` / `tier` / `suite` / `build-method` | 架构、档位、建自源的哪个 suite、走的哪条构建路径 |
| `cn.internal.kylin-commit` / `buildkit-commit` | 哪份配置与哪份脚本建的 |
| `cn.internal.build-run` | 跳回当时的 CI 日志与测试报告 |
| `org.opencontainers.image.*` | 标准溯源字段：源仓库、revision、version |

跨架构时这几个 label 尤其有用，原因见下节。要确认手里两个镜像是不是同一份，只能比 digest——镜像里没有记录源快照的 label，那个值只存在于构建机上。

## 架构与 ABI 分叉

amd64 与 arm64 由原生 runner 构建；V11 另有 loong64（QEMU 模拟构建，实测 micro 约 3 分钟、devel 约 30 分钟）。

**同一个版本号在不同架构下的 ABI 并不一致。**

| | amd64 / arm64 | loong64 |
|---|---|---|
| V11 | glibc `2.38-1ok6.9k0.5` · libstdc++ `6.0.32`（`GLIBCXX_3.4.32`） | glibc **`2.38-1ok6.12`** · libstdc++ **`6.0.33`**（**`GLIBCXX_3.4.33`**） |
| V10 SP1 | glibc `2.31` · gcc 9.3 | glibc **`2.28`** · gcc **8.3**（Loongnix 血脉） |

两支的差距不是一个量级。V10 SP1 差着一整个工具链世代；V11 的两支编译器同为 GCC 12.3，分叉在运行时——loong64 那支的 libstdc++ 反而更新一档，多出 `GLIBCXX_3.4.33` 这一级符号。

所以在 amd64 上验过的构建结论不能直接套用到 loong64。这是厂商产品线的现状，本仓库只是如实反映；每个镜像都把实测值写进了 label，`docker inspect` 一看就知道拉到的是哪一档。

**V10 SP1 与 V4 没有 LoongArch 镜像。** V10 SP1 的 `loongarch64` 是旧世界 ABI：它的 glibc 2.28 仍在调用 `syscall 79/80`（`newfstatat` / `fstat`），而 LoongArch 新世界把这两个调用去掉了，上游 QEMU 的 loongarch64 target 因此没有实现。最小 rootfs 实测的表现是打开 `libtinfo.so.6`、读出 ELF 头后报 `Unknown syscall 80` 退出 127。没有龙芯补丁版 QEMU 或真机就造不出来，所以直接不列入，免得留一个永远红着的 job。V4 则是源里根本没有任何 LoongArch 架构。

## 已知的怪癖与期望失败

**V11 的 `gcc` 是厂商的包装脚本，每次调用往 stderr 吐一行。**

```
grep: /CurrentlyBuilding: No such file or directory
```

`/usr/bin/gcc` 不是原生 ELF 而是麒麟的 shell 包装，它 grep 一个只存在于厂商构建系统里的 `/CurrentlyBuilding` 来决定要不要加安全加固选项。镜像里没有这个文件，于是每次调用都报一行。**编译本身照常成功**，这行噪声可以忽略；但如果你的 CI 把 stderr 当失败信号，需要单独放行。V10 SP1 与 V4 的 `gcc` 是原生 ELF，没有这个现象。

**V4 有一项注定红的检查。** 它比其余版本老一个时代，这类项在 `distros/v4.conf` 的 `XFAIL` 里声明，报告中标为 🟡 而不计为失败：

- `elf_broken`：V4 的 gnupg 会装一个链接 `libldap` 的 `gpgkeys_ldap`，而本项目不装 Recommends，`ldd` 必报缺库；ircd 的可选模块 `m_xt.so` 同理。**真实的 V4 装机同样如此**，镜像是忠实的。这条不是「允许有坏 ELF」的通行证，白名单机制仍在，只是这两个文件的缺库状态与真机一致。

判据是「这个现象在真机上也一样吗」。一样就进 `XFAIL`，不一样就是真缺陷。

## 镜像是怎么造的

三个版本**都不经过 ISO**，直接从麒麟的 apt 归档 `archive.kylinos.cn/kylin/KYLIN-ALL/` bootstrap。V11 走 `mmdebstrap`；V10 SP1 与 V4 走两段式自举，因为宿主 dpkg 写出的 `status` 这两个系统自带的旧 dpkg 读不了。

有两处容易踩：V10 SP1 的 suite 是 `10.1` 而不是 `10.0`（后者的 `Release` 自述 `Label: v10-离在线更新推送`，是给已装机器推更新的差异源，`universe` 有四万多个包却没有 `libc6`）；V4 要用 `4.0.2-desktop` 而不是 `4.0.2`（`debootstrap` 会核对请求的 suite 名与 `Release` 自述是否一致，而 apt 不核对）。glibc 在这台归档上位于 `universe` 而非 `main`，只配 `main` 会得到一个没有 libc6 的源。

每个镜像发布前都在**干净机器上**装载、真正启动、跑完整检查集——构建阶段的机器状态（builder 容器、本地源、宿主装的包）会掩盖镜像自身的缺陷。最近一轮 21 个镜像、905 项检查：899 通过、6 项期望不通过、**零异常**。测试报告与完整日志按系统打包在每次 run 的 artifact 里，成功与失败的都在。

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

三个阶段严格串行：**构建**（每个「版本 × 架构 × 档位」一个 job，产物走 artifact）→ **测试**（在干净机器上装载、启动、跑完整检查集，出按系统分文件的报告）→ **发布**（要求 `test` 与 `report` 都成功才执行）。

## 仓库结构

构建机器码在 [`distrotwin/buildkit`](https://github.com/distrotwin/buildkit)，本仓库以 submodule 引用并钉住 commit。本仓库自己只有三个 `distros/*.conf` 和一份调用 workflow。

```
distros/v4.conf  v10sp1.conf  v11.conf   # 各版本的源、suite、包列表、ABI 基线、XFAIL
.github/workflows/build.yml              # 调用 buildkit 的可复用 workflow
buildkit/                                # submodule：构建、测试、门禁、报告、发布
```
