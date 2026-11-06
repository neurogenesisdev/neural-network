---
title: "Browser lanes for the coding harnesses: headless VPS Chromium, laptop Claude in Chrome, laptop debugging-port tunnel"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-11-06
commit: pending
created: 2026-11-06
updated: 2026-11-06
tags: [runbook, chromium, mcp, claude-code, codex, design, ssh]
---

# Browser lanes: headless VPS, laptop extension, laptop tunnel

> [!NOTE] Summary
> Three ways an agent on this VPS gets a browser, each for a different job.
> **Lane 1 (default):** headless system Chromium on the VPS through the user-scope
> MCP servers `chrome-devtools` and `dembrandt`. Deterministic extraction of CSS,
> tokens, keyframes and screenshots; needs nothing from the laptop. **Lane 2:**
> Claude in Chrome on the laptop, driven from the VPS session with `claude --chrome`,
> for sites behind Luis's logins and anything interactive. **Lane 3:** the laptop's
> own Chromium with a remote-debugging port, reached over an SSH reverse tunnel by
> the user-scope MCP server `laptop-chrome` (disabled by default). For real-GPU
> rendering, system fonts and inspector-level access to logged-in references.
> Lanes 1 and 2 are `measured` 2026-11-06; lane 3 is configured but `inferred` until
> first run from the laptop. Background: the research report at
> `/home/dev/code/reports/AI tools for aesthetic web UI.md` (2026-11-04).

Lanes: coding = `dev`. The only admin step, the font packages in §Fonts, is done.
Supersedes the "chrome-devtools-mcp fails with Missing X server … uninstalled" note in
[[coding-rig-stage-1|coding-rig-stage-1 §Headless Chromium]] (2026-09-24): the failure
was the server launching headful; `--headless` in the server args fixes it. The
`/opt/google/chrome/chrome` symlink from that note no longer exists after the 2026-10
rebuild and is not needed (`measured` 2026-11-06).

## Which lane

| Lane | Use it for | Needs the laptop | Deterministic |
|------|-----------|------------------|---------------|
| 1 `chrome-devtools` + `dembrandt` (headless VPS) | URL → tokens / DESIGN.md; authored CSS, cascade and `@keyframes` for one element; snapshots and screenshots of public sites and of the project's own dev server; CI and hooks | No | Yes |
| 2 Claude in Chrome (laptop extension) | Web apps with Luis's sessions (Pen.dev, html.to.design, Claude Design, Mobbin, Figma); flows that hit logins or CAPTCHAs; Luis watching and intervening | Yes, Chrome open and extension signed in | No, it acts like a user |
| 3 `laptop-chrome` (debugging-port tunnel) | WebGL/shader backgrounds and performance traces on real GPU; screenshots with the fonts a visitor sees; authored CSS and keyframes from a reference behind a login | Yes, tunnel open | Yes |

Rule of thumb: start in lane 1. Move to lane 2 only when a login or a human
decision is in the way. Move to lane 3 only when lane 1's rendering is misleading
(GPU, fonts) or lane 2 cannot see under the hood.

## Lane 1 — headless system Chromium on the VPS — `measured` 2026-11-06

**Binary.** `/usr/bin/chromium` is the Debian launcher wrapper around
`Chromium 156.0.8078.73 built on Ubuntu 26.04.1 LTS` (`/usr/bin/chromium --version`).
`CHROMIUM_FLAGS` is no longer set in `~dev/.zshenv` after the rebuild, so every
flag is passed explicitly in the server args.

**Servers** (user scope, stored in `~dev/.claude.json` under `mcpServers`; nothing
in any repo). Written with `claude mcp add --scope user --transport stdio …`:

```
chrome-devtools  npx -y chrome-devtools-mcp@latest --headless --isolated
                 --executablePath /usr/bin/chromium
                 --chromeArg=--disable-dev-shm-usage
dembrandt        npx -y --package dembrandt dembrandt-mcp
                 env DEMBRANDT_NO_HEADED_FALLBACK=1
```

- `--isolated`: temporary profile, deleted on close, so no profile lock survives a
  crashed run. `--no-sandbox` from the 2026-09-24 note is **dropped**: the admin
  confirmed the Chromium sandbox works for `dev` on this host, and the screenshot run
  below succeeds without it (`measured` 2026-11-06). `--disable-dev-shm-usage` is
  kept; whether the VPS's `/dev/shm` is small enough to need it is `inferred`.
