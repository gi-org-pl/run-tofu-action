# run-tofu-action

A GitHub Action that plans, applies or checks for drift in an OpenTofu configuration, and reports the plan with [report-tofu-plan-action](https://github.com/gi-org-pl/report-tofu-plan-action).

It runs the same steps in [github-iac](https://github.com/gi-org-pl/github-iac), [aws-iac](https://github.com/gi-org-pl/aws-iac) and [cloudflare-iac](https://github.com/gi-org-pl/cloudflare-iac). Each repository keeps its own workflows, with the trigger, environment, permissions and AWS role, and calls this action for everything else. Action versions and the OpenTofu version are set here once.

## Commands

| `command` | Steps |
|---|---|
| `plan` | checkout, AWS role, GitHub App token, OpenTofu, `fmt -check`, `init`, `validate`, `plan`, report |
| `apply` | checkout, AWS role, GitHub App token, OpenTofu, `init`, `plan`, report, `apply` of that plan |
| `drift` | checkout, AWS role, GitHub App token, OpenTofu, `init`, `plan -detailed-exitcode`, report, then `drift-check` |

`drift` fails when the plan is not empty, so a scheduled run fails when someone changed something by hand. `drift-check` still runs then, to also report resources that OpenTofu doesn't know about.

## Requirements

- `id-token: write` to assume an AWS role, and `pull-requests: write` to comment the plan on pull requests.
- Secrets and provider settings come from the job's `env`, for example `TF_VAR_state_passphrase` or `CLOUDFLARE_API_TOKEN`.

## Usage

```yaml
name: plan

on:
  pull_request:

jobs:
  plan:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    env:
      TF_VAR_state_passphrase: ${{ secrets.TOFU_STATE_PASSPHRASE }}
    steps:
      - uses: gi-org-pl/run-tofu-action@<commit SHA> # v1.0.0
        with:
          command: plan
          aws-role-arn: arn:aws:iam::123456789012:role/aws-iac-github-plan
```

The job has to be called `plan`: the "Require a passing plan" ruleset in each repository waits for a check with that name.

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `command` | `plan`, `apply` or `drift` | Yes | |
| `working-directory` | Directory of the OpenTofu root module | No | `infra` |
| `backend-config` | Backend settings file passed to `tofu init`, relative to `working-directory`; empty for none | No | `backend.hcl` |
| `aws-role-arn` | AWS role assumed through GitHub OIDC; empty to skip | No | |
| `aws-region` | AWS region of the role session | No | `eu-central-1` |
| `github-app-client-id` | GitHub App whose installation token OpenTofu gets as `GITHUB_TOKEN`; empty to skip | No | |
| `github-app-private-key` | Private key of that GitHub App | No | |
| `drift-check` | Shell command run from the repository root after a `drift` plan; gets the App token as `GH_TOKEN` | No | |
| `tofu-version` | OpenTofu version | No | `1.13.9` |
| `comment` | Whether to comment the plan on pull requests | No | `true` |

## Updating

Dependabot keeps the actions used here up to date. A new OpenTofu version is a change to the `tofu-version` default. After a release, Dependabot moves the IaC repositories to it.

Every merge to `main` that changes the action releases the next patch version and moves the `v1` tag. Run the `release` workflow by hand to release a minor or major version instead.
