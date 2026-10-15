# Web-tree/actions

Shared GitHub Actions for Web-tree repositories. Public, so public repositories can use
it too: a public repository cannot call a workflow or action from a private one.

Anything here is read by the world. Keep secrets, internal hostnames and
repository-specific logic out; pass them in as inputs.

## Contents

| Path | Kind | What it does |
|---|---|---|
| `release-app-token` | composite action | Mints a token for the org's `webtree-release` GitHub App, scoped to the calling repository. |
| `.github/workflows/release-please.yml` | reusable workflow | Runs release-please with that token, so release PRs get their checks without approval and release tags start `push: tags` workflows. |

### release-please

```yaml
on:
  push:
    branches: [main]

jobs:
  release-please:
    permissions:
      id-token: write
    uses: Web-tree/actions/.github/workflows/release-please.yml@v1
```

Outputs: `release_created`, `releases_created`, `tag_name`, `version`, `paths_released`,
`prs_created`, `pr`, and `outputs` (everything release-please returned, as JSON).

### release-app-token

```yaml
permissions:
  id-token: write
steps:
  - uses: Web-tree/actions/release-app-token@v1
    id: app
  - run: gh pr list
    env:
      GH_TOKEN: ${{ steps.app.outputs.token }}
```

### Enabling a repository

The token comes from AWS SSM through an OIDC role that reads only the App's id and key.
Add the repository's subject, normally `repo:Web-tree/<repo>:ref:refs/heads/main`, to
`webtree-release-app` in `github_ssm_reader_roles`
(`Web-tree/infrastructure`, `tf/aws/accounts/prod/modules.tf`). Never add a
`pull_request` subject: the key is org-wide.

## Versioning

Callers pin a major tag, `@v1`. A breaking change to an input, output or required
permission gets the next major. Release: tag `v1.x.y`, then move `v1` to it.
