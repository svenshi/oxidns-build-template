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
                                          ▼  uses: svenshi/oxidns/...@<同源码 ref>
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

模版在编译每个 ref 时,会优先用**同一个 ref 上的** `custom-build.yml`(例如
编译 `v1.2.0` 时用 `custom-build.yml@v1.2.0`,编译 `main` 分支时用
`custom-build.yml@main`)。这样工作流逻辑和源码版本严格绑定:
上游在 `main` 改了构建流程不会反过来污染历史 release 的构建结果,新功能也只
在切 tag 时随源码一起生效。当 ref 上不存在 `custom-build.yml`(例如非常老的
release)时,自动回退到 `@main`。

## 快速开始

1. 在 GitHub 点 **Use this template** → **Create a new repository**
2. 编辑 [`build.config.yml`](build.config.yml):
   - `bundle`: 选 `full` / `standard` / `minimal` / `custom`
   - `features`: 仅 `bundle: custom` 时生效,填 Cargo feature 列表
   - `targets`: 注释掉不需要的平台
3. push 到 main。在 Actions 页手动 Run 一次 **Watch Upstream** 触发首次构建。

之后每 30 分钟轮询一次上游 latest release,有新版本就自动编译并发布到你的仓库。

### 指定 branch / tag / commit 编译

上游还没发布过 release,或者需要测试某个分支 / PR / commit 时,直接手动触发
**Build OxiDNS Release**,在 `ref` 输入框里填:

| `ref` 取值 | 行为 |
|---|---|
| 留空(默认) | 取上游 latest release tag;若上游一个 release 都没有,回退到默认分支 |
| `v1.2.0`(语义化版本 tag) | 编译该 tag,发布为正式 release |
| `main` / `feature/foo`(分支名) | 编译该分支当前 HEAD,发布为 prerelease,tag 格式 `branch-<分支>-<sha7>` |
| `abc1234`(commit SHA) | 编译该 commit,发布为 prerelease,tag 格式同上 |

> 分支 / commit 构建发布的是 **prerelease**,既不会覆盖正式版,客户端也要显式
> 加 `--allow-prerelease` 才会被升级到。

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

升级到分支 / commit 构建时,加上 `--target` 指向具体的 prerelease tag:

```bash
oxidns upgrade apply \
  --repository your-name/oxidns-build \
  --target branch-main-abc1234 \
  --bundle full \
  --allow-prerelease
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
A: 通常什么都不用做。下次新 tag 发布时,新 tag 自带新的 `custom-build.yml`,
你的模版会自动用上去。如果想立刻在分支构建上试用,手动 `workflow_dispatch`
传 `ref: main` 即可,会用 `custom-build.yml@main`。

**Q: 工作流逻辑会被强制锁死在某个 ref 吗?**
A: 默认严格跟随源码 ref。如果想强制某个 ref 始终用 `@main` 的编译逻辑(例如
回填一个老 release 但希望用上新的 cross 工具链),编辑 `build.yml` 里
`uses:` 那一行,把 `@${{ needs.plan.outputs.workflow_ref }}` 改成 `@main`。

**Q: 能编译上游 PR 分支或某个 commit 吗?**
A: 直接手动 `workflow_dispatch` 给 build.yml 传 `ref: feature/foo` 或
`ref: abc1234`,会发布为 prerelease。详见上文"指定 branch / tag / commit 编译"。

**Q: GitHub Actions 调度延迟?**
A: cron 不保证准点触发,实际延迟可能到 10 分钟以上。需要实时性把改用上游的
`repository_dispatch` 主动推送。

**Q: 怎么验证产物的完整性?**
A: GitHub 给每个 release asset 自动生成 SHA256 digest,`oxidns upgrade` 会自动
校验,无需手动签名。
