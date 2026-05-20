# dependency-sanity-action

GitHub Action wrapping the [camel-sanity-analyzer](https://github.com/lhein/camel-sanity-analyzer)
CLI. On every PR that touches a `pom.xml`, it analyzes the dependencies of the
changed POM(s), diffs them against the base branch, and posts a Markdown report
as a PR comment.

## Quick start

Add the following workflow to your repo at `.github/workflows/camel-sanity.yml`:

```yaml
name: Camel Sanity

on:
  pull_request:
    paths: ['**/pom.xml']

permissions:
  pull-requests: write
  contents: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: lhein/dependency-sanity-action@v0.1.0
        with:
          pom-path: pom.xml
```

## Inputs

| Name | Description | Default |
|---|---|---|
| `pom-path` | Path to the pom.xml to analyze. | `pom.xml` |
| `mode` | `diff` (PR-only) or `full` (entire tree). | `diff` |
| `fail-on` | `none` / `critical` / `warning` / `cve` — the action exits non-zero when the condition is met. | `none` |
| `include-transitive-test` | Follow transitive test-scope deps. Result tree gets much larger. | `false` |
| `comment-mode` | `update` (edit the previous bot comment) or `new` (always add). | `update` |
| `cli-version` | CLI release tag to download. | `latest` |
| `github-token` | Token used for posting the comment. | `${{ github.token }}` |

## Outputs

| Name | Description |
|---|---|
| `report` | Path to the rendered Markdown file (also posted as PR comment). |

## Modes

- **diff** (default): compares the head pom against its base-branch version and only
  surfaces added/removed/version-changed deps plus health regressions. Best signal/noise
  for PR comments.
- **full**: full health report for the whole transitive tree. Useful for cron-style scans.

## Fail-on policy

Set `fail-on` to make the action fail the PR check when issues are found:

- `none` — always pass (informational only).
- `critical` — fail on any `CRITICAL` dependency (archived repo, high/critical CVE,
  no commit for 24+ months).
- `warning` — fail on `CRITICAL` *or* `WARNING`.
- `cve` — fail on any dependency with a known CVE.

## How it works

1. Downloads the latest `camel-sanity-cli.jar` from the analyzer's releases.
2. In diff mode: extracts the base-branch version of `pom-path` via `git show`.
3. Runs the CLI to produce a Markdown report.
4. Posts the report as a PR comment, updating the previous bot comment in-place
   when `comment-mode: update`.
5. Honors the `fail-on` policy via the CLI's exit code.

## See also

- The analyzer itself (web UI + CLI source): https://github.com/lhein/camel-sanity-analyzer
- Sample workflow: [`.github/workflows/test.yml`](.github/workflows/test.yml)
