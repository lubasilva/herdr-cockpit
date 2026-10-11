# Herdr Cockpit — Stitch Visual Direction (V1 main screen)

Date: 2026-10-10 · Status: **PROPOSAL — awaiting owner approval**

> **Do not implement the visual redesign until the owner approves this direction.**
> Batch 9 (UI) is blocked on that approval. The Stitch HTML is reference only; it is
> Tailwind output, not code to copy. Implementation uses the existing tokens in
> `src/styles.css`, colocated component CSS, lucide icons and i18n strings.

## Artifacts

| | Value |
|---|---|
| Stitch project | `3911610958165352778` ("Herdr Cockpit — Primary Workspace (V1 direction)") |
| Design system | `assets/5888524796046612090` ("Herdr Cockpit — Graphite & Amber") |
| Screen 1 — live workspace (final) | `e3e7999813ca45429cd66856188a240a` → `stitch/01-live-workspace.png` / `.html` |
| Screen 2 — historical session (final) | `dc693278318046d983301a97c8f69194` → `stitch/02-historical-session.png` / `.html` |
| Screen 3 — Mac disconnected (final) | `99718431ea944a918f7517fd642cea98` → `stitch/03-mac-offline.png` / `.html` |
| Superseded | `06ce62e4beb744da9b7ff233bb941ee1`, `8fd75583664d4d90943c712065f2084a`, `aacb2db80f0e4037bf7ad422031d8b72` (first generation, PNGs in `stitch/v1-raw/`); `f72502f766fc4a62ad54cacba5b8c5ba` (screen 1 after the cleanup edit, header regressed) |

Model: `GEMINI_3_8_FLASH`, desktop. Generation path: design system → three screens →
one consistency edit over all three → one targeted fix on screen 1.

## Direction: "lamp-lit graphite console"

A tool for eight-hour days, not a dashboard:

- **Built on the upstream tokens.** Base `#12100e`, panel `#181613`, elevated `#211e1a`, hairline
  `#2d2924`, text `#d8d0c3`/dim `#9b9183`, one amber `#f0a830`, status colors working `#6cb8d6`,
  blocked `#ff7b70`, done `#93c36b`, idle `#9b9183`. This is the existing herdr-web-ui palette,
  so the direction ships **without a global restyle** and without forking `src/styles.css`.
- **Amber is the user's hand only.** Selection marker, focus, active lens, Send/Resume. Machine and
  agent state never use amber.
- **Project-first sidebar.** Project → sessions (live and historical mixed, sorted by recency).
  Each session is a two-line row (title / mono `Runtime · Machine · time`). The machine is a word
  in mono, never a chip. The glyph carries state: filled dot = live (status color), hollow
  circle = historical, dashed circle = PC offline/stale.
- **Hierarchy through tone and hairlines.** No cards on cards, no shadows in flow, no KPI tiles.
- **Conversation reads like a chat app.** 760 px column; user turns in a neutral raised bubble;
  assistant plain text; tool activity folded into one mono row.
- **Live vs history differs in structure, not alarm color.** The history view drops the lens
  toggle and the composer, adds a dashed `HISTORY · read-only` tag and one notice line, and puts
  an action bar where the composer was: **Resume on <PC>** (amber) and **Start fresh
  continuation** (ghost), plus a one-line explanation that nothing is replayed.
- **Offline is quiet.** Dashed glyph, dimmed rows, a plain-language inline state, a retry
  countdown. No modal, no red banner, the rest of the app stays usable, and the cockpit says
  plainly that it holds no transcript copy.

## Per-screen notes

1. **Live workspace.** Accepted shape: top bar (wordmark · breadcrumb · `Linux ●` `Mac ●` ·
   ⌘⇧K · gear), Projects|PCs switch, project tree, `PLAN LIMITS` footer, Chat|Terminal lens,
   folded tool row, composer with an "Esc to interrupt" hint.
2. **Historical session.** Read-only tag, notice strip linking to the live session of the same
   project, mono gutter timestamps, "Load earlier turns", action bar with Resume/Fresh.
