# `gha-npmjs/release`

Release your npm package with [release-please](https://github.com/googleapis/release-please) and publish it to
[npmjs.com](https://www.npmjs.com) with [trusted publishing](https://docs.npmjs.com/trusted-publishers). No npm token
needed.

## Contents

- [How it works](#how-it-works)
- [Setup](#setup)
  - [Step 1: Pin your toolchain with mise](#step-1-pin-your-toolchain-with-mise)
  - [Step 2: Add the release workflow](#step-2-add-the-release-workflow)
  - [Step 3: Configure the trusted publisher on npmjs](#step-3-configure-the-trusted-publisher-on-npmjs)
  - [Step 4: Test it from a pull request (optional)](#step-4-test-it-from-a-pull-request-optional)
  - [Step 5: Release](#step-5-release)
- [Examples](#examples)
  - [Package in a subdirectory](#package-in-a-subdirectory)
  - [pnpm, yarn or bun](#pnpm-yarn-or-bun)
  - [Monorepo](#monorepo)
  - [Prerelease dist-tag](#prerelease-dist-tag)
  - [Provenance](#provenance)
  - [Custom install or build command](#custom-install-or-build-command)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Good to know](#good-to-know)
- [Troubleshooting](#troubleshooting)

## How it works

1. You push to `main`. release-please opens or updates a release pull request, with the version bump taken from your
   [conventional commits](https://www.conventionalcommits.org).
2. You merge the release pull request. release-please creates the tag and the GitHub release.
3. The workflow checks out the tag, installs and builds with your package manager, and publishes to npmjs.

## Setup

### Step 1: Pin your toolchain with mise

The workflow installs node and your package manager with [mise](https://mise.jdx.dev). Add a `mise.toml` to your
repository:

```toml
[tools]
node = "24.21.0"
pnpm = "12.4.2" # leave out if you use npm
```

Commit your lockfile too. Dependencies are installed with a frozen lockfile.

> [!IMPORTANT]
> Dependabot does not update `mise.toml`. Bump the node and package manager versions by hand.

### Step 2: Add the release workflow

Create `.github/workflows/cd.yml`:

```yml
name: CD

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write # trusted publishing (OIDC)

jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
```

All four permissions are required.

### Step 3: Configure the trusted publisher on npmjs

On npmjs.com, open your package → **Settings** → **Trusted publisher** → **GitHub Actions**:

| Field | Value |
| --- | --- |
| Organization or user | `entur` |
| Repository | your repository name |
| Workflow filename | `cd.yml` (the file from step 2) |
| Environment | leave empty |

> [!NOTE]
> A brand new package must be published once by hand before you can configure a trusted publisher.

### Step 4: Test it from a pull request (optional)

A dry run installs, builds and runs `npm publish --dry-run`. Nothing is released or published. Create
`.github/workflows/ci.yml`:

```yml
name: CI

on:
  pull_request:

permissions: # same as cd.yml, or the run fails to start
  contents: write
  pull-requests: write
  issues: write
  id-token: write

jobs:
  publish-dry-run:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      dry_run: true
```

### Step 5: Release

Merge to `main`, then merge the release pull request that release-please opens. The new version is on npmjs a few
minutes later.

## Examples

Each example only shows the `jobs` part of `cd.yml`. Keep the `on` and `permissions` from
[step 2](#step-2-add-the-release-workflow).

### Package in a subdirectory

```yml
jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      path: packages/amazing-lib
```

### pnpm, yarn or bun

```yml
jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      package_manager: pnpm
```

| `package_manager` | Install command | Example |
| --- | --- | --- |
| `npm` (default) | `npm ci` | [`fixture/npm-package`](fixture/npm-package) |
| `pnpm` | `pnpm install --frozen-lockfile` | [`fixture/pnpm-package`](fixture/pnpm-package) |
| `yarn` | `yarn install --immutable` | [`fixture/yarn-package`](fixture/yarn-package) |
| `bun` | `bun install --frozen-lockfile` | [`fixture/bun-package`](fixture/bun-package) |

Pin the same package manager in `mise.toml`.

### Monorepo

Use release-please [manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md).
Each package gets its own version, tag and changelog.

```yml
jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      release_type: manifest
      package_manager: yarn
```

Add these two files at the repository root:

```sh
.
├── mise.toml
├── package.json
├── release-please-config.json
├── .release-please-manifest.json
└── packages
    ├── common
    │   └── package.json
    └── util
        └── package.json
```

`release-please-config.json`:

```json
{
  "release-type": "node",
  "tag-separator": "@",
  "include-v-in-tag": false,
  "packages": {
    "packages/common": { "package-name": "@entur/common", "component": "@entur/common" },
    "packages/util": { "package-name": "@entur/util", "component": "@entur/util" }
  },
  "plugins": ["node-workspace"]
}
```

`.release-please-manifest.json`:

```json
{
  "packages/common": "1.0.0",
  "packages/util": "1.0.0"
}
```

What gets published:

- Every package in `.release-please-manifest.json` whose version is not on npmjs yet.
- Packages already on npmjs are skipped. Set `skip_published: false` to fail instead.
- Packages with `"private": true` are always skipped.

To publish a fixed list instead of the whole manifest, add `packages`:

```yml
    with:
      release_type: manifest
      packages: |
        packages/util
        packages/common
```

To release one package in a subdirectory with manifest mode, set `path` to the package and `packages: "."`:

```yml
    with:
      path: packages/amazing-lib
      release_type: manifest
      packages: "."
```

Working examples: [`fixture/monorepo-npm`](fixture/monorepo-npm), [`fixture/monorepo-pnpm`](fixture/monorepo-pnpm),
[`fixture/monorepo-yarn`](fixture/monorepo-yarn) and [`fixture/monorepo-bun`](fixture/monorepo-bun).

### Prerelease dist-tag

```yml
    with:
      dist_tag: next
```

### Provenance

Provenance is off by default, because it only works from a public repository. To turn it on:

```yml
    with:
      provenance: true
```

### Custom install or build command

```yml
    with:
      install_command: npm ci --ignore-scripts
      build_command: npm run build:prod
```

Without `build_command`, the `build` script in `package.json` runs if it exists. Otherwise the build is skipped.

## Inputs

<!-- AUTO-DOC-INPUT:START - Do not remove or modify this section -->

Generated by `tj-actions/auto-doc` in the CD workflow.

<!-- AUTO-DOC-INPUT:END -->

## Outputs

<!-- AUTO-DOC-OUTPUT:START - Do not remove or modify this section -->

Generated by `tj-actions/auto-doc` in the CD workflow.

<!-- AUTO-DOC-OUTPUT:END -->

## Good to know

- **Publishing always uses the npm CLI.** Trusted publishing is an npm CLI feature. After install and build, the
  workflow installs npm `npm_version` (default `12.1.0`) for the publish step.
- **`workspace:` ranges are replaced before publishing.** With pnpm, yarn or bun, each package is packed by your
  package manager, which turns `workspace:*` into the real version. The workflow fails if a `workspace:` range is left.
- **`prepublishOnly` does not run with pnpm, yarn or bun.** npm skips it when publishing a tarball. Move build or test
  steps to `build` or `prepack`.
- **Dependencies are published first.** In a monorepo, a package is never published before a package it depends on.
- **mise config can be in a parent directory.** Any file name mise supports works. Use `mise_working_directory` if it
  lives somewhere else.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Run fails with `startup_failure` and no jobs start | Add all four permissions from [step 2](#step-2-add-the-release-workflow) to the calling workflow, also for dry runs. |
| `node is not pinned in a mise configuration` | Add `node` to `mise.toml`, see [step 1](#step-1-pin-your-toolchain-with-mise). |
| `npm publish` fails with an authentication error | Check the trusted publisher on npmjs. The workflow filename must match the file that calls this workflow. |
| `release_type is 'manifest' but .release-please-manifest.json was not found` | Put the manifest at `path`, set `manifest_file`, or list packages with `packages`. |
| `release-please reported a release but no tag_name` | With manifest mode, `path` is not one of the released packages. See [Monorepo](#monorepo). |
| `depend on each other in a cycle` | Remove the dependency cycle between the released packages. |
| `was packed with unresolved workspace: ranges` | Set `package_manager` to the package manager that owns the workspace. |
| Provenance error when publishing | Your repository is not public. Remove `provenance: true`. |
