# Monorepo fixture

Mirrors the layout of [`entur/entur-partner-packages`](https://github.com/entur/entur-partner-packages):
workspaces under `packages/`, `lerna.json` for linking and running tasks, and release-please in
[manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md) with the
`node-workspace` plugin, so each package gets its own version, tag and changelog.

`.github/workflows/ci.yml` runs the `prepare-packages` action and `publish.yml` against this fixture with
`dry_run: true`. `prepare-packages` reads the packages from `.release-please-manifest.json`, and `publish.yml` skips
the ones already on npmjs, so a release publishes exactly the packages release-please bumped.

The root `build` script runs `npm run build --workspaces` instead of `lerna run build` to keep the fixture's
lockfile small — real consumers keep `lerna` as a devDependency and the workflow runs whatever the root
`build` script is.
