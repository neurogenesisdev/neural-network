---
type: decision
id: FDR-004
aliases: [FDR-004]
status: accepted
impact: medium
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-13
updated: 2026-09-20
tags: [decision, marketing, business]
---

# FDR-004 — Business identity lives on neurogenesis.dev; email via Cloudflare Email Routing

> [!NOTE] Summary
> The practice gets no second domain or site. neurogenesis.dev carries a
> business section; the business address is `luiswg@neurogenesis.dev`,
> received via Cloudflare Email Routing into Luis's hub Gmail and sent via
> Gmail "Send mail as". Revisit at 3 published case studies.

## Context

Zero paying clients, 10–20 h/week, Chantal pro bono in progress. Email inboxes
(three Gmail accounts, one Proton) are a graveyard; the Architect's stage-2
plan needs one primary address before the n8n classifier (plan 2.6) can be
built. A second brand would mean a second site, second Impressum, second
thing to keep alive.

## Decision

On 2026-09-13 Luis decided:

1. **Site:** extend neurogenesis.dev with a business section (services, work,
   contact). No second domain, no separate site.
2. **Business email:** `luiswg@neurogenesis.dev`. Cloudflare Email Routing
   forwards it into the personal hub Gmail; replies go out through Gmail
   "Send mail as" (smtp.gmail.com, app password, SPF record includes Google).
   Cost: 0 €.
3. **Personal consolidation:** one Gmail account is the hub; the other two
   auto-forward into it and are no longer read. KeePass entries move to the
   hub opportunistically, on next login per service — not as a project.
4. **Proton:** stays as a manual, low-volume mailbox. Not automated (no API;
   Proton Bridge on the VPS rejected as a second principal for no gain).
5. **Priority:** making the inboxes usable (unsubscribe, archive, one place to
   look) comes before any automation on top of them.

## Alternatives considered

1. **Google Workspace Business Starter on the domain (~7–8 €/month)** — real
   separation of business and personal mail, DKIM on own domain, smaller
   OAuth blast radius for n8n. Rejected for now: recurring cost at zero
   revenue and a fourth account to manage; the first candidate when revisiting.
2. **Plain gmail.com address for business** — free, but reads as a hobby on
   an invoice or Impressum. Rejected.
3. **Second domain + separate business site** — clean brand separation.
   Rejected: doubles maintenance and legal surface with nothing to show yet.

## Consequences

- Positive: nothing new to host or pay for; the Architect's classifier plan
  is unchanged (hub Gmail has the Gmail API); business mail is a first-class
  address from day one — use it for every business login going forward
  (Cloudflare, GitHub, IHK, Finanzamt/ELSTER).
- Negative / cost: business and personal mail share one mailbox, so the
  n8n OAuth grant on netcup-vps covers personal mail too (accepted, see
  infra-docs ADR-0034). DKIM signs for gmail.com, not the domain — slightly
  weaker deliverability with strict receivers. Any LLM step in the
  classifier sees client mail: keep client-facing mail out of providers
  without a DPA (no DeepSeek for that lane) — Architect to decide when
  building.
- The Impressum on neurogenesis.dev must list a real contact address and
  this email once the business section goes live.

## Status (Architect, 2026-09-15)

Built and measured 2026-09-14/15: Cloudflare Email Routing → hub Gmail, Gmail
Send-as, DMARC `p=none`; Proton's stale custom-domain records removed; business
section live (/impressum, /datenschutz, /contact), /services gated per
[[../business/web-presence|web-presence]]. Known weakness confirmed: Proton
flags Send-as mail as failing the domain's authentication (DKIM signs
`gmail.com`). Runbooks: `infra-docs/shared/runbooks/`.

## Status (Architect, 2026-09-20)

Weakness closed: Gmail Send-as now relays through **Brevo** (EU, free tier 300/day),
which DKIM-signs as `neurogenesis.dev`; measured at Proton `dkim=pass`, `dmarc=pass`,
verified-sender padlock. DMARC policy raised to `p=quarantine`. Cost still 0 €. The
FDR's negative consequence "DKIM signs for gmail.com" no longer applies. Cosmetic
leftover: Brevo adds a `List-Unsubscribe` header (Proton shows a mailing-list
banner) — backlog. Details: `infra-docs/shared/runbooks/neurogenesis-dev-dns-and-email.md` §4.

## Revisit when

Three case studies published, or the first client on a retainer holding
customer data (AVV) — then Workspace or equivalent, and possibly a name domain.
