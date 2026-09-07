# sengol-github-action

GitHub Action that runs a [Sengol](https://github.com/sengol-io/sengol) evaluation suite and gates CI/CD on compliance obligations.

The action installs `sengol`, runs `sengol gate` against your `sengol.yaml`, and writes the results to the GitHub Actions step summary, a sticky PR comment, or a JUnit XML file. Workflow-command annotations are emitted alongside all modes so failed obligations surface inline on the PR diff. Your pipeline fails when an obligation pass rate falls below its declared threshold.

The gate verdict and the signed audit record are the same object — the PR comment is the developer view of evidence that already exists, never a second source of truth.

## Usage

Minimal — runs the suite, fails the workflow when the gate fails, writes a markdown summary and posts a sticky PR comment:

```yaml
- uses: sengol-io/sengol-github-action@2.0.0
  with:
    config: sengol.yaml
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

> **Required permission:** The action posts a PR comment via `GITHUB_TOKEN`. Add `pull-requests: write` to your workflow's permissions block:
> ```yaml
> permissions:
>   pull-requests: write
>   contents: read
> ```
> Fork PRs receive a read-only token automatically — the gate degrades gracefully (::notice + step summary) and never fails because of the comment.

JUnit XML output (for surfacing failures in test-report integrations):

```yaml
- uses: sengol-io/sengol-github-action@2.0.0
  with:
    output-format: junit
    junit-output-path: eval-results.xml
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
```

Flake tolerance — retry LLM evaluators 3 times, require 2/3 to pass:

```yaml
- uses: sengol-io/sengol-github-action@2.0.0
  with:
    repeat: '3'
    min-pass: '2'
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Push results to a downstream exporter (e.g. MLflow):

```yaml
- uses: sengol-io/sengol-github-action@2.0.0
  with:
    export: mlflow
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
    MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
    MLFLOW_TRACKING_TOKEN: ${{ secrets.MLFLOW_TRACKING_TOKEN }}
```

### Python caching

Pin `sengol-version` and enable pip caching via `actions/setup-python` to avoid re-downloading on every run:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'

- uses: sengol-io/sengol-github-action@2.0.0
  with:
    sengol-version: '2.0.0'   # pin to a specific release
    config: sengol.yaml
```

> **Docker prebuilt-image variant** (no install step, sub-second startup) is Phase 7 backlog.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `config` | no | `sengol.yaml` | Path to your `sengol.yaml`. |
| `python-version` | no | `3.11` | Python version installed before `pip install sengol`. |
| `sengol-version` | no | `>=2.0,<3` | Pin a specific release or range (e.g. `2.0.0`, `>=2.0,<3`). |
| `extras` | no | `anthropic-judge,openai-judge` | Comma-separated sengol extras to install. The default ships both LLM-judge provider SDKs so judges wire up out of the box. Add `model` for statistical evaluators (PSI/AUC/Fairness); set to `""` for a bare install (deterministic evaluators only). |
| `fail-on-gate-failure` | no | `true` | Exit non-zero when the gate fails. |
| `output-format` | no | `github-summary` | One of `github-summary`, `junit`. |
| `junit-output-path` | no | `eval-results.xml` | Path for JUnit XML when `output-format=junit`. |
| `export` | no | *(none)* | Comma-separated result-exporter names (e.g. `mlflow`). Options come from the `sengol.yaml` `exporters:` block. Exporters are best-effort sinks — a failure never fails the gate. |
| `pr-comment` | no | `true` | Post/update a sticky PR comment with gate results on `pull_request` events. Requires `pull-requests: write`. Degrades gracefully on fork PRs. |
| `repeat` | no | `1` | Run LLM evaluators N times per input for flake tolerance. Deterministic evaluators always run once. All attempt records are written to evidence. |
| `min-pass` | no | `0` | Require M of N (`repeat`) LLM-evaluator attempts to pass. `0` means all must pass (equivalent to `repeat` value). |
| `dataset-path` | no | `''` | Path to a JSONL trace dataset (`sengol gate --dataset-path`). Empty uses the dataset from `sengol.yaml`. |
| `dataset-id` | no | `''` | Dataset identifier recorded on the evaluation run. Empty derives it from config. |

## Outputs

| Output | Description |
|---|---|
| `passed` | `"true"` / `"false"` — gate verdict for downstream steps. Always set. |
| `score` | GoldScore (0.0–1.0) for the evaluation period. Empty string (`''`) when GoldScore is not configured. |
| `report_url` | Path or URL to the generated PDF evidence pack. Empty string (`''`) when PDF generation is not enabled. |
| `no_agent_activity` | `"true"` when at least one obligation found no agent activity in the evaluation period. Empty string (`''`) otherwise. |
| `regression` | `"true"` when current pass-rate regressed vs the configured `suite.baseline_pass_rate` in `sengol.yaml`. `"false"` when at or above baseline. Empty string (`''`) when no baseline is configured. |

> **GitHub Actions composite action note:** All declared outputs are resolved by the runner even when the underlying step does not emit a value, producing an empty string rather than a truly absent output. Check with `!= ''` rather than testing for absence:
>
> ```yaml
> - if: steps.sengol.outputs.regression == 'true'
>   run: echo "Pass-rate regressed vs baseline"
>
> - if: steps.sengol.outputs.no_agent_activity != ''
>   run: echo "Agent was offline during evaluation period"
> ```

The SDK's `set_outputs()` never writes an empty string to `$GITHUB_OUTPUT` — the empty string consumers see is the GitHub Actions runner's own default for unmapped outputs.

## Regression detection

Add `baseline_pass_rate` to your `sengol.yaml` to enable regression detection:

```yaml
suite:
  name: acme-compliance
  evaluators:
    - FaithfulnessEvaluator
  baseline_pass_rate: 0.95   # alert when pass-rate drops below this
```

The `regression` output is then `"true"` / `"false"` on every run, and the pass-rate delta appears in the step summary and PR comment.

## Required secrets

The action does not inject defaults for audit-store credentials — your workflow must supply:

- `SENGOL_AUDIT_URI` — Postgres URI (or `memory://` for ephemeral local runs)
- `SENGOL_SIGNING_KEY` — 32-byte HMAC key for signing audit records
- `SENGOL_TENANT_ID` — tenant identifier
- `ANTHROPIC_API_KEY` — only when your suite includes LLM-judge evaluators

## Authentication: GitHub OIDC federation (no long-lived authentication token)

This removes the long-lived **`SENGOL_API_TOKEN`** from CI. It does not make the workflow
"secret-free" — `SENGOL_AUDIT_URI`, `SENGOL_SIGNING_KEY`, `SENGOL_TENANT_ID`, and (for
LLM-judge suites) `ANTHROPIC_API_KEY` are still required secrets.

**Before (every workflow):** a long-lived `SENGOL_API_TOKEN` stored as a repo/org secret,
shared across every run, with no per-run identity.

```yaml
- uses: sengol-io/sengol-github-action@2.0.0
  with:
    config: sengol.yaml
  env:
    SENGOL_API_TOKEN: ${{ secrets.SENGOL_API_TOKEN }}   # stored secret, never rotates itself
```

**After (Phase 7c — ADR-0077):** GitHub already issues a short-lived OIDC token to every
workflow run. The action exchanges it for a Sengol JWT — no long-lived Sengol auth token
stored in CI. Add `id-token: write` and drop `SENGOL_API_TOKEN`. The example below is a
complete, runnable job (`permissions:` is a job-level key — it cannot sit next to a
`- uses:` step):

```yaml
jobs:
  governance:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # required for the OIDC exchange
      pull-requests: write  # required for the sticky PR comment
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: sengol-io/sengol-github-action@2.0.0
        with:
          config: sengol.yaml
        env:
          SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
          SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
          SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Audience.** The exchange uses an OIDC audience that must match on both sides. The default
is `sengol`. If your Sengol server is configured with a custom `SENGOL_GITHUB_OIDC_AUDIENCE`,
the workflow must request a GitHub token with the **same** audience by setting
`SENGOL_GITHUB_OIDC_AUDIENCE` as a step env var — GitHub only changes the token's `aud`
claim when the action requests it explicitly. The action.yml exposes no `audience` input;
it is configured purely through this env var:

```yaml
      env:
        SENGOL_GITHUB_OIDC_AUDIENCE: my-org-audience   # only if the server uses a non-default audience
        # ...plus the secrets above
```

If the server still has the default `SENGOL_GITHUB_OIDC_AUDIENCE` unset, the exchange
returns 503 and the action falls back to `SENGOL_API_TOKEN` (see below).

**Fallback.** The action detects OIDC availability (`ACTIONS_ID_TOKEN_REQUEST_URL`) at
runtime: if `id-token: write` isn't granted, or the server doesn't accept the exchange, it
**falls back to `SENGOL_API_TOKEN`** automatically. For that fallback to work, the
`SENGOL_API_TOKEN` secret must remain set in the workflow env — only remove it once you
have confirmed the OIDC exchange succeeds. Every exchange is logged server-side with the
repository, ref, commit, and workflow run ID (`github_workflow_audit_log` — see
`docs/reference/console.md` in the `sengol` repo).

## License

Apache 2.0. See [LICENSE](LICENSE).
