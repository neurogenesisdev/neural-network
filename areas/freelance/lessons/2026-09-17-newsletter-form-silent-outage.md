---
type: lesson
date: 2026-09-17
severity: medium
client:
changed: standards/delivery (launch checklist + new "after every pipeline change" rule)
tags: [lesson, delivery, deploy]
---

# A deploy-pipeline change silently removed the newsletter form

> [!NOTE] Summary
> When neurogenesis.dev moved to GitHub Actions deploys, the Kit form id
> (a build-time config value) never reached the CI build. The site built and
> deployed green; the signup form was simply gone. Nobody noticed until the
> Architect's site-ops pass on 2026-09-17. Window: from the deploy.yml merge
> until 2026-09-17 — check the merge date for the exact length. Also found the
> same day: both GitHub tokens behind the blog pipeline had expired in spring,
> which was the "daily publish fails" mystery.

## What happened

- Build-time secret/config (Kit form id) existed locally but was not set in the Actions environment. The build did not fail; it rendered the page without the form.
- No post-deploy check existed. "Green pipeline" was treated as "site works".
- Expiring tokens had no owner or reminder; failures were noticed as noise, not as a broken pipeline.

## Rule extracted

1. **After every change to how a site is built or deployed, run the launch checklist's form test on the live URL** — not only at launch. A green build proves the build ran, not that the page has everything.
2. Every build-time config value and secret a site needs is listed in the project's runbook (name, where it is set, who owns it). The Architect checks that list when a pipeline changes.
3. Every token or credential with an expiry gets the expiry date in its KeePass entry and a calendar reminder 2 weeks before. Failing silently for months is the failure mode.

## Why it matters for clients

On Chantal's site the same gap would be a contact or booking form silently dead for weeks — the one thing the site is for. For a paying client that is a refund conversation.

## Where it went

- standards/delivery: launch checklist item reworded; new section "After pipeline changes" (Mentor, 2026-09-19).
- Runbook/config list and token-expiry reminders: Architect (technical side).
