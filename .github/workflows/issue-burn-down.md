---
description: Quarterly Adoptium issue burn-down report.
on:
  schedule:
    - cron: "0 9 1 */3 *"
  workflow_dispatch:
engine:
  id: copilot
  model: gpt-4o
permissions:
  contents: read
  issues: read
  pull-requests: read
  copilot-requests: write
tools:
  github:
    min-integrity: none
  bash:
    - "gh"
    - "jq"
    - "date"
    - "sleep"
    - "echo"
network:
  allowed:
    - defaults
safe-outputs:
  create-issue:
    title-prefix: "[burndown] "
    labels: [report, burndown]
    close-older-issues: true
---
# Adoptium Issue Burn-down Report
Create a single quarterly issue burn-down report covering the public repositories in the **adoptium** GitHub organization, posted as one GitHub issue in this repository.
## Scope
- Look across the public repositories in the `adoptium` organization, not just this one.
- Cover a custom date range (default: the last full calendar quarter relative to today). Count issues only — exclude pull requests.
## What to include
- For each repository: issues opened in the period, issues closed in the period, a burn-down score (`closed - opened`, where negative means the backlog is growing), and the count of stale open issues (no activity in the last 30 days).
- Up to 5 stale issues per repository, listed with title, link, and last-updated date, ordered most-stale first.
- An org-level list of the top 10 potentially under-served labels — labels where open issues outpace closed issues — with their open/closed counts.
## Tooling and data handling (important — avoid large JSON payloads)
Perform every GitHub read with the `gh` CLI, which is already authenticated in the runner. Do **not** use the github MCP search tools for tallies: they return full issue objects that bloat the context window and previously caused both oversized-JSON failures and malformed tool calls. The `gh api` approach below reads only a single count per query, so issue bodies never enter the context.

Rules:
- Read counts from the search endpoint's `total_count` field with `--jq '.total_count'`. Never download issue lists to obtain a count.
- The authenticated search API is limited to 30 requests per minute. Pace your queries — insert a short `sleep` between calls if you approach that rate — and stop querying a repository as soon as a single count shows it has no activity in the period.
- Run one small command at a time. Keep any intermediate output small and validate it with `jq` before relying on it.
## Process
1. Determine the reporting window. Default to the last full calendar quarter relative to today. Compute `START` and `END` (as `YYYY-MM-DD`, the first and last day of that quarter) and `STALE` (the date 30 days before today):
```bash
STALE=$(date -u -d '30 days ago' +%Y-%m-%d)
```
2. Enumerate active public repositories in `adoptium`, skipping archived repositories and any with issues disabled:
```bash
gh api --paginate 'orgs/adoptium/repos?per_page=100&type=public' \
  --jq '.[] | select(.archived == false and .has_issues == true) | .name'
```
3. For each repository `NAME`, read the three tallies using `total_count` only:
```bash
# issues opened in the period
gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue created:$START..$END" \
  -f per_page=1 --jq '.total_count'

# issues closed in the period
gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue closed:$START..$END" \
  -f per_page=1 --jq '.total_count'

# stale open issues (no activity in the last 30 days)
gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue is:open updated:<$STALE" \
  -f per_page=1 --jq '.total_count'
```
   If all three counts are zero, skip the repository entirely and issue no further queries for it.
4. For each repository that has stale open issues, list up to 5, oldest-activity first, selecting only the fields the report uses:
```bash
gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue is:open updated:<$STALE" \
  -f sort=updated -f order=asc -f per_page=5 \
  --jq '.items[] | {title: .title, url: .html_url, updated: .updated_at}'
```
5. For label statistics, query the open and closed counts per candidate label rather than aggregating full issue data:
```bash
gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue is:open label:\"$LABEL\"" \
  -f per_page=1 --jq '.total_count'

gh api -X GET search/issues \
  -f q="repo:adoptium/$NAME is:issue is:closed label:\"$LABEL\"" \
  -f per_page=1 --jq '.total_count'
```
6. Compute the per-repository burn-down score (`closed - opened`), rank repositories by stale-issue count descending, aggregate label open/closed counts across all repositories, rank by `open - closed` descending, and keep the top 10.
7. Create a single GitHub issue in this repository containing the full report: the period covered, the per-repository summary (sorted by stale count), and the under-served labels list.
## Constraints
- Do not spawn sub-agents or use any task-delegation tool. Perform all analysis yourself in a single sequential pass, processing one repository at a time.
- Read-only: do not close, comment on, or relabel any issue you analyze — this workflow only produces a report.
- Create exactly one issue.
- Skip any repository with zero issues in the period rather than listing it with empty stats.
