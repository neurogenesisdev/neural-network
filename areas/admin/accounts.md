---
type: area
status: active
created: 2026-09-29
updated: 2026-09-29
tags: [accounts]
---

# Accounts — email consolidation map

> [!NOTE] Summary
> Four inboxes hold official and financial mail. Goal: one primary address per
> purpose, forwarding from the rest, so triage automation sees everything.
> Fill the table by searching each inbox for `Rechnung OR Bescheid OR Konto`.

## Inboxes

| Address | Role today | Automation | Target |
|---------|-----------|------------|--------|
| luiswgordon@gmail.com | primary; n8n classifier runs here (luis) | yes | primary — keep |
| luiswgordon@protonmail.com | possibly holds FM Safe & Protect / older employer mail (luis) | none — no API | forward to primary Gmail with label, or Proton Bridge on a machine that stays on |
| luiswillg@gmail.com | PayPal (luis) | none | forward to primary; move PayPal login later |
| luiswilliamgordon@gmail.com | unknown (luis) | none | audit, then forward or close |

## Services by address

| Service | Address | Move to primary? | Done |
|---------|---------|------------------|------|
| PayPal | luiswillg | yes, after forwarding works | [ ] |
| Sparkasse Heidelberg | ? | | [ ] |
| Service Credit Union | ? | | [ ] |
| Telekom | luiswgordon (invoices seen) | — | [x] |
| AOK | ? | | [ ] |
| Jobcenter / jobcenter.digital | ? | | [ ] |
| ELSTER | ? | | [ ] |

## Next action
- [ ] Set up Gmail forwarding from luiswillg and luiswilliamgordon → luiswgordon, filter → label `Konten/<source>`
- [ ] Proton: enable forwarding to Gmail (Settings → Auto-forward; requires paid plan — otherwise manual check weekly)
- [ ] Search Proton for FM Safe & Protect / Lohnabrechnung → feed [[cases/fm-safe-protect-insolvenz]]
