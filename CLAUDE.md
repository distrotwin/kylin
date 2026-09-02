# 开发指引

银河麒麟桌面 V4 / V10 SP1 / V11 的构建与测试镜像。定位是「与真实桌面系统尽可能一致的环境，用于软件构建与测试」，不是服务器镜像，不面向生产部署。动手前先读 README 开头那两节，它决定了什么算缺陷、什么算忠实还原。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引。源索引里 `gcc` 的版本是元包版本（V11 是 `4:12.2.0-ok1k0.2`），实际编译器是 `12.3.0`

## 这个仓库放什么

很薄的一层：三个配置加一份调用 workflow，机器码全在 submodule 里。

```
distros/{v4,v10sp1,v11}.conf   # 各版本的源、suite、包列表、ABI 基线、XFAIL
.github/workflows/build.yml    # 只负责调用 buildkit 的可复用 workflow
buildkit/                      # submodule → distrotwin/buildkit
CLAUDE.md / AGENTS.md          # 后者是指向前者的符号链接
```

判断改动该落在哪边：**只跟某个麒麟版本自己的事实有关**（源地址、suite、包列表、ABI 基线、这一版特有的怪癖）就改 `distros/*.conf`；**跟怎么构建、怎么测、怎么发有关**就改 `buildkit`。绝大多数改动属于后者。

## 配置文件都有什么

| 字段 | 作用 | 改的时候注意 |
|---|---|---|
| `METHOD` | `mmdebstrap` 或 `selfhost`，决定走哪条构建路径 | 换路径等于换执行环境，见下面「必知事实」 |
| `MIRROR` / `SUITE` / `COMPONENTS` | 源、套件、组件 | suite 名有坑，见下 |
| `IMAGE` | 本地镜像 tag 前缀 | **必须按版本唯一**，共用会让不同版本互相覆盖 |
| `EXPECT_GLIBC` / `EXPECT_LIBSTDCPP` / `EXPECT_GLIBCXX` | ABI 基线，验收时逐项对账 | 留空会因「期望空 == 实际空」被判通过，宁可填推导值让它报错 |
| `case "$ARCH" in ...` | 按架构覆盖上面的基线 | LoongArch 常常是独立产品线，基线与另两个架构不同 |
| `MICRO_INCLUDE` / `BASE_INCLUDE` / `DEVEL_INCLUDE` | 三个档位的包列表 | 不装 Recommends，缺可选依赖是预期行为 |
| `REPACK_DEBS` / `STUB_PROVIDES` / `PIN_NEVER` | 绕厂商坏包、顶替内核态组件、钉死不许装的包 | 每一条都该有注释说明绕的是哪个具体缺陷 |
| `XFAIL` | 声明注定红且红得有道理的检查项 | 判据是「这个现象在真机上也一样吗」 |

`EXPECT_GLIBC` 只按 `^[0-9]+\.[0-9]+` 前缀对账，所以厂商修订号（`2.38-1ok6.9k0.5` 与 `2.38-1ok6.12`）不在它的覆盖范围内——那两个值只出现在实测结果与镜像 label 里。

## 改动到跑通的完整回路

改 `buildkit` 之后必须同步两处 pin：submodule 钉脚本，`uses: ...@<sha>` 钉 workflow，**两者必须是同一个 commit**。CI 第一步就断言这件事：从 `.github/workflows/` 里 grep 出所有 40 位 SHA，要求唯一且等于 submodule 的 HEAD。不要用 `github.job_workflow_sha` 做这个断言，实测是空串。

```bash
git -C buildkit push origin HEAD
NEW=$(git -C buildkit rev-parse HEAD)
sed -i "s/<旧SHA>/$NEW/g" .github/workflows/build.yml
git -C buildkit checkout "$NEW"
git add buildkit .github/workflows/build.yml

# 验证而不是假设：把两边实际取到的值打出来比对
git ls-files -s buildkit | awk '{print $2}'
grep -o '\.yml@[0-9a-f]*' .github/workflows/build.yml | cut -d@ -f2 | sort -u
```

**同时维护两个仓库时一律用绝对路径或 `git -C`**，不要靠 `cd` 建立目录上下文。Bash 的工作目录跨调用会漂，麻烦的不是报错，而是命令静默作用到错误的仓库——打印出来的 short SHA 读起来像是另一个仓库的，于是把「已同步」当成事实继续往下走。

