---
title: "neurogenesis.dev — DNS state and business email (Cloudflare Email Routing + Gmail Send-as via Brevo)"
type: runbook
provenance: measured
device: shared
stale: false
last_verified: 2026-09-20
commit: pending
created: 2026-09-14
updated: 2026-09-20
tags: [runbook, dns, email, cloudflare, gmail, brevo, dmarc, neurogenesis.dev, freelance]
---

# neurogenesis.dev — DNS and business email

> [!NOTE] Summary
> Decision: `freelance-practice` FDR-004 (2026-09-13) — no second domain, no
> Google Workspace; `luiswg@neurogenesis.dev` **receives** through Cloudflare
> Email Routing into the hub Gmail (`luiswgordon@gmail.com`) and **sends** as a
> Gmail alias through **Brevo's SMTP relay** (since 2026-09-20; was `smtp.gmail.com`).
> Built and measured 2026-09-14; DMARC alignment fixed 2026-09-20 (§4), policy
> now `p=quarantine`.

The domain is registered at Cloudflare Registrar and its zone is on Cloudflare
DNS (`olga`/`pablo.ns.cloudflare.com`). Zone exports taken during the change are
the source for §1 (`neurogenesis.dev.txt` … `neurogenesis.dev3.txt`, not kept in
the vault — the DKIM public key and the Proton verification token in them are
harmless but noise).

## 1. Zone after 2026-09-14 (measured, Cloudflare export 11:14)

