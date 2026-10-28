---
type: standard
status: active
owner: mentor
created: 2026-09-15
updated: 2026-09-15
tags: [marketing]
---

# Socials — accounts, handles, setup order

> [!NOTE] Summary
> Which platforms get a business presence, under which handle and login, in
> which order. Handle: `neurogenesisdev`. Login for every new business account:
> `luiswg@neurogenesis.dev` (FDR-004), 2FA + recovery codes in KeePassXC.
> Per-platform bio copy comes from [[positioning]]. Checklists per platform
> are filled in as each one is set up.

## Rules

- New accounts use `luiswg@neurogenesis.dev`. Exception: Google-owned properties (YouTube, Google Business Profile, OAuth consent) live on the hub Gmail — the business address is a forwarding alias, not a Google account. Existing business-relevant accounts on other addresses (Kit → Proton, Cal.com → personal Gmail) move to it opportunistically, on next login.
- One account per platform. Existing personal accounts (Instagram @luiswg, X, LinkedIn — all inactive, < 100 followers, luis 2026-09-15) are *renamed and repurposed*, not duplicated: the few followers are the warm network, and two accounts per platform is double the upkeep for zero audience. Never a second LinkedIn profile (ToS; the personal profile *is* the business asset).
- Claim the handle even where the platform is not used yet — handles cannot be recovered later.
- **Rename vs. new account.** Rename where the account is a shopfront: reach, no recreational use, no history worth keeping (Instagram, LinkedIn, GitHub, Facebook, Substack). Create a new account where it is a living room: friends, servers, activity history, private use (Discord, Reddit, Twitch, Snapchat). Ask first whether the business needs that platform at all — if not, the business account is handle protection only and stays empty.
- Every account: KeePassXC entry named by domain at creation, login email noted, 2FA (authenticator) with recovery codes in the entry. Luis is retrofitting 2FA on all accounts that lacked it (2026-09-15).
- Kit: login and default sender moved to the business email (2026-09-15).
- Notifications: email digests off, push off, except DMs/mentions on the platforms clients use (LinkedIn, Instagram).
- Every bio: first sentence of the one-liner + link to neurogenesis.dev. No "AI" in the first line.

## Platforms

| # | Platform | Account | Handle / URL | Status 2026-09-15 | Why |
|---|----------|---------|--------------|-------------------|-----|
| 1 | Instagram | existing (was @luiswg), renamed | @neurogenesisdev | done 2026-09-15 — renamed; Business account, category Entrepreneur; business email added in Accounts Center and shown as public contact; bio/link pending | Creator clients live here; also where Chantal's audience is |
| 2 | LinkedIn | existing personal profile; business email added + primary; website + GitHub links added (2026-09-15) | www.linkedin.com/in/neurogenesis.dev | headline/About/contact visibility/2FA pending | Small-business clients and the warm network check here first |
| 3 | GitHub | existing, **renamed `Luis-Gordon` → `neurogenesisdev` on 2026-09-15** (luis) | github.com/neurogenesisdev | primary email = business address, backup = Proton, public email set, URL + social links set; Name field → "Luis Gordon"; README + pinned repos pending. **Architect: check git remotes (laptop, VPS), Cloudflare Pages repo connection, Actions/workflow references, any hard-coded `Luis-Gordon` URLs (site, vault, README)** | Proof of work |
| 4 | TikTok | new, business email | @neurogenesisdev | created 2026-09-15; bio/link/business switch pending | Second prospect and creator clients are here; content later |
| 5 | YouTube | existing channel on the hub Gmail | @neurogenesisdev (10 subscribers) | done — already set up earlier; display name "neurogenesis", green "n" avatar; rename to Luis Gordon + bio with the copy pass | Demo videos later, after the first case study |
| 6 | X | existing @neurogenesisdev (created March 2026 for Memogenesis, Proton login) | @neurogenesisdev | access regained 2026-09-15, credentials in KeePass, 2FA on; email change blocked by X's post-reset restriction period — retry in ~2 weeks; to do: name → Luis Gordon, bio from [[positioning]], drop Memogenesis avatar/banner. Old personal X account: leave or delete, never run two | Low priority; dev/AI audience, not clients |
| 7 | Facebook | existing personal; business email added in Meta Accounts Center, only the .dev address public (2026-09-15) | — | — | No business page. Only if a client wants FB/IG ads run; a Meta Business account can be added then |
| 8 | Snapchat | existing personal | renamed 2026-09-15 | Private; not a business channel — revert or leave, irrelevant either way | — |
| 9 | Threads | via the Instagram account | @neurogenesisdev | automatic with the IG rename | Comes free with Instagram |
| 10 | Reddit | new, business email | u/neurogenesisdev | claimed 2026-09-15; never post as the business (Reddit punishes self-promotion) | Handle protection |
| 11 | Twitch | new, business email | neurogenesisdev | claimed 2026-09-15 | Handle protection; the second prospect's circle |
| 12 | Bluesky | new, business email | @neurogenesisdev.bsky.social | claimed 2026-09-15. **To do: switch the handle to the domain (@neurogenesis.dev) via a DNS TXT record in Cloudflare** — Architect, 5 min | dev/AI audience; the one platform where the domain becomes the handle |
| 13 | Substack | existing | neurogenesisdev | renamed 2026-09-15; login email → .dev pending | Publishing = shopfront; only if blog posts get republished |
| 14 | Discord | existing personal | — | Personal handle was renamed in error — **revert**. Park the business name in a separate empty account only if wanted (usernames are global) | Private platform; business has no use for it |
| — | Mastodon | skip | — | Federated — no global handle exists to claim | Not a client channel |

## Setup order

1. Handles: Instagram, TikTok, YouTube done 2026-09-15; X recovered, housekeeping pending (rate-limited, retry in 1–2 days). Log each in KeePassXC with the recovery codes. Existing accounts keep their current login email; switch to the business email on the next login, not as a task.
2. LinkedIn: headline + About from [[positioning]], custom URL, contact info shows neurogenesis.dev and the business email, notifications trimmed.
3. GitHub: profile README (positioning EN paragraph, 3 pinned repos: neurogenesis-dev, the SaaS if public, one tool), bio link to the site.
4. Instagram full setup: bio, link, business/creator account type, highlights later.
5. Everything else stays claimed-and-empty until there is a case study to post.

## Per-platform checklists

Filled in when each platform is set up (handle, bio text used, links, settings changed, notification choices, date).
