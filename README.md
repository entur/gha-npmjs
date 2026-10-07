# `gha-npmjs`

Release your npm package with [release-please](https://github.com/googleapis/release-please) and publish it to
[npmjs.com](https://www.npmjs.com) with [trusted publishing](https://docs.npmjs.com/trusted-publishers). No npm token
needed.

You build your package the way you like. gha-npmjs packs and publishes it. Use it together with
[`entur/gha-meta`'s `release.yml`](https://github.com/entur/gha-meta), which runs release-please:

| Piece | Does | Permissions |
| --- | --- | --- |
| [`gha-meta/release.yml`](https://github.com/entur/gha-meta) | Runs release-please: opens the release pull request, creates the tag and GitHub release | `contents`/`pull-requests`/`issues`: write |
| Your build job + [`prepare-packages` action](README-prepare-packages.md) | You check out, install and build. `prepare-packages` resolves the packages, packs them in dependency order and uploads them | `contents: read` |
| [`publish.yml`](README-publish.md) | Uploads the tarballs to npmjs. No checkout, runs nothing from your repository | `id-token: write` |

## Contents

- [How it works](#how-it-works)
- [Setup](#setup)
  - [Step 1: Add the release workflow](#step-1-add-the-release-workflow)
  - [Step 2: Configure the trusted publisher on npmjs](#step-2-configure-the-trusted-publisher-on-npmjs)
  - [Step 3: Test it from a pull request (optional)](#step-3-test-it-from-a-pull-request-optional)
  - [Step 4: Release](#step-4-release)
- [Examples](#examples)
  - [Package in a subdirectory](#package-in-a-subdirectory)
  - [pnpm, yarn or bun](#pnpm-yarn-or-bun)
  - [Monorepo](#monorepo)
  - [Prerelease dist-tag](#prerelease-dist-tag)
  - [Provenance](#provenance)
  - [Test the tarballs before publishing](#test-the-tarballs-before-publishing)
- [Good to know](#good-to-know)
- [Troubleshooting](#troubleshooting)

## How it works

1. You push to `main`. release-please opens or updates a release pull request, with the version bump taken from your
   [conventional commits](https://www.conventionalcommits.org).
2. You merge the release pull request. release-please creates the tag and the GitHub release.
3. Your build job checks out the tag, installs and builds. Its last step, the `prepare-packages` action, packs the packages.
4. `publish.yml` uploads the tarballs to npmjs.

Build and publish only run when a release was created. Pull requests can build and pack, see
[step 3](#step-3-test-it-from-a-pull-request-optional).

## Setup

### Step 1: Add the release workflow

Create `.github/workflows/cd.yml`. The build job is yours, this one uses [mise](https://mise.jdx.dev) and npm:

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

jobs:
  release:
    uses: entur/gha-meta/.github/workflows/release.yml@v1
    with:
      release_type: node # bumps the version in package.json. gha-meta's default is simple

  build:
    needs: release
    if: ${{ needs.release.outputs.releases_created == 'true' }}
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    outputs:
      artifact_name: ${{ steps.prepare-packages.outputs.artifact_name }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ needs.release.outputs.tag_name }} # build the release tag, not the branch head
      - uses: jdx/mise-action@7a4e45a543138629540c9a1616d08632b893e492 # v5.0.1
        with:
          cache: false
      - run: npm ci
      - run: npm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: npm

  publish:
    needs: build
    permissions:
      contents: read
      id-token: write # trusted publishing (OIDC)
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
```

The three top-level permissions are required by gha-meta's `release.yml`. Only the `publish` job asks for
`id-token: write`, so it is the only job that can get an id-token. See [gha-meta](https://github.com/entur/gha-meta) for the release
inputs and outputs.

### Step 2: Configure the trusted publisher on npmjs

On npmjs.com, open your package → **Settings** → **Trusted publisher** → **GitHub Actions**:

| Field | Value |
| --- | --- |
| Organization or user | `entur` |
| Repository | your repository name |
| Workflow filename | `cd.yml` (the file from step 1) |
| Environment | leave empty |

> [!NOTE]
> A brand new package must be published once by hand before you can configure a trusted publisher.

### Step 3: Test it from a pull request (optional)

Build and pack on every pull request, so a broken build or an unresolved `workspace:` range fails before release.
Nothing is released or published. Create `.github/workflows/ci.yml` with the same `build` steps as `cd.yml`,
without `needs`, `if` and the checkout `ref`:

```yml
name: CI

on:
  pull_request:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: jdx/mise-action@7a4e45a543138629540c9a1616d08632b893e492 # v5.0.1
        with:
          cache: false
      - run: npm ci
      - run: npm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: npm
```

### Step 4: Release

Merge to `main`, then merge the release pull request that release-please opens. The new version is on npmjs a few
minutes later.

## Examples

Each example only shows what changes in `cd.yml`. Keep the rest from [step 1](#step-1-add-the-release-workflow).

### Package in a subdirectory

Set `path` on `release` and on the `prepare-packages` step, and run your install and build there:

```yml
  release:
    with:
      release_type: node
      path: packages/amazing-lib

  build:
    defaults:
      run:
        working-directory: packages/amazing-lib
    steps:
      # ...checkout, toolchain, install, build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: npm
          path: packages/amazing-lib
```

### pnpm, yarn or bun

Install and build with your package manager, and tell `prepare-packages` which one it is. `package_manager` is
required, there is no default:

```yml
      - run: pnpm install --frozen-lockfile
      - run: pnpm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: pnpm
```

| `package_manager` | Example |
| --- | --- |
| `npm` | [`fixture/npm-package`](fixture/npm-package) |
| `pnpm` | [`fixture/pnpm-package`](fixture/pnpm-package) |
| `yarn` | [`fixture/yarn-package`](fixture/yarn-package) |
| `bun` | [`fixture/bun-package`](fixture/bun-package) |

Packing uses your package manager, so pnpm, yarn and bun replace `workspace:` ranges. Publishing always uses the npm
CLI.

### Monorepo

Use release-please [manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md).
Each package gets its own version, tag and changelog. Set `release_type: manifest` on `release` and on the `prepare-packages` step:

```yml
  release:
    with:
      release_type: manifest

  build:
    steps:
      # No single tag_name in manifest mode, so build the commit that triggered the release
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ github.sha }}
      # ...toolchain
      - run: yarn install --immutable
      - run: yarn workspaces foreach --all --topological run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          release_type: manifest
          package_manager: yarn
```

Add these two files at the repository root:

```sh
.
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
- Packages already on npmjs are skipped. Set `skip_published: false` on `publish` to fail instead.
- Packages with `"private": true` are always skipped.

To pack a fixed list instead of the whole manifest, set `packages` on the `prepare-packages` step:

```yml
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: npm
          packages: |
            packages/util
            packages/common
```

Working examples: [`fixture/monorepo-npm`](fixture/monorepo-npm), [`fixture/monorepo-pnpm`](fixture/monorepo-pnpm),
[`fixture/monorepo-yarn`](fixture/monorepo-yarn) and [`fixture/monorepo-bun`](fixture/monorepo-bun).

### Prerelease dist-tag

By default no `--tag` is passed, so npm publishes under `latest` and refuses to publish a prerelease version such as
`2.0.0-beta.1`. Set `dist_tag` to publish prereleases. `publish.yml` also fails if a prerelease would be published as
`latest`:

```yml
  publish:
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
      dist_tag: next
```

### Provenance

Provenance is off by default, because it only works from a public repository. To turn it on:

```yml
  publish:
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
      provenance: true
```

### Test the tarballs before publishing

Add a job between `build` and `publish`. It downloads the artifact, so it tests exactly what gets published:

```yml
  test-tarballs:
    needs: build
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - uses: actions/download-artifact@3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c # v8.0.1
        with:
          name: ${{ needs.build.outputs.artifact_name }}
          path: tarballs
      - run: ls tarballs # your tests here

  publish:
    needs: [build, test-tarballs]
```

## Good to know

- **Build and publish are separate jobs.** Your build job installs, builds and packs without an id-token.
  `publish.yml` only uploads those tarballs. It has no checkout and runs no scripts from your repository or
  dependencies, so a compromised dependency can never publish. Never add `id-token: write` to the build job.
- **Publishing always uses the npm CLI.** Trusted publishing is an npm CLI feature. The publish job uses the npm
  bundled with node 26.10.0 (npm 11.19.1).
- **`workspace:` ranges are replaced before publishing.** `prepare-packages` packs each package with your package manager, and
  pnpm, yarn and bun turn `workspace:*` into the real version. It fails if a `workspace:` range is left.
- **`prepublishOnly` does not run.** Packages are published as tarballs, and npm skips it then. Move build or test
  steps to your build job or `prepack`.
- **Dependencies are published first.** In a monorepo, a package is never published before a package it depends on.
- **Build the release tag.** Check out `needs.release.outputs.tag_name`. An empty `ref` makes `actions/checkout` build
  the default branch. In manifest mode there is no single tag, so check out `github.sha`, the release commit.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Run fails with `startup_failure` and no jobs start | Add the top-level permissions and the `publish` job's `id-token: write` from [step 1](#step-1-add-the-release-workflow). |
| `npm publish` fails with an authentication error | Check the trusted publisher on npmjs. The workflow filename must match the file that calls `publish.yml`. |
| Build checks out the default branch instead of the tag | Manifest mode has no single `tag_name`, so check out `github.sha` instead, see [Monorepo](#monorepo). |
| `release_type is 'manifest' but .release-please-manifest.json was not found` | Put the manifest at `path`, set `manifest_file`, or list packages with `packages` on the `prepare-packages` step. |
| `depend on each other in a cycle` | Remove the dependency cycle between the released packages. |
| `was packed with unresolved workspace: ranges` | Set `package_manager` on the `prepare-packages` step to the package manager that owns the workspace. |
| Provenance error when publishing | Your repository is not public. Remove `provenance: true`. |
