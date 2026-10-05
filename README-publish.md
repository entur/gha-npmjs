# `gha-npmjs/publish`

Publish the tarballs from the [`prepare-packages` action](README-prepare-packages.md) to [npmjs.com](https://www.npmjs.com) with
[trusted publishing](https://docs.npmjs.com/trusted-publishers). No npm token needed. The job has no checkout and runs
no scripts from your repository or dependencies. It is the only job with `id-token: write`.

Configure the trusted publisher on npmjs with the filename of your top-level workflow, the one that calls
`publish.yml`, see [setup](README.md#step-2-configure-the-trusted-publisher-on-npmjs).

## Usage

See the [setup guide](README.md#setup) for the full `cd.yml`.

```yml
jobs:
  publish:
    needs: build
    permissions:
      contents: read
      id-token: write # trusted publishing (OIDC)
    uses: entur/gha-npmjs/.github/workflows/publish.yml@v1
    with:
      artifact_name: ${{ needs.build.outputs.artifact_name }}
```

## Inputs

<!-- AUTO-DOC-INPUT:START - Do not remove or modify this section -->

|                                     INPUT                                     |  TYPE   | REQUIRED | DEFAULT |                                                                                         DESCRIPTION                                                                                          |
|-------------------------------------------------------------------------------|---------|----------|---------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|    <a name="input_artifact_name"></a>[artifact_name](#input_artifact_name)    | string  |   true   |         |                                   Name of the GitHub artifact <br>with the tarballs, from the <br>artifact_name output of the prepare-packages <br>action                                    |
|           <a name="input_dist_tag"></a>[dist_tag](#input_dist_tag)            | string  |  false   |         |                    npm dist-tag to publish under. <br>Default: empty, so npm uses <br>latest and refuses to publish <br>a prerelease version without an <br>explicit tag                     |
|             <a name="input_dry_run"></a>[dry_run](#input_dry_run)             | boolean |  false   | `false` |                                                                Run npm publish with --dry-run, <br>so nothing is published.                                                                  |
|        <a name="input_provenance"></a>[provenance](#input_provenance)         | boolean |  false   | `false` |                                Publish with a provenance attestation <br>(npm publish --provenance). Requires a public source <br>repository. Default: false                                 |
|  <a name="input_skip_published"></a>[skip_published](#input_skip_published)   | boolean |  false   | `true`  | Skip packages whose version already <br>exists on npmjs, instead of <br>failing. Keeps monorepo releases idempotent, <br>since only some packages are <br>bumped per release. Default: true  |
| <a name="input_timeout_minutes"></a>[timeout_minutes](#input_timeout_minutes) | number  |  false   |  `15`   |                                                                         Timeout in minutes for the <br>publish job                                                                           |

<!-- AUTO-DOC-INPUT:END -->
