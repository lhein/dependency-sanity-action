# dependency-sanity-action

GitHub Action for Maven projects. On every PR that touches a `pom.xml`, it
analyzes the project's dependencies, diffs them against the base branch, and
posts a Markdown health report as a PR comment.

For each declared dependency the report covers: latest available version
(with patch/minor/major classification), GitHub activity (last commit,
stars, archived state), OpenSSF Scorecard, license, and CVEs from OSV.
Newly added critical dependencies, new CVEs, or health regressions are
called out in a banner at the top of the comment so reviewers can spot
them without scrolling.

The analysis engine is the standalone CLI from
[camel-sanity-analyzer](https://github.com/lhein/camel-sanity-analyzer)
— that repo is the engine source despite its historical name; the
analyzer itself works on any Maven project, not just Apache Camel.

## Quick start

Add the following workflow to your repo at `.github/workflows/dependency-sanity.yml`:

```yaml
name: Dependency Sanity

on:
  pull_request:
    paths: ['**/pom.xml']

permissions:
  pull-requests: write       # required so the bot can post the PR comment
  contents: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0     # required: diff mode reads the base-branch pom via `git show`
      - uses: lhein/dependency-sanity-action@v0.1.2
        with:
          pom-path: pom.xml
```

Two non-obvious requirements:

- `permissions: pull-requests: write` — without it the action can't post the
  comment and the run fails with a 403.
- `fetch-depth: 0` on `actions/checkout` — diff mode runs `git show <base-sha>:pom.xml`
  to grab the base-branch version of the POM; that needs the full git history.
  With `fetch-depth: 1` (the default), the action falls back to a full scan.

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

## Sample output

When a PR adds a known-vulnerable dependency (e.g. log4j-core 2.14.1), the
comment looks like:

> ## Dependency Sanity Analyzer — PR diff
>
> Project: **com.example:my-app:1.0.0-SNAPSHOT**
>
> > ### 🚨 Breaking changes detected
> >
> > - **1** newly added dependency is `CRITICAL`
> > - **7** new CVEs introduced by added dependencies
> >
> > See sections below for details.
>
> ### Summary of changes
>
> | | Base | Head | Δ |
> |---|---:|---:|---:|
> | Total deps | 11 | 13 | +2 |
> | 🔴 Critical | 0 | 1 | +1 |
> | 🛡 With CVEs | 0 | 1 | +1 |
> | 🟡 Warning | 0 | 0 | — |
> | 🟠 Outdated | 0 | 0 | — |
> | 🟢 Healthy | 10 | 11 | +1 |
>
> ### ➕ Added (2)
>
> - **log4j-core** `2.14.1` (latest `2.26.0`) — score 50/100
>   - 🛡 **🚨 Critical** (2)
>     - [GHSA-jfh8-c2jp-5v3q](https://osv.dev/vulnerability/GHSA-jfh8-c2jp-5v3q) — Remote code injection in Log4j
>     - …
>   - 🛡 **🔴 High** (1)
>     - …

Real example in the dev-test repo: [PR #2](https://github.com/lhein/dependency-sanity-test/pull/2).

## Breaking-change banner

The action automatically surfaces a `> ### 🚨 Breaking changes detected` quote
block at the top of the comment whenever **any** of these conditions are met:

- a newly added dependency has status `CRITICAL` (archived repo, high CVE, no
  commit for 24+ months, …)
- a newly added dependency carries one or more CVEs
- an existing dependency's health status got worse compared to the base branch

The banner gives a one-line tally per condition so reviewers don't need to
read the full report to know whether the PR introduces risk.

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

- Engine source (CLI + web UI for ad-hoc analyses):
  https://github.com/lhein/camel-sanity-analyzer — historical name, generic
  Maven engine.
- Sample workflow that this repo runs against itself:
  [`.github/workflows/test.yml`](.github/workflows/test.yml)
