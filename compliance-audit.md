# Creao Design Compliance Audit

| | |
|---|---|
| **Audited** | The production build at agent.creao.ai: `index-BTZeeHx5.js` (16.8 MB), `index-BXYlNekV.css` (675 KB) and 184 lazy-loaded chunks |
| **Fetched** | 2026-11-09 |
| **Against** | `design-system-guideline.md` v1.1. Rule IDs below (T, C, L, P, GU) refer to it. P1–P8 are always screen patterns. Severity uses S1–S3, so the two never collide. |
| **Result** | 18 findings: 3 S1, 6 S2, 9 S3. Each one maps to a rule and to the gate that would have blocked it. |

**Contents:**
- [1. Summary](#1-summary)
- [2. Method and limits](#2-method-and-limits)
- [3. Findings](#3-findings)
- [4. Proof: the enforcement mechanism on real output](#4-proof-the-enforcement-mechanism-on-real-output)
- [5. What the gates will not catch](#5-what-the-gates-will-not-catch)
- [6. Fix order](#6-fix-order)

---

## 1. Summary

Creao's design system works where people use it. The Creao `Button` appears at 1,278 call sites. `Card`, `StatusPill` and `ComposerShell` are fully token-driven, and all 9 destructive confirmations use `AlertDialog`. The problems sit around the system rather than inside it:

1. **Two component libraries ship.** A stock shadcn/ui copy, bundled next to the generative-UI catalog, redefines 21 of Creao's primitives (62 parts), Button included. It even carries its own copy of the class-merge helper. The agent's generated UI is drawn with this copy, not Creao's.
2. **The agent's own UI is off-system.** Generated blocks in chat use the legacy `ox-*` palette and a hard-coded cream background, plus 35 CSS overrides that restyle library components from outside.
3. **Four render modes, no checks.** Classic and v2, each light and dark, change button heights, radii and title sizes. Hard-coded values are right in at most one mode.
4. **Nothing above the component level.** Page titles come in 31 hand-written styles. 27 of 32 empty states are hand-rolled. Form dialogs use two different overlay types, and there is no width system: production uses 68 distinct arbitrary max-widths.

**Baseline metrics.** These come from the Gate 2 scanner and count only classes that both the CSS and the JS reference:

| Metric | Count | Rule |
|---|---|---|
| Raw Tailwind palette colour classes (`amber-*`, `gray-*`, …) | 370 | T1.1 |
| Manual `dark:` colour patches | 489 | T2.1 |
| Direct `grey-*` ramp classes | 264 | T1.2 |
| Arbitrary spacing classes | 105 | T7.1 |
| Arbitrary max-widths (68 distinct values) | 74 | T9.1 |
| Legacy `ox-*` classes | 61 | T1.4 |
| Hand-written shadows (42 distinct) | 47 | T5.1 |
| Descendant selectors restyling library components | 47 | GU3 |
| Arbitrary font sizes (32 distinct, 7 px to 52 px) | 44 | T3.1 |
| Arbitrary radii (13 distinct) | 23 | T4.1 |
| z-index classes outside the feature scale `z-0`/`10`/`20` (18 distinct levels, 0–300) | 22 | T6.1 |
| `<h1>` elements not using the page-title tokens | 40 of 71 | C8.1 |
| Empty states not using `EmptyState` | 27 of 32 | C6.1 |
| Primitives defined in both libraries | 21 (62 parts) | C1.2 |
| Extra `data-slot` definitions in the build (the slot count) | 81 | C1.2 |

**What already works (keep it):**
- `StatusPill` has 7 tokenised tones.
- Component dimension tokens exist (`--button-h`, `--card-radius`, …).
- Confirmations consistently use `AlertDialog`.
- The chat column uses container queries.
- Safe-area variables are used.
- Library primitives emit `data-slot`. Button and TabsList also emit `data-variant`, and Button emits `data-size`. That's what makes DOM-level checks (Gate 3) cheap to build.
- On touch screens, v2 raises button heights to the classic values (`@media (pointer: coarse)`).

---

## 2. Method and limits

**How the audit was done:**
1. **Tokens.** Every theme block was parsed from the shipped CSS: `:root`, `:root[data-design=v2]`, `.dark`, `:root.dark[data-design=v2]` and `.auth-legacy-theme`.
2. **Counts.** The Gate 2 scanner from guideline §7.4 ran on the downloaded build. It counts a class only if the CSS defines it *and* the shipped JS references it. That excluded 193 "phantom" classes Tailwind had generated from non-UI text (F-18).
3. **Screens.** Each finding is tied to a screen through evidence inside the minified component itself: the route definition (for example `Ca("/status")`), `data-testid` values and i18n keys (for example `agentDetail.runFilterNoMatches`). Class strings and code are quoted verbatim, with only minified variable names shortened.
4. **Libraries.** Every `data-slot` definition in the JS was assigned to a library by position. The stock copy is one contiguous region of about 140 KB in `index.js`. That region holds every call to the copy's class-merge helper and none to Creao's.
5. **Gate 1.** The guideline's ESLint config ran on code rebuilt from those verbatim class strings (§4.3).

**Limits:**
- **No signed-in session, so no screenshots.** Statements about how something looks are derived from token values and marked *(derived)*. Gate 3 snapshots would confirm them.
- **The assessment's screen set wasn't available in this workspace.** The production bundle contains every shipped screen, so findings are anchored to routes and components instead.
- **Empty states and dialogs were mapped to their components heuristically.** For empty states: whether `EmptyState` appears within ±400 characters of the test id. For dialogs: which overlay component renders around the dialog's test id. The empty states quoted in F-05 were confirmed by reading the code.
- **Component names are inferred from `data-slot`** (for example `empty-state` → `EmptyState`).

**Severity:**
- **S1 (critical):** degrades the core chat surface, or breaks a whole render mode.
- **S2 (major):** a visible inconsistency on a specific screen, or on mobile.
- **S3 (debt):** systemic debt. The ratchet holds it and each sprint burns it down.

---

## 3. Findings

### F-01 · Two component libraries ship in production · S1

**Where:** app-wide. The stock copy is bundled beside the generative-UI chart catalog (`LineChart`, `AreaChart`, `BarChart` and `CandlestickChart` schemas) and is also used for carousel controls.

**Evidence:**
- Two `Button` variant tables are in the bundle:

```js
// Creao Button: 1,278 JSX call sites
size: { default: "h-(--button-h) gap-1.5 px-3.5 py-2 text-sm …", sm: "h-(--button-h-sm) …" }
variant: default | destructive | outline | secondary | raised | raised-outline | ghost | link

// Stock shadcn Button: 5 call sites (carousel prev/next + 3 outline buttons)
size:    { default: "h-9 px-4 py-2 has-[>svg]:px-3", sm: "h-8 rounded-md …", lg: "h-10 rounded-md …" }
variant: { ghost: "hover:bg-accent hover:text-accent-foreground …",
           destructive: "bg-destructive text-white hover:bg-destructive/90 …" }
```

- **Two class-merge helpers.** The copy ships its own `cn`, a second copy of the same one-line function Creao uses. Both call the same merge and join functions. 1,741 `className` props go through Creao's and 77 through the copy's.
- **21 primitives (62 `data-slot` parts) are defined in both:** Avatar, Badge, Button, Card, Checkbox, Collapsible, Dialog (8 parts), DropdownMenu, Input, Label, Popover, Progress, Select (8 parts), Separator, Skeleton, Slider, Switch, Table (7 parts), Tabs, Textarea and Tooltip.
- **Only in the stock copy:** Accordion, Alert, Carousel, Drawer, Pagination, RadioGroup, Toggle, ToggleGroup, TableCaption, and Card's `CardHeader`, `CardTitle` and `CardDescription`.
- **Generated UI is drawn with the copy.** The `.jr-block` CSS restyles `[data-slot=card-title]`, `card-header`, `card-description` and `alert`. Only the stock copy defines those four parts, so the json-render catalog renders stock components.

**Impact:**
- **Hover:** in Creao, `accent` is ink (`#16181a` classic, `#37352f` v2). A stock ghost or outline button therefore hovers to a near-black fill *(derived)*.
- **Height:** in v2 the stock button is 36 px next to Creao's 28 px buttons.
- **Fixes don't propagate:** a fix to Dialog, Select or Table in one library never reaches the other.

**Breaks:** C1.1, C1.2, GU1; token traps 1, 4 and 5 (guideline §2.1).

**How enforcement would have caught it:**
- **Gate 2, slot count (`C1.2` in `design/scan.mjs`):**
  - Removing the copy's region from today's build takes `C1.2` from 81 to 17.
  - So the PR that added the copy raised the count by 64 and would have failed the ratchet with `C1.2 primitive defined again: button badge card …`. Both numbers are measured (§4.2).
  - The count reads the build, so it works even though the copy may arrive through an npm package rather than `src/`.
- **Gate 3:** the generated-UI fixture conversation shows the height and hover mismatch in v2 snapshots.

**Fix:**
1. Add `CardHeader`, `CardTitle` and `CardDescription` to `src/components/ui`, plus a `Callout` to replace `Alert` (guideline §9).
2. Point the catalog at `src/components/ui` and delete the copy.
3. Run `--update-baseline` so `C1.2` locks in at 17.

If generated UI needs a denser look, the library adds `density="compact"` (guideline §9).

---

### F-02 · Agent-generated UI renders on the legacy palette · S1

**Where:** every json-render block the agent produces in `/chat` and rooms. That covers the inline `.jr-block`, previews (`.jr-preview`) and the full-screen block dialog.

**Evidence:**

```css
.jr-block      { --jr-container-bg: #f8f6f3; --jr-card-bg: var(--color-ox-paper); --jr-card-border: var(--color-ox-rule);
                 --jr-ink: var(--color-ox-ink); --jr-pill-bg: var(--color-ox-btn-bg); … }
.dark .jr-block { --jr-container-bg: var(--color-ox-tile); --jr-card-bg: var(--color-ox-paper) }
/* no [data-design=v2] override exists */
.jr-block :is(h1,h2,h3,h4) { font-family: var(--font-display); font-weight: 800; line-height: 1.05 }
.jr-block h1 { font-size: 2rem }
.jr-block [data-slot=card]       { border-radius: 6px; padding-top: .875rem; … }
.jr-block [data-slot=card-title] { letter-spacing: .1em; text-transform: uppercase; font-size: .6875rem }
.jr-block [data-slot=badge][data-variant=secondary] { background: #ebe7e2a6 }
```

```js
className: "jr-block my-3 rounded-[6px] border border-ox-rule"      // inline block container
className: "jr-block w-auto! fixed inset-8 z-50 … rounded-[6px] border border-ox-rule bg-background shadow-2xl"  // full screen
```

**Impact (derived):**
- In v2 light, the page and conversation are `#fff`, but every generated block sits in a `#f8f6f3` cream box. It's the only cream surface on an otherwise white screen.
- Block headings are weight 800, heavier than anything the product itself uses (the token maximum is 700).
- Card titles become 11 px uppercase labels, a style product cards don't use.
- The result: in v2, the agent's UI looks like a different product from the chat around it.

**Breaks:** GU2, GU3, GU5, T1.4, T1.6, T4.1, C2.3.

**How enforcement would have caught it:**
- **Gate 1** on the container: `[T4.1] rounded-[6px]` and `[T1.4] border-ox-rule` (real output, §4.3).
- **Gate 2:** `T1.6` counts the literal `--jr-container-bg: #f8f6f3`. `GU3` counts every `.jr-block [data-slot=…]` rule (35 of the 47). Under the ratchet, the second such rule would have failed CI.
- **Gate 3:** the v2-light snapshot of the generated-UI fixture.

**Fix:** apply the GU2 alias table (CSS only), delete the `.dark .jr-block` override block, demote block headings to `h3`/`h4` (GU5), and burn down `GU3` through library density variants.

---

### F-03 · Sizes that are right in only one mode · S1

**Where:** the chat composer, plus any screen with hard-coded heights or generic radii.

**Evidence:** component dimension tokens differ by mode (guideline §1.2). For example, `--composer-radius` is 24 px in classic and 8 px in v2, and `--button-h` is 36 px and 28 px. The queued-messages tray attached to the composer ignores this:

```js
// chat input frame, shown while messages are queued ("input.queued")
className: "mb-2 w-full rounded-xl border border-border bg-surface"                        // 12 px in both modes
// ComposerShell directly below it
className: "relative z-10 w-full rounded-(--composer-radius) border bg-composer shadow-composer"  // 24 px classic, 8 px v2
```

**Impact:** the tray's corners match the composer in neither mode: 12 px against 24 px in classic, and 12 px against 8 px in v2. The same class of problem appears in 39 generic `rounded-*` usages (T4.2), 15 generic shadows (T5.2) and the stock button's fixed `h-9` (F-01).

**Breaks:** P1.3, T4.2, C3.2, T2.2.

**How enforcement would have caught it:**
- **Gate 1** advisory `[T4.2] rounded-xl`. Advisory rules block changed files through `--max-warnings=0`.
- **Gate 3:** P1 snapshots in classic and v2.

**Fix:** set the tray to `rounded-(--composer-radius)`, then migrate the T4.2 list.

---

### F-04 · Page titles: 31 hand-written styles · S2

**Evidence:** 71 `<h1>` elements ship. 31 use the page-title tokens. The other 40 use 31 distinct class strings across 4 font families.

| Screen (how it was located) | Shipped `<h1>` classes |
|---|---|
| `/status` (route definition `Ca("/status")`) | `text-[22px] font-semibold text-gray-900 tracking-normal` |
| App header: title, description and coloured icon tile (`apps` namespace) | `my-1 font-sans text-[28px] font-semibold leading-[1.3] tracking-normal text-grey-950 dark:text-grey-100` |
| CRM account and CRM header (`crm.*` keys) | `mt-2 font-data text-3xl font-medium tracking-tight [word-spacing:-0.2em]`; `truncate font-data text-2xl …` |
| Login and signup (`login.*`, `signup.*` keys) | `text-xl font-medium`, `text-xl font-[500]` and `text-xl font-semibold`: three spellings for one job |
| App install page (`install.*` keys) | `text-xl font-semibold text-foreground` |
| Personal agent details header (`agent-details`) | `truncate text-[15px] leading-5 font-semibold` |
| Room private chat header (`room-private-chat-*`) | `truncate text-[15px] font-semibold` |
| Markdown renderer | `text-lg font-bold tracking-normal mt-4 mb-3`. This is a document heading rendered as the page's `h1`. |

The most common title style (20 uses) is token-based but colours itself with `text-grey-950 dark:text-grey-100` instead of `text-foreground`. In classic dark, `grey-100` (`#f8f6f3`) isn't the same as `foreground` (`#ece2dd`), so titles are a different white from the body text *(derived)*.

**Breaks:** C8.1, C8.2, T3.1, T3.2, T3.4, T1.1, T1.2, T2.1.

**How enforcement would have caught it:**
- **Gate 1** flags every raw heading: `[C8.1] Page titles come from <PageHeader>`. On `/status` it also reports `[T3.1] text-[22px]` and `[T1.1] text-gray-900` (real output, §4.3).
- **Gate 3, D2:** exactly one `h1`, inside `[data-slot=page-header]`.

**Fix:**
- P2, P3 and P5 screens use `PageHeader`.
- The agent details header becomes `PageHeader` with `leading` (P3).
- The private chat header is a P1 header, so its title becomes `text-sm font-semibold` and is no longer an `h1`.
- Markdown headings are demoted (C8.2).
- Monospace titles are removed (T3.4).

---

### F-05 · 27 of 32 empty states are hand-rolled · S2

**Evidence:** 32 `*-empty` test ids ship.
- **5 use `EmptyState`:** `crm-accounts-empty`, `crm-empty-suggestions`, `crm-suggestions-empty`, `room-files-empty` and `room-empty-invite`.
- **27 don't.** Three confirmed by reading the code:

```js
// Agent detail → run filter, "agentDetail.runFilterNoMatches"
jsx("p", { className: "px-1 py-6 text-center text-sm text-muted", "data-testid": "run-filter-empty" })

// Room secrets, "secrets.empty" (also the second spelling of muted)
jsx("p", { className: "text-sm text-muted-foreground", "data-testid": "room-secrets-empty" })

// Room announcements: EmptyState rebuilt by hand, minus its title and container
jsxs("div", { className: "flex flex-col items-center gap-3 px-3 py-8 text-center", children: [
  jsx(Icon, { className: "size-6 text-muted" }),
  jsx("p",  { className: "text-sm text-muted", "data-testid": "room-announcements-empty" }),
  jsxs(Button, { variant: "outline", size: "sm" }) ] })
```

**Root cause:** the library's only `EmptyState` is a large dashed card (`border-dashed bg-surface-elevated px-8 py-12`). That's wrong for "no matches" inside a list, so people build their own. The guideline addresses this with two kinds of empty state (C6.1) and a `size="sm"` request to the library (guideline §9).

**Breaks:** C6.1.

**How enforcement would have caught it:**
- **Gate 1:** `[C6.1] Zero-data states render <EmptyState>` on `run-filter-empty` (real output).
- **Gate 3, D6:** every `[data-testid$="-empty"]` must be `[data-slot=empty-state]`.

---

### F-06 · The same job uses two overlay types · S2

**Evidence:** form dialogs are split between two primitives that behave differently on phones.

| `ResponsiveDialog`: becomes a bottom sheet on mobile | Plain `Dialog`: stays a centred modal on mobile |
|---|---|
| `create-room`, `add-agent`, `edit-agent`, `invite-members`, `room-settings`, `room-automation`, `room-api`, `room-shortcut`, `room-file-edit`, `artifact-share`, `agent-invite-setup`, `start-plan`, `slide-settings`, `room-decision`, `file-version-history`, `room-file-version`, `room-forward`, `room-members-bring-personal-agent` | `crm-new-account`, `crm-edit`, `save-workflow-as-agent`, `move-to`, `dashboard-move`, `large-upload-upgrade`, `custom-skills-paywall` |

`workflow-dag-dialog` also uses `Dialog`, but legitimately: it's a P8 viewer.

**Compliant:** all 9 confirmations use `AlertDialog`: `room-file-delete`, `room-secret-delete`, `room-shortcut-delete`, `room-task-cancel`, `deck-preview-annotate-discard`, `deck-preview-annotate-unapplied`, `deck-preview-stale-edit`, `deck-preview-stale-annotate` and `deck-export-stale-edit`.

**Impact:** on phones, 7 form dialogs open as centred desktop modals, with no bottom sheet, no drag-to-dismiss and the keyboard covering fields. The same job behaves two different ways.

**Breaks:** C4.2.

**How enforcement would have caught it:**
- **Gate 1:** `[C4.2] Forms use <ResponsiveDialog>; confirmations use <AlertDialog>` (real output on `crm-new-account`).
- **Gate 3, D9:** mobile viewport, `[role=dialog]` must be `responsive-dialog-content`, `mobile-sheet-content` or `alert-dialog-content`.

---

### F-07 · `/status` is built entirely outside the system · S2

**Evidence:**

```js
Ca("/status")({ component: StatusPage })
const serviceTone = { operational: "text-[#4a8c5c]", degraded: "text-[#b8860b]", outage: "text-[#c0392b]", unknown: "text-gray-400" }
const overall = { operational: { label: "All Systems Operational", bg: "bg-[#6ba368]" }, degraded: { bg: "bg-[#daa520]" },
                  partial_outage: { bg: "bg-[#e67e22]" }, major_outage: { bg: "bg-[#c0392b]" } }
jsx("h1", { className: "text-[22px] font-semibold text-gray-900 tracking-normal" })
// root, tooltip and labels:
"min-h-screen bg-grey-0" · "rounded-md bg-gray-900 px-3 py-2 text-xs text-white shadow-lg" · "text-[13px] text-gray-400" · "text-[11px] text-gray-500"
```

**Impact:**
- A private status palette of 6 hex values (used in 7 classes) duplicates `status-done`, `status-attention` and `status-failed`.
- Tailwind's cool `gray` family sits against Creao's warm greys.
- Font sizes are arbitrary: 11, 13, 15 and 22 px.
- The page stays light in dark mode only because `--color-grey-0` is never redefined by a dark theme. That's an accident, not a design decision.
- This page accounts for 7 of the 11 live `T1.3` violations. The other 4 are three decorative macOS window dots (`bg-[#ff5f57]/60`, …), which belong under `src/assets/brands/`, and one `ring-offset-[#fafafa]`.

**Breaks:** P7.2, C5.1, C8.1, T1.1, T1.3, T1.5, T3.1.

**How enforcement would have caught it:**
- **Gate 1** blocks the commit: `[T1.3] text-[#c0392b]`, `[T1.3] bg-[#6ba368]`, `[T3.1] text-[22px]`, `[T1.1] text-gray-900`, `[C8.1]` (real output).
- **Gate 3:** the dark-mode snapshot.

**Fix:** rebuild as P7, with `StatusPill` tones for each service and day, and `PageHeader`.

---

### F-08 · Persistent notices built from raw palette colours · S2

**Where:** the pending account deletion banner (`accountSettings.deleteAccount.pendingBanner`).

**Evidence:**

```js
"… border-b border-amber-300 bg-amber-50 px-4 py-2 text-sm dark:border-amber-900/60 dark:bg-amber-950/30"
icon:   "size-4 shrink-0 text-amber-600 dark:text-amber-400"
text:   "text-center text-amber-900 dark:text-amber-200"
jsxs("button", { className: "inline-flex … font-medium text-amber-900 underline … hover:text-amber-700 … dark:text-amber-200" })
```

**Scale:** 370 raw palette classes across 20 hues, with `amber` the most common. Most of them describe states and notices that `status-*` tokens and the Callout recipe already cover.

**Breaks:** C7.2, C2.1, T1.1, T2.1.

**How enforcement would have caught it:** Gate 1 reports 8 errors on the rebuilt banner, including `[C2.1] Styled <button>` (real output). The full original has more, because it also uses amber icon and hover classes.

**Fix:** use the Callout recipe with the `attention` tone, and `Button variant="link"`.

---

### F-09 · Colours in JavaScript, invisible to any CSS check · S2

**Evidence:**

```js
// Billing → credits statement chart
jsx(Area, { type: "monotone", dataKey: "credits", stroke: "#6366f1", activeDot: { stroke: "#6366f1", fill: "#fff" } })

// App header icon tile (apps namespace)
const color = appColor || "#6b7280";
jsx("div", { className: "flex size-10 … rounded-lg", style: { backgroundColor: `${color}20` } })
```

**Impact:**
- `#6366f1` is Tailwind indigo-500, even though `--color-credit-*` tokens exist for exactly this chart.
- `#6b7280` is Tailwind gray-500.
- `${color}20` appends a hex alpha. It breaks for any colour that isn't 6-digit hex, and it ignores dark mode.

The bundle holds 640 hex literals in props and styles. Most are third-party brand marks (Google `#4285F4`, PayPal `#003087`, Mastercard `#EB001B`), which are legitimate under `src/assets/brands/`.

**Breaks:** T1.7, GU7.

**How enforcement would have caught it:** **Gate 1 only.** `[T1.7] Hex colour in code` fires on both (real output). None of this reaches CSS, so Gate 2 can't see it. That's why lint and the build scan are both needed.

**Fix:** use `stroke="var(--color-credit-purchased)"`, and `bg-tile-*` / `text-tile-*-fg` for icon tiles.

---

### F-10 · Theme-unsafe greys and 489 manual dark patches · S3

**Evidence:** 264 `grey-*` classes and 489 `dark:` colour overrides. A typical case is the agent detail run row:

```js
className: "overflow-hidden rounded-[6px] bg-grey-100 dark:bg-surface-alt"
```

**Root cause:** both dark themes redefine only `grey-300`, `-500` and `-600` (F-17). `bg-grey-100` stays `#f8f6f3` in dark mode, so every use needs a hand-written dark patch.

**Breaks:** T1.2, T2.1.

**How enforcement would have caught it:**
- **Gate 1:** `[T1.2] bg-grey-100`, `[T2.1] dark:bg-surface-alt`, `[T4.1] rounded-[6px]` (real output).
- **Gate 2** holds the totals at 264 and 489.

**Fix:** `bg-surface-alt` alone. It already adapts.

---

### F-11 · Typography sprawl · S3

- **44 arbitrary font-size classes with 32 distinct values (31 distinct pixel sizes):** 7, 8, 9, 10, 10.5, 11, 11.5, 12, 12.5, 12.8, 13, 13.5, 14, 14.5, 15, 16, 17, 18, 20, 22, 26, 28, 31, 34, 37, 39, 40, 44, 45, 51 and 52 px.
- **Four arbitrary weights:** `font-[450]`, `font-[500]`, `font-[550]` and `font-[600]`.
- **36 arbitrary leading or tracking values.**
- **Six font families declared:** Instrument Sans, Inter (fallback), DM Sans (display), EB Garamond (title), JetBrains Mono and DM Mono (data). v2 adds the system stack.

**Breaks:** T3.1–T3.4. **Caught by:** Gate 1, Gate 2 (44/4/36), Gate 3 D3 (computed font size must be a token size).

### F-12 · Shape and depth sprawl · S3

- **23 arbitrary radius classes with 13 distinct values** (1–22 px), plus 39 generic `rounded-*` classes that don't adapt between modes.
- **47 hand-written shadows with 42 distinct values**, plus 15 generic Tailwind shadows. The system has 6 shadow tokens.
- Many shadows are written twice, once with a `dark:` twin.

**Breaks:** T4, T5. **Caught by:** Gate 1, Gate 2, Gate 3 D4.

### F-13 · No layer scale · S3

22 z-index classes sit outside the feature scale (`z-0`, `z-10`, `z-20`). Across all classes there are 18 distinct levels: 0, 1, 2, 3, 4, 5, 10, 15, 20, 30, 40, 50, 60, 80, 100, 200, 210 and 300. Level 1 is even written two ways, `z-1` and `z-[1]`. Radix overlays portal at `z-50`, so anything at 60–300 can cover menus and tooltips.

**Breaks:** T6.1. **Caught by:** Gate 1, Gate 2, Gate 3 D5.

### F-14 · No width system · S3

74 arbitrary max-width classes with 68 distinct values. They range from truncation limits (`max-w-[120px]`) to page containers: `40.2rem`, `46.2rem`, `46.8rem`, `52.8rem`, `58rem`, `70rem`, `1080px` and `1600px`. Meanwhile the workspace chat layout uses `max-w-7xl` and the room composer `max-w-3xl`. Every feature picks its own width.

**Breaks:** T9.1, L2. **Caught by:** Gate 1, Gate 2. **Fix:** `Page width=` (guideline §4.2).

### F-15 · Motion sprawl · S3

23 custom duration, easing or animation classes, with 8 distinct `cubic-bezier` curves, against 2 easing tokens.

**Breaks:** T8.1. **Caught by:** Gate 1, Gate 2.

### F-16 · Legacy layers still live · S3

- **61 `ox-*` classes.**
- **`.auth-legacy-theme`** wraps login and signup. It's effectively a fifth theme, with its own background (`#f6f6f5`) and focus ring (`#475569`).
- **47 `white` and `black` classes.**

**Breaks:** T1.4, T1.5, P7.1. **Caught by:** Gate 1, Gate 2.

### F-17 · Defects in the token library itself · S3

These are reported to the library owner. They aren't redesigned here.

| Defect | Evidence | Effect |
|---|---|---|
| Grey ramp isn't monotonic | Classic `grey-600 #46423f` is darker than `grey-700 #4a463f`. In v2, `grey-600` = `grey-700` = `#5f5e5b`. | A "darker" step isn't darker |
| Dark ramps are incomplete | Both dark themes redefine only `grey-300`, `-500` and `-600` | The cause of F-10's 489 patches |
| Hover inverts in classic dark | `--color-action-strong: var(--color-foreground)` (light, `#ece2dd`), but `--color-action-strong-hover: #16181ae6` (near-black) | A strong action flips from light to black on hover *(derived)* |
| Light card on a dark page | `.dark { --color-embedded-card: #efebe6 }` on a `#161615` background | Confirm whether intended |
| Aliases | `#fefdfb` has 6 names (`surface`, `card`, `embedded-card`, `popover`, `grey-0`, `ox-paper`). `muted` = `muted-foreground` in every mode. | People and agents pick between them at random |
| Categorical colours collapse in v2 | 13 tile names map to 6 colours (violet = indigo, amber = orange, green = teal = olive, red = pink, navy = sky, neutral = slate) | Categories become indistinguishable in v2 |
| v2 doesn't define `memory-*` or `credit-*` | They fall back to classic values in v2 | Confirm whether intended |
| Icon button sizes ignore the mode | Button `size` maps `icon` → `size-8`, `icon-md` → `size-9` and `icon-lg` → `size-10` in every mode, while text sizes use `h-(--button-h…)`. One component maps a default-size icon-only button to `icon-md`. | In v2 that icon button is 36 px beside 28 px text buttons *(derived)* |

**Caught going forward by:** the token contract test (guideline §7.4): monotonic ramps, complete mode coverage and aliased `--jr-*` values.

### F-18 · Phantom CSS from non-UI text · S3

193 arbitrary-value classes in the shipped CSS aren't referenced by any shipped code. Examples are `text-[Npx|rem|em]`, `z-[N]`, `text-[0-9a-f]` and `text-[color:var(--color-fg)]/60` (that last one points at a token that doesn't exist). Tailwind is scanning docs, comments or agent instructions.

Strings like `text-[Npx|rem|em]` read like a written rule about arbitrary values. So a rule seems to exist somewhere in prose, and nothing enforces it.

**Caught by:** Gate 2 (`phantomClasses` is ratcheted). **Fix:** exclude non-UI folders with Tailwind's `@source not`.

---

## 4. Proof: the enforcement mechanism on real output

Everything in this section was run on the build fetched 2026-11-09. It was re-run on 2026-11-10 with the code as it now appears in guideline v1.1, extracted straight from the markdown file.

### 4.1 Gate 2 on the production build

`node design/scan.mjs <production assets>`. This output became `design/baseline.json`:

```
T1.1    370    T1.5     47    T3.3     36    T6.1     22    T9.1     74    T4.2     39    GU3      47
T1.2    264    T2.1    489    T4.1     23    T7.1    105    C2.3     13    T7.2     63    T1.6     20
T1.3     11    T3.1     44    T5.1     47    T8.1     23    T5.2     15    C1.2     81    phantom 193
T1.4     61    T3.2      4
```

### 4.2 The ratchet blocks a regression

A simulated PR added one arbitrary font size and one hex background to the same build:

```
$ node design/scan.mjs dist/assets
T1.3               12  base    11  <-- REGRESSION
T3.1               45  base    44  <-- REGRESSION
FAIL: design metrics increased. Fix, or file an exception (guideline 7.7).
exit code 1
```

Two more runs tested the slot count (`C1.2`):

```
# 1. the same build with the stock copy's region cut out of index.js (the F-01 fix)
C1.2               17  base    81
#    (Only the C1.2 line is shown. This run also reported phantomClasses 197 against 193. That's an
#    artifact of the simulation: cutting JS without rebuilding leaves the copy's CSS classes
#    unreferenced. A real build regenerates the CSS.)

# 2. a PR that adds one more Button definition
C1.2               82  base    81  <-- REGRESSION
C1.2 primitive defined again: button
FAIL: design metrics increased. Fix, or file an exception (guideline 7.7).
exit code 1
```

### 4.3 Gate 1 on code rebuilt from production

The guideline's ESLint config (guideline §7.3, used unchanged: ESLint 9.39 with `typescript-eslint`) ran on code rebuilt from the verbatim production class strings quoted in F-02 and F-04 to F-10. Lines are abridged: where two errors share a line here, the real output prints one error per line. The right-hand column maps each error to its finding.

```
  3:1   error  '@radix-ui/react-dialog' import is restricted … [C1.1] Import the wrapper from src/components/ui  no-restricted-imports
  7:25  error  [T4.1] rounded-[6px] - Arbitrary radius. Use a radius token (guideline 2.4)               creao/classes      F-02
  7:25  error  [T1.4] border-ox-rule - ox-* is a frozen legacy palette …                                 creao/classes      F-02
 12:10  error  [C8.1] Page titles come from <PageHeader>. Do not render <h1> directly                     no-restricted-syntax  F-04
 12:24  error  [T3.1] text-[22px] - Arbitrary font size. Use the type scale (guideline 2.3)              creao/classes      F-04/F-07
 12:24  error  [T1.1] text-gray-900 - Raw Tailwind palette colour …                                      creao/classes      F-04/F-07
 18:61  error  [C6.1] Zero-data states render <EmptyState>                                               no-restricted-syntax  F-05
 27:5   error  [C4.2] Forms use <ResponsiveDialog>; confirmations use <AlertDialog> …                    no-restricted-syntax  F-06
 36:21  error  [T1.3] text-[#c0392b] - Arbitrary colour value …                                          creao/classes      F-07
 37:20  error  [T1.3] bg-[#6ba368] - Arbitrary colour value …                                            creao/classes      F-07
 45:20  error  [T1.1] border-amber-300 - Raw Tailwind palette colour …                                   creao/classes      F-08
 45:20  error  [T1.1] bg-amber-50 - Raw Tailwind palette colour …                                        creao/classes      F-08
 45:20  error  [T1.1] dark:border-amber-900/60 …   [T2.1] dark:border-amber-900/60 …                     creao/classes      F-08
 45:20  error  [T1.1] dark:bg-amber-950/30 …       [T2.1] dark:bg-amber-950/30 …                         creao/classes      F-08
 46:15  error  [C2.1] Styled <button>. Use <Button> (asChild for custom elements)                        no-restricted-syntax  F-08
 46:25  error  [T1.1] text-amber-900 - Raw Tailwind palette colour …                                     creao/classes      F-08
 55:57  error  [T1.7] Hex colour in code …  (stroke="#6366f1")                                           no-restricted-syntax  F-09
 58:22  error  [T1.7] Hex colour in code …  ('#6b7280' tile fallback)                                    no-restricted-syntax  F-09
 64:25  error  [T4.1] rounded-[6px] …  [T1.2] bg-grey-100 …  [T2.1] dark:bg-surface-alt …                creao/classes      F-10
✖ 23 problems (23 errors, 0 warnings)
```

A file written to the guideline produced **0 problems**. It used `rounded-(--card-radius)`, `hover:bg-secondary`, `z-10`, `<EmptyState data-testid="apps-empty">` and the allowed safe-area `pb-[calc(0.75rem+var(--sab))]`. The rules catch real violations without flagging compliant code. They also leave UI copy alone: a string such as "Totals are rounded to the nearest credit" passes. The full `/status` tone map from F-07 is still caught: all three hex values plus `text-gray-400`.

### 4.4 Coverage matrix

| Finding | Gate 1 (lint) | Gate 2 (scan, ratchet, slots, tokens) | Gate 3 (render matrix) | Gate 4 (review) |
|---|---|---|---|---|
| F-01 Two libraries | C1.1 | **`C1.2` slot count** | v2 snapshot | CODEOWNERS |
| F-02 Generated UI off-system | **T4.1, T1.4** | **T1.6, GU3** | **v2-light fixture** | |
| F-03 Mode-specific sizes | **T4.2** (advisory) | T4.2 count | **classic and v2 snapshots**; D10 for button heights | |
| F-04 Page titles | **C8.1, T3.1** | T3.1 count | **D2** | |
| F-05 Empty states | **C6.1** | | **D6** | |
| F-06 Overlay split | **C4.2** | | **D9 (mobile)** | |
| F-07 `/status` | **T1.3, T1.1, T3.1, C8.1** | T1.3 count | dark snapshot | |
| F-08 Raw-palette notice | **T1.1, T2.1, C2.1** | T1.1 count | | |
| F-09 Colour in JS | **T1.7** (only gate) | — | snapshot | |
| F-10 Greys and dark patches | **T1.2, T2.1** | **ratchet** | 4-mode snapshots | |
| F-11–F-16 Systemic sprawl | **T3–T9, T1.4, T1.5** | **ratchet** | D3–D5 | |
| F-17 Token defects | | **Token contract test** | 4-mode snapshots | Library owner |
| F-18 Phantom CSS | | **phantomClasses** | | |

**Bold** marks the gate that fails first. Every finding is caught by at least one deterministic gate, either lint or the build scan, before merge.

---

## 5. What the gates will not catch

| Risk | Why the gates miss it | Mitigation |
|---|---|---|
| Choosing the wrong pattern in a structurally valid way, for example a card grid where people need to compare columns | It's a judgement call | Gate 4 review; the P2.2 grid-or-table rule; the PR template's pattern declaration |
| A `className` on a library component that uses allowed tokens but overrides its design, for example `className="h-12"` on `Button` | Each class is valid on its own | Gate 3 D10 checks every button's rendered height against its `data-size` (C3.2); review (C2.4) |
| Class names built dynamically (`"bg-" + tone`) | Neither lint nor the scan sees the final string, and Tailwind doesn't generate the class | It renders unstyled, which Gate 3 snapshots show |
| Screens that aren't exemplars | Gate 3 only renders exemplars | Gates 1 and 2 still cover all code. Add each new pattern instance to `exemplars.json`. |
| Copy, information hierarchy, the feel of motion | Not mechanically checkable | Design review; C8.3 |

---

## 6. Fix order

| # | Finding | Change | Size | Proof it's done |
|---|---|---|---|---|
| 1 | F-02 | GU2 alias table, delete `.dark .jr-block`, demote block headings | S (CSS only) | `T1.6` −1; v2-light fixture snapshot matches the surrounding chat |
| 2 | F-01 | Add `CardHeader`, `CardTitle`, `CardDescription` and `Callout` to the library; point the catalog at `src/components/ui`; delete the stock copy | M | `C1.2` 81 → 17; stock Button gone from the bundle |
| 3 | F-03 | Composer tray to `rounded-(--composer-radius)`; T4.2 on P1 surfaces | S | P1 snapshots in classic and v2 |
| 4 | F-06 | 7 form dialogs to `ResponsiveDialog` | S | D9 passes on mobile |
| 5 | F-07, F-08, F-09 | Rebuild `/status` on tokens; Callout recipe; chart and tile colours from tokens | S | `T1.3` 11 → 4; `T1.7` clean |
| 6 | F-04 | Adopt `PageHeader` on P2, P3 and P5 screens | M | D2 passes; raw `h1` count → 0 outside P7 one-offs |
| 7 | F-05 | `EmptyState size="sm"`, then migrate 27 sites | M | D6 passes; C6.1 clean |
| 8 | F-17 | Token fixes (library owner) | M | Token contract test passes; `T2.1` falls as patches are deleted |
| 9 | F-10 to F-16, F-18 | Sprint-by-sprint burn-down | Ongoing | `baseline.json` decreases every sprint (target −25% per quarter on T1.1, T1.2, T2.1) |
