# Preset Design System Check

Validate PR code against your design system rules in CI. Catches AI slop, hardcoded colors, forbidden primitives, token misuse, and more — with inline PR annotations.

## Quick Setup

The fastest way to add Preset CI to your project:

```bash
npx preset ci init
```

This generates a `.github/workflows/preset.yml` configured for your project. See [CLI Setup](#cli-setup) for details.

## Manual Setup

Add to your workflow (`.github/workflows/preset.yml`):

```yaml
name: Design System Check

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  preset:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check Design System
        uses: preset-ai/compliance-check@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

That's it! The action will:
- Validate changed files against 16 default design system patterns
- Post inline annotations on the PR diff
- Post a summary comment on the PR
- Fail the check if errors are found

## Validation Modes

### Defaults (always runs, no credentials needed)

Runs `afterGeneration()` from `@preset/core`. Catches:
- Native HTML elements (`<select>`, `<input>`, `<button>`)
- Browser APIs (`window.alert`, `window.confirm`, `window.prompt`)
- Hardcoded Tailwind colors (`bg-red-500`, `text-blue-400`)
- AI slop patterns (generic gradients, excessive shadows, emojis, Lorem ipsum)
- Arbitrary spacing values

### Full (when Supabase credentials provided)

Also runs `DesignSystemValidator` from `@preset/mcp` against your team's actual database rules:
- Forbidden primitives
- Context rule violations
- Pattern compliance
- Intent route mismatches
- Token usage (hardcoded hex → semantic token)

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    supabase-url: ${{ secrets.PRESET_SUPABASE_URL }}
    supabase-key: ${{ secrets.PRESET_SUPABASE_KEY }}
    design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
```

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `github-token` | no | `${{ github.token }}` | Token for PR comments and annotations |
| `supabase-url` | no | — | Supabase project URL (enables full validation) |
| `supabase-key` | no | — | Supabase anon key |
| `design-system-id` | no | — | Design system UUID |
| `layer-id` | no | — | Optional layer for multi-layer systems |
| `include` | no | `**/*.{tsx,jsx,ts,vue,svelte}` | Comma-separated globs for files to check |
| `exclude` | no | `**/node_modules/**` | Comma-separated globs for files to skip |
| `fail-on-errors` | no | `true` | Fail the check if errors found |
| `fail-on-warnings` | no | `false` | Fail the check if warnings found |
| `comment-on-pr` | no | `true` | Post/update a PR comment |
| `upload-sarif` | no | `false` | Generate SARIF file |
| `score-threshold` | no | `0` | Minimum health score to pass (0-100, 0 = disabled) |
| `regression-threshold` | no | `0` | Max allowed score drop from previous snapshot (0 = disabled) |
| `fail-on-new-only` | no | `false` | Only fail on new violations not in baseline |
| `preset-api-key` | no | — | Legacy fallback for the per-run callback. With the [Preset GitHub App](#authenticating-the-callback) installed and `permissions: id-token: write` set, no key is needed. |
| `mcp-gateway-url` | no | `https://mcp.presetai.dev` | Override base URL for the callback target (e.g., self-hosted gateway) |
| `mcp-oidc-audience` | no | `https://presetai.dev` | Audience claim requested when minting the OIDC token. Override only for staging/self-hosted gateways. |
| `disable-callback` | no | `false` | Disable callback regardless of available auth |

## Outputs

| Output | Description |
|--------|-------------|
| `violations-count` | Total number of violations found |
| `errors-count` | Number of errors |
| `warnings-count` | Number of warnings |
| `passed` | Whether the check passed (`true`/`false`) |
| `health-score` | Overall health score (0-100, empty if not enabled) |
| `health-grade` | Health score grade (A-F, empty if not enabled) |
| `health-delta` | Score change from previous snapshot (empty if no history) |

## Health Scoring

Enable health scoring to track your design system's overall quality over time. The action runs a multi-dimensional audit covering token adoption, preset adoption, accessibility, and consistency.

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    supabase-url: ${{ secrets.PRESET_SUPABASE_URL }}
    supabase-key: ${{ secrets.PRESET_SUPABASE_KEY }}
    design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
    score-threshold: '70'
    regression-threshold: '5'
```

This will:
- Run the full health audit on each PR
- Fail if the overall score drops below 70
- Fail if the score regresses more than 5 points from the previous snapshot
- Include dimension scores in the PR comment

### Baseline Comparison

Use `fail-on-new-only` to adopt the action incrementally. It only fails on _new_ violations introduced in the PR, letting legacy violations pass until they're addressed:

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    # ...credentials...
    fail-on-new-only: 'true'
```

Requires a baseline snapshot saved in the database.

## Configuration File

Create a `.presetrc` file in your repo root to override defaults without changing action inputs:

```json
{
  "include": ["src/**/*.tsx", "src/**/*.ts"],
  "exclude": ["**/node_modules/**", "**/*.test.*"],
  "failOnErrors": true,
  "failOnWarnings": false,
  "scoreThreshold": 70,
  "regressionThreshold": 5
}
```

Only the seven fields above are recognized. If you add other keys to `.presetrc`, they're ignored locally and never reported via the [per-run telemetry callback](#per-run-telemetry).

## Per-Run Telemetry

After each run, the Action sends a single non-blocking request to your Preset workspace summarizing the run. The callback is what populates your dashboard's enforcement view.

### Authenticating the callback

The recommended path uses GitHub Actions OIDC — no secret in repo settings.

1. Install the [Preset GitHub App](https://github.com/apps/preset-ai) on your org or selected repos. Pick which design system the install binds to during the bind flow on `app.presetai.dev`.
2. Grant the workflow the `id-token: write` permission and `design-system-id` for the bound DS. The Action mints a fresh OIDC token per run, scoped to your repo and the configured audience.

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write   # required for OIDC

jobs:
  preset:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: preset-ai/compliance-check@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
```

#### Legacy API key fallback

If your runner can't grant `id-token: write` (some self-hosted setups) or you haven't installed the App yet, you can still authenticate with an API key:

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
    preset-api-key: ${{ secrets.PRESET_API_KEY }}
```

The Action prefers OIDC when both are configured. If the OIDC mint fails (no permission, non-GHA runner), it falls back to the API key automatically.

### What gets reported

One row per Action invocation, written to `agent_validation_outcome` in your workspace:

- **Run identity:** `trigger`, `outcome` (`pass` / `fail` / `skip`), `repo`, `run_id`, `run_attempt`, `check_run_id`, `commit_sha`, `commit_author_id`
- **Override flags:** your `fail-on-errors`, `fail-on-warnings`, `fail-on-new-only`, `score-threshold`, and `regression-threshold` values for that run
- **`.presetrc` state:** `presetrc_loaded` (boolean) and `presetrc_contents` filtered to a 7-field allowlist (see below)
- **Violation summary:** counts only: `{ errors, warnings, info }`
- **Run metadata:** `duration_ms`, `action_version`

### `.presetrc` allowlist

If a `.presetrc` is present on the runner, only these seven fields are reported:

`failOnErrors`, `failOnWarnings`, `failOnNewOnly`, `scoreThreshold`, `regressionThreshold`, `include`, `exclude`

Any other key is filtered out twice: once by the Action before sending, and again by the gateway before the row is inserted. This is intentional defense-in-depth so the privacy boundary doesn't depend on either layer alone.

### What we don't report

- **File contents:** never sent
- **Per-violation details:** rule IDs, line numbers, and code snippets stay in PR annotations and SARIF only
- **Environment variables and secrets:** never sent
- **Non-allowlist `.presetrc` keys:** filtered out before sending

### Opting out

Three ways to disable the callback:

| Goal | How |
|---|---|
| Never enable telemetry | Omit `permissions: id-token: write` and leave `preset-api-key` unset. The rest of the Action still works. |
| Disable it for one run | Set `disable-callback: true` on the step. |
| Forward to a self-hosted gateway | Set `mcp-gateway-url` to your endpoint and `mcp-oidc-audience` to the audience the gateway expects. |

The callback is non-blocking with a 3-second timeout, so it cannot fail your build. Errors are logged as Action warnings only.

### Why we collect this

- **Bypass detection:** distinguish a clean run ("Action found nothing") from a soft-mode run ("Action found violations but `fail-on-warnings: false` let them through")
- **Override visibility:** workflow YAML audits can miss `.presetrc` overrides; this surface makes them visible to your workspace owners
- **Enforcement signal:** feeds the enforcement pipeline so your team can flag rules that drift in real-world usage

## SARIF Integration

Generate SARIF output for GitHub Code Scanning:

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    upload-sarif: 'true'

- uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: .preset-results/results.sarif
    category: preset
```

Requires `security-events: write` permission.

## CLI Setup

The Preset CLI can generate the workflow file for you:

```bash
# Interactive setup
npx preset ci init

# Non-interactive with auto-detection
npx preset ci init --yes

# Preview without writing
npx preset ci init --dry-run

# With specific options
npx preset ci init --fail-on-warnings --score-threshold 70 --sarif
```

The CLI is also offered during `preset init` and `preset setup` flows.

## PR Comment

The action posts a formatted comment on the PR with:
- Pass/fail status
- Health score with grade and delta (when enabled)
- Summary table (files checked, errors, warnings, mode)
- Per-rule breakdown
- Expandable violation details (capped at 30)

The comment is updated on subsequent pushes (not duplicated).

## Troubleshooting

### No PR comment appears

Check that:
1. `comment-on-pr` is `true` (default)
2. The workflow is triggered by `pull_request` event
3. The token has `pull-requests: write` permission

### SARIF upload fails

Ensure `security-events: write` permission is set in your workflow.

### No files checked

The action only validates files changed in the PR with design-system-relevant extensions (`.tsx`, `.jsx`, `.ts`, `.vue`, `.svelte`, `.css`). Check your `include`/`exclude` patterns.

### Health score not appearing

Health scoring requires full mode (Supabase credentials) and at least one threshold enabled (`score-threshold` > 0 or `regression-threshold` > 0).
