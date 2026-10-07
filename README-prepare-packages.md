# `gha-npmjs/prepare-packages`

A composite action for the last step of your build job. It resolves which packages to publish, packs them into
tarballs in dependency order and uploads them as a GitHub artifact for [`publish.yml`](README-publish.md).

You own the rest of the build job: checkout, toolchain, install and build. Keep it at `contents: read` and without
`id-token`. Install scripts, your build and the `prepack`/`prepare` lifecycle scripts all run here, so none of them
can ever get an npm token.

## Usage

See the [setup guide](README.md#setup) for the full `cd.yml`.

```yml
jobs:
  build:
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
```

### pnpm

Pin pnpm next to node in `mise.toml`, so `mise-action` installs both:

```toml
[tools]
node = "24.21.0"
pnpm = "12.4.2"
```

Then install, build and prepare with pnpm. `pnpm pack` replaces `workspace:` ranges with real versions:

```yml
      - run: pnpm install --frozen-lockfile
      - run: pnpm run build
      - id: prepare-packages
        uses: entur/gha-npmjs/.github/actions/prepare-packages@v1
        with:
          package_manager: pnpm
```

A pnpm monorepo in release-please manifest mode adds `release_type: manifest`. See
[`fixture/pnpm-package`](fixture/pnpm-package) and [`fixture/monorepo-pnpm`](fixture/monorepo-pnpm).

## What it does

1. Resolves the package directories: `packages` if set, otherwise every package in `manifest_file` when
   `release_type: manifest`, otherwise `path`.
2. Orders them so dependencies come first. Fails on a dependency cycle.
3. Packs each package with `package_manager`. pnpm, yarn and bun replace `workspace:` ranges with real versions. Fails
   if a `workspace:` range is left, or if a file that `main`, `module`, `types`, `typings`, `bin` or `exports` points
   at is not in the tarball.
4. Uploads the tarballs and `publish-order.tsv` as an artifact, kept for 1 day. Packages with `"private": true` are
   listed but not packed.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `path` | `.` | Path to the package in the repository (where `package.json` lives) |
| `release_type` | `node` | release-please release type. With `manifest`, the packages are read from `manifest_file` |
| `manifest_file` | `.release-please-manifest.json` | release-please versions manifest, relative to `path` |
| `packages` | | Newline-separated package directories, relative to `path`. Overrides the manifest |
| `package_manager` | required | `npm`, `pnpm`, `yarn` or `bun`. Must be the one that installed the workspace |
| `artifact_name` | generated | Name of the uploaded artifact. Must be unique within the run |

## Outputs

| Output | Description |
| --- | --- |
| `artifact_name` | Name of the uploaded artifact. Pass it to `publish.yml` |
