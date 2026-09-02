# 在这个仓库里干活

银河麒麟桌面 V4 / V10 SP1 / V11 的**构建与测试镜像**。定位是「与真实桌面系统尽可能一致的环境，用于软件构建与测试」，不是服务器镜像、不面向生产部署。改动前先读 README 的「这是什么，不是什么」。

## 硬性约定

- **commit 一律用 HansBug 身份**（`HansBug <hansbug@buaa.edu.cn>`），**不准带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务，那不是本仓库的议题

## 仓库结构

本仓库很薄：三个配置 + 一份调用 workflow，机器码全在 submodule 里。

```
distros/{v4,v10sp1,v11}.conf   # 源、suite、包列表、ABI 基线、XFAIL
.github/workflows/build.yml    # 只负责调用 buildkit 的可复用 workflow
buildkit/                      # submodule → distrotwin/buildkit
```

绝大多数改动应该落在 `buildkit`，本仓库只在「某个麒麟版本自己的事实变了」时才改。

## 两处 pin 必须同步

submodule 钉脚本，`uses: ...@<sha>` 钉 workflow，**两者必须是同一个 commit**。CI 第一步就断言这件事（从 `.github/workflows/` 里 grep 出所有 40 位 SHA，要求唯一且等于 submodule 的 HEAD）。注意不要用 `github.job_workflow_sha` 做这个断言，实测是空串。

改完 buildkit 之后的完整流程：

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

**同时维护两个仓库时一律用绝对路径或 `git -C`**，不要靠 `cd` 建立目录上下文。Bash 的工作目录跨调用会漂，最危险的不是报错，而是命令静默作用到错误的仓库——打印出来的 short SHA 读起来像是另一个仓库的。

## 跑 CI

只能手动触发：

```bash
gh workflow run build.yml --repo distrotwin/kylin -f publish=true -f include-loongarch=true
```

- `include-loongarch` 默认关：loong64 要 QEMU 模拟，devel 档约 30 分钟
- `publish` 默认关：只有明确要发布时才开，且要有用户授权

三阶段串行：构建（每个「版本 × 架构 × 档位」一个 job）→ 测试（干净机器上装载、启动、跑完整检查集）→ 发布（要求 `test` 与 `report` 都成功）。改了 `buildkit` 之后**不能用 `gh run rerun --failed`**：`uses:` 的 pin 钉在调用方 commit 上，重跑会照旧用旧版脚本。

## 动手前必须知道的事实

这些都是实测踩出来的，不要凭直觉推翻：

**suite 名有坑。** V10 SP1 用 `10.1` 不是 `10.0`（后者是给已装机器推更新的差异源，`universe` 四万多个包却没有 `libc6`）；V4 用 `4.0.2-desktop` 不是 `4.0.2`（`debootstrap` 核对请求的 suite 名与 `Release` 自述是否一致，apt 不核对）。glibc 在这台归档上位于 `universe` 而非 `main`。

**mmdebstrap 必须在 Debian 13 容器里跑。** 在宿主 Ubuntu 上会以 `NO_PUBKEY` 告终，尽管 keyring 里就有那把钥匙——真因是执行环境不对，不是 key 不对。查这类问题时先跟能跑通的调用逐字对比，不要发明新的 apt 选项。

**LoongArch 有两个不通用的世界。** V10 SP1 是 `loongarch64` 旧世界（`/lib64/ld.so.1`，glibc 2.28 仍调 syscall 79/80），上游 QEMU 跑不了，所以不列入而不是留着让它失败；V11 是 `loong64` 新世界，可以模拟构建。用户态模拟要 QEMU 7.1 以上，Ubuntu 22.04 只带 6.2，所以 loong 的 job 跑在 ubuntu-24.04。**不要引入 `tonistiigi/binfmt` 容器**，它会破坏本来可用的 binfmt 注册。

