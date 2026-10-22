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
  push:
    branches:
    - main

jobs:
  hydration:
    uses: https://github.com/steadforce/gitea-workflows/.gitea/workflows/helm-hydration.yaml@v0.1.0
    with:
      # Pinned, so a new Helm release can't change the manifests without a commit
      helm-version: v4.2.1
      # Render only on pull requests, as a check; push and open pull requests on main
      create-pull-request: ${{ github.event_name == 'push' }}
    secrets:
      # Bot token, needed before Gitea 28, see Setup
      token: ${{ secrets.HYDRATION_TOKEN }}
```

Every environment in `helm-config.yaml` is rendered into `hydrated-manifests/` on the branch
`environments/<environment>`, through a pull request from
`hydration-pull-request/<environment>[-<chart>]-<version>`.

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
| `create-pull-request` | Push to the environment branch and open a PR. `false` renders only | `true` |

### Setup

- **Token**: the workflow pushes branches, creates labels and opens pull requests.
  - **Gitea 1.24**: the job token can push and create labels, but not open pull requests ("user
    must be a collaborator"). Create a bot user, give it write access to the repositories (e.g.
    through an organization team), and store a token of it with the scopes `write:repository` and
    `write:issue` as the organization secret `HYDRATION_TOKEN`.
  - **Gitea 28** (tested with 28.2.0): the job token opens pull requests itself, so no bot is
    needed. Leave out `secrets:` and grant the permissions in the caller, as on GitHub:
    ```yaml
    permissions:
      contents: write
      pull-requests: write
      issues: write
    ```
- **Runner**: a runner with the label `ubuntu-latest` whose image has bash, git, Node, curl, jq
  and yq, such as Gitea's default `gitea/runner-images:ubuntu-latest`. It needs to reach
  github.com for this repository, `actions/checkout` and `azure/setup-helm`, get.helm.sh for Helm,
  and the chart repositories the charts depend on. Where github.com isn't reachable, mirror this
  repository and the two actions into Gitea and reference them from there.

### What it does on each event

| Event | Result |
|---|---|
| Pull request | Renders every chart and environment as a check; pushes nothing |
| Push to `main`, manifests changed | Opens a pull request to `environments/<environment>`, or updates the open one |
| Push to `main`, manifests unchanged | Nothing |
| Manifests back to what `environments/<environment>` holds | Closes the open pull request |
