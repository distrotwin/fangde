# 开发指引

这个仓库只做一件事：把中科方德桌面操作系统（tiger 线）的公开 apt 源变成可用于软件构建与测试的容器镜像。它不是方德系统的替代品，回答的是「编出来的东西对不对」而不是「跑起来的系统对不对」。定位边界见 README 的前两节。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引里的元包版本

## 这个仓库放什么

```
fangde/
├── buildkit/                     # submodule，钉住一个 commit
├── distros/tiger.conf            # 一个产品线一个，文件名即 DID
├── .github/workflows/build.yml   # 只定义矩阵、调用 buildkit 的可复用 workflow
├── README.md                     # 面向使用者
├── CLAUDE.md                     # 本文件
└── AGENTS.md                     # 与本文件同步
```

判断改动落在哪边只有一条：**只跟方德自己的事实有关**（数据镜像指纹、种子包、ABI 基线、这一线特有的怪癖）就进 `distros/tiger.conf`；**跟怎么构建、怎么测、怎么发有关**就进 buildkit。第二类占绝大多数。

## 必知事实

- **版本标识用代号 tiger，不用市场版本号。** 厂商 os-release 自述 `Debian NFSDesktop 11 (bullseye)`，tiger 对应 4.0 还是 5.0 没有公开可验的依据，宁可用代号也不写猜的数字
- **取材走数据镜像，不走厂商站。** 方德两台源主机对 GitHub runner 全量 135 秒超时（官网同 runner 1.7 秒通），介质由本机把 `tiger`+`tiger-security`+`tiger-upstream` 三 suite 合并切出后推 `ghcr.io/distrotwin/scratch`，判据见 `buildkit/docs/downstream-repo.md`、机制见 `buildkit/docs/srcdata.md`
- **`SRCDATA_MANIFEST_SHA256` 必须钉**（两个架构各一），不钉的话数据镜像被换掉不会被任何检查发现
- **METHOD 必须是 selfhost。** 厂商 libcrypt1 依赖 libssl1.1，形成 `libssl1.1→debconf→perl-base→libcrypt1` 的环，apt 的 immediate-configure 拆不开（mmdebstrap 两种配置实测都失败），dpkg 能拆
- **`STAGE1_TOUCH="etc/apt/apt.conf"` 不能删。** 厂商 apt 的 postinst 直接 `sed /etc/apt/apt.conf`，裸自举时它不存在 → exit 2
- **`PIN_NEVER=360epp` 不能删。** 厂商把 425 MB 的 360 浏览器标成 `Priority: required`，不拦会整个进镜像
- **介质 dists 代号记 bullseye，厂商原代号是 base。** `base` 不在 debootstrap 的年代白名单里，会被当成 bookworm+ 而强求 `usr-is-merged` 包；改名依据在数据镜像 `.origin` 里
- **厂商包有签名但公钥不可得**（`nfs-gpg-keys` 里装的是 Oracle 公钥和 OBS 默认无口令 key），完整性锚点是校验和链不是签名，README 里不能混写
- **loongarch64 是旧世界**（`/lib64/ld.so.1`、glibc 2.28、`.lns8`），托管 runner 造不出来，不进矩阵

## 改动到跑通的完整回路

改 conf → 本地跑一个档位（**别拿 CI 当实验台**）→ 推仓库跑 `publish=false` 的完整 CI → 看报告 artifact 确认零异常 → `publish=true` 发一轮 → 匿名视角验收 registry → 把 README 的数字对齐到这一轮实际结果。
