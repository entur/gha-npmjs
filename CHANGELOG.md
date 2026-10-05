# Changelog

## 1.0.0 (2026-10-05)


### ⚠ BREAKING CHANGES

* release.yml is removed, along with its build inputs. Call entur/gha-meta's release.yml, build in your own job ending with the prepare-packages action, then call publish.yml.

### Features

* add support for stage publishing ([bdd4f66](https://github.com/entur/gha-npmjs/commit/bdd4f6605e39f3d9030f89aae514ca29e38ade70))
* add support for workspace: ^ so .lock files are correct before release ([02af67e](https://github.com/entur/gha-npmjs/commit/02af67e09f1b92118a93c66caf3b0149e3635902))
* add workflow for initalizing tools, building packages and release to npmjs ([a88683e](https://github.com/entur/gha-npmjs/commit/a88683e83c1794e1364351f4a6471189f98a7573))
* default provenance to off ([6cf6dfa](https://github.com/entur/gha-npmjs/commit/6cf6dfaf01d86fce126bd32252dcc5ccf6d4145a))
* initialize baseline npmjs release workflow ([d25cfd8](https://github.com/entur/gha-npmjs/commit/d25cfd8d6acc95cf37e8028ab4c797d87b1cb756))
* split into prepare-packages action and publish workflow ([7bba955](https://github.com/entur/gha-npmjs/commit/7bba955fdd0e43a0e3b735cfac492f8b9e197d00))


### Bug Fixes

* PR comments ([0125605](https://github.com/entur/gha-npmjs/commit/012560547cacc61e607d1defb08ffa9db359d774))
* PR comments (prerelease dist-tag, build job permissions, portable artifact name, manifest checkout ref) ([68cdad1](https://github.com/entur/gha-npmjs/commit/68cdad104b3580e79362c509f6a41701d1a3a2b3))
* remove private publishing since we do not have it in Entur ([1694796](https://github.com/entur/gha-npmjs/commit/16947966f00d95371d92ea95e305a09329a75c4b))
* verify mise with mise which, drop manifest path check, sync docs and CI ([bba085a](https://github.com/entur/gha-npmjs/commit/bba085a55e3b3fd540da873a751a51a7e4fed8c8))
