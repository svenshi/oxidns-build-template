# OxiDNS 自定义编译模版

中文 | [English](README_EN.md)

这是一个 **GitHub Template Repository** —— 用 "Use this template" 生成你自己的副本,
就能定制 [OxiDNS](https://github.com/svenshi/oxidns) 的 features 和目标平台,
在上游每次发布 release 时自动重新编译并发布到你自己的仓库 release。

## 工作方式

```
svenshi/oxidns                    your-name/oxidns-build (从模版生成)
─────────────                     ──────────────────────────────────
 发布 vX.Y.Z  ─每 30 分钟轮询─▶   watch-upstream.yml
                                          │
                                          ▼  读 build.config.yml
                                  build.yml (薄壳)
                                          │
                                          ▼  uses: svenshi/oxidns/...@main
                          ┌───────────────────────────────────┐
                          │ svenshi/oxidns/.github/workflows/ │
                          │      custom-build.yml             │  ← 编译矩阵在这里
                          │  (在调用方 runner 上执行)          │
                          └───────────────────┬───────────────┘
                                              │
                                              ▼
                                   发布到 your-name/oxidns-build releases
                                              │
                                              ▼
                       oxidns upgrade --repository your-name/oxidns-build
```

**关键点**:实际的编译矩阵、命名、打包逻辑全部在上游 `svenshi/oxidns`
的 [`.github/workflows/custom-build.yml`](https://github.com/svenshi/oxidns/blob/main/.github/workflows/custom-build.yml) 中维护。
派生仓库只持有一个薄壳,不需要自己复制粘贴这套矩阵。上游改了构建流程,
所有派生仓库通常**不需要更新 workflow** 就能自动跟上 —— 这也是把这套机制叫
"reusable workflow" 的原因。

但 `build.config.yml` 中的 custom feature 列表属于派生仓库自己的静态配置,
不会随上游自动迁移。上游新增、重命名或调整 feature 后,仍需参照目标 release tag
的 [`Cargo.toml`](https://github.com/svenshi/oxidns/blob/main/Cargo.toml) 检查这份列表。

## 快速开始

1. 在 GitHub 点 **Use this template** → **Create a new repository**
2. 编辑 [`build.config.yml`](build.config.yml):
   - `bundle`: 选 `full` / `standard` / `minimal` / `custom`
   - `features`: 仅 `bundle: custom` 时生效,填 Cargo feature 列表
   - `targets`: 注释掉不需要的平台
3. push 到 main。在 Actions 页手动 Run 一次 **Watch Upstream** 触发首次构建。

之后每 30 分钟轮询一次上游 latest release,有新版本就自动编译并发布到你的仓库。

常规部署优先使用 `minimal`、`standard` 或 `full` 预设。只有确实需要裁剪或追加
能力时才使用 `custom`;修改 custom features 后,应同时检查运行时 `config.yaml`
没有引用未编译的协议或插件。构建完成后可运行 `oxidns build-info` 核对实际能力。

协议 feature 按用途分组:`server-*` 控制入站 DNS 服务,`upstream-*` 控制
`forward` 等 DNS upstream,`resolver-*` 控制
`network.outbound.resolver.nameservers`。三组名称相近但不能互相替代。

完整的 bundle 能力矩阵、feature 清单和本地构建方式见
[OxiDNS 自定义编译文档](https://oxidns.org/custom-build)。

## 在客户端使用自定义编译

```bash
oxidns upgrade apply \
  --repository your-name/oxidns-build \
  --bundle full
```

**为什么 custom bundle 也要传 `--bundle full`?** 客户端本地二进制的 `PRIMARY_BUNDLE`
为 `custom`,而 `--bundle auto` 在 custom 上会拒绝执行(防止猜错 asset 名)。
custom 构建产物文件名格式是 `oxidns-{target}.{ext}`,刚好与 `--bundle full` 匹配。

如果想跳过 bundle 推断,直接指定 asset:

```bash
oxidns upgrade apply \
  --repository your-name/oxidns-build \
  --asset oxidns-x86_64-unknown-linux-musl.tar.gz
```

也可以把这套参数写进 `config.yaml`,让 `upgrade` 执行器插件自动使用:

```yaml
plugins:
  - tag: my_upgrader
    type: upgrade
    args:
      repository: your-name/oxidns-build
      bundle: full
```

## 产物命名 (必须与上游对齐,否则升级会失败)

| bundle | 文件名 | 压缩包内容 | 用的 config |
|---|---|---|---|
| `full` / `custom` | `oxidns-{target}.tar.gz` / `.zip` | `oxidns`/`oxidns.exe`、`config.yaml`、`LICENSE`、`webui/` | `config.yaml` |
| `standard` | `oxidns-standard-{target}.tar.gz` | 同上 | `config.yaml` |
| `minimal` | `oxidns-minimal-{target}.tar.gz` | `oxidns`、`config.yaml`、`LICENSE` | `config.minimal.yaml` |

所有文件都在 tarball **根目录**,与 `oxidns upgrade` 解包时的硬编码路径一致。

## 支持的 targets

与上游 `release.yml` 完全一致,共 13 个:

- `x86_64-unknown-linux-gnu` / `x86_64-unknown-linux-musl`
- `aarch64-unknown-linux-gnu` / `aarch64-unknown-linux-musl`
- `i686-unknown-linux-musl` / `arm-unknown-linux-musleabihf` / `armv7-unknown-linux-musleabihf`
- `x86_64-apple-darwin` / `aarch64-apple-darwin`
- `x86_64-unknown-freebsd`
- `x86_64-pc-windows-msvc` / `i686-pc-windows-msvc` / `aarch64-pc-windows-msvc`

## FAQ

**Q: 上游改了编译流程,我要做什么?**
A: 通常什么都不用做。模版里的 `build.yml` 默认 pin 到 `svenshi/oxidns@main`,
下次触发就自动用最新逻辑。要锁版本就把 `@main` 改成 `@vX.Y.Z` 形式的实际 tag。
但如果使用 `bundle: custom`,仍需检查自己的 feature 列表是否兼容新的 release tag。

**Q: GitHub Actions 调度延迟?**
A: cron 不保证准点触发,实际延迟可能到 10 分钟以上。需要实时性就改用上游的
`repository_dispatch` 主动推送。

**Q: 怎么验证产物的完整性?**
A: GitHub 给每个 release asset 自动生成 SHA256 digest,`oxidns upgrade` 会自动
校验,无需手动签名。
