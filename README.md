# download-npm-package-action

A GitHub Action and reusable workflow to install npm packages and upload `node_modules` as an artifact, separating the job that requires `id-token: write` from the build job.

[Reusable Workflow](.github/workflows/download.yaml), [restore action](restore/action.yaml)

## Motivation

We use [flatt-security/setup-takumi-guard-npm](https://github.com/flatt-security/setup-takumi-guard-npm), a GitHub Action that routes `npm ci` through a security registry.
setup-takumi-guard-npm requires the `id-token: write` permission for authentication, but this permission should not be granted to the build job.
If it were, malicious code running in the build job, such as install scripts of npm packages or the code of the repository, could abuse it to sign malicious artifacts or assume AWS IAM roles or Google Cloud service accounts via OIDC.
To avoid this, this action installs npm packages in a separate job and uploads `node_modules` as an artifact, which the build job then restores.

`node_modules` is archived with tar to keep file permissions and symbolic links such as `node_modules/.bin`.

Install scripts of npm packages (e.g. `postinstall`) aren't run in the job with `id-token: write`, because `npm ci` is run with `--ignore-scripts`.
Instead, the restore action runs them by `npm rebuild` in the job restoring `node_modules`.

The restore action also sets `NPM_CONFIG_OFFLINE=true` for subsequent steps by default, so npm commands such as `npx` can't download packages from the registry without Takumi Guard.
Install scripts that download files by themselves aren't affected.

This is the npm version of [download-go-module-action](https://github.com/suzuki-shunsuke/download-go-module-action).

## Usage

### Reusable Workflow

```yaml
jobs:
  npm:
    uses: suzuki-shunsuke/download-npm-package-action/.github/workflows/download.yaml@main
    permissions:
      contents: read
      id-token: write
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}

  build:
    timeout-minutes: 30
    runs-on: ubuntu-26.04
    needs: npm
    permissions:
      contents: read
    steps:
      - name: Checkout the commit where npm packages were installed
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{needs.npm.outputs.sha}}
          persist-credentials: false

      - name: Set up Node.js
        uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version-file: .node-version

      - name: Restore node_modules
        uses: suzuki-shunsuke/download-npm-package-action/restore@main

      - run: npm run build
```

The reusable workflow outputs `sha`, the commit SHA it checked out.
Check out the same commit in the job restoring `node_modules`, so both jobs use the same `package-lock.json` even if the pull request is updated in between.

#### Inputs

- `working_directory`: The directory where package.json and package-lock.json exist. The default is `.`
- `node-version-file`: The default is `.node-version`
- `artifact_name`: The default is `node_modules`. Artifact names must be unique in a workflow run, so set this if this workflow is called multiple times in a workflow run
- `ref`: The git ref to checkout. By default, the default ref of actions/checkout is used, which is the merge commit on `pull_request` events

### GitHub Action

```yaml
jobs:
  npm:
    timeout-minutes: 20
    runs-on: ubuntu-26.04
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - uses: suzuki-shunsuke/download-npm-package-action@main
        with:
          takumi_guard_bot_id: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```

### Restore action

The restore action doesn't set up Node.js. Set it up yourself before running the restore action, because it runs `npm rebuild` by default.

```yaml
- uses: suzuki-shunsuke/download-npm-package-action/restore@main
  with:
    # working_directory: . # Optional. The directory where node_modules is extracted
    # artifact_name: node_modules # Optional
    # rebuild: "true" # Optional. If this is false, npm rebuild isn't run, so install scripts of npm packages aren't run
    # offline: "true" # Optional. If this is true, NPM_CONFIG_OFFLINE=true is set for subsequent steps
```
