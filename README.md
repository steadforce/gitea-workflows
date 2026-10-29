# Gitea Workflows

Reusable [Gitea Actions](https://docs.gitea.com/usage/actions/overview) workflows.

| Workflow | Purpose |
|---|---|
| [Helm hydration](#helm-hydration) | Render charts per environment, open one pull request per changed environment |

Gitea's runner loads them from GitHub by their absolute URL, e.g.
`uses: https://github.com/steadforce/gitea-workflows/.gitea/workflows/helm-hydration.yaml@v0.1.0`.
Reference them by tag: the runner caches a called workflow by its ref and doesn't fetch a branch
like `@main` again, so a branch ref keeps running an old version.

## Helm Hydration

`.gitea/workflows/helm-hydration.yaml` renders Helm charts into plain Kubernetes manifests before
deployment, so the manifests Argo CD applies are reviewed as a pull request. It's the Gitea variant
of `helm-hydration.yaml` in [steadforce/steadops-workflows](https://github.com/steadforce/steadops-workflows).

### Usage

```yaml
# .gitea/workflows/helm-hydration.yaml
name: Helm hydration

on:
  pull_request:
  # main opens pull requests, every other branch gets a preview, Renovate's are skipped
  push:
  # Deletes the preview of a deleted branch
  delete:

# For the job token, from Gitea 1.26 on
permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  hydration:
    uses: https://github.com/steadforce/gitea-workflows/.gitea/workflows/helm-hydration.yaml@v0.2.0
    with:
      # Pinned, so a new Helm release can't change the manifests without a commit
      helm-version: v4.2.1
      # Render only on pull requests, as a check; push and open pull requests on main
      create-pull-request: ${{ github.event_name == 'push' }}
      # Pushes to other branches publish this environment on preview/local/<branch>
      preview-environment: local
    # Before Gitea 1.26 only, instead of permissions: a bot token, see Setup
    # secrets:
    #   token: ${{ secrets.HYDRATION_TOKEN }}
```

Every environment in `helm-config.yaml` is rendered into `hydrated-manifests/` on the branch
`environments/<environment>`, through a pull request from
`hydration-pull-request/<environment>[-<chart>]-<version>`.

A push to any other branch, with `preview-environment` set, renders that environment only and
publishes it on `preview/<environment>/<branch>`, e.g. `preview/local/feature/kafka` for
`feature/kafka`, without a pull request. A local Argo CD Application follows the branch with
`targetRevision: preview/local/feature/kafka`. The preview is one commit with the same layout as
`environments/<environment>`, replaced on every push, and deleted with its branch.

Pushes to Renovate's branches are skipped (`renovate-branch-prefix`, Renovate's default
`branchPrefix`): their pull requests are checked, and their changes reach the environments
through `main`. This workflow's own branches, `environments/*`, `hydration-pull-request/*` and
`preview/*`, hold manifests only, without a workflow, so pushes to them don't start it at all.

```yaml
# helm-config.yaml
releaseName: &name external-secrets
namespace: *name
environments:
  dev:
    # API groups for `.Capabilities.APIVersions` checks, since nothing is rendered against a cluster
    apis:
    - monitoring.coreos.com/v1/ServiceMonitor
    valueFiles:
    - values-subchart-overrides.yaml
    - values-dev.yaml
```

### Inputs

| Input | Description | Default |
|---|---|---|
| `helm-version` | Helm CLI version to install, e.g. `v4.2.1`. Pin it, `latest` can change the manifests | `latest` |
| `bundle-patches-in-one-pr` | Group patch-level subchart updates into one PR per minor version | `true` |
| `charts-root` | Directory the chart search starts from | `.` |
| `chart-discovery-depth` | Maximum search depth for `helm-config.yaml` below `charts-root` | `1` |
| `create-pull-request` | Push to the environment branch and open a PR, for the default branch. `false` renders only | `true` |
| `preview-environment` | Environment a push to another branch publishes on `preview/<environment>/<branch>`. Empty for none | `""` |
| `renovate-branch-prefix` | Prefix of Renovate's branches, whose pushes are skipped. Empty skips none | `renovate/` |

### Setup

- **Token**: the workflow pushes branches, creates labels and opens pull requests.
  - **Gitea 1.26 and later** (tested with 1.26.1 and 28.2.0): the job token does all of it, with
    the `permissions:` of the caller as in Usage ([#36173](https://github.com/go-gitea/gitea/pull/36173)).
    No bot is needed. The maximum permissions in the owner's or repository's Settings → Actions →
    General must allow `write`, which they do unless changed.
  - **Before Gitea 1.26** (tested with 1.24.5): the job token can push and create labels, but not
    open pull requests ("user must be a collaborator"). Create a bot user, give it write access to
    the repositories (e.g. through an organization team), store a token of it with the scopes
    `write:repository` and `write:issue` as the organization secret `HYDRATION_TOKEN`. Pass it with
    the commented-out `secrets:` in Usage, instead of `permissions:`.
- **Runner**: a runner with the label `ubuntu-latest` whose image has bash, git, Node, curl, jq
  and yq, such as Gitea's default `gitea/runner-images:ubuntu-latest`. It needs to reach
  github.com for this repository, `actions/checkout` and `azure/setup-helm`, get.helm.sh for Helm,
  and the chart repositories the charts depend on. Where github.com isn't reachable, mirror this
  repository and the two actions into Gitea and reference them from there.

### What it does on each event

| Event | Result |
|---|---|
| Pull request | Renders every chart and environment as a check; pushes nothing |
| Push to another branch, `preview-environment` set | Renders that environment, force-pushes it to `preview/<environment>/<branch>` |
| Push to a Renovate branch | Nothing, its pull request is checked |
| Branch deleted, `preview-environment` set | Deletes `preview/<environment>/<branch>` |
| Push to `main`, manifests changed | Opens a pull request to `environments/<environment>`, or updates the open one |
| Push to `main`, manifests unchanged | Nothing |
| Manifests back to what `environments/<environment>` holds | Closes the open pull request |
