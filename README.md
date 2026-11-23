# Preset Design System Check

Validate PR code against your design system rules in CI. Catches AI slop, hardcoded colors, forbidden primitives, token misuse, and more — with inline PR annotations.

## Quick Setup

The fastest way to add Preset CI to your project. Install the Preset CLI (a standalone binary, no npm), then generate the workflow:

```bash
curl -fsSL https://presetai.dev/install.sh | sh
preset ci init
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

### Full (when `design-system-id` is set)

Also checks each changed file against your design system's own rules:
- Forbidden primitives
- Context rule violations
- Pattern compliance
- Intent route mismatches
- Token usage (hardcoded hex → semantic token)

Full mode runs through the preset AI gateway. The Action signs in with a GitHub
OIDC token minted for the run, so no secret is stored in your repository. You
need the [preset AI GitHub App](https://github.com/apps/preset-ai) installed on the
repository and bound to your design system, and `permissions: id-token: write`
on the job:

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write

steps:
  - uses: actions/checkout@v4
  - uses: preset-ai/compliance-check@v1
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
```

If your runner can't grant `id-token: write`, pass `preset-api-key` instead.
Checking needs `author` scope: for the GitHub App, ask preset AI to enable it
for your installation; for an API key, use a Standard key.

`design-system-id` must be the design system your credential is bound to. The
gateway always checks against the bound one, so the Action fails the run when
the two differ rather than report results for a design system you didn't name.

### How the mode is chosen

| `design-system-id` | `supabase-url` / `supabase-key` | Mode |
|---|---|---|
| not set | anything | Basic |
| set | neither | Full, through the preset AI gateway |
| set | both | Full, reading Supabase directly (deprecated, see below) |
| set | only one | Configuration error: the check fails |

In full mode through the gateway, the check fails if the gateway can't be
reached, rejects the credential, has used up its monthly allowance, or returns
a result the Action can't read. It never falls back to basic mode, because a
run that couldn't check your rules isn't a pass. Short rate limits are waited
out (up to 5 minutes per run). These checks run even when no relevant file
changed, so a broken setup shows up on the next PR.

### Deprecated: `supabase-url` and `supabase-key`

Setting both still reads your design system directly with the Supabase key,
and the Action warns on every run. An anonymous key can't read a private
design system (the default), so in that setup full mode checks nothing against
your rules and fails on unchecked files (below). Remove both inputs to use the
gateway. They will be removed in a future major version. Health scoring still
needs them for now.

### Full mode and unchecked files

When full mode is configured, the Action checks whether your design system's own
rules actually applied to each changed file. If one or more changed files had no
binding rule behind them at all (the `unchecked-files` output is non-zero), the
check fails by default, even if no violations were found. A run that checked
nothing against your own rules is not a compliance pass.

This most often means one of two things:

- `design-system-id` points at a design system with no rules at warn or block
  for these files yet.
- You use the deprecated `supabase-key`, which can't read a private design
  system. Switch to the gateway (above).

If you want the Action to keep running without failing on this (for example,
while your design system intentionally has no rules yet), set
`allow-unchecked: true`:

```yaml
- uses: preset-ai/compliance-check@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    design-system-id: ${{ secrets.PRESET_DESIGN_SYSTEM_ID }}
    allow-unchecked: 'true'
```

`allow-unchecked` accepts "rules checked, none bound". It does not cover a
gateway failure, which always fails the check.

This has no effect in basic mode: basic mode never ran against a design system
in the first place, so it never fails on this.

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `github-token` | no | `${{ github.token }}` | Token for PR comments and annotations |
| `supabase-url` | no | — | Deprecated. With `supabase-key`, reads the design system directly instead of through the gateway |
| `supabase-key` | no | — | Deprecated. Supabase anon key; cannot read a private design system |
| `design-system-id` | no | — | Design system UUID. Turns on full mode; must match the design system your credential is bound to |
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
| `preset-api-key` | no | — | Legacy fallback for the per-run callback. With the [preset AI GitHub App](#authenticating-the-callback) installed and `permissions: id-token: write` set, no key is needed. |
| `mcp-gateway-url` | no | `https://mcp.presetai.dev` | Override base URL for the callback target (e.g., self-hosted gateway) |
| `mcp-oidc-audience` | no | `https://presetai.dev` | Audience claim requested when minting the OIDC token. Override only for staging/self-hosted gateways. |
| `disable-callback` | no | `false` | Disable callback regardless of available auth |
| `allow-unchecked` | no | `false` | In full mode, accept a run where none of your design system's binding rules applied to any changed file instead of failing it. See [Full mode and unchecked files](#full-mode-and-unchecked-files). |

## Outputs

| Output | Description |
|--------|-------------|
| `violations-count` | Total number of violations found |
| `errors-count` | Number of errors |
| `warnings-count` | Number of warnings |
| `passed` | Whether the check passed (`true`/`false`) |
| `unchecked-files` | Number of changed files checked against zero binding design system rules. See [Full mode and unchecked files](#full-mode-and-unchecked-files). |
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

1. Install the [preset AI GitHub App](https://github.com/apps/preset-ai) on your org or selected repos. Pick which design system the install binds to during the bind flow on `app.presetai.dev`.
2. Grant the workflow the `id-token: write` permission and `design-system-id` for the bound DS. The Action mints a fresh OIDC token per run, scoped to your repo and the configured audience. The same token is used for [full mode](#full-when-design-system-id-is-set).

> **Checking and result reporting need `author` scope.** An OIDC-authenticated installation defaults to `read`, which can neither run full-mode checks nor report results back: full mode fails with a message naming author scope, and the callback is refused with a 403 that explains why. There's no self-serve toggle for this yet: ask preset AI to enable `author` scope for your installation.

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
- **Override flags:** your `fail-on-errors`, `fail-on-warnings`, `fail-on-new-only`, `score-threshold`, `regression-threshold`, and `allow-unchecked` values for that run
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

The Preset CLI (a standalone binary; install with `curl -fsSL https://presetai.dev/install.sh | sh`) can generate the workflow file for you:

```bash
# Interactive setup
preset ci init

# Non-interactive with auto-detection
preset ci init --yes

# Preview without writing
preset ci init --dry-run

# With specific options
preset ci init --fail-on-warnings --score-threshold 70 --sarif
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

Health scoring still requires the deprecated Supabase inputs (`supabase-url`, `supabase-key`, `design-system-id`) and at least one threshold enabled (`score-threshold` > 0 or `regression-threshold` > 0). Moving it to the gateway is planned.

### Check fails with "no binding design system rules applied"

See [Full mode and unchecked files](#full-mode-and-unchecked-files). Check `design-system-id` (and stop using `supabase-key` if you still do), or set `allow-unchecked: true` if this is expected.

### Check fails with "the preset AI gateway could not check this run"

The error line above it says why. The usual causes:

- **No credential.** Add `permissions: id-token: write` to the job, or set `preset-api-key`.
- **`design-system-id` does not match.** The message names both design systems. Use the id of the design system your GitHub App installation or API key is bound to.
- **Missing author scope.** The message names it. See [Authenticating the callback](#authenticating-the-callback).
- **Monthly allowance used up.** The message includes the gateway's hint.

### Check fails with "Design system check misconfigured"

You set one of `supabase-url` and `supabase-key` but not the other. Remove both to use the gateway (recommended), or set both.
