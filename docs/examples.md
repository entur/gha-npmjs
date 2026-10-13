# Examples

Each example only shows what changes in `cd.yml`. Keep the rest from the [README setup](../README.md#setup).

- [pnpm, yarn or bun](#pnpm-yarn-or-bun)
- [Monorepo](#monorepo)
- [Package in a subfolder](#package-in-a-subfolder)
- [Prerelease versions](#prerelease-versions)
- [Provenance](#provenance)
- [Test on pull requests](#test-on-pull-requests)
- [Test the tarballs before publishing](#test-the-tarballs-before-publishing)

Full working repositories: [`entur/npmjs-packages-release-tests`](https://github.com/entur/npmjs-packages-release-tests).

## pnpm, yarn or bun

Install and build with your package manager. Set `package_manager` to the same one. It is required.

```yml
      - run: pnpm install --frozen-lockfile
      - run: pnpm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: pnpm
```

With mise, pin the package manager next to node in `mise.toml`:

```toml
[tools]
node = "24.21.0"
pnpm = "12.4.2"
```

| `package_manager` | Example |
| --- | --- |
| `npm` | [`fixture/npm-package`](../fixture/npm-package) |
| `pnpm` | [`fixture/pnpm-package`](../fixture/pnpm-package) |
| `yarn` | [`fixture/yarn-package`](../fixture/yarn-package) |
| `bun` | [`fixture/bun-package`](../fixture/bun-package) |

## Monorepo

Each package gets its own version, tag and changelog. This uses release-please
[manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md).

**1. Change `cd.yml`:**

```yml
  release:
    with:
      release_type: manifest

  build:
    steps:
      # A monorepo has no single tag, so build the commit that triggered the release
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

**2. Add `release-please-config.json`** at the repository root:

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

**3. Add `.release-please-manifest.json`** at the repository root:

```json
{
  "packages/common": "1.0.0",
  "packages/util": "1.0.0"
}
```

**What gets published:**

- Every package in the manifest whose version is not on npmjs yet.
- Packages already on npmjs are skipped. Set `skip_published: false` on `publish` to fail instead.
- Packages with `"private": true` are always skipped.

**Only some packages?** Set `packages` on `prepare-packages`:

```yml
        with:
          package_manager: npm
          packages: |
            packages/util
            packages/common
```

Examples: [`monorepo-npm`](../fixture/monorepo-npm), [`monorepo-pnpm`](../fixture/monorepo-pnpm),
[`monorepo-yarn`](../fixture/monorepo-yarn), [`monorepo-bun`](../fixture/monorepo-bun).

## Package in a subfolder

release-please only looks for `package.json` at the repository root. For a package in a subfolder, use manifest mode
with one package.

**1. Change `cd.yml`:**

```yml
  release:
    with:
      release_type: manifest

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
          release_type: manifest
```

Keep the checkout `ref` from the README. With one package, `tag_name` is still set.

**2. Add `release-please-config.json`** at the repository root:

```json
{
  "packages": {
    "packages/amazing-lib": { "release-type": "node" }
  }
}
```

**3. Add `.release-please-manifest.json`** at the repository root:

```json
{
  "packages/amazing-lib": "1.0.0"
}
```

## Prerelease versions

npm refuses to publish a version like `2.0.0-beta.1` as `latest`. Set `dist_tag`:

```yml
  publish:
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
      dist_tag: next
```

## Provenance

Off by default. It only works from a public repository.

```yml
  publish:
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
      provenance: true
```

## Test on pull requests

Build and pack on every pull request. A broken build fails before release. Nothing is published.

Create `.github/workflows/ci.yml`. It is the `build` job from `cd.yml` without `needs`, `if` and the checkout `ref`:

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

## Test the tarballs before publishing

Add a job between `build` and `publish`. It tests exactly what gets published:

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