- Dembrandt has no executable-path option. It drives Playwright's own Chromium from
  `~dev/.cache/ms-playwright/` (revision 1243, "Chrome Headless Shell 153.0.8010.12",
  fetched by `npx dembrandt install-browser`, 114 MiB, 2026-11-06). Its only hook for
  another browser is `BROWSER_CDP_ENDPOINT`, so it does not use `/usr/bin/chromium`.
  `DEMBRANDT_NO_HEADED_FALLBACK=1` stops its retry "with visible browser" on a
  navigation timeout, which has no display to open here (`measured` in the CLI source;
  that the MCP entry point reads the same variable is `inferred`).
- chrome-devtools-mcp 1.12.0 restricts file-writing tools (`take_screenshot
  filePath`, `take_snapshot filePath`) to the OS temp directory when the client does not
  negotiate MCP roots (server stderr, `measured` with a bare stdio client). Whether
  Claude Code negotiates roots is `inferred`; if a screenshot to a repo path is refused,
  add `--allow-unrestricted-paths` or a `--filesystemRoot <dir>` to the args.

**Verification record, 2026-11-06, bare stdio client against the same command:**
`new_page https://neurogenesis.dev` → `take_screenshot` (206 KB PNG, viewport) →
`take_snapshot` (64 elements) → `get_css_styles uid=1_1` returned the authored
rules in cascade order with `@layer utilities` / `@layer base` wrappers, source file
`copy.sVkb6Amk.css:1`, `[overloaded]` markers and the inherited `body` rule with
`font-family: var(--font-sans)`. Dembrandt CLI
`npx dembrandt neurogenesis.dev --design-md` → 9 colors, 14 text styles, Tailwind v4
detected, `DESIGN.md` with `primary: "#1F6FEB"`, `fontFamily: "Inter"`.

**Agent use.** Extraction: ask for `dembrandt` on the URL (flags `pages`, `darkMode`,
`basicAuth`, `header`, `noSandbox` are tool arguments). Per-element: `take_snapshot`,
then `get_css_styles` on the uid; `evaluate_script` with `document.getAnimations()`
or `document.styleSheets` for motion. Checks: `take_screenshot` at three widths via
`resize_page`, `emulate` for dark mode and reduced motion, `performance_start_trace`.

## Lane 2 — Claude in Chrome on the laptop — `measured` 2026-11-06

Setup done 2026-11-06: extension installed in the laptop's Chromium-family browser,
signed in to the same claude.ai account and organization as `dev`'s Claude Code
(`/login`, not an API key). On the VPS, `/chrome` → **Enabled by default**, or start
with `claude --chrome`. The connection is relayed through Anthropic
(`bridge.claudeusercontent.com` per the Claude Code docs), so it works from a remote
session; `measured`: this VPS session listed the laptop as "Browser 1" (Linux), opened
a tab, loaded a page, read its accessibility tree at 1920×891, closed the tab.

Limits: only while the laptop is online with Chrome running; site permissions are
granted per site in the extension; the agent sees the page as a user, not the
inspector. Never make client work depend on this lane.

## Lane 3 — laptop Chromium behind an SSH reverse tunnel — `inferred` until first run

**On the laptop** (Arch, `chromium` package). A dedicated profile directory is
required, not optional: since Chrome 136 (Chrome Developers blog, 2025-03-17, read
2026-11-06) `--remote-debugging-port` is ignored on the default data directory and
"must now be accompanied by the `--user-data-dir` switch to point to a non-standard
directory". Whether a non-browser client also needs `--remote-allow-origins` was
not stated on that page (`inferred`: not needed for the HTTP endpoint; add
`--remote-allow-origins=http://127.0.0.1:9222` if the WebSocket upgrade is refused).

```
chromium --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.local/share/chromium-debug" --no-first-run
```

Then, in a second terminal, the tunnel to this VPS (replace `vps` with the existing
SSH alias used for `ssh <user>@vps`):

```
ssh -N -R 9222:127.0.0.1:9222 dev@vps
```

