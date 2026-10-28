---
type: standard
status: active
created: 2026-09-09
updated: 2026-09-19
tags: [standard, delivery, ownership, hosting]
---

# Delivery

> [!NOTE] Summary
> Scope before work, ownership in the client's name, no personal hosting,
> launch checklist, formal handover.

## Before work starts
- Anything with a lead time (consent, appointments, registrations) is sent the day it is written. Presentability never gates paperwork. ([[../lessons/2026-09-15-admin-blockers-first]])
- A rename of a handle other things depend on (GitHub user, domain, email) is announced to the Architect before it happens, not after.
- No work without a written scope and an explicit out-of-scope list the client has confirmed.
- Every project has a target launch date and a milestone table in the client note.
- Content collection starts in week 1 with a deadline. Build with placeholders; never wait for content.

## Ownership
- Domain, hosting, and every third-party account are in the client's name, created with the client's email, by the client, on the client's device.
- Luis is a member/collaborator where the platform allows it. Where it does not, Luis holds the credential during setup only and hands it over at launch.
- The client's password manager holds the credentials. Luis's never holds a client's primary credential after handover.
- Never bind 2FA on a client account to Luis's phone.

## Hosting
- Do not host client sites or services on personal infrastructure. Default: static hosting inside the client's own account, e.g. Cloudflare Pages. ([[../decisions/fdr-001-no-hosting-on-personal-infra|FDR-001]])
- Any exception is a paid retainer with a written maintenance agreement.

## Launch checklist
- Impressum and Datenschutzerklärung present and linked from every page.
- Cookie consent present if the site has embeds (YouTube, Instagram, Maps) or tracking.
- Forms tested end to end on the live URL — the submission arrives at the client. A green build is not a working page.
- Mobile check on a real phone.
- Domain connected; preview URL retired or redirected.

## After every build or deploy change
- Re-run the form test on the live URL the same day. Build-time config (form ids, keys) that never reaches CI removes features silently. ([[../lessons/2026-09-17-newsletter-form-silent-outage]])
- Every build-time config value and secret is listed in the site's runbook with its owner and expiry; tokens that expire get a reminder 2 weeks before.

## Handover
- Client rotates every credential; Luis confirms in writing that his access is removed or reduced to member.
- Client receives a one-page "how to update your site" note.
- The maintenance option is stated in writing, even if it is "none — here is who to ask".
- Testimonial request goes out within one week of launch.
