# sengol-github-action

GitHub Action that runs a [Sengol](https://github.com/sengol-io/sengol) evaluation suite and gates CI/CD on compliance obligations.

The action installs `sengol`, runs `sengol run-suite` against your `sengol.yaml`, and writes the results to either the GitHub Actions step summary or a JUnit XML file. Workflow-command `::error::` annotations are emitted alongside both modes so failed obligations surface inline on the PR diff. Your pipeline fails when an obligation pass rate falls below its declared threshold.

## Usage

Minimal — runs the suite, fails the workflow when the gate fails, writes a markdown summary to the run page:

```yaml
- uses: sengol-io/sengol-github-action@v0.1.0
  with:
    config: sengol.yaml
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

JUnit XML output (for surfacing failures in test-report integrations):

```yaml
- uses: sengol-io/sengol-github-action@v0.1.0
  with:
    output-format: junit
    junit-output-path: eval-results.xml
  env:
    SENGOL_AUDIT_URI: ${{ secrets.SENGOL_AUDIT_URI }}
    SENGOL_SIGNING_KEY: ${{ secrets.SENGOL_SIGNING_KEY }}
    SENGOL_TENANT_ID: ${{ secrets.SENGOL_TENANT_ID }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `config` | no | `sengol.yaml` | Path to your `sengol.yaml`. |
| `python-version` | no | `3.11` | Python version installed before `pip install sengol`. |
| `sengol-version` | no | *(latest)* | Pin a specific release (e.g. `0.2.0`). |
| `fail-on-gate-failure` | no | `true` | Exit non-zero when the gate fails. |
| `output-format` | no | `github-summary` | One of `github-summary`, `junit`. |
| `junit-output-path` | no | `eval-results.xml` | Path for JUnit XML when `output-format=junit`. |

## Outputs

| Output | Description |
|---|---|
| `passed` | `"true"` / `"false"` — gate verdict for downstream steps. Always set. |
| `score` | GoldScore (0.0–1.0) for the evaluation period. Empty string (`''`) when GoldScore is not configured. |
| `report_url` | Path or URL to the generated PDF evidence pack. Empty string (`''`) when PDF generation is not enabled. |
| `no_agent_activity` | `"true"` when at least one obligation found no agent activity in the evaluation period. Empty string (`''`) otherwise. Use to branch on agent-offline vs. evaluation-failure scenarios. |

> **GitHub Actions composite action note:** All declared outputs are resolved by the runner even when the underlying step does not emit a value, producing an empty string rather than a truly absent output. Check with `!= ''` rather than testing for absence:
>
> ```yaml
> - if: steps.sengol.outputs.score != ''
>   run: echo "Score is ${{ steps.sengol.outputs.score }}"
>
> - if: steps.sengol.outputs.no_agent_activity != ''
>   run: echo "Agent was offline during evaluation period"
> ```

The SDK's `write_outputs()` never writes an empty string to `$GITHUB_OUTPUT` — the empty string consumers see is the GitHub Actions runner's own default for unmapped outputs.

## Required secrets

The action does not inject defaults for audit-store credentials — your workflow must supply:

- `SENGOL_AUDIT_URI` — Postgres URI (or `memory://` for ephemeral local runs)
- `SENGOL_SIGNING_KEY` — 32-byte HMAC key for signing audit records
- `SENGOL_TENANT_ID` — tenant identifier
- `ANTHROPIC_API_KEY` — only when your suite includes LLM-judge evaluators

## License

Apache 2.0. See [LICENSE](LICENSE).