3. **Mac disconnected.** Mac rows dim with "last seen"; the needs-you row becomes a grey `?`
   (state unknown, not "blocked"); inline "Mac is offline" state; both actions disabled with
   reasons; explicit "Start in Projeto EAD on Linux…" link.

## Stitch artifacts to ignore (do not implement)

- Extra chips the model added (e.g. "standby", "host offline" duplicates of the dashed glyph),
  any "Plan Details"/footer links, "3 ACTIVE" style counters, and the bell icon unless the owner
  keeps it (O6).
- Exact pixel sizes, Tailwind classes and Material icon names: map to existing tokens and
  lucide icons.
- Sample content (project names, titles, paths, model names) is fictional.

## Owner decisions (resolved 2026-10-10)

O1 `Projects | PCs` switch, Projects default · O2 two lines · O3 titles cached locally, shown
stale/last seen · O4 compact usage in the header, detail on demand · O5 keep the explicit
other-PC fallback · O6 keep machine pills, drop the bell · O7 keep a discreet Esc hint ·
O8 direction approved.

Original options:

| # | Decision | Recommendation |
|---|---|---|
| O1 | `Projects \| PCs` switch inside the existing sidebar vs replacing the PC tree | Switch (keeps upstream `Sidebar` untouched; PCs view remains for terminal/workspace management) |
| O2 | Two-line session rows (denser alternative: one line + trailing mono) | Two lines |
| O3 | Offline cache may store session titles (≤120 chars) on the Linux server, or only runtime · time · branch | Titles (otherwise screen 3 is unreadable); file is 0600 and disposable |
| O4 | Plan-limit meters in the sidebar footer vs top bar vs settings only | Sidebar footer, only after the telemetry follow-on; V1 keeps upstream's existing usage UI |
| O5 | Keep "Start in <project> on <other PC>…" as an explicit fallback link | Keep, explicit only |
| O6 | Top-bar machine pills (`Linux ●` `Mac ●`) — duplicate of the PCs view? Keep the bell? | Keep the pills (they are the only always-visible connection state); drop the bell |
| O7 | Composer hint "Esc to interrupt · ⇧↵ newline" | Keep only if it matches the existing Composer keys; no new behavior |
| O8 | Approve the direction as a whole (approve / approve with edits / reject) | — |

## Exact prompts sent

Verbatim inputs, in call order.

### create_project

Title: Herdr Cockpit — Primary Workspace (V1 direction)

### create_design_system

Theme: dark, IBM Plex Sans / JetBrains Mono labels, ROUND_FOUR, primary `#f0a830`, neutral `#12100e`. designMd:

```markdown
# Herdr Cockpit — Graphite & Amber

A warm developer console used for many hours a day: lamp-lit graphite surfaces, one amber accent, chat-app legibility. Derived from the existing herdr-web-ui token system so it can ship without forking it.

## Surfaces (dark)
- base #12100e, panel #181613, elevated #211e1a, hover #2a2621, input #1c1916
- border #2d2924 (hairline 1px), border-strong #3e3830
- No drop shadows on in-flow content. Hierarchy comes from tonal surface steps and hairlines, never from cards floating on cards.

## Text
- primary #d8d0c3, dim #9b9183, strong #f2ebdf

## Accent
- ONE chrome color: amber #f0a830 (tint rgba(240,168,48,0.13)). Used only for focus, the selected item marker, the terminal cursor and the user's own primary action (Send, Resume). Never for decoration.

## Agent / machine state colors (never amber)
- working #6cb8d6, blocked/needs-you #ff7b70, done #93c36b, idle #9b9183
- Offline/stale = dim text + dashed outline glyph, never red unless it is an error.

## Typography
- UI: IBM Plex Sans 13px/1.45 (compact desktop density). Section heads 11px uppercase letter-spacing .06em dim.
- Metadata, IDs, branches, model names, timestamps: JetBrains Mono 11.5px dim.
- Conversation body: 14.5px/1.6, max line length ~76ch.

## Shape & spacing
- 4px base unit, radius 4–6px, rows 26–28px high in the sidebar.
- Sections separated by space and a dim head, not by rules or boxes.

## Do not
- No neon, no gradients, no glassmorphism, no big hero numbers, no KPI tiles, no grid of cards, no emoji.
```

