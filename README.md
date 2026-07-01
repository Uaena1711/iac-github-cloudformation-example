# iac-github-cloudformation-example

Working consumer of the [`Uaena1711/iac-github`](https://github.com/Uaena1711/iac-github)
**CloudFormation** pipeline (`cfn-env.yml@v2`). One job per environment; the change set is the
plan and `apply` executes it after the GitHub Environment gate.

```
.github/workflows/cloudformation.yml   # dev + prod jobs -> cfn-env.yml@v2
templates/demo.yaml                    # ONE shared template (referenced by every env, not copied)
stacks/
  dev/  { cfn-ci.env, cfn-params.env, parameters.json }
  prod/ { cfn-ci.env, cfn-params.env, parameters.json }
```

`cfn-ci.env` points `TEMPLATE` at the shared `templates/demo.yaml` (repo-root-relative); per-env
differences live only in `parameters.json` (the `EnvName`) and the sensitive `cfn-params.env`.

- **`cfn-ci.env`** — per-stack identity + deploy config (role, region, stack name, template,
  parameters). Parsed, never sourced. `SECRETS_PROVIDER=github` resolves `${REF}` from repo
  vars/secrets.
- **`parameters.json`** — non-secret CloudFormation parameters.
- **`cfn-params.env`** — sensitive `NoEcho` parameters (`Name=${REF}`), resolved from a GitHub
  secret, masked, merged into the change set in memory (never on disk).

The demo stack creates one SSM parameter `/iac-github/demo/cfn-<env>` whose value comes from a
NoEcho template parameter.

## Setup
- GitHub **Environments** `dev` (no reviewers → auto) and `prod` (required reviewer → gated).
- Repo **secrets** `DEMO_SECRET_DEV` / `DEMO_SECRET_PROD` (the sensitive parameter value).
- An AWS IAM role (`AWS_ROLE_ARN` in each `cfn-ci.env`) federated to this repo via GitHub OIDC,
  with CloudFormation + the demo SSM permissions.

## Flow
- **PR** → change-set preview per env (no execution).
- **Push to `main`** → dev applies automatically; prod applies after its reviewer approves.
- **Destroy** → run the workflow manually with `mode: destroy`.
