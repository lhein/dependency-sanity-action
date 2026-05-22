# dependency-sanity-action

GitHub Action for Maven and Gradle projects. On every PR that touches a
`pom.xml`, `build.gradle` or `build.gradle.kts`, it analyzes the project's
dependencies, diffs them against the base branch, and posts a Markdown health
report as a PR comment.

For each declared dependency the report covers: latest available version
(with patch/minor/major classification), GitHub activity (last commit,
stars, archived state), OpenSSF Scorecard, license, and CVEs from OSV.
Newly added critical dependencies, new CVEs, or health regressions are
called out in a banner at the top of the comment so reviewers can spot
them without scrolling.

The analysis engine is the standalone CLI from
[camel-sanity-analyzer](https://github.com/lhein/camel-sanity-analyzer)
— that repo is the engine source despite its historical name. The
analyzer itself works on any Maven or Gradle project, not just Apache
Camel. Gradle resolution goes through the official Gradle Tooling API,
so both the Groovy DSL (`build.gradle`) and the Kotlin DSL
(`build.gradle.kts`) are fully supported.

## Quick start

Add the following workflow to your repo at `.github/workflows/dependency-sanity.yml`:

```yaml
name: Dependency Sanity

on:
  pull_request:
    paths:
      - '**/pom.xml'
      - '**/build.gradle'
      - '**/build.gradle.kts'

permissions:
  pull-requests: write       # required so the bot can post the PR comment
  contents: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0     # required: diff mode needs full git history
      - uses: lhein/dependency-sanity-action@v0.2.0
```

That's it — by default the action auto-detects which build files to
analyze (see [Auto-detection](#auto-detection) below). For most projects
you don't need to set `build-files` at all.

Two non-obvious requirements:

- `permissions: pull-requests: write` — without it the action can't post the
  comment and the run fails with a 403.
- `fetch-depth: 0` on `actions/checkout` — diff mode either runs
  `git show <base-sha>:pom.xml` (Maven) or sets up a detached `git worktree`
  at the base SHA (Gradle, because resolution needs the full project tree).
  Both paths require the full git history. With `fetch-depth: 1` (the
  default), the action falls back to a full scan.

## Inputs

| Name | Description | Default |
|---|---|---|
| `build-files` | Build file(s) to analyze. `auto` (default) discovers them automatically — see below. To override, pass a comma-separated list of paths/globs, e.g. `pom.xml,modules/*/pom.xml` or `app/build.gradle.kts,lib/build.gradle`. | `auto` |
| `pom-path` | Deprecated alias of `build-files`. Still honored — if set to anything other than `auto`, overrides `build-files`. Will be removed in a future release. | `auto` |
| `mode` | `diff` (PR-only) or `full` (entire tree). | `diff` |
| `fail-on` | `none` / `critical` / `warning` / `cve` — the action exits non-zero when the condition is met. | `none` |
| `include-transitive-test` | Follow transitive test-scope deps. Maven-only — Gradle resolution always reflects the runtime classpath that Gradle itself produces. | `false` |
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
> ### 🚨 Breaking changes detected
>
> - **1** newly added dependency is `CRITICAL`
> - **7** new CVEs introduced by added dependencies
>
> See sections below for details.
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

## Auto-detection

With the default `build-files: auto`, the action figures out which build
files to analyze based on the run context:

- **In diff mode (PR runs)**: only build files touched by this PR
  (`git diff --name-only base..head -- '*pom.xml' '*build.gradle' '*build.gradle.kts'`).
  If no build file changed, the workflow logs that and exits cleanly
  without posting a comment.
- **In full mode (or non-PR runs)**: every `pom.xml`, `build.gradle` and
  `build.gradle.kts` in the working tree, excluding `target/`, `build/`,
  `.git/`, and `node_modules/`.

Override by passing an explicit list/glob:

```yaml
- uses: lhein/dependency-sanity-action@v0.2.0
  with:
    build-files: 'pom.xml, modules/*/build.gradle'
```

## Modes

- **diff** (default): compares the head build file against its base-branch version and only
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

For a multi-module Maven reactor or a multi-project Gradle build the
default `build-files: auto` already does the right thing — in diff mode
it runs the analyzer on the modules whose build file changed and
aggregates the reports into a single comment. Pass an explicit pattern
only when you want to scope narrower or wider than that.

For Gradle specifically, the analyzer is invoked once per `build.gradle`
or `build.gradle.kts` — each invocation resolves that project's runtime
classpath via the Gradle Tooling API. Sub-projects with their own build
file appear as separate entries in the aggregated comment.

## How it works

1. Downloads the analyzer CLI jar from its releases.
2. Expands `build-files` to a concrete list of build files in the working tree.
3. For each file in diff mode:
   - **Maven**: extracts the base-branch version of that `pom.xml` via `git show`.
   - **Gradle**: provisions a detached `git worktree` at the base SHA so the
     Tooling API can resolve the full base-state project (worktree is reused
     across all Gradle files in the run, then torn down on exit).
4. Runs the CLI per file (Maven POMs via Maven Resolver, Gradle projects via
   the Gradle Tooling API), then aggregates the markdown into one comment.
5. Posts (or updates) a single PR comment per workflow.
6. Honors the `fail-on` policy via the CLI's worst exit code across files.

### Gradle wrapper

The Tooling API resolves Gradle's distribution from the `gradle/wrapper`
directory of the target project. If the target repo doesn't ship a
wrapper, the Tooling API downloads a default distribution on first use
(adds ~30 s to the first run, cached afterwards by the GitHub runner).

## See also

- Engine source (CLI + web UI for ad-hoc analyses):
  https://github.com/lhein/camel-sanity-analyzer — historical name, generic
  Maven engine.
- Sample workflow that this repo runs against itself:
  [`.github/workflows/test.yml`](.github/workflows/test.yml)