### generate_screen_from_text — screen 1 (live workspace)

```text
Desktop app screen, 1440x900, dark. "Herdr Cockpit": a developer's cockpit for coding agents (Claude Code, Codex, OpenCode) running in terminal panes on two computers (a Linux PC and a MacBook) that feel like ONE workspace. It must feel like a calm professional tool used 8 hours a day (think the discipline of Linear, the reading comfort of a chat app, the project tree of VS Code) — NOT a SaaS admin dashboard. No cards grid, no KPI tiles, no charts, no gradients, no neon.

LAYOUT — three zones, edge to edge, separated by 1px hairlines only:

1) TOP BAR (36px, panel surface): left "Herdr Cockpit" wordmark small; center breadcrumb "Projeto EAD › Claude · Opus 4 · Linux · main"; right: two tiny machine dots with labels "Linux ●" (green) and "Mac ●" (green), a command-palette hint "⌘⇧K", settings gear icon.

2) LEFT SIDEBAR (272px, panel surface). Header row: segmented toggle "Projects | PCs" (Projects selected), a "+" icon (New workspace). Section head "PROJECTS" (11px uppercase dim). Project tree, project-first:
 ▼ Projeto EAD (expanded, bold) with a small "2 PCs" mono badge
   • row selected (amber left marker, hover-fill): animated-looking blue dot "working" · "Refactor enrollment API" · trailing mono "Claude · Linux · now"
   • row: red dot "needs you" · "Fix flaky checkout test" · "Codex · Mac · 2m"
   • row historical (hollow circle, dim): "Migrate auth to Lucia" · "Claude · Mac · 21:14"
   • row historical: "Seed script cleanup" · "Codex2 · Linux · 18:20"
   • row historical: "Course export bug" · "OpenCode · Linux · Yesterday"
   • "Show 14 older…" link dim
 ▶ CotaPay (collapsed, trailing "1 live")
 ▶ Transporte
 ▶ Echo
 ▶ Fabrica
 Each row: leading 14px agent brand glyph or status dot, title text 13px, trailing metadata in JetBrains Mono 11px dim. Rows 28px. Machine shown as a tiny mono tag "Mac"/"Linux", never as a big chip.
 Bottom of sidebar, footer strip: plan meters as thin 2px bars with mono labels "Claude 5h 72%", "Codex 18%".

3) MAIN AREA (base surface). Session header row (40px): title "Refactor enrollment API", then mono metadata "Claude · Opus 4 · Linux · ~/Projects/projeto-ead · ⎇ main", right side a segmented lens toggle "Chat | Terminal" (Chat selected, amber underline) and a "⋯" menu.
 Conversation column centered, max 760px wide:
  - User turn: neutral raised bubble (elevated surface, no border), right aligned text "Split the enrollment controller into service + repository and keep the API contract identical."
  - Assistant turn: plain text on base, no bubble, readable 14.5px, a short paragraph and a small bulleted list.
  - A FOLDED tool-activity block between them: one 32px row with chevron "▸ Worked 1m 12s · 6 tools · Read 4 files · Edit 2 · Bash 1" in mono dim, collapsed.
  - Second assistant paragraph with an inline code span and a 6-line code block (darker surface, mono).
  - Live status line at the bottom of the thread: blue pulse dot "Working · 23s".
 COMPOSER pinned to bottom, same 760px column: input surface, 1px border, placeholder "Message Claude on Linux…", left icons (paperclip, slash "/" commands, "@" files, mic), right a "Send ↵" primary button in amber. Above composer a tiny dim hint "Esc to interrupt · ⇧↵ newline".

Overall: compact density, generous whitespace inside the conversation, strict alignment, everything legible at a glance. Amber is used ONLY for the selected row marker, the active lens underline, focus ring and Send.
```

### generate_screen_from_text — screen 2 (historical session)

