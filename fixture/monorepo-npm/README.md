# Monorepo fixture

Mirrors the layout of [`entur/entur-partner-packages`](https://github.com/entur/entur-partner-packages):
workspaces under `packages/`, `lerna.json` for linking and running tasks, and release-please in
[manifest mode](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md) with the
`node-workspace` plugin, so each package gets its own version, tag and changelog.

`gha-npmjs/release` is exercised against this fixture in `.github/workflows/ci.yml` with `dry_run: true`. The
publish step discovers the packages from `.release-please-manifest.json`, so a release publishes exactly the
packages release-please bumped and skips the ones already on npmjs.

The root `build` script runs `npm run build --workspaces` instead of `lerna run build` to keep the fixture's
lockfile small — real consumers keep `lerna` as a devDependency and the workflow runs whatever the root
`build` script is.
