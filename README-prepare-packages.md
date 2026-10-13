# `gha-npmjs/prepare-packages`

A composite action for the last step of your build job. It resolves which packages to publish, packs them into
tarballs in dependency order and uploads them as a GitHub artifact for [`publish.yml`](README-publish.md).

You own the rest of the build job: checkout, toolchain, install and build. Keep it at `contents: read` and without
`id-token`. Install scripts, your build and the `prepack`/`prepare` lifecycle scripts all run here, so none of them
can ever get an npm token.

## Usage

Full `cd.yml`: [README setup](README.md#setup).

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

For pnpm, yarn, bun or a monorepo, see [examples](docs/examples.md). For what the action does step by step, see
[How it works](docs/how-it-works.md#what-prepare-packages-does).

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
