# Sync Upstream schedule mitigation

## Context

Linear HPB-4257 was opened after the `sync-upstream.yml` workflow had no observed
`schedule` run on 2026-10-26. Current GitHub evidence shows daily schedule runs
from 2026-10-26 through 2026-11-03, but they started 4h47m–6h35m after the
configured 01:00 UTC time. GitHub documents that scheduled events may be delayed
during high load, especially at the start of an hour, and recommends scheduling
at a different minute.

Runs have also failed since 2026-10-29 when they attempted to create an upstream
sync PR. The failure was `Resource not accessible by integration
(createPullRequest)` because the repository disallowed Actions from creating
pull requests. After explicit approval, the repository setting was enabled;
the default `GITHUB_TOKEN` permissions remain read-only, while the workflow
continues to declare its existing job-specific permissions.

The PR's `composer audit --locked` check also exposed `symfony/mime` v7.4.19
affected by CVE-2026-45070. The lockfile now uses the fixed v7.4.22 release and
the compatible `symfony/polyfill-mbstring` v1.43.3.

After the Composer advisory was fixed, Gitleaks failed because the security
workflow's checkout had only the triggering commit and could not resolve the
requested parent range (`cd48a59^`). The checkout now fetches full history so
the secret scan can inspect the complete PR range.

## Decision

Move the daily cron from `01:00 UTC` to `01:17 UTC`. This follows GitHub's
documented mitigation for top-of-hour schedule congestion. It reduces exposure
to the known high-load minute but does not guarantee exact-time delivery.

Update the minimum dependency set needed to clear the newly surfaced Composer
advisory. Enable the repository Actions PR-creation setting while keeping
`default_workflow_permissions` at `read`.

Set `fetch-depth: 0` for the security-audit checkout, and enable repository
auto-merge after explicit approval while leaving the default token permission
read-only.

## Alternatives considered

1. Keep `01:00 UTC`: no source change and no mitigation for documented queue
   congestion.
2. Move to `01:17 UTC` (selected): minimal schedule change with low operational
   risk.
3. Add an external watchdog or a separate retry workflow: broader operational
   scope and unnecessary for the observed schedule delay.

## Acceptance

- [x] `sync-upstream.yml` schedules daily at `01:17 UTC` (`09:17 UTC+8`).
- [x] `workflow_dispatch`, permissions, and sync behavior remain unchanged.
- [x] A static regression check verifies the cron expression and the diff passes
  whitespace validation.
- [x] `composer audit --locked` passes with the fixed Symfony MIME version.
- [x] The Actions PR-creation setting is enabled and the default token permission
  remains `read`.
- [x] The security-audit checkout fetches full history for Gitleaks.
- [x] Repository auto-merge is enabled after explicit approval.
- [x] Record that scheduled execution remains best-effort.
