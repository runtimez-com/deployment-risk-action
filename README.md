# runtimez deployment-risk gate

A GitHub Action that catches risky Kubernetes changes before they ship, and checks that they landed well after.

- **Pre-apply** (`pull_request`): scores the PR's rendered manifests against the base branch — weakened probes,
  removed PodDisruptionBudgets or NetworkPolicies, requests above limits, risky add-on settings and more — posts one
  sticky comment on the PR, and fails the check at or above `fail-on`.
- **Post-deploy** (`push` to your deploy branch): waits until Argo CD has deployed the commit, then passes or fails
  the job on how the deploy went (`PASS` / `FAIL` / `INCONCLUSIVE` / `TIMEOUT` in the `verdict` output).

Pre-apply works on its own, offline. Connect it to runtimez to add the cluster-aware checks — will the new pods fit on
the target cluster, does the change clash with the add-on versions running there. Post-deploy always needs runtimez.

## Setup

1. **API key** — in runtimez, create an API key with the `api:read` scope. Its creator must be able to read the
   target cluster.
2. **Secrets** — in the repository (or organisation) settings → *Secrets and variables* → *Actions*, add:

   | Secret | Value |
   |---|---|
   | `RUNTIMEZ_URL` | your runtimez API URL (`https://…`) |
   | `RUNTIMEZ_API_KEY` | the `rk_…` key from step 1 |
   | `RUNTIMEZ_ORG_ID` | your organisation id (`org_…`) |
   | `RUNTIMEZ_CLUSTER_ID` | the cluster this repo deploys to (`cluster_…`) |

   Use secrets, not variables, for all four: GitHub prints a step's inputs in the run log, and a public repo's logs
   are public.
3. **Workflows** — copy [`examples/pr-gate.yml`](examples/pr-gate.yml) and, if you deploy with Argo CD,
   [`examples/post-deploy-gate.yml`](examples/post-deploy-gate.yml) into `.github/workflows/`, and point `manifests`,
   `baseline-manifests` and `argocd-app` at your own paths and Application.

Every PR comment ends with one line saying whether the cluster-aware checks ran: *on*, *off* (and which input is
missing or empty), or how many runtimez lookups failed and why. A missing or rejected key never fails the check, but
it is never silent either — the line and a warning annotation say so.

The gate reads **rendered** manifests. For Helm or Kustomize, render the head and base sides in a step before it
(see the comment in `examples/pr-gate.yml`).

## Inputs

| Input | Mode | Default | |
|---|---|---|---|
| `mode` | both | `pre-apply` | `pre-apply` or `post-deploy` |
| `manifests` | pre-apply | | rendered YAML file or directory (required) |
| `baseline-manifests` | pre-apply | | the base branch's rendered manifests — turns on the diff checks |
| `changed-files` | pre-apply | | newline-separated `git diff --name-only` output |
| `fail-on` | pre-apply | `HIGH` | `LOW` · `MEDIUM` · `HIGH` · `CRITICAL` |
| `github-token` | pre-apply | `github.token` | posts the PR comment (needs `pull-requests: write`) |
| `runtimez-url` | both | | `https://` (or `http://localhost`); required for post-deploy |
| `api-key` | both | | `rk_…` key with `api:read`; required for post-deploy |
| `org-id` | both | | required for post-deploy |
| `cluster-id` | both | | the target cluster |
| `argocd-app` | post-deploy | | `<namespace>/<name>` |
| `revision` | post-deploy | `github.sha` | commit to wait for |
| `timeout` | post-deploy | `900` | seconds before `TIMEOUT` |
| `gate-mode` | post-deploy | `fast` | `fast` (Synced + Healthy) or `full` (also release verification) |
| `fail-on-inconclusive` | post-deploy | `false` | fail the job when runtimez cannot judge the deploy |

## Versions

`@v1` follows the latest 1.x release; pin `@v1.0.0` (or a commit SHA) for a fixed gate. Each release runs one
exact image, `ghcr.io/runtimez-com/deployment-risk-action:<version>` — never `:latest`.
