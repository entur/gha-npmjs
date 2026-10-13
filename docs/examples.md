# Examples

- [npm, pnpm, yarn or bun](#npm-pnpm-yarn-or-bun)
- [Monorepo](#monorepo)
- [Package in a subfolder](#package-in-a-subfolder)
- [Prerelease versions](#prerelease-versions)
- [Provenance](#provenance)
- [Test on pull requests](#test-on-pull-requests)
- [Test the tarballs before publishing](#test-the-tarballs-before-publishing)

Full working repositories: [`entur/npmjs-packages-release-tests`](https://github.com/entur/npmjs-packages-release-tests).

## npm, pnpm, yarn or bun

Install and build with your package manager. Set `package_manager` to the same one.

```yml
  release:
    uses: entur/gha-meta/.github/workflows/release.yml@v1
    with:
      release_type: node

  build:
    steps:
      # ...checkout, toolchain
      - run: pnpm install --frozen-lockfile
      - run: pnpm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: pnpm # Supports npm, pnpm, yarn or bun.

  publish:
    needs: build
    permissions:
      contents: read
      id-token: write
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
```

| `package_manager` | Example |
| --- | --- |
| `npm` | [`release-npm.yml`](https://github.com/entur/npmjs-packages-release-tests/blob/main/.github/workflows/release-npm.yml) |
| `yarn` | [`release-yarn.yml`](https://github.com/entur/npmjs-packages-release-tests/blob/main/.github/workflows/release-yarn.yml) |
| `bun` | [`release-bun.yml`](https://github.com/entur/npmjs-packages-release-tests/blob/main/.github/workflows/release-bun.yml) |

## Monorepo

Each package gets its own version, tag and changelog. This uses release-please
[manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md).

**1. Change `cd.yml`:**

```yml
  release:
    uses: entur/gha-meta/.github/workflows/release.yml@v1
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
      - run: yarn run build # root script, for example: lerna run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          release_type: manifest
          package_manager: yarn

  publish:
    needs: build
    permissions:
      contents: read
      id-token: write
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
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

**Only some packages?** Set `packages` on `prepare-packages`. Only the listed packages are published, even if the
manifest has more:

```yml
  build:
    steps:
      # ...checkout, toolchain, install, build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          release_type: manifest
          package_manager: yarn
          # Specify which packages to publish
          packages: |
            packages/util
            packages/common
```

Example: [`release-lerna.yml`](https://github.com/entur/npmjs-packages-release-tests/blob/main/.github/workflows/release-lerna.yml) (yarn 4 workspaces and Lerna).

## Package in a subfolder

release-please only looks for `package.json` at the repository root. For a package in a subfolder, use manifest mode
with one package.

**1. Change `cd.yml`:**

```yml
  release:
    uses: entur/gha-meta/.github/workflows/release.yml@v1
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

  publish:
    needs: build
    permissions:
      contents: read
      id-token: write
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
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

Example: [`entur/uniformen`](https://github.com/entur/uniformen/tree/main/packages/uniformen) publishes
`packages/uniformen` with bun and GHA-NpmJS. See its [`release.yml`](https://github.com/entur/uniformen/blob/main/.github/workflows/release.yml).

## Provenance

Off by default. It only works from a public repository.

```yml
  publish:
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
      provenance: true
```

## Test on pull requests

Pack your packages on every pull request, so packing problems show up early. Without this, they only show up after
you merge the release pull request. Nothing is published.

It catches:

- a build that fails
- `workspace:` ranges that were not replaced
- entry points (`main`, `module`, `types`, `bin`, `exports`) that point at files missing from the tarball
- dependency cycles between your packages

Create `.github/workflows/ci.yml`. The steps are the same as the `build` job in `cd.yml`:

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
