# `gha-npmjs`

Publish your npm package to [npmjs.com](https://www.npmjs.com) when you merge a release pull request. No npm token
needed.

- **You** build the package.
- **gha-npmjs** packs it and publishes it with [trusted publishing](https://docs.npmjs.com/trusted-publishers).
- **[release-please](https://github.com/googleapis/release-please)** (through [`entur/gha-meta`](https://github.com/entur/gha-meta)) picks the version.

Want the details? Read [How it works](docs/how-it-works.md).

## Setup

### 1. Add `.github/workflows/cd.yml`

Copy this. Change the `npm ci` and `npm run build` lines if you build differently.

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
      release_type: node

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
          ref: ${{ needs.release.outputs.tag_name }}
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
      id-token: write
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
```

> [!IMPORTANT]
> Only the `publish` job gets `id-token: write`. Never add it to the `build` job.

### 2. Add the trusted publisher on npmjs

On npmjs.com: your package → **Settings** → **Trusted publisher** → **GitHub Actions**.

| Field | Value |
| --- | --- |
| Organization or user | `entur` |
| Repository | your repository name |
| Workflow filename | `cd.yml` |
| Environment | leave empty |

> [!NOTE]
> A brand new package must be published once by hand first.

### 3. Release

1. Merge a pull request to `main`.
2. release-please opens a release pull request. Merge it.
3. The new version is on npmjs a few minutes later.

## Using something else?

| You use | See |
| --- | --- |
| pnpm, yarn or bun | [Examples: pnpm, yarn or bun](docs/examples.md#pnpm-yarn-or-bun) |
| A monorepo | [Examples: monorepo](docs/examples.md#monorepo) |
| A package in a subfolder | [Examples: package in a subfolder](docs/examples.md#package-in-a-subfolder) |
| Beta / prerelease versions | [Examples: prerelease](docs/examples.md#prerelease-versions) |
| Checks on pull requests | [Examples: test on pull requests](docs/examples.md#test-on-pull-requests) |

All examples: [docs/examples.md](docs/examples.md).

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Run fails with `startup_failure` and no jobs start | Copy the `permissions` from [step 1](#1-add-githubworkflowscdyml) exactly. |
| `npm publish` fails with an authentication error | The workflow filename on npmjs must match your file, for example `cd.yml`. |
| Build checks out the default branch instead of the tag | In a monorepo, check out `github.sha`. See [Monorepo](docs/examples.md#monorepo). |
| `release_type is 'manifest' but .release-please-manifest.json was not found` | Put the manifest at `path`, set `manifest_file`, or set `packages` on `prepare-packages`. |
| `depend on each other in a cycle` | Remove the dependency cycle between your packages. |
| `was packed with unresolved workspace: ranges` | Set `package_manager` to the one that installed your workspace. |
| `points at files that are not in the tarball` | Build before `prepare-packages`, and add the build output to `files` in `package.json`. |
| Provenance error | Your repository is not public. Remove `provenance: true`. |

## Reference

- [`prepare-packages` action](README-prepare-packages.md): inputs and outputs
- [`publish.yml` workflow](README-publish.md): inputs
- [`gha-meta`](https://github.com/entur/gha-meta): release inputs and outputs
- [`entur/npmjs-packages-release-tests`](https://github.com/entur/npmjs-packages-release-tests): real repositories that use this
