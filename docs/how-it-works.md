# How it works

Read this if you want to understand what happens. For setup, see the [README](../README.md).

## The flow

1. You push to `main`. release-please opens or updates a release pull request. The version bump comes from your
   [conventional commits](https://www.conventionalcommits.org).
2. You merge the release pull request. release-please creates the tag and the GitHub release.
3. Your `build` job checks out the tag, installs and builds. Its last step, `prepare-packages`, packs the packages.
4. `publish.yml` uploads the tarballs to npmjs.

`build` and `publish` only run when a release was created. To catch errors earlier, also run the build on pull
requests. See [Test on pull requests](examples.md#test-on-pull-requests).

## The three pieces

| Piece | What it does | Permissions |
| --- | --- | --- |
| [`gha-meta/release.yml`](https://github.com/entur/gha-meta) | Runs release-please: opens the release pull request, creates the tag and GitHub release | `contents`, `pull-requests`, `issues`: write |
| Your `build` job + [`prepare-packages`](../README-prepare-packages.md) | You check out, install and build. `prepare-packages` packs the packages and uploads them | `contents: read` |
| [`publish.yml`](../README-publish.md) | Uploads the tarballs to npmjs. No checkout, runs nothing from your repository | `id-token: write` |

The three top-level permissions in your `release.yml` are required by `gha-meta/release.yml`.

## Why build and publish are separate jobs

Your `build` job runs install scripts, your build and the `prepack`/`prepare` scripts. Any dependency could be
compromised, so this job never gets an id-token.

`publish.yml` only uploads the tarballs. It has no checkout and runs no scripts from your repository or your
dependencies. So a compromised dependency can never publish.

This is why you must never add `id-token: write` to the `build` job.

## What `prepare-packages` does

1. **Finds the packages.** `packages` if set. Otherwise every package in the manifest when `release_type: manifest`.
   Otherwise `path`.
2. **Orders them.** Dependencies come first. Fails on a dependency cycle.
3. **Packs each one** with your `package_manager`. Fails if:
   - a `workspace:` range is left, or
   - `main`, `module`, `types`, `typings`, `bin` or `exports` points at a file that is not in the tarball.
4. **Uploads** the tarballs and `publish-order.tsv` as an artifact, kept for 1 day.

Packages with `"private": true` are listed but not packed.

## What `publish.yml` does

1. Downloads the artifact.
2. Publishes each tarball in order, with the npm CLI.
3. Skips versions that are already on npmjs. Set `skip_published: false` to fail instead.

## Good to know

- **Publishing always uses the npm CLI.** Trusted publishing is an npm CLI feature. The publish job uses the npm
  bundled with node 26.10.0 (npm 11.19.1).
- **`workspace:` ranges are replaced.** pnpm, yarn and bun turn `workspace:*` into the real version when they pack.
- **`prepublishOnly` does not run.** npm skips it when publishing a tarball. Move those steps to your build job or
  to `prepack`.
- **`prepack` runs.** If your package builds in `prepack`, you can drop the `npm run build` step.
- **Build the release tag.** Check out `needs.release.outputs.tag_name`. An empty `ref` builds the default branch.
  A monorepo has no single tag, so check out `github.sha` instead.
- **No `--tag` by default.** npm publishes under `latest` and refuses prerelease versions. See
  [Prerelease versions](examples.md#prerelease-versions).
