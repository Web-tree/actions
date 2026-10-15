# Web-tree/actions

Shared GitHub Actions for Web-tree repositories. Public, so public repositories can use
it too: a public repository cannot call a workflow or action from a private one.

Anything here is read by the world. Keep secrets, internal hostnames and
repository-specific logic out; pass them in as inputs.

## Contents

| Path | Kind | What it does |
|---|---|---|
| `app-token-from-ssm` | composite action | Mints a GitHub App token from an App id and key in AWS SSM, read through an OIDC role. |
| `.github/workflows/release-please.yml` | reusable workflow | Runs release-please as a GitHub App, so release PRs get their checks without approval and release tags start `push: tags` workflows. |

## Secrets and environments

Nothing here holds an account id, role, parameter name or hostname: callers pass them
as inputs, and anything that grants access comes from a secret.

Put such secrets in an **environment**, not the repository, and give the environment a
deployment branch policy that admits only the branches that need them (for a release,
`main`). A job on any other branch, or from a fork, then cannot read them. For AWS,
trust the environment's OIDC subject (`repo:<owner>/<repo>:environment:<name>`), not a
branch subject, so the GitHub-side policy and the AWS trust agree.

A reusable workflow reads environment secrets only in a job that names the
environment, and only when the caller passes `secrets: inherit`.

### release-please

```yaml
on:
  push:
    branches: [main]

jobs:
  release-please:
    permissions:
      id-token: write   # AWS_ROLE only
    uses: Web-tree/actions/.github/workflows/release-please.yml@v1
    secrets: inherit
```

Secrets of the `release` environment (input `environment`), one of:

- `APP_ID` and `APP_PRIVATE_KEY`: the App's id and private key;
- `AWS_ROLE`: an OIDC role that reads them from SSM, with inputs `aws-region`,
  `app-id-parameter` and `private-key-parameter`.

Outputs: `release_created`, `releases_created`, `tag_name`, `version`, `paths_released`,
`prs_created`, `pr`, and `outputs` (everything release-please returned, as JSON).

### app-token-from-ssm

```yaml
environment: release
permissions:
  id-token: write
steps:
  - uses: Web-tree/actions/app-token-from-ssm@v1
    id: app
    with:
      aws-role: ${{ secrets.AWS_ROLE }}
      aws-region: eu-central-1
      app-id-parameter: /my/app/app-id
      private-key-parameter: /my/app/private-key
  - run: gh pr list
    env:
      GH_TOKEN: ${{ steps.app.outputs.token }}
```

## Versioning

Callers pin a major tag, `@v1`. A breaking change to an input, output or required
permission gets the next major. Release: tag `v1.x.y`, then move `v1` to it.