```text
Same "Herdr Cockpit" desktop screen as before (1440x900, dark warm graphite #12100e/#181613/#211e1a, hairlines #2d2924, single amber accent #f0a830, IBM Plex Sans UI, JetBrains Mono metadata, compact, calm, no cards grid, no gradients, no neon). Identical top bar (Herdr Cockpit · breadcrumb · "Linux ●" "Mac ●" both connected green · ⌘⇧K · gear) and identical left sidebar project tree (Projects | PCs toggle; ▼ Projeto EAD with rows: blue "working" Refactor enrollment API · Claude · Linux · now; red "needs you" Fix flaky checkout test · Codex · Mac · 2m; hollow-circle historical rows Migrate auth to Lucia · Claude · Mac · 21:14; Seed script cleanup · Codex2 · Linux · 18:20; Course export bug · OpenCode · Linux · Yesterday; collapsed CotaPay, Transporte, Echo, Fabrica; thin plan meters footer).

STATE: an OLD HISTORICAL session is selected — "Migrate auth to Lucia" (Claude · Mac · 21:14) has the amber left marker. The live session rows still show their own blue/red dots so the user sees the live agent is elsewhere.

MAIN AREA must read unmistakably as a READ-ONLY ARCHIVE, not the live agent:
- Session header: title "Migrate auth to Lucia", then a small outlined mono tag "HISTORY · read-only" (dim, dashed 1px border), then mono metadata "Claude · Opus 4 · Mac (MacBook Pro) · /Users/ipremium/Projects/projeto-ead · ⎇ feat/lucia · Oct 9, 21:14 → 22:03 · 48 turns". No Chat|Terminal toggle here (there is no live terminal); instead a dim "Details" disclosure that would reveal the native session ID.
- A thin full-width notice strip under the header (elevated surface, no color fill, a small clock-history icon): "You are reading a past Claude session from Mac. Nothing you type here is sent. The live agent for this project is “Refactor enrollment API” on Linux →" (the last part is a link).
- Conversation column (max 760px): same chat typography as the live view but slightly desaturated: user bubbles neutral, assistant text plain; two folded tool blocks "▸ Worked 3m 40s · 14 tools · Edit 6 · Bash 3"; a "Load earlier turns" dim button at the top; timestamps in mono at the left gutter of each turn (21:14, 21:22, 21:40). No live status line.
- In place of the composer, a ACTION BAR pinned to the bottom of the 760px column, input-surface background, 1px border:
  left text block: "Continue this session?" (13px strong) and below in mono dim "Resumes natively on Mac · Claude · ~/Projects/projeto-ead · branch feat/lucia".
  right: primary amber button "Resume on Mac" with a small play-history icon, and a secondary ghost button "Start fresh continuation" (border only). Under the buttons a tiny dim note: "Fresh starts an empty Claude session in the same project. The old transcript is not replayed."
Make the difference between live and historical obvious through the read-only tag, the notice strip, the missing composer and missing lens toggle — not through loud colors.
```

### generate_screen_from_text — screen 3 (Mac disconnected)

