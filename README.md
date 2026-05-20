# dependency-sanity-action

GitHub Action for Maven projects. On every PR that touches a `pom.xml`, it
analyzes the project's dependencies, diffs them against the base branch, and
posts a Markdown health report as a PR comment.

For each declared dependency the report covers: latest available version
(with patch/minor/major classification), GitHub activity (last commit,
stars, archived state), OpenSSF Scorecard, license, and CVEs from OSV.

The analysis engine is the standalone [camel-sanity-analyzer
CLI](https://github.com/lhein/camel-sanity-analyzer); despite the historical
name it works on any Maven project, not just Apache Camel.

## Quick start

Add the following workflow to your repo at `.github/workflows/dependency-sanity.yml`:

```yaml
name: Dependency Sanity

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
      - uses: lhein/dependency-sanity-action@v0.1.2
        with:
          pom-path: pom.xml
```

## Inputs

| Name | Description | Default |
|---|---|---|
| `pom-path` | Pom file(s) to analyze — comma-separated, each entry may be a glob (e.g. `modules/*/pom.xml`). | `pom.xml` |
| `mode` | `diff` (PR-only) or `full` (entire tree). | `diff` |
| `fail-on` | `none` / `critical` / `warning` / `cve` — the action exits non-zero when the condition is met. | `none` |
| `include-transitive-test` | Follow transitive test-scope deps. Result tree gets much larger. | `false` |
| `comment-mode` | `update` (edit the previous bot comment) or `new` (always add). | `update` |
| `cli-version` | Analyzer CLI release tag to download. | `latest` |
| `github-token` | Token used for posting the comment. | `${{ github.token }}` |

## Outputs

| Name | Description |
|---|---|
| `report` | Path to the rendered Markdown file (also posted as PR comment). |

## Modes

- **diff** (default): compares the head pom against its base-branch version and only
  surfaces added / removed / version-changed deps plus health regressions. Best
  signal/noise for PR comments.
- **full**: full health report for the whole transitive tree. Useful for cron-style scans.

## Fail-on policy

Set `fail-on` to make the action fail the PR check when issues are found:

- `none` — always pass (informational only).
- `critical` — fail on any `CRITICAL` dependency (archived repo, high/critical CVE,
  no commit for 24+ months).
- `warning` — fail on `CRITICAL` *or* `WARNING`.
- `cve` — fail on any dependency with a known CVE.

## Multi-module projects

Pass a comma-separated list or a glob to `pom-path` and the action runs the
analyzer per POM, aggregating the results into one comment:

```yaml
- uses: lhein/dependency-sanity-action@v0.1.2
  with:
    pom-path: 'pom.xml, modules/*/pom.xml'
```

## How it works

1. Downloads the analyzer CLI jar from its releases.
2. Expands `pom-path` to a concrete list of POM files in the working tree.
3. For each POM in diff mode: extracts the base-branch version of that file via `git show`.
4. Runs the CLI per POM, then aggregates the markdown into one comment.
5. Posts (or updates) a single PR comment per workflow.
6. Honors the `fail-on` policy via the CLI's worst exit code across POMs.

## See also

- Analyzer source + web UI: https://github.com/lhein/camel-sanity-analyzer
- Sample workflow: [`.github/workflows/test.yml`](.github/workflows/test.yml)
