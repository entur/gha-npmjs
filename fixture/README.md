# Fixtures

Test packages the reusable workflows run against in [`.github/workflows/ci.yml`](../.github/workflows/ci.yml), always
with `dry_run: true` so nothing reaches npmjs. Every fixture pins its toolchain in `mise.toml` and commits its
lockfile, so the workflow's frozen-lockfile install is exercised for real.

| Fixture | Package manager | Covers |
| --- | --- | --- |
| [`single-package`](single-package) | npm | `npm ci`, the default single-package path, restricted access and dist-tag overrides |
| [`pnpm-package`](pnpm-package) | pnpm | `pnpm install --frozen-lockfile` |
| [`yarn-package`](yarn-package) | yarn 4 (`node-modules` linker) | `yarn install --immutable` |
| [`bun-package`](bun-package) | bun | `bun install --frozen-lockfile` |
| [`monorepo`](monorepo) | npm workspaces + lerna | release-please manifest mode, publishing several packages from one release |

The publish step always runs through the npm CLI, since trusted publishing (OIDC) is an npm CLI feature — the package
manager only installs and builds.

## Minimum release age

Every fixture refuses package versions published less than **5 days** ago, so a compromised release has time to be
caught and pulled before any install picks it up. The setting is per package manager, and the unit differs:

| Package manager | File | Setting |
| --- | --- | --- |
| npm (>= 12) | [`.npmrc`](single-package/.npmrc) | `min-release-age=5` (days) |
| pnpm | [`pnpm-workspace.yaml`](pnpm-package/pnpm-workspace.yaml) | `minimumReleaseAge: 7200` (minutes) |
| yarn (>= 4.12) | [`.yarnrc.yml`](yarn-package/.yarnrc.yml) | `npmMinimalAgeGate: "5d"` |
| bun | [`bunfig.toml`](bun-package/bunfig.toml) | `minimumReleaseAge = 432000` (seconds) |

Internally published packages do not need the delay, so pnpm exempts them with `minimumReleaseAgeExclude` and yarn
with `npmPreapprovedPackages`. npm's equivalent is `min-release-age-exclude[]=@entur/*` and bun's is
`minimumReleaseAgeExcludes = ["pkg"]` (exact names, no globs); neither fixture needs one.

The npm fixtures pin `"npm:npm" = "12.0.2"` in `mise.toml`, since `min-release-age` is silently ignored by npm 11.
