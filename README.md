# OxiDNS 自定义编译模版

中文 | [English](README_EN.md)

这是一个 **GitHub Template Repository** —— 用 "Use this template" 生成你自己的副本,
就能定制 [OxiDNS](https://github.com/svenshi/oxidns) 的 features 和目标平台,
在上游每次发布 release 时自动重新编译并发布到你自己的仓库 release。

## 工作方式

```
svenshi/oxidns                    your-name/oxidns-build (从模版生成)
─────────────                     ──────────────────────────────────
 发布 v1.2.0  ─每 30 分钟轮询─▶   watch-upstream.yml
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
派生仓库永远只持有一个 50 行的薄壳,不需要自己复制粘贴这套矩阵。
上游改了构建流程,所有派生仓库**不需要更新**就自动跟上 —— 这也是把这套机制叫
"reusable workflow" 的原因。

## 快速开始

1. 在 GitHub 点 **Use this template** → **Create a new repository**
2. 编辑 [`build.config.yml`](build.config.yml):
   - `bundle`: 选 `full` / `standard` / `minimal` / `custom`
   - `features`: 仅 `bundle: custom` 时生效,填 Cargo feature 列表
   - `targets`: 注释掉不需要的平台
3. push 到 main。在 Actions 页手动 Run 一次 **Watch Upstream** 触发首次构建。

之后每 30 分钟轮询一次上游 latest release,有新版本就自动编译并发布到你的仓库。

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

与上游 `release.yml` 完全一致:

- `x86_64-unknown-linux-gnu` / `x86_64-unknown-linux-musl`
- `aarch64-unknown-linux-gnu` / `aarch64-unknown-linux-musl`
- `i686-unknown-linux-musl` / `arm-unknown-linux-musleabihf`
- `x86_64-apple-darwin` / `aarch64-apple-darwin`
- `x86_64-unknown-freebsd`
- `x86_64-pc-windows-msvc` / `i686-pc-windows-msvc` / `aarch64-pc-windows-msvc`

## FAQ

**Q: 上游改了编译流程,我要做什么?**
A: 通常什么都不用做。模版里的 `build.yml` 默认 pin 到 `svenshi/oxidns@main`,
下次触发就自动用最新逻辑。要锁版本就把 `@main` 改成 `@v1.2.0` 之类的 tag。

**Q: GitHub Actions 调度延迟?**
A: cron 不保证准点触发,实际延迟可能到 10 分钟以上。需要实时性就改用上游的
`repository_dispatch` 主动推送。

**Q: 怎么验证产物的完整性?**
A: GitHub 给每个 release asset 自动生成 SHA256 digest,`oxidns upgrade` 会自动
校验,无需手动签名。
