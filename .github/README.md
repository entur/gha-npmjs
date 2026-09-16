<h1 align="center">entur/gha-npmjs</h1>

[![Entur/Npmjs/CI](https://github.com/entur/gha-npmjs/actions/workflows/ci.yml/badge.svg)](https://github.com/entur/gha-npmjs/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/entur/gha-npmjs)](https://github.com/entur/gha-npmjs)

GitHub reusable workflows to help Entur teams release npm packages to [npmjs.com](https://www.npmjs.com):

- [Release and publish to npmjs](../README-release.md)

The workflow wraps [`entur/gha-meta/release.yml`](https://github.com/entur/gha-meta) (release-please) and publishes the
released version to npmjs using [trusted publishing](https://docs.npmjs.com/trusted-publishers) — OIDC, no npm token.

## Requirements

1. **mise** — the toolchain is resolved with [mise](https://mise.jdx.dev). Your repository must contain a mise
   configuration (`mise.toml`, `.mise.toml`, `.config/mise/config.toml` or `.tool-versions`) pinning node and your
   package manager. The workflow fails early if it finds none.

   ```toml
   # mise.toml
   [tools]
   node = "24.21.0"
   pnpm = "12.4.2"
   ```

2. **Trusted publisher on npmjs** — configure the package on npmjs.com with a trusted publisher pointing at
   `entur/<your-repo>` and the workflow file that calls this workflow (see [Setting up the trusted publisher](#setting-up-the-trusted-publisher)).

3. **Permissions in the calling workflow** — a called workflow can never hold more permissions than its caller, so the
   caller must grant `id-token: write` (OIDC) along with the permissions release-please needs.

4. **Conventional commits** — release-please derives the version bump from commit messages.

## Golden Path

### Example

Our repo is called `amazing-lib` and publishes a single package from the repository root.

```sh
λ amazing-lib ❯ tree
.
├── mise.toml
├── package.json
└── .github
    └── workflows
        ├── ci.yml
        └── cd.yml
```

Release and publish on every push to `main`:

```yaml
# cd.yml
name: CD

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write # Required for trusted publishing (OIDC)

jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
```

The first push opens a release pull request. Merging that pull request creates the release, the tag and the GitHub
release, and then publishes the new version to npmjs with a provenance attestation.

Verify the setup from a pull request without publishing anything:

```yaml
# ci.yml
name: CI

on:
  pull_request:

permissions:
  contents: read
  id-token: write

jobs:
  publish-dry-run:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      dry_run: true
```

### A single package in a subdirectory

```yaml
jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      path: packages/amazing-lib
      package_manager: pnpm
```

### Monorepo with workspaces and lerna

Run release-please in manifest mode and every package it bumps is published, the rest are skipped:

```yaml
jobs:
  release:
    uses: entur/gha-npmjs/.github/workflows/release.yml@v1
    with:
      release_type: manifest
      package_manager: yarn
```

See [Monorepos](../README-release.md#monorepos-release-please-manifest-mode) for the release-please configuration,
and [`fixture/monorepo`](../fixture/monorepo) for a working layout modelled on
[`entur/entur-partner-packages`](https://github.com/entur/entur-partner-packages).

Dependencies and the build run with your package manager; the publish itself always runs through the npm CLI, because
trusted publishing is an npm CLI feature. The workflow upgrades npm automatically when the toolchain ships a version
older than 11.5.1.

### Setting up the trusted publisher

On npmjs.com, open the package → **Settings** → **Trusted publisher** → GitHub Actions, and fill in:

| Field | Value |
| --- | --- |
| Organization or user | `entur` |
| Repository | your repository name |
| Workflow filename | the file that calls this workflow, e.g. `cd.yml` |
| Environment | leave empty unless your calling job uses one |

For a brand new package, the first version has to be published manually (or with a granular token) before the trusted
publisher can be configured.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