## 本地构建与验证

两条路径都要一个 Debian 13 的 builder 容器：

```bash
docker build -f buildkit/Dockerfile.builder -t dosbuild-cache buildkit/
docker run -d --name dosb --privileged --init -v "$PWD:/w" dosbuild-cache sleep infinity
```

V10 SP1 与 V4 走 selfhost，在宿主起，脚本自己进容器完成阶段一，并自己把产物导成镜像：

```bash
sudo ARCH=amd64 ROOT=$PWD DID=v10sp1 buildkit/build/build-selfhost.sh micro base devel
sudo ARCH=amd64 ROOT=$PWD buildkit/test/verify.sh v10sp1
```

V11 走 mmdebstrap，整个跑在容器里，而且 `build.sh` 只落 tarball 不建镜像，导入是单独一步：

```bash
sudo ARCH=amd64 ROOT=$PWD buildkit/tools/mk-localrepo.sh v11
docker exec dosb bash -c 'umask 022; ROOT=/w BK=/w/buildkit ARCH=amd64 \
  /w/buildkit/build/build.sh v11 micro base devel'
sudo ARCH=amd64 ROOT=$PWD buildkit/build/import.sh v11 micro base devel
sudo ARCH=amd64 ROOT=$PWD buildkit/test/verify.sh v11
```

`verify.sh` 认 `TIERS` 与 `DISTROS` 两个环境变量，调试时可以只跑一个档位。

## 跑 CI

只能手动触发：

```bash
gh workflow run build.yml --repo distrotwin/kylin -f publish=true -f include-loongarch=true
```

- `include-loongarch` 默认关：loong64 要 QEMU 模拟，devel 档约 30 分钟
- `publish` 默认关：只有明确要发布、且拿到授权时才开

三阶段串行：构建（每个「版本 × 架构 × 档位」一个 job，产物走 artifact）→ 测试（在干净机器上装载、启动、跑完整检查集）→ 发布（要求 `test` 与 `report` 都成功）。

改了 `buildkit` 之后**不能用 `gh run rerun --failed`**：`uses:` 的 pin 钉在调用方 commit 上，重跑会照旧用旧版脚本，看起来像修的没生效。

## 必知事实

下面这些都是实测踩出来的，别凭直觉推翻。

**suite 名有坑。** V10 SP1 用 `10.1` 不是 `10.0`：后者的 `Release` 自述 `Label: v10-离在线更新推送`，是给已装机器推更新的差异源，`universe` 四万多个包却没有 `libc6`。V4 用 `4.0.2-desktop` 不是 `4.0.2`：`debootstrap` 会核对请求的 suite 名与 `Release` 自述是否一致，apt 不核对。glibc 在这台归档上位于 `universe` 而非 `main`，只配 `main` 会得到一个没有 libc6 的源。

**mmdebstrap 必须在 Debian 13 容器里跑。** 在宿主 Ubuntu 上会以 `NO_PUBKEY` 告终，尽管 keyring 里就有那把钥匙、`gpgv` 单独验也能过。真因是执行环境不对，不是 key 不对。查这类问题先跟已知能跑通的调用逐字对比，别发明新的 apt 选项。

**LoongArch 有两个不通用的世界。** V10 SP1 是 `loongarch64` 旧世界（`/lib64/ld.so.1`，glibc 2.28 仍调 syscall 79/80），上游 QEMU 没实现那两个调用，最小 rootfs 实测表现是读出 ELF 头后报 `Unknown syscall 80` 退出 127，所以直接不列入；V11 是 `loong64` 新世界，可以模拟构建。用户态模拟要 QEMU 7.1 以上，Ubuntu 22.04 只带 6.2，所以 loong 的 job 跑在 ubuntu-24.04。**不要引入 `tonistiigi/binfmt` 容器**，它会破坏本来可用的 binfmt 注册。

**本地镜像 tag 必须带架构后缀。** 发布阶段要把同一档位的多个架构装进同一个守护进程，共用 `repo:tag` 会让 `docker load` 把 tag 从先装的身上摘走，日志里只留一句 `renaming the old one ... to empty string`。后果是同一份 rootfs 被推成多个架构 tag，而 CI 全绿。这个缺陷真发生过，六次 push 引用同一层。

