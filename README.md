# navno-ci

Reusable GitHub Actions workflows for the navno team's repositories.

Reference a workflow as `navikt/navno-ci/.github/workflows/<name>.yml@v1`.

| Workflow | Does | Output |
|---|---|---|
| `node-build.yml` | pnpm install, lint, build, test; then CDN upload, prune to prod deps and push a Docker image | `image` |
| `jvm-build.yml` | Gradle build; then push a Docker image | `image` |
| `nais-deploy.yml` | Apply one NAIS resource to a cluster | |
| `release.yml` | Create a GitHub release with a timestamped tag and the deployed image ([rollback](#rollback)) | |
| `dependabot-lockfile.yml` | Regenerate `pnpm-lock.yaml` on dependabot PRs | |

`image` is empty when `push-image: false`.

Each section below shows an example call, then lists every input and secret
under a collapsed heading. Inputs are optional unless the default column says
`required`.

## Permissions and secrets

The calling job, or the calling workflow, must grant what the called job
declares:

| Workflow | Permissions |
|---|---|
| `node-build.yml`, `nais-deploy.yml` | `contents: read`, `id-token: write` |
| `jvm-build.yml` | `contents: read`, `id-token: write`, `packages: read` |
| `release.yml`, `dependabot-lockfile.yml` | `contents: write` |

Pass secrets with `secrets: inherit`. A `with:` value cannot reference
`secrets`; GitHub does not allow it in reusable-workflow calls.

## Node app

```yaml
# pr-checks.yml
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
  id-token: write
jobs:
  check:
    uses: navikt/navno-ci/.github/workflows/node-build.yml@v1
    with:
      push-image: false
    secrets: inherit
```

```yaml
# deploy.prod.yml
on:
  push:
    branches: [main]
jobs:
  build:
    uses: navikt/navno-ci/.github/workflows/node-build.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      image-suffix: release-${{ github.ref_name }}
      cdn-source: ./server/dist/client/assets     # omit to skip the CDN
      cdn-destination: my-app/prod
      prune-remove: node_modules server/node_modules
      env-file: |                                  # omit if the build needs no .env
        ENV=prod
        VITE_APP_ORIGIN=https://www.nav.no
    secrets: inherit

  deploy:
    needs: build
    uses: navikt/navno-ci/.github/workflows/nais-deploy.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      cluster: prod-gcp
      resource: .nais/config.yml
      vars: .nais/vars-prod.yml
      var: image=${{ needs.build.outputs.image }}

  release:
    needs: [build, deploy]
    uses: navikt/navno-ci/.github/workflows/release.yml@v1
    permissions: { contents: write }
    with:
      image: ${{ needs.build.outputs.image }}   # lets a rollback redeploy this release
    secrets: inherit
```

The build runs `lint`, `build` and `test` in that order, each only if
package.json has the script. package.json must declare the Node version in
`engines.node` and the pnpm version in `packageManager`; setup fails without
`engines.node`.

<details>
<summary>Inputs and secrets</summary>

| Input | Default | Description |
|---|---|---|
| `push-image` | `true` | Upload to the CDN, prune to production dependencies and push the image. `false` for PR checks. |
| `image-suffix` | `''` | Suffix appended to the image name, e.g. `dev-deploy`. |
| `image-tag` | `''` | Extra tag applied to the image, e.g. `latest`. |
| `platforms` | `linux/amd64` | Docker build platforms. |
| `dockerfile` | `Dockerfile` | Path to the Dockerfile. |
| `team` | `navno` | Nav team owning the image registry and the CDN bucket. |
| `github-environment` | `''` | GitHub environment to run under, for scoped secrets and variables. |
| `env-file` | `''` | `KEY=VALUE` lines written to `env-file-path` before install. Blank lines and `#` comments are allowed; any other line fails the build. |
| `env-file-path` | `.env` | Where to write `env-file`. |
| `env-file-copy-to` | `''` | Extra paths the env file is copied to, space- or newline-separated. |
| `build-cache-paths` | `''` | Directories kept between runs, keyed on the lockfile, one per line. E.g. `packages/nextjs/.next/cache`. Written afresh every run, so keep it to small, mutable build output. |
| `playwright` | `''` | Command that runs the Playwright suite, with the browsers restored from a cache keyed on the installed Playwright version. Empty skips Playwright. |
| `post-build-commands` | `''` | Bash script run after lint, build and test with `-e -o pipefail`. For a repo-specific check such as a Playwright suite. |
| `cdn-source` | `''` | Static directory to upload to the Nav CDN. Empty skips the upload. |
| `cdn-destination` | `''` | CDN destination path, e.g. `my-app/prod`. |
| `prune-remove` | `node_modules` | Directories deleted before reinstalling with `--prod`, space- or newline-separated, globs allowed. Empty skips the prune. |
| `pnpm-deploy` | `''` | `<package> <output-dir>` pairs, one per line, each run as `pnpm --filter <package> deploy --prod <output-dir> --legacy`. |

| Secret | Required | Description |
|---|---|---|
| `READER_TOKEN` | no | GitHub Packages token for private @navikt packages. |
| `SERVICE_SECRET` | no | Set as `$SERVICE_SECRET` during `pnpm run build`. |
| `NAIS_WORKLOAD_IDENTITY_PROVIDER` | no | Fallback only; `nais/login` resolves the navikt value itself. |

`post-build-commands` sees every secret the caller passes as an environment
variable of the same name. To hand one to a tool under another name, set it
inline:

```yaml
    with:
      post-build-commands: |
        PLAYWRIGHT_TOKEN=$MY_SECRET pnpm exec james bond
    secrets: inherit
```

Gate the `playwright` input with the same condition as the suite, or a skipped run still pays for
the browser download:

```yaml
    with:
      playwright: ${{ !inputs.SKIP_TESTS && 'pnpm run test:e2e' || '' }}
```

</details>

## JVM app

```yaml
jobs:
  build:
    uses: navikt/navno-ci/.github/workflows/jvm-build.yml@v1
    permissions: { contents: read, id-token: write, packages: read }
    secrets: inherit

  deploy:
    needs: build
    uses: navikt/navno-ci/.github/workflows/nais-deploy.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      cluster: prod-gcp
      resource: .nais/nais.yaml
      vars: .nais/prod-gcp/navno.json
      workload-image: ${{ needs.build.outputs.image }}
```

<details>
<summary>Inputs and secrets</summary>

| Input | Default | Description |
|---|---|---|
| `push-image` | `true` | Build and push the Docker image. `false` for PR checks. |
| `java-version` | `21` | JDK major version. |
| `gradle-args` | `build --no-daemon` | Arguments for the Gradle invocation. |
| `team` | `navno` | Nav team owning the image registry. |
| `image-suffix` | `''` | Suffix appended to the image name. |
| `dockerfile` | `Dockerfile` | Path to the Dockerfile. |
| `github-environment` | `''` | GitHub environment to run under, for scoped secrets and variables. |

| Secret | Required | Description |
|---|---|---|
| `NAIS_WORKLOAD_IDENTITY_PROVIDER` | no | Fallback only; `nais/login` resolves the navikt value itself. |

</details>

## Deploy a NAIS resource

Use on its own for alerts, OpenSearch or a failover instance, or chain it
after a build as above.

```yaml
on:
  push:
    branches: [main]
    paths: [.nais/alerts.yaml]
permissions:
  contents: read
  id-token: write
jobs:
  alerts:
    uses: navikt/navno-ci/.github/workflows/nais-deploy.yml@v1
    with:
      cluster: prod-gcp
      resource: .nais/alerts.yaml
```

Pass the image with `var: image=...` when the manifest templates
`{{ image }}`, and with `workload-image:` when it does not. They are not
interchangeable.

<details>
<summary>Inputs</summary>

| Input | Default | Description |
|---|---|---|
| `cluster` | required | NAIS cluster, e.g. `dev-gcp` or `prod-gcp`. |
| `resource` | required | Path to the resource to apply, relative to the repo root. |
| `vars` | `''` | Path to a template variables file. |
| `var` | `''` | Inline template variables, comma-separated `key=value` pairs. |
| `workload-image` | `''` | Image to inject as the workload image, for manifests that do not template it. |
| `hpa-resource` | `''` | HPA resource applied before `resource`. Its failure is ignored. |
| `environment` | `''` | GitHub environment to run under, for approvals and scoped variables. |
| `ref` | `''` | Git ref to check out. Defaults to the triggering ref. |

No secrets.

</details>

## Release

```yaml
  release:
    needs: [build, deploy]
    uses: navikt/navno-ci/.github/workflows/release.yml@v1
    permissions: { contents: write }
    with:
      image: ${{ needs.build.outputs.image }}
    secrets: inherit
```

Creates a release tagged `<tag-prefix><unix timestamp>` with generated
release notes. The tag points at the built commit (`github.sha`), not
wherever `main` has moved to since. `RELEASE_TOKEN` is optional; without it
the release is created with `github.token`, which does not trigger other
workflows.

`image` is recorded as the first line of the release body,
`` Deployed image: `<image>` ``, above `body` and the generated notes. It must
be digest-pinned (`...@sha256:<digest>`), like the `image` output of
`node-build.yml` and `jvm-build.yml`. The job then checks the release with
`navikt/navno-actions/find-release@v1` and fails if a [rollback](#rollback)
couldn't use it. A release without `image` cannot be rolled back to.

<details>
<summary>Inputs and secrets</summary>

| Input | Default | Description |
|---|---|---|
| `target-commitish` | `github.sha` | Commit or branch the tag points at. |
| `tag-prefix` | `release/prod@` | Prefix for the generated tag. |
| `name` | `Release <ref name>` | Release title. |
| `image` | `''` | Deployed image, digest-pinned. Required for rollback. |
| `body` | `''` | Text placed below the image line and above the generated release notes. |

| Secret | Required | Description |
|---|---|---|
| `RELEASE_TOKEN` | no | PAT with `contents: write`, used to create the release. Falls back to `github.token`. |

</details>

## Rollback

A rollback redeploys an earlier release's image without building or testing.
There is no shared rollback workflow: the repo's own workflow uses
`navikt/navno-actions/find-release@v1` to look up the release's commit and
recorded image, then deploys them with `nais-deploy.yml`.

```yaml
# rollback.prod.yml
on:
  workflow_dispatch:
    inputs:
      release-tag:
        description: Release tag. Empty = the release before the newest one.
        type: string
        required: false
        default: ''
jobs:
  find:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      contents: read
    outputs:
      sha: ${{ steps.release.outputs.sha }}
      image: ${{ steps.release.outputs.image }}
    steps:
      - id: release
        uses: navikt/navno-actions/find-release@v1
        with:
          tag: ${{ inputs.release-tag }}
          # tag-prefix: ...   # if release.yml sets a custom tag-prefix

  deploy:
    needs: find
    uses: navikt/navno-ci/.github/workflows/nais-deploy.yml@v1
    permissions: { contents: read, id-token: write }
    with:
      cluster: prod-gcp
      ref: ${{ needs.find.outputs.sha }}
      resource: .nais/config.yml
      vars: .nais/vars-prod.yml
      var: image=${{ needs.find.outputs.image }}   # or workload-image: for JVM apps
```

An empty tag picks the release before the newest one, skipping releases of
the same commit; see navno-actions for details.

- The `.nais` files come from the release's commit, since `ref` checks it
  out, so later changes to the manifest or vars are rolled back too.
- A rollback creates no release, so running it again with an empty tag
  redeploys the same release. Pass a tag to go further back.
- The next push to `main` deploys `main` again. Revert or fix on `main`
  before merging anything else.
- If the ordinary deploy sets `environment`, set the same one here so its
  approvals and branch restrictions apply.

## Dependabot lockfile

```yaml
on:
  pull_request:
    branches: [main]
permissions:
  contents: write
jobs:
  update-lockfile:
    uses: navikt/navno-ci/.github/workflows/dependabot-lockfile.yml@v1
    secrets: inherit
```

Runs only when the PR author is `dependabot[bot]`. Reinstalls without a
frozen lockfile and pushes the updated `pnpm-lock.yaml` to the PR branch.

<details>
<summary>Secrets</summary>

No inputs.

| Secret | Required | Description |
|---|---|---|
| `READER_TOKEN` | no | GitHub Packages token for private @navikt packages. On dependabot PRs this is the Dependabot secret of the same name. |

</details>

## Toolchain in your own job

For a job a repo owns itself, such as Storybook, CodeQL or publishing a
library, use the setup actions directly:

```yaml
steps:
  - uses: actions/checkout@v7
  - uses: navikt/navno-actions/setup-node-pnpm@v1
    with:
      reader-token: ${{ secrets.READER_TOKEN }}
  - run: pnpm run build-storybook
```

The other actions are documented in navno-actions.

<details>
<summary>Inputs</summary>

`navikt/navno-actions/setup-node-pnpm@v1`

| Input | Default | Description |
|---|---|---|
| `reader-token` | `''` | Token used as `NODE_AUTH_TOKEN` when installing private packages. |
| `install` | `true` | Run `pnpm install` after setup. |
| `frozen-lockfile` | `true` | Use `--frozen-lockfile`. Dependabot wants `false`. |

`navikt/navno-actions/setup-jvm@v1`

| Input | Default | Description |
|---|---|---|
| `java-version` | `21` | JDK major version. |
| `distribution` | `temurin` | JDK distribution passed to `actions/setup-java`. |
| `dependency-graph` | `auto` | One of `auto`, `disabled`, `generate`, `generate-and-submit`. `auto` submits on the default branch, which needs `contents: write`, and disables it elsewhere. |

</details>

## Concurrency and timeouts

Every workflow declares both:

- Builds: one per calling workflow and ref. A superseded PR build is
  cancelled; everything else queues. 30 minutes.
- Deploys: one per repository, cluster, resource and vars file. Queued; a
  running deploy is never cancelled, but a newer queued run replaces an
  older queued one. 15 minutes.
- Releases: one per repository and ref. 10 minutes.
- Lockfile: one per repository and PR branch, cancelled when superseded.
  15 minutes.

Builds and deploys never share a group, so a PR check cannot block a deploy.

Rollbacks go through `nais-deploy.yml` and share these groups.

## `dependabot.yml` for consumer repos

```yaml
version: 2

registries:
  npm-github-packages:
    type: npm-registry
    url: https://npm.pkg.github.com
    token: ${{secrets.READER_TOKEN}}

updates:
  - package-ecosystem: "npm"        # or "gradle"
    directory: "/"
    registries:
      - npm-github-packages         # drop for gradle-only repos
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Europe/Oslo"
    open-pull-requests-limit: 10
    pull-request-branch-name:
      separator: "-"
    cooldown:
      semver-major-days: 14
      semver-minor-days: 7
      semver-patch-days: 7
    groups:
      minor-and-patch:
        update-types: ["minor", "patch"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Europe/Oslo"
    open-pull-requests-limit: 10
    pull-request-branch-name:
      separator: "-"
    groups:
      actions:
        patterns: ["*"]
```
