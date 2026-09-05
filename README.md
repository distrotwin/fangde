# 方德桌面操作系统 · 构建与测试镜像

对着中科方德公开 apt 源自举出来的容器环境，用于**软件构建、打包与兼容性测试**。覆盖三代桌面线：`v3.1`（cdos，glibc 2.24）、`panda`（Debian 10 系，2.28）、`tiger`（Debian 11 系，2.31），最多 amd64 / arm64 两架构、各三档，公开在 GHCR。最近一轮 15 个镜像、630 项检查全部通过，零异常。

```bash
docker run --rm ghcr.io/distrotwin/fangde:tiger-devel \
  bash -c 'grep PRETTY /etc/os-release; ldd --version | head -1; gcc -dumpfullversion'
```

## 这是什么，不是什么

镜像里没有内核——这是所有 Linux 容器镜像的常态，容器共享宿主内核。方德的 IMA 度量、安全标签这类机制依赖内核态，装在镜像里也不生效；`systemd` 有二进制但不是 PID 1。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在方德桌面上跑的二进制与 deb；验证依赖闭包；检查产物需要的 glibc / libstdc++ 符号版本目标系统能否满足；复现只在这个系统上出现的编译问题。

**别用它**：当生产运行时基础镜像；复现内核相关行为；当作系统的完整替代品做验收测试。

## 先跑一遍

进容器，写个 A+B，编了跑，再看符号天花板。

```bash
docker run -it --rm ghcr.io/distrotwin/fangde:tiger-devel /bin/bash
```

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

最后那行是这套镜像最有用的一句：**它直接告诉你产物需要目标系统多新的 glibc**。在 tiger 线上答案不该超过 `GLIBC_2.31`。

## 选哪一个

| 代 | 底座（实测身份） | glibc / gcc | 架构 | 状态 |
|---|---|---|---|---|
| `v3.1` | cdos 3.0（Mint/Ubuntu 包混于 Debian 9 底盘，`lsb-base 4.1+Debian11ubuntu6mint1+1cdos2`） | 2.24 / 6.2.1 | amd64 | 2022-01 停更 |
| `panda` | Debian 10 buster 系 | 2.28 / 8.3.0 | amd64+arm64 | 2022-11 停更 |
| `tiger` | Debian NFSDesktop 11 (bullseye) | 2.31 / 10.2.1 | amd64+arm64 | 现役（2025-10） |

每代三档：`<代>-micro`（最小根系统，无 apt）、`<代>-base`（+apt/python3/systemd）、`<代>-devel`（+gcc/g++/make/dpkg-dev）。`latest` 指向 `tiger-devel`。

panda 的 suite 拓扑要如实说明：通用 `panda` 是厂商覆盖层（不含 libc6/dpkg，不自洽），它的 upstream 索引是空目录；完整 buster 上游取自设备线（amd64 用 g120，arm64 用 kylin990），security 同线，合并时厂商覆盖层按版本正常胜出。panda 镜像的 pam `common-*` 由构建注入 buster 规范默认值——厂商 libpam-modules 的 postinst 假定它们已存在，而那本该由 pam-auth-update 生成。

## 为什么叫 tiger 而不是某个版本号

厂商的 `os-release` 自述是 `Debian NFSDesktop 11 (bullseye)`，公开源里的产品线代号是 `tiger`（前代 `panda` 对应桌面 3.1）。tiger 对应哪个市场版本号（4.0 还是 5.0）没有公开可验的依据，所以版本标识如实用代号，不用猜出来的数字。

## 龙芯为什么不在

方德全部三处龙芯树都查过：`update.os` 的 NFS4.0/LoongarchOS、`repos.os` 的 tiger-loongarch、`rpm/nfs` 的 nfs-soaring-loongarch——全是**旧世界** ABI（动态链接器 `/lib64/ld.so.1`、glibc 2.28，新世界的 `ld-linux-loongarch-*` 零命中），上游 QEMU 不支持旧世界的信号系统调用，托管 runner 上造不出来。判据与完整排查记录见 buildkit 的 `docs/downstream-repo.md`。新世界 loong64 在方德没有公开材料，查无。

## 镜像是怎么造的

方德的源主机对 GitHub 托管 runner 不可达（官网同一 runner 1.7 秒通，源主机全部 135 秒超时；判据见 buildkit 的 `docs/downstream-repo.md`）。所以取材走**数据镜像**：在能连通厂商站的机器上把 `tiger` + `tiger-security` + `tiger-upstream` 三个 suite 合并（同名包取版本最高者），算依赖闭包裁到每架构约 140 MB，逐文件核对厂商索引的 SHA256 后推成 `ghcr.io/distrotwin/scratch:nfsdesktop-11-20251011-<arch>`；CI 构建时取回，先对 conf 钉死的 manifest 指纹、再逐文件核对后才使用。构建走两阶段 debootstrap，用厂商自己的 dpkg 完成自举。

两点如实标注。其一，厂商的包有 PGP 签名但**公钥不可得**：`nfs-gpg-keys` 包里装的是 Oracle 的公钥和一把 OBS 默认无口令 key，厂商自己的 repo 配置就写着 `gpgcheck=0`——所以完整性锚点是「conf 钉 manifest → manifest 覆盖每个文件 → `.origin` 记厂商三个 suite 的 Release 指纹与逐包 SHA256 来源」这条校验和链，不是签名，两者不是一回事。其二，厂商把 425 MB 的 360 安全浏览器标成了 `Priority: required`，构建镜像不该带浏览器，它被从入口排除（`PIN_NEVER`）。

## 镜像与厂商源的关系

镜像等于对公开源快照（2025-10-11）按依赖闭包装出来的最小系统，不等于装好的方德桌面。被排除的只有 `360epp`（见上）与内核相关包；`libcrypt1` 等厂商改过的基础库（`+m*+*nfs5` 版本）原样保留。厂商装机介质走网盘分发、无从公开校验，所以对照的基准是公开源而不是介质。

## 认出自己在哪个系统上

```bash
docker run --rm ghcr.io/distrotwin/fangde:tiger-micro cat /etc/os-release
```

## tag 与钉版

`tiger-<tier>` 是活动 tag，指向最近一次发布；`tiger-<tier>-<日期>` 是不可变 tag。CI 里建议钉日期 tag 或直接钉 digest。

## 镜像自带的溯源信息

每个镜像的 label 里带着：数据镜像 tag 与 manifest 指纹（`cn.internal.srcdata-image` / `cn.internal.srcdata-manifest-sha256`）、构建仓库 commit 与 run。`docker inspect` 即可读。

## 本地构建

需要能连通厂商站的网络位置才能重造数据镜像（`buildkit/tools/srcdata-make.sh`）；只重建容器镜像的话，任何机器都可以：

```bash
git clone --recursive https://github.com/distrotwin/fangde && cd fangde
ROOT=$PWD ARCH=amd64 buildkit/tools/srcdata-fetch.sh tiger
sudo DID=tiger ARCH=amd64 buildkit/build/build-selfhost.sh micro
```

## CI

手动触发 `构建镜像` workflow；勾选 `publish` 才会推 GHCR。构建、测试、汇总、发布四段全部来自 buildkit 的可复用 workflow，本仓库只有矩阵。

## 仓库结构

```
fangde/
├── distros/tiger.conf  # 版本定义：数据镜像指纹、种子、基线，全部钉死
├── .github/workflows/  # 只有矩阵，实现都在 buildkit
└── buildkit/           # submodule，构建、测试、发布的全部实现
```
