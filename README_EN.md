# OxiDNS Custom Build Template

[中文](README.md) | English

A **GitHub Template Repository** — click "Use this template" to spawn your
own copy, customize [OxiDNS](https://github.com/svenshi/oxidns) features and
target platforms, and let it auto-rebuild whenever upstream publishes a new
release. The matching binaries get published to your own repo's releases.

## How it works

```
svenshi/oxidns                    your-name/oxidns-build (from template)
─────────────                     ──────────────────────────────────────
 release v1.2.0  ─poll every 30m─▶ watch-upstream.yml
                                            │
                                            ▼  reads build.config.yml
                                    build.yml (thin shell)
                                            │
                                            ▼  uses: svenshi/oxidns/...@main
                          ┌────────────────────────────────────┐
                          │ svenshi/oxidns/.github/workflows/  │
                          │      custom-build.yml              │  ← build matrix
                          │  (runs on the caller's runner)     │
                          └─────────────────┬──────────────────┘
                                            ▼
                                  publishes to your-name/oxidns-build releases
                                            ▼
                       oxidns upgrade --repository your-name/oxidns-build
```

**Key point**: the actual build matrix, naming, and packaging logic all
live upstream in
[`.github/workflows/custom-build.yml`](https://github.com/svenshi/oxidns/blob/main/.github/workflows/custom-build.yml).
Derivative repos only carry a ~50-line shell — they don't need to clone
the matrix. When upstream improves the build pipeline, every derivative
repo **automatically picks it up** without local updates — that's the
whole point of calling this a "reusable workflow".

## Quick start

1. On GitHub click **Use this template** → **Create a new repository**
2. Edit [`build.config.yml`](build.config.yml):
   - `bundle`: pick `full` / `standard` / `minimal` / `custom`
   - `features`: only used when `bundle: custom`; list the Cargo feature names
   - `targets`: comment out the platforms you don't need
3. Push to `main`. Run **Watch Upstream** once manually from the Actions tab
   to trigger the first build.

The watcher polls upstream's latest release every 30 minutes and rebuilds
+ publishes to your repo whenever a new version appears.

## Using a custom build on the client

```bash
oxidns upgrade apply \
  --repository your-name/oxidns-build \
  --bundle full
```

**Why does the custom bundle still need `--bundle full`?** The client
binary's `PRIMARY_BUNDLE` reports as `custom`, and `--bundle auto` refuses
to run on custom builds (to avoid guessing the wrong asset name). Custom
builds use the `oxidns-{target}.{ext}` filename, which matches
`--bundle full` exactly.

You can bypass bundle inference by specifying the asset directly:

```bash
oxidns upgrade apply \
  --repository your-name/oxidns-build \
  --asset oxidns-x86_64-unknown-linux-musl.tar.gz
```

You can also wire it into `config.yaml` so the `upgrade` executor plugin
picks it up automatically:

```yaml
plugins:
  - tag: my_upgrader
    type: upgrade
    args:
      repository: your-name/oxidns-build
      bundle: full
```

## Asset naming (must match upstream, otherwise upgrade fails)

| bundle | filename | archive contents | config used |
|---|---|---|---|
| `full` / `custom` | `oxidns-{target}.tar.gz` / `.zip` | `oxidns`/`oxidns.exe`, `config.yaml`, `LICENSE`, `webui/` | `config.yaml` |
| `standard` | `oxidns-standard-{target}.tar.gz` | same as above | `config.yaml` |
| `minimal` | `oxidns-minimal-{target}.tar.gz` | `oxidns`, `config.yaml`, `LICENSE` | `config.minimal.yaml` |

Every file sits at the tarball **root** — same convention as the hardcoded
extraction paths in `oxidns upgrade`.

## Supported targets

Identical to upstream `release.yml`:

- `x86_64-unknown-linux-gnu` / `x86_64-unknown-linux-musl`
- `aarch64-unknown-linux-gnu` / `aarch64-unknown-linux-musl`
- `i686-unknown-linux-musl` / `arm-unknown-linux-musleabihf`
- `x86_64-apple-darwin` / `aarch64-apple-darwin`
- `x86_64-unknown-freebsd`
- `x86_64-pc-windows-msvc` / `i686-pc-windows-msvc` / `aarch64-pc-windows-msvc`

## FAQ

**Q: Upstream changed the build pipeline — what do I do?**
A: Usually nothing. The template's `build.yml` pins to
`svenshi/oxidns@main`, so the next trigger picks up the latest logic
automatically. Pin to `@v1.2.0` (or any tag) if you want a fixed version.

**Q: How long is the cron delay?**
A: GitHub cron does not guarantee on-time firing — delays of 10+ minutes
are normal. For real-time triggering, switch to upstream-pushed
`repository_dispatch`.

**Q: How is artifact integrity verified?**
A: GitHub auto-generates a SHA256 digest for each release asset, and
`oxidns upgrade` validates it on download. No manual signing needed.
