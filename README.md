# lotof.cloud.ci

Shared GitHub Actions and CI decisions for the LOTOF repositories. This repository is public so that every service repository can reference its actions.

## Contents

```
.github/actions/update-state/action.yml   composite action that deploys by bumping an image tag
docs/adr/0001-graphql-as-the-gateway-api.md   architecture decision record on the gateway API
```

## `update-state` action

Rewrites the image tag of one service in the GitOps overlay repository, which makes ArgoCD roll it out (see `lotof.cloud.state` and `lotof.cloud.state.main`).

What it does:

1. Writes the SSH key to a throwaway directory (never to the runner's home; the self-hosted runner is a persistent machine) and uses it for git.
2. Clones `git@github.com:<owner>/lotof.cloud.state.<branch>.git`. Branch `main` maps to `lotof.cloud.state.main`.
3. If `<repo-name>/patch.yaml` does not exist there, it prints a notice and exits successfully (the `lotof.sample.*` templates build an image nobody deploys).
4. Replaces the `image:` line with `ghcr.io/<owner>/<repo-name>:<short-sha>` (first 7 characters of the commit).
5. Exits quietly if nothing changed; otherwise commits "Update image for <repo> to <sha>" and pushes. When the push is rejected because other services deployed at the same moment it rebases and retries up to five times, then fails.

### Inputs

| Input | Required | Meaning |
|---|---|---|
| `repo-name` | yes | Name of the service repository (directory name in the state repo) |
| `commit-sha` | yes | Full commit SHA that was built |
| `branch` | yes | Environment branch; selects the state repository (`main` -> `lotof.cloud.state.main`) |
| `ssh-key` | yes | Private deploy key with write access to the state repository |

### Usage

```yaml
deploy:
  runs-on: self-hosted
  needs: build-and-push
  environment: ${{ github.ref_name }}
  steps:
    - name: Update state repo
      uses: pieceowater-dev/lotof.cloud.ci/.github/actions/update-state@main
      with:
        repo-name: ${{ github.event.repository.name }}
        commit-sha: ${{ github.sha }}
        branch: ${{ github.ref_name }}
        ssh-key: ${{ secrets.STATE_REPO_SSH_KEY }}
```

The secret `STATE_REPO_SSH_KEY` must exist in the `main` environment of each service repository.

## Conventions of the service pipelines

The service repositories (`*.gtw`, `*.msvc.*`, `pages.edge`, `web.client`) share the same shape: test, build and publish the image to ghcr, then deploy through this action. They run on a single self-hosted runner, so a newer push cancels the older run of the same branch (`concurrency` with `cancel-in-progress`), and documentation-only commits should carry `[skip ci]` to avoid flooding the queue. The library `lotof.lib.gossiper` runs its checks on GitHub-hosted runners and auto-tags releases.

## Architecture decisions

`docs/adr/0001-graphql-as-the-gateway-api.md` proposes (status: proposed) that the gateways keep GraphQL (typed client generation, no N+1 by construction, subscriptions where needed) and records what it costs (generated code volume, flat one-RPC-per-field resolvers).

## Licence

MIT