| Name | Type | Content | Note |
|---|---|---|---|
| `@` | CNAME | `neurogenesis-blog.pages.dev`, proxied | the site (Cloudflare Pages project `neurogenesis-blog`) |
| `www` | CNAME | `neurogenesis-blog.pages.dev`, proxied | **was a dead Vercel CNAME** from the discarded first site — `curl -sI https://www…` returned nothing; fixed + added as Pages custom domain |
| `@` | MX ×3 | `route1/2/3.mx.cloudflare.net` (20/3/7) | Email Routing |
| `cf2024-1._domainkey` | TXT | `v=DKIM1; …` | Cloudflare's DKIM for forwarded mail |
| `@` | TXT | `v=spf1 include:_spf.mx.cloudflare.net include:_spf.google.com ~all` | Cloudflare's SPF + Google added for Send-as |
| `_dmarc` | TXT | `v=DMARC1; p=quarantine; rua=mailto:rua@dmarc.brevo.com` | since 2026-09-20 — see §4. (Before: the record had degraded to a bare `p=none` without `v=DMARC1;`, i.e. no valid DMARC at all — found and fixed the same day) |
| `@` | TXT | `brevo-code:…` | Brevo domain verification, 2026-09-20 |
| `brevo1._domainkey`, `brevo2._domainkey` | CNAME | `b1`/`b2.neurogenesis-dev.dkim.brevo.com`, DNS only | Brevo DKIM (key rotation on Brevo's side, nothing to do here) |
| `@` | CAA ×7 | digicert, pki.goog, letsencrypt (issue + issuewild), iodef | Cloudflare's set for Universal SSL; the dashboard's warning triangles are cosmetic |
| `@` | TXT | `google-site-verification=…` | Search Console domain property, added 2026-09-15 for the OAuth consent-screen branding check; keep |

Changed 2026-09-15: `@` and `www` now CNAME → `neurogenesis-dev.pages.dev`, the
new Pages project `neurogenesis-dev` that replaced `neurogenesis-blog`
(inherited-unverified — from [[neurogenesis-dev-site]] §1; no zone export
taken since the one above).

Removed 2026-09-14: Proton custom-domain set (2 MX `mail`/`mailsec.protonmail.ch`,
3 `protonmail*._domainkey` CNAMEs, `protonmail-verification=` TXT, Proton SPF,
`_dmarc p=quarantine`) — the domain was still attached to the Proton account from
an earlier attempt; and two stale A records `n8n.` / `alerts.` → `128.140.12.216`
(the decommissioned Hetzner `hive`).

## 2. Receive — Cloudflare Email Routing

Dashboard → neurogenesis.dev → Email → Email Routing. *Get started* adds the
MX/SPF/DKIM records itself but **refuses while any foreign MX exists** ("Existing
non-Cloudflare MX records conflict") — delete the old MX first.

- **Destination address** = the real mailbox mail is delivered *to*:
  `luiswgordon@gmail.com`, verified by the link Cloudflare mails there. (Trap
  hit: entering the *new* address as destination — its verification mail can
  never arrive.)
- **Routing rule**: custom address `luiswg` → *Send to an email* → the hub.
  Catch-all: **Drop** (forwarding everything invites spam; add names by rule).
- Cost 0. Limits: forwarding only, no mailbox, no SMTP submission.

Measured 13:28: mail from `luiswgordon@protonmail.com` → hub Inbox within a
minute; Gmail header `mailed-by: neurogenesis.dev`, TLS. First attempt from the
Proton **app** was refused as "not a proper address" — Proton still had the
domain registered and treated the address as an internal one that does not
exist. Fix: Proton → Settings → Domain names → remove `neurogenesis.dev`.

## 3. Send — Gmail "Send mail as"

Hub account, 2-step verification on. **App password** (page is hidden from the
menus: `myaccount.google.com/apppasswords`, not *Third-party apps & services*)
named `gmail send-as neurogenesis` → KeePassXC.

Gmail → Settings → Accounts and Import → *Send mail as* → Add: name `Luis
Gordon`, `luiswg@neurogenesis.dev`, **Treat as an alias** ✓. SMTP: **overwrite
Gmail's prefill** — it guesses the MX host (`route2.mx.cloudflare.net`), which
cannot send. Use `smtp.gmail.com`, `587`, TLS, username = the full hub address,
password = the app password. Confirmation code arrives at the new address via §2.
Then *When replying*: **Reply from the same address the message was sent to**;
hub stays the compose default for now.

Measured 13:37: "The Gmail user may now send mail as luiswg@neurogenesis.dev";
test mails from the alias reached Proton and the hub, From shows the alias.

## 4. DMARC alignment — fixed 2026-09-20 with a Brevo relay

**Built 2026-09-20 (~40 min, €0):** Brevo account on the business address (EU
provider; free tier 300 mails/day). Senders, Domains & Dedicated IPs → Domains →
add `neurogenesis.dev` → records from §1 (Brevo code TXT, two DKIM CNAMEs, DMARC) →
authenticated; sender `Luis Gordon <luiswg@neurogenesis.dev>` verified. SMTP & API →
SMTP key `gmail-send-as` (expires 2027-09-20; KeePassXC entry `Brevo`, attribute).
Gmail *Send mail as* → edit → SMTP server **`smtp-relay.brevo.com`, 587, TLS**,
username = Brevo SMTP login (`…@smtp-brevo.com`, shown on the SMTP page), password =
the key.

Traps hit: (1) Brevo's SMTP keys are **IP-restricted by default** — Gmail's servers
returned `525 5.7.1 Unauthorized IP address`; disabled the authorized-IP restriction
in Brevo (Security → Authorized IPs). Trade-off: the key alone authenticates from
anywhere; bounded by the daily cap and revocation. (2) Brevo adds `List-Unsubscribe`
headers to relayed mail and DKIM-signs them; Proton shows a "this message is from a
mailing list" banner. Cosmetic; no toggle found in the transactional settings on
2026-09-20 — backlog. (3) Brevo's current domain setup needs **no SPF include** (it
uses its own return-path domain `hb.d.sender-sib.com`); alignment comes from DKIM.

Measured 2026-09-20 22:36 (Proton headers): `dkim=pass header.d=neurogenesis.dev
(s=brevo2)`, `dmarc=pass`, `spf=pass smtp.mailfrom=hb.d.sender-sib.com`; Proton shows
the verified-sender padlock, no authentication banner. Then `_dmarc` →
`p=quarantine`. Consequence: any mail for the domain that is not DKIM-aligned (e.g.
Send-as falling back to `smtp.gmail.com`) will be junked by receivers — intended.

Credential row (ADR-0034 pattern, **not on netcup-vps**): Brevo SMTP key
`gmail-send-as` — lives in the hub Gmail's Send-as config and KeePassXC; scope: send
as `@neurogenesis.dev` through Brevo, 300/day; revoke at Brevo → SMTP & API → delete
key (Send-as then fails loudly on the next send).

### 4a. The original weakness (measured 2026-09-14, historical)

Gmail signs alias mail with `d=gmail.com` and uses the gmail.com envelope
sender, so SPF and DKIM pass — for **gmail.com** — and DMARC for
`neurogenesis.dev` has nothing aligned to it: **FAIL**. At `p=none` nothing is
rejected, but Proton Mail shows a "failed its domain's authentication" banner on
every such mail (seen on the test), and some clients show "via gmail.com".
Unacceptable for client-facing mail long-term.

Resolved by the Brevo relay above (2026-09-20).

## 5. Operate

- New address on the domain: Email Routing → Routing rules → add; for sending, also a
  Brevo sender + a Gmail Send-as entry with the same relay settings.
- Brevo SMTP key expires 2027-09-20: regenerate, update Gmail Send-as + KeePassXC.
- Hub Gmail is the single point of failure for both directions; its recovery
  options are the hub's own (see the account entry in KeePassXC).
- The n8n Gmail classifier (`netcup-vps/runbooks/automations-stage-2.md` §2.9)
  labels forwarded business mail like any other — `Topics/Work` is where client
  mail should land; adjust the prompt if it doesn't.
- Zone changes: export the zone (DNS → Export) before and after, as done here;
  keep the exports out of the vault.
- Site operations (repo, deploy, legal pages): [[neurogenesis-dev-site]].

## Related

- `freelance-practice` FDR-004 — the decision (business address, no Workspace)
- [[../../netcup-vps/runbooks/automations-stage-2|automations stage 2]] §2.9 — the classifier the mail lands in
- [[../../netcup-vps/decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — credential table pattern for the relay key