```text
Same "Herdr Cockpit" desktop screen (1440x900, dark warm graphite #12100e/#181613/#211e1a, hairlines #2d2924, single amber accent #f0a830, IBM Plex Sans UI, JetBrains Mono metadata; compact, calm; no emoji, no folder icons, no cards grid, no gradients, no neon, no modal dialogs). Use ONLY the content described here — do not invent cluster names, hostnames or extra footer links.

STATE: the MacBook is DISCONNECTED (asleep / off network). Linux is fine.

TOP BAR: "Herdr Cockpit" · breadcrumb "Projeto EAD › Claude · Opus 4 · Mac" · right side machine indicators: "Linux ●" green dot, and "Mac" with a HOLLOW dashed dim circle and dim text "offline · retrying in 32s" (no red, no banner across the whole app). ⌘⇧K hint, gear.

LEFT SIDEBAR (Projects | PCs toggle, Projects selected; section head "PROJECTS"):
▼ Projeto EAD (badge "2 PCs", second PC dimmed)
  • Linux rows stay fully normal: blue working dot "Refactor enrollment API · Claude · Linux · now"; hollow-circle "Seed script cleanup · Codex2 · Linux · 18:20"; "Course export bug · OpenCode · Linux · Yesterday".
  • Mac rows remain VISIBLE but STALE: text at ~55% opacity, status glyph a dashed hollow circle, trailing mono "Mac · offline". Rows: "Fix flaky checkout test · Codex · Mac · last seen 14:02" (was needs-you; now shows a grey "?" instead of red, meaning unknown), and the SELECTED row "Migrate auth to Lucia · Claude · Mac · 21:14" with the amber left selection marker still on it.
▶ CotaPay, ▶ Transporte, ▶ Echo, ▶ Fabrica (plain text, chevrons only).
Footer: thin plan meters "Claude 5h 72%" "Codex 18%" plus a tiny dim line "Mac usage unavailable".

MAIN AREA — the selected Mac historical session cannot be read right now:
- Header: title "Migrate auth to Lucia", outlined mono tag "HISTORY · read-only", metadata "Claude · Opus 4 · Mac (MacBook Pro) · ⎇ feat/lucia · Oct 9, 21:14 · 48 turns" and a dim mono "metadata from 14:02" stamp.
- Inline unavailable state (NOT a modal), centered in the 760px column, quiet: a small dashed-circle icon, heading "Mac is offline" (15px strong), body "This conversation lives on the MacBook. It will load when the Mac reconnects. The cockpit does not keep copies of transcripts and will not open it on Linux." Two small buttons: ghost "Retry now" and text-link "Open PCs settings". A mono dim line: "Last contact 14:02 · SSH macbook · retrying every 60s".
- Bottom action bar (same position as composer, disabled look, 50% opacity): "Resume on Mac" button disabled with tooltip-style caption "Unavailable while Mac is offline"; "Start fresh continuation" ALSO disabled with caption "would start on Mac — choose a Linux checkout instead" and a small enabled link "Start in Projeto EAD on Linux…" (explicit, never automatic).
The whole app remains usable: Linux sessions clearly live and clickable. Disconnected state is understandable at a glance through dimming, dashed glyphs and plain words — no alarm colors.
```

### edit_screens on 06ce62e4beb744da9b7ff233bb941ee1, 8fd75583664d4d90943c712065f2084a, aacb2db80f0e4037bf7ad422031d8b72 (consistency cleanup)

```text
Consistency cleanup across all three screens, keep everything else exactly as it is (layout, colors, typography, conversation content, action bars):
1. TOP BAR left: show ONLY the "Herdr Cockpit" wordmark. Remove "Cluster / prod-linux-01", "Session #8420" and the "operator: root" text entirely — these concepts do not exist. Keep the center breadcrumb, the machine indicators, ⌘⇧K, bell and gear.
2. SIDEBAR rows: use the SAME two-line session row on all three screens: line 1 = session title (13px), line 2 = mono dim "Runtime · Machine · time" (e.g. "Claude · Linux · now"). Leading cell = status glyph (filled dot live, hollow circle historical, dashed circle offline). Remove any "archived" badge; historical rows need no badge.
3. Remove the folder icons in front of project names; project rows are just chevron + name + small mono badge.
4. Offline (Mac) rows on the disconnected screen: line 2 reads "Codex · Mac · last seen 14:02" / "Claude · Mac · 21:14 · offline"; nothing may overlap or be truncated mid-word — drop the separate floating "offline" label.
5. Sidebar footer: section label "PLAN LIMITS" (not "Host quotas"), keep the two thin meters; remove the "Shell" and "Docs" footer links and version text.
```

### edit_screens on f72502f766fc4a62ad54cacba5b8c5ba (screen 1 header and row fix)

```text
Fix only two things, change nothing else:
1. The session header row in the main area (left of the Chat | Terminal toggle) must show the session, not the app name: title "Refactor enrollment API" (13px strong, no dot), then mono dim metadata "Claude · Opus 4 · Linux · ~/Projects/projeto-ead · ⎇ main". Remove the "Herdr Cockpit" text and amber dot from that row.
2. Sidebar session rows under Projeto EAD, line 2 must read exactly: "Refactor enrollment API" → "Claude · Linux · now"; "Fix flaky checkout test" → "Codex · Mac · 2m"; "Migrate auth to Lucia" → "Claude · Mac · 21:14"; "Seed script cleanup" → "Codex2 · Linux · 18:20"; "Course export bug" → "OpenCode · Linux · Yesterday". Historical rows use a clearly visible hollow circle glyph (same size as the live dots).
```

