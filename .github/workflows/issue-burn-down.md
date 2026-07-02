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

## Process

1. Enumerate the public repositories in the `adoptium` organization.
2. For each repository, gather issues opened and closed during the reporting period, identify stale open issues, and tally label counts (open vs. closed) for issues opened in the period.
3. Rank repositories by stale issue count, descending.
4. Aggregate label stats across all repositories and rank by `open - closed`, descending, keeping the top 10.
5. Create a single GitHub issue in this repository containing the full report: period covered, per-repo summary (sorted by stale count), and the under-served labels list.

## Constraints

- Read-only: do not close, comment on, or relabel any issue you analyze — this workflow only produces a report.
- Create exactly one issue.
- Skip any repository with zero issues in the period rather than listing it with empty stats.