**本地镜像 tag 必须带架构后缀。** 发布阶段要把同一档位的多个架构装进同一个守护进程，共用 `repo:tag` 会让 `docker load` 把 tag 从先装的身上摘走（日志里只有一句 `renaming the old one ... to empty string`），结果是同一份 rootfs 被推成多个架构 tag，而 CI 全绿。这个缺陷真发生过。

**`docker import` 要显式给 `--platform linux/$ARCH`。** 默认按守护进程架构写 config，loong64 在 amd64 runner 上导入会被标成 amd64，而 `docker manifest create` 的平台正是取自这个字段。

**给已有镜像打 label 用 `docker create` + `docker commit`，不能用 `docker build`。** BuildKit 解析 `FROM` 时要求目标平台与本地镜像一致，发布 job 在 amd64 上要给 arm64/loong64 也打 label，它会绕开本地镜像去 registry 拉同名镜像，报 `pull access denied`——看起来像依赖了 Docker Hub 上的同名镜像。用 commit 时注意两点：`docker create` 不能带命令（commit 会把它写成新的 CMD）；commit 记录的是**容器**配置，docker CLI 会把 `config.json` 里的 proxies 注入成容器环境变量并一起烧进镜像。

**V11 的 `gcc` 是厂商包装脚本**，每次调用往 stderr 吐一行 `grep: /CurrentlyBuilding: No such file or directory`，编译本身正常。

**元包版本不等于编译器版本。** 源索引里 `gcc` 的版本是元包版本（V11 是 `4:12.2.0-ok1k0.2`），实际编译器是 12.3.0。要写进文档的版本号一律跑镜像实测：`gcc -dumpfullversion`。

## 门禁

发布路径上的门禁都是为已经发生过的缺陷加的，不要因为「看起来多余」就删：

| 门禁 | 防的是什么 |
|---|---|
| 各架构制品层指纹互不相同 + 平台戳符合预期 | 本地 tag 撞车导致多个架构指向同一份 rootfs |
| 打 label 前后语义字段逐项不变 | commit 把容器状态（如宿主的 proxy 环境变量）烧进镜像 |
| 任何 label 的值不得为空 | 空值 label 看起来像「量过了，值就是空」，比缺失更糟 |
| 发布后校验 manifest list 的平台集合与成员 digest 数 | 各架构指向同一份镜像时，manifest list 照样建得出来 |
| 构建出口断言：不允许「退出码 0 但没有产物」 | selfhost 路径曾打印一句提示就 exit 0 |

**加门禁时要对门禁本身做变异测试**：制造它该拦的情形，确认它真的拦；制造它该放行的情形，确认它不误伤。本会话有一道门禁只验了单向，上线后把九个发布 job 全卡死——判据写成了「除 Labels 外全等」，而 `docker commit` 合法地会改 `Hostname`、`Image`、`AttachStdout/Stderr`。

## 验收

不要拿 CI 自己的日志当发布成功的证据，用匿名视角查 registry 里真实躺着什么：

```bash
TOK=$(curl -s "https://ghcr.io/token?scope=repository:distrotwin/kylin:pull&service=ghcr.io" \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
curl -s -H "Authorization: Bearer $TOK" \
  -H 'Accept: application/vnd.oci.image.index.v1+json' \
  https://ghcr.io/v2/distrotwin/kylin/manifests/v11-devel | python3 -m json.tool
```

要看的是：平台集合对不对、成员 digest 是否两两不同、config blob 里的 label 有没有空值、平台戳与 `cn.internal.arch` 是否一致。

## 排错时的一般原则

这个项目里的缺陷大多**不在报错的地方**：静默跳过、被 `|| true` 吞掉的错误、空着传下去的变量、症状离真因三步远。所以：

- 先在本地复现，不要拿 CI 当实验台
- 看到「源不可达」「key 不对」「需要一小时」这类结论时，先确认它是观察还是推断——本会话三次差点据此做出错误的架构决策，真因分别是环境变量没传、执行环境不对、网络偶发卡顿
- 日志里的 warning 要读。有四个 label 空了一整轮，警告当时就打在日志里