`-R 9222:127.0.0.1:9222` makes the VPS's own `127.0.0.1:9222` reach the laptop's
port; the port is bound to loopback on both ends and never crosses the tailnet or
the public interface unencrypted. sshd on the VPS has no `AllowTcpForwarding` or
`PermitOpen` override in `/etc/ssh/sshd_config*` (`measured` 2026-11-06), so the
default `yes` applies (`inferred`; confirm with `sudo sshd -T | grep -i forwarding`).

**On the VPS.** The server entry exists but is disabled:

```
laptop-chrome    npx -y chrome-devtools-mcp@latest --browserUrl http://127.0.0.1:9222
```

Disabled by `disabledMcpServers: ["laptop-chrome"]` in `~dev/.claude.json` for
every project known on 2026-11-06 (`/home/dev/code`, `/home/dev/code/neurogenesis-dev`,
`/srv/vault` and its four area folders). Claude Code stores this list **per
project**, so a new project directory sees the server enabled until it is toggled
off once in `/mcp`. To use the lane: open the tunnel, then `/mcp` → `laptop-chrome` →
enable; `claude mcp get laptop-chrome` should report connected. Disable again when
the tunnel closes, or every browser tool call will fail with a connection error.

Security of the lane: a debugging port is full control of that profile, including
its cookies. Hence the dedicated profile with nothing signed in except what the
task needs, and loopback binding only. Close Chromium when done.

## Fonts for lane-1 screenshots — `measured` 2026-11-06

Before (morning of 2026-11-06): 50 fonts from `fonts-liberation`, `fonts-freefont-ttf`,
`fonts-noto-color-emoji`, `fonts-ipafont-gothic`, `fonts-wqy-zenhei`, `fonts-unifont`,
`fonts-tlwg-loma-otf`; `fc-match` resolved `sans-serif`, `serif`, `monospace`,
`system-ui`, `Inter`, `Roboto`, `Segoe UI` and `Georgia` **all to WenQuanYi Zen Hei**,
a CJK face. Any page relying on a generic family or a system stack rendered Latin
text in the wrong face. Web fonts loaded by the page were unaffected.

Fixed the same day (root, from Ubuntu's archive; the apt pins exclude the xtradeb PPA):

```
apt install -y fonts-dejavu-core fonts-noto-core fonts-noto-mono \
  fonts-crosextra-carlito fonts-crosextra-caladea     # + fonts-dejavu-mono pulled in
fc-cache -f
```

After (`measured` as `dev`): `sans-serif`, `system-ui`, `Inter`, `Roboto`, `Segoe UI`
→ Noto Sans; `serif`, `Georgia` → Noto Serif; `monospace` → DejaVu Sans Mono;
`Calibri` → Carlito; `Cambria` → Caladea (metric clones, so documents set in those
faces keep their layout). If a screenshot ever shows CJK-looking Latin text again,
rerun this `fc-match` list first.

## Codex — not configured

The three servers were added to Claude Code's user config only. Codex
(`codex-cli 0.163.0` under `dev`'s mise Node 22, `~dev/.codex/config.toml`) reads its
own `[mcp_servers]` table. Adding the same three commands there is a follow-up;
`inferred` syntax, look it up in the Codex docs before writing it.

## References

- Claude Code MCP docs: `claude mcp add --scope user`, the `/mcp` toggle and
  `disabledMcpServers` in `~/.claude.json` (read 2026-11-06).
- Chrome DevTools MCP `docs/configuration.md` and `docs/tool-reference.md`
  (`--headless`, `--isolated`, `--executablePath`, `--browserUrl`, `--chromeArg`,
  `--allow-unrestricted-paths`, `--filesystemRoot`; `get_css_styles` returns matched
  rules, inherited styles, custom properties and cascade wrappers).
- Dembrandt README and `--help` (`install-browser`, `--no-headed-fallback`,
  `BROWSER_CDP_ENDPOINT`, MCP entry `dembrandt-mcp`).
- Chrome Developers blog, "Changes to remote debugging switches to improve
  security" (2025-03-17): Chrome 136+ ignores `--remote-debugging-port` and
  `--remote-debugging-pipe` on the default data directory; a non-standard
  `--user-data-dir` is required; Chrome for Testing keeps the old behaviour.
- Screenshot check 2026-11-06: the lane-1 capture of neurogenesis.dev rendered its
  web font (Inter) correctly; the font gap in §Fonts affects only pages that fall
  back to generic or system families.