**`docker import` 要显式给 `--platform linux/$ARCH`。** 默认按守护进程的架构写 config，loong64 在 amd64 runner 上导入会被标成 amd64，而 `docker manifest create` 的平台正是取自这个字段。

**给已有镜像打 label 用 `docker create` + `docker commit`，不能用 `docker build`。** BuildKit 解析 `FROM` 时要求目标平台与本地镜像一致，而发布 job 在 amd64 上还要给 arm64/loong64 打 label，它会绕开本地镜像去 registry 拉同名镜像，报 `pull access denied`，看起来像依赖了 Docker Hub 上的同名镜像。用 commit 时注意两点：`docker create` 不能带命令，否则 commit 会把它写成新的 `CMD`；commit 记录的是**容器**配置，docker CLI 会把 `config.json` 里的 proxies 注入成容器环境变量并一起烧进镜像。

**V11 的 `gcc` 是厂商包装脚本**，每次调用往 stderr 吐一行 `grep: /CurrentlyBuilding: No such file or directory`，编译本身正常。V10 SP1 与 V4 的 `gcc` 是原生 ELF。

**容器里 `uname` 报的是宿主内核**，跟麒麟无关，判断身份要读 `/etc/os-release`。V11 不声明 `ID_LIKE`，V10 SP1 与 V4 声明 `debian`。

## 门禁

发布路径上的每道门禁都对应一个已经发生过的缺陷，别因为「看起来多余」就删。

| 门禁 | 防的是什么 |
|---|---|
| 各架构制品的层指纹互不相同、平台戳符合预期 | 本地 tag 撞车导致多个架构指向同一份 rootfs |
| 打 label 前后语义字段逐项不变 | commit 把容器状态（如宿主的 proxy 环境变量）烧进镜像 |
| 任何 label 的值不得为空 | 空值 label 看起来像「量过了，值就是空」，比缺失更糟 |
| 发布后校验 manifest list 的平台集合与成员 digest 数 | 各架构指向同一份镜像时，manifest list 照样建得出来 |
| 构建出口断言：不允许「退出码 0 但没有产物」 | selfhost 路径曾打印一句提示就 exit 0 |
| submodule 与 workflow 的 pin 相等 | 两处版本不一致时，跑的脚本不是你以为的那份 |

加门禁时要**对门禁本身做双向变异测试**：制造它该拦的情形确认真拦得住，制造它该放行的情形确认不误伤。有一道门禁只验了单向就上线，把九个发布 job 全卡死——判据写成了「除 Labels 外全等」，而 `docker commit` 合法地会改 `Hostname`、`Image`、`AttachStdout/Stderr`。正确的写法是显式列出必须不变的语义字段。

## 发布与验收

别拿 CI 自己的日志当发布成功的证据，用匿名视角查 registry 里真实躺着什么：

```bash
TOK=$(curl -s "https://ghcr.io/token?scope=repository:distrotwin/kylin:pull&service=ghcr.io" \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
curl -s -H "Authorization: Bearer $TOK" \
  -H 'Accept: application/vnd.oci.image.index.v1+json' \
  https://ghcr.io/v2/distrotwin/kylin/manifests/v11-devel | python3 -m json.tool
```

要看四件事：平台集合对不对、成员 digest 是否两两不同、config blob 里的 label 有没有空值、平台戳与 `cn.internal.arch` 是否一致。

日期 tag 取的是发布时仓库 HEAD 的 commit 日期（UTC），同一天内重复发布会覆盖它——允许，但发布日志会打印被覆盖的旧 manifest 摘要。

## 排错

这个项目里的缺陷大多**不在报错的地方**：静默跳过、被 `|| true` 吞掉的错误、空着传下去的变量、症状离真因三步远。

- 先在本地复现，别拿 CI 当实验台
- 看到「源不可达」「key 不对」「需要一小时」这类结论，先分清它是观察还是推断。这三条都真出现过，真因分别是环境变量没传到、执行环境不对、网络偶发卡顿，照着推断改会走到自建镜像源、换 keyring、加超时上限这些完全错误的方向
- 日志里的 warning 要读。有四个 label 空了一整轮才被发现，警告当时就打在日志里
- 同一现象在两条构建路径上可能由不同原因造成，改完两边都要验；只改一边的话，另一边会以「看起来无关」的形式在很久以后炸开
