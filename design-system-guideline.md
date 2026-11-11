# Creao Product Design Guideline

| | |
|---|---|
| **Version** | 1.1 (2026-11-10) |
| **Owner** | Product Design (reports to CTO) |
| **Applies to** | Every screen, overlay and agent-generated UI block in the Creao web and desktop app, whether a person or a coding agent wrote it |
| **Builds on** | The existing token set (Tailwind v4 theme in the app's CSS) and component library (`src/components/ui`). This document does not redesign either. It says how to compose them and how compliance is checked. |
| **Verified against** | Production build at agent.creao.ai, `index-BTZeeHx5.js` / `index-BXYlNekV.css`, fetched 2026-11-09. See `compliance-audit.md`. |

**Contents:**
- [0. How to use this document](#0-how-to-use-this-document)
- [1. What exists today](#1-what-exists-today)
- [2. Token rules (T)](#2-token-rules-t)
- [3. Component rules (C)](#3-component-rules-c)
- [4. Layout system (L)](#4-layout-system-l)
- [5. Screen patterns (P1–P8)](#5-screen-patterns-p1p8)
- [6. Agent-generated UI (GU)](#6-agent-generated-ui-gu)
- [7. Enforcement](#7-enforcement)
- [8. Decisions for the CTO](#8-decisions-for-the-cto)
- [9. Library requests](#9-library-requests)

---

## 0. How to use this document

### 0.1 Who this is for

- **Engineers** building or changing UI.
- **Coding agents** working in this repo. Load this file before writing UI code. Every rule has an ID and a check, so you can verify your own output.
- **Reviewers.** Cite rule IDs in review comments, for example "T4.1: use `rounded-(--card-radius)`".

### 0.2 Keywords

- **MUST / MUST NOT**: a hard rule. An automated gate blocks the merge, or CODEOWNERS review does.
- **SHOULD**: the default. Deviating needs a one-line reason in the PR description.
- **MAY**: allowed, not required.

When a rule and your instinct disagree, the rule wins. When a rule blocks you, use the exception process (§7.7). Never work around a rule.

### 0.3 The workflow (every UI change)

1. **Pick the pattern.** Classify every screen you touch as one of P1–P8 using the decision tree in §5.0. If nothing fits, stop and open a design review (§5.10). Do not invent a layout.
2. **Compose from what exists.** Use the pattern components (§4.7) for page structure and the UI library (§3) for everything inside it. Do not create new primitives.
3. **Use only tokens.** Colour, type, radius, shadow, layer, spacing, motion and width all come from §2. Never use arbitrary `[...]` values, raw palette colours, `grey-*`, `ox-*`, hex, or `dark:` colour patches.
4. **Run the checks.** Run `npm run design:check` (lint + build scan) and fix every error. Do not suppress `creao/*` rules.
5. **Declare it.** Fill in the PR template's Design section: the pattern, any new components (expected: none) and any exceptions (expected: none).

### 0.4 The 12 rules that prevent most problems

| # | Do | Never | Rule |
|---|---|---|---|
| 1 | `bg-surface`, `text-muted`, `border-edge` | `bg-amber-50`, `text-gray-900`, `bg-grey-100`, `bg-[#6ba368]` | T1 |
| 2 | Let tokens handle dark mode | `dark:bg-…`, `dark:text-…` | T2.1 |
| 3 | `text-xs` / `text-sm` / `text-base` | `text-[13px]`, `font-[550]` | T3 |
| 4 | `rounded-(--card-radius)` and other component radius tokens | `rounded-[6px]`, `rounded-xl` on cards | T4 |
| 5 | `shadow-card`, `shadow-card-hover` | `shadow-[0_1px_3px…]`, `shadow-lg` | T5 |
| 6 | `z-0`, `z-10`, `z-20` | `z-[200]`, `z-50` in feature code | T6.1 |
| 7 | `<PageHeader title=…>` | a raw `<h1 className=…>` | C8.1 |
| 8 | `<Button variant=… size=…>` | a styled `<button>`, or `h-9` on a Button | C2.1, C3.2 |
| 9 | `<EmptyState title=…>` | a `<p>` that says "Nothing here" | C6.1 |
| 10 | `<ResponsiveDialog>` for forms, `<AlertDialog>` for confirms | a plain `<Dialog>` holding a form | C4.2 |
| 11 | `<StatusPill tone="running">` | coloured text or dots for state | C5.1 |
| 12 | Import from `src/components/ui` | `@radix-ui/*`, `vaul`, `cmdk`, or a second component copy | C1 |

### 0.5 Paths used in this document

| Path | Contents |
|---|---|
| `src/components/ui/` | The existing UI library. Only the library owner changes it (CODEOWNERS). |
| `src/components/patterns/` | Layout pattern components (§4.7). New; owned by Product Design. |
| `design/` | `rules.json`, `match.mjs`, `eslint-plugin.mjs`, `scan.mjs`, `baseline.json`, `slots.json`, `exemplars.json` (§7) |
| `e2e/design/` | Render-matrix tests (§7.5) |

If the repo's paths differ, edit `design/rules.json → scope` and this table. Nothing else in the document depends on them.

Component names in this document are the PascalCase form of each primitive's `data-slot`. For example, `data-slot="empty-state"` is `EmptyState`, and `data-slot="responsive-dialog-content"` is `ResponsiveDialogContent`.

---

## 1. What exists today

These facts were read from the production build. Rules below refer back to them.

### 1.1 Stack

- React with Tailwind CSS v4. Tokens are CSS custom properties exposed as utilities, for example `bg-surface` and `rounded-(--card-radius)`.
- A Radix-based component library that tags every part with `data-slot`.
- `sonner` for toasts.
- The Creao `Button` is used at **1,278** call sites. It is the canonical library.

### 1.2 Four render modes

Creao ships two design modes. Each has a light and a dark theme.

| Mode | How it's set | Character |
|---|---|---|
| `classic` (default) | no attribute | Warm paper (`#f8f6f3`) and ink (`#16181a`). Instrument Sans. Primary = ink. |
| `v2` (opt-in) | `<html data-design="v2">`, persisted as `localStorage['creao.designSystem']` | White and neutral grey (`#37352f`). System sans. Primary = indigo `#4f5bd5`. |
| `dark` (either mode) | `.dark` on `<html>`, persisted as `localStorage['creao-ui-theme']` (`light`, `dark` or `system`) | Separate dark values for both modes |

**Every screen MUST render correctly in all four combinations:** classic light, classic dark, v2 light and v2 dark. The component dimension tokens differ by mode, so any hard-coded size is right in at most one of them.

| Token | classic | v2 |
|---|---|---|
| `--button-h` / `-sm` / `-xs` / `-lg` | 36 / 32 / 24 / 44 px | 28 / 24 / 20 / 36 px |
| `--input-h` / `-sm` | 40 / 32 px | 32 / 28 px |
| `--card-radius` | 8 px | 4 px |
| `--composer-radius` | 24 px | 8 px |
| `--callout-radius` | 8 px | 4 px |
| `--chip-radius` | 6 px | 3 px |
| `--pill-radius` | full | 6 px |
| `--page-title-size` / weight | 24 px / 600 | 40 px / 700 |
| `--status-text` | 11 px | 12 px |

Two details the tokens hide:
- On touch screens (`@media (pointer: coarse)`), v2 raises `--button-h` to 36 px and `--button-h-sm` to 32 px, the classic values. Using the `size` prop gets this for free; a hard-coded height doesn't.
- Icon-only button sizes are fixed in every mode: `icon-xs` 24 px, `icon` and `icon-sm` 32 px, `icon-md` 36 px, `icon-lg` 40 px. They don't follow `--button-h` (library request in §9).

### 1.3 Frozen and deprecated layers

Do not add usages of any of these. Migrate existing ones when you touch the file.

| Layer | Status | Replacement |
|---|---|---|
| `ox-*` colours (`ox-paper`, `ox-ink`, `ox-rule`, `ox-btn-bg`, …) | Frozen | `surface`/`card`, `foreground`, `edge`/`border`, `primary` (see the table in §2.1) |
| `.auth-legacy-theme` (login and signup) | Deprecated | The app's normal modes (P7) |
| The second, stock shadcn/ui component copy bundled with the generative-UI catalog | Delete | `src/components/ui` (C1.2, GU1) |
| Tailwind default palettes (`gray-*`, `amber-*`, `emerald-*`, …) in feature code | Banned | Semantic tokens (T1) |

---

## 2. Token rules (T)

All rules in this section apply to feature code: everything under `src/` except `src/components/ui/`, `src/styles/` and `src/assets/brands/`. They are enforced by Gate 1 (lint) and counted by Gate 2 (build scan).

### 2.1 Colour (T1)

**T1 (MUST):** feature code uses semantic colour tokens only, chosen by role:

| Role | Use | Notes |
|---|---|---|
| Page background | `bg-background` | `Page` sets it. Don't repeat it. |
| App chrome | `bg-sidebar`, `bg-chrome` | Shell only |
| Raised surface: cards, panels, artifact panel | `bg-surface`; `bg-card` inside `Card` only | |
| Inset or well: table header, code, logs, read-only fields | `bg-surface-alt` | |
| Hover on a surface | `hover:bg-secondary` | Never a solid `bg-accent` (trap 1 below) |
| Selected or pressed on a surface | `bg-surface-elevated` (selected), `bg-surface-raised` (pressed) | |
| Persistent notice | Callout recipe (C7.2) | `bg-callout border-callout-border` |
| Primary text | `text-foreground` | |
| Secondary text: metadata, descriptions, placeholders | `text-muted` | `text-muted-foreground` is the same value. Write `text-muted`. |
| Error and success text | `text-destructive`, `text-success` | |
| Dividers | `border-border`, `divide-border` | |
| Edges of panes, cards and blocks | `border-edge` | |
| State of an agent, run, task or automation | `StatusPill tone=…` | Never apply `status-*` classes by hand |
| Categorical identity: agent and app icon tiles, tags | `bg-tile-{name} text-tile-{name}-fg` | In v2 the 13 tile names collapse to 6 distinct colours, so never rely on tile colour alone to tell items apart |
| Chart series and swimlanes | `var(--color-lane-1)` … `var(--color-lane-8)`, in order | |
| Domain palettes | `memory-*` (memory types), `credit-*` (billing) | Only inside their own domain |

| ID | Rule | Check |
|---|---|---|
| T1.1 | MUST NOT use Tailwind palette colours (`red-500`, `gray-900`, `amber-50`, …) | Gate 1 + Gate 2 |
| T1.2 | MUST NOT use the `grey-*` ramp in feature code. It is not theme-safe: dark modes redefine only grey-300, -500 and -600. | Gate 1 + Gate 2 |
| T1.3 | MUST NOT use arbitrary colours (`bg-[#…]`, `text-[rgb(…)]`, `bg-[color-mix(…)]`, `text-[color:var(…)]`) | Gate 1 + Gate 2 |
| T1.4 | MUST NOT add `ox-*` usages | Gate 1 + Gate 2 |
| T1.5 | MUST NOT use `white`/`black`. Use the `*-foreground` partner of the surface you're on. Brand marks under `src/assets/brands/` are exempt. | Gate 1 + Gate 2 |
| T1.6 | CSS custom properties outside the theme files MUST alias a token (`--x: var(--color-…)`), never a literal | Gate 2 |
| T1.7 | MUST NOT put hex colours in JS/TS: props, inline `style`, chart config, or `${color}20` alpha concatenation. For SVG and canvas use `var(--color-…)` or read it with `getComputedStyle`. | Gate 1 |

**Token semantics traps.** Coding agents default to stock shadcn/ui conventions. These are the places where Creao's tokens mean something different:

| # | Stock instinct | In Creao, write | Why |
|---|---|---|---|
| 1 | `hover:bg-accent` as a subtle hover | `hover:bg-secondary` (`hover:bg-sidebar-hover` in the sidebar) | `accent` is ink: `#16181a` in classic, `#37352f` in v2. Solid `bg-accent` is a near-black block. Low-alpha tints (`bg-accent/5`, `/10`) are fine. |
| 2 | `bg-muted` as a subtle background | `bg-surface-alt` | `muted` is a text colour (`#6e6863`) |
| 3 | `text-muted-foreground` | `text-muted` | Same value in every mode. Use one spelling. |
| 4 | `bg-primary text-white` for a CTA | `<Button>` (default variant) | Primary is ink in classic and indigo in v2. Its foreground is cream in classic, not white. |
| 5 | `h-9` / `h-10` on a button or input | the `size` prop | Heights are mode tokens (§1.2) |
| 6 | `gray-*` | a semantic token | Tailwind's `gray` is cool. Creao's neutrals are warm. |
| 7 | `rounded-xl` on a composer, card or tray | `rounded-(--composer-radius)`, `rounded-(--card-radius)` | Radius differs by mode |

### 2.2 Dark mode (T2)

| ID | Rule | Check |
|---|---|---|
| T2.1 | MUST NOT write `dark:` colour overrides (`dark:bg-*`, `dark:text-*`, `dark:border-*`, …) in feature code. Semantic tokens already adapt. If one renders wrong in dark, file a token fix (§9) instead of patching the screen. | Gate 1 + Gate 2 |
| T2.2 | MUST verify the change in all four modes (§1.2). Gate 3 screenshots exemplar routes in each. | Gate 3 |

### 2.3 Typography (T3)

| Use | Class | Notes |
|---|---|---|
| Metadata, captions, timestamps, badge text, table meta | `text-xs` (12 px) | |
| Default UI text: buttons, labels, table cells, list rows, descriptions | `text-sm` (14 px) | The app's base size |
| Reading text: chat messages, generated documents, long descriptions | `text-base` (16 px) or `text-body-lg` | |
| Section title (PageSection `h2`) | `text-base font-semibold` | Set by `PageSection` |
| Dialog title, EmptyState title | `text-lg font-medium` | Set by the library |
| Page title | `PageHeader` only (`--page-title-*` tokens) | Never by hand |
| Hero text: registered P7 one-offs and hero slots (T3.4) | `text-2xl`–`text-5xl` | The display and serif families are allowed only here |

| ID | Rule | Check |
|---|---|---|
| T3.1 | MUST NOT use arbitrary font sizes (`text-[13px]`, `text-[0.8rem]`) | Gate 1 + Gate 2 |
| T3.2 | Weights MUST be `font-normal`, `font-medium` or `font-semibold`. Bold comes only from tokens. Never `font-[450]`. | Gate 1 + Gate 2 |
| T3.3 | MUST NOT use arbitrary leading or tracking. Allowed: `leading-tight`, `-snug`, `-normal`, `-relaxed`; `tracking-tight` (titles ≥ `text-2xl`); `tracking-wide` (uppercase `text-xs` eyebrows only). | Gate 1 + Gate 2 |
| T3.4 | Families: `font-sans` for all UI, `font-mono` for code, IDs, keys and logs, and `font-data` for tabular numbers only, never headings. `font-display`, `font-title` and `font-serif` are reserved for P7 heroes, the EmptyState glyph and registered hero slots. A hero slot is one serif headline per screen; today there are two, the chat home greeting ("How can I help, …?") and the Discover hero. A new hero slot is added through Gate 4 review. | Gate 3 (D3) + review |
| T3.5 | SHOULD use `tabular-nums` on numbers that update: durations, credits, counters | Review |

### 2.4 Radius (T4)

| Surface | Class |
|---|---|
| Card, panel, list container, generated block | `rounded-(--card-radius)` |
| Callout or notice | `rounded-(--callout-radius)` |
| Chip, tag, hover row, small interactive row | `rounded-(--chip-radius)` |
| Badge, count bubble | `rounded-(--badge-radius)`, `rounded-(--count-radius)` |
| Pill, segmented control | `rounded-(--pill-radius)`, `rounded-(--segmented-radius)` |
| Composer and anything attached to it (queued tray, mention tray, attachment strip) | `rounded-(--composer-radius)` |
| Avatars, dots, circular icon buttons | `rounded-full` |
| Edge-to-edge content (a table flush inside a card) | `rounded-none` |

| ID | Rule | Check |
|---|---|---|
| T4.1 | MUST NOT use arbitrary radii (`rounded-[6px]`) | Gate 1 + Gate 2 |
| T4.2 | MUST use the component radius tokens above. Generic `rounded`, `rounded-xs`…`rounded-4xl` belong to the library, because they don't adapt between classic and v2. | Gate 1 (warning now; error after rollout week 4) |

### 2.5 Elevation (T5)

Allowed: `shadow-card` (cards at rest), `hover:shadow-card-hover` (interactive cards), `shadow-composer`, `shadow-segmented`, `shadow-panel`. `shadow-popover` is for library overlays only. `shadow-none` is always allowed.

| ID | Rule | Check |
|---|---|---|
| T5.1 | MUST NOT write shadows by hand (`shadow-[0_1px_3px…]`) | Gate 1 + Gate 2 |
| T5.2 | MUST NOT use Tailwind's generic `shadow-sm`…`shadow-2xl` | Gate 1 (warning now; error after week 4) |

### 2.6 Layers (T6)

| Layer | Class | Who may use it |
|---|---|---|
| Content | `z-0` | Feature code |
| Sticky inside a pane: table header, toolbar, ChatHeader | `z-10` | Feature code |
| Floating inside a pane: jump-to-latest, resize handles, floating action | `z-20` | Feature code |
| App shell sidebar | `z-30` | Shell only |
| Mobile navigation drawer | `z-40` | Shell only |
| All portalled overlays: dialogs, sheets, popovers, menus, tooltips, toasts | `z-50` | Library only |

| ID | Rule | Check |
|---|---|---|
| T6.1 | Feature code MUST use only `z-0`, `z-10`, `z-20` and `z-auto`. That excludes in-between steps such as `z-1` or `z-[15]`. Nothing anywhere, library included, may exceed `z-50`. The existing `z-[60]`…`z-[300]` must be migrated (§9). | Gate 1 + Gate 2 + Gate 3 (D5) |

### 2.7 Spacing (T7)

| ID | Rule | Check |
|---|---|---|
| T7.1 | MUST NOT use arbitrary spacing (`mt-[13px]`, `top-[4.25rem]`). The only exception is safe-area maths using the shell variables: `pb-[calc(0.75rem+var(--sab))]` with `--sat`, `--sab`, `--sal`, `--sar`, `--sat-content`, `--keyboard-height` or `--top-chrome-h`. | Gate 1 + Gate 2 |
| T7.2 | Rhythm steps are `1` (4 px), `1.5`, `2` (8), `3` (12), `4` (16), `6` (24), `8` (32) and `12` (48). `0.5` and `2.5` are for optical alignment inside a component. Avoid `5`, `7`, `9`, `10`, `11` and `14`. | Gate 1 (warning) |
| T7.3 | Page-level spacing belongs to the pattern components. Feature code MUST NOT add margins between `PageHeader`, `PageToolbar` and `PageSection`. | Review + Gate 3 snapshot |

### 2.8 Motion (T8)

- **Durations:** `duration-150` for hover, press and colour changes. `duration-200` for small overlays and expand/collapse. `duration-300` for panels and sheets.
- **Easing:** `ease-out` for entering, `ease-in-out` for moving or resizing.
- **Reduced motion:** any transform or opacity animation that isn't essential feedback (spinners and skeletons are essential) is wrapped in `motion-safe:`.
- **Keyframes:** named keyframes such as artifact enter, shimmer and thinking belong to the library.

| ID | Rule | Check |
|---|---|---|
| T8.1 | MUST NOT write custom durations, easings or animations (`duration-[260ms]`, `ease-[cubic-bezier(…)]`, `animate-[…]`) | Gate 1 + Gate 2 |

### 2.9 Width (T9)

| ID | Rule | Check |
|---|---|---|
| T9.1 | MUST NOT use arbitrary max-widths. Page width comes from `Page width=…` (§4.2). Dialog width comes from the dialog size set (C4.5). | Gate 1 + Gate 2 |

---

## 3. Component rules (C)

### 3.1 One library (C1)

| ID | Rule | Check |
|---|---|---|
| C1.1 | Feature code MUST import UI primitives only from `src/components/ui`. Never import `@radix-ui/*`, `vaul` or `cmdk` directly. | Gate 1 |
| C1.2 | Each primitive MUST be defined once: a given `data-slot` value is defined in exactly one file. The stock shadcn copy bundled with the generative-UI catalog is deleted, and the catalog imports from `src/components/ui` (GU1). | Gate 2 (`C1.2` slot count, ratcheted) |
| C1.3 | Only the library owner adds primitives (anything that renders a new `data-slot`) | Gate 4 (CODEOWNERS) |

### 3.2 Don't rebuild what exists (C2)

| ID | Rule | Check |
|---|---|---|
| C2.1 | MUST NOT style a raw `<button>`. Use `Button`, with `asChild` to render a link or another element. | Gate 1 |
| C2.2 | MUST use the library component for each job in the table below | Review + Gate 1 where noted |
| C2.3 | MUST NOT restyle library internals from outside: no `[&_[data-slot=…]]:` variants and no descendant CSS on `[data-slot=…]`. If you need a different look, request a variant (§9). | Gate 1 + Gate 2 (`GU3`) |
| C2.4 | `className` on a library component is for placement only: width, flex or grid placement, `shrink-0`, `min-w-0`, responsive visibility. Never colour, radius, type, height or padding. | Review + Gate 3 snapshot |

| Job | Component |
|---|---|
| Any clickable action | `Button` (variants: `default`, `secondary`, `outline`, `ghost`, `link`, `destructive`) |
| Live state of an agent, run, task or automation | `StatusPill` (tones: `running`, `scheduled`, `done`, `failed`, `attention`, `new`, `neutral`) |
| Static attribute: plan, visibility, version, count | `Badge` (`default`, `secondary`, `outline`, `success`, `warning`, `scheduled`, `destructive`, `ghost`) |
| Content container | `Card` with `CardContent`. A card title is an `h3` (C8.2). Creao's library has no `CardHeader`, `CardTitle` or `CardDescription`; those exist only in the stock copy (§9). |
| Zero-data region | `EmptyState` (`title`, `description`, `icon` or `glyph`, `action`) |
| Region loading | `Skeleton` |
| Region failed | `ErrorState`: the existing retry card (`message`, `onRetry`, `retryLabel`) |
| Tabular data | `Table` inside a table container |
| Section switching inside a page | `Tabs variant="line"` |
| Scope filter (All, Mine, Team) | `Tabs variant="pill"` |
| 2–4 mutually exclusive view options (Grid, List) | `Tabs variant="default"`, the segmented control. The library has no `ToggleGroup`. |
| Choose one of many | `Select`; `Command` when searchable or longer than 10 items |
| Row and overflow actions | `DropdownMenu` (right-click: `ContextMenu`) |
| Free text and tag entry | `Input`, `Textarea`, `TaggedInput` |
| Agent message input | `ComposerShell` |
| Navigation trail | `Breadcrumb` |
| Paging | A "Load more" `Button variant="outline"`. The library has no `Pagination`. |
| Disclosure | `Collapsible`, one per section. The library has no `Accordion`. |
| Identity | `Avatar`, `AvatarGroup`, or a tile (`bg-tile-*`) |

`ToggleGroup`, `Pagination`, `Accordion`, `RadioGroup` and `Alert` exist in the bundle only as part of the stock copy that C1.2 deletes. Feature code MUST NOT use them. If one is needed, request it (§9).

### 3.3 Buttons (C3)

| ID | Rule | Check |
|---|---|---|
| C3.1 | MUST have at most one `variant="default"` (primary) button per region: page header, toolbar, dialog footer, card, generated block | Gate 3 (counts `[data-slot=button][data-variant=default]` per region) |
| C3.2 | MUST set size with the `size` prop (`xs`, `sm`, `default`, `lg`, `icon-xs`, `icon-sm`, `icon`, `icon-md`, `icon-lg`), never with `h-*`, `px-*` or `text-*`. Sizes by context: page header and toolbar `default`; table rows and dense lists `sm` or `xs`; dialog footers `default`. | Gate 3 (D10: rendered height matches the token for the button's `data-size`) |
| C3.3 | Icon-only buttons MUST have an `aria-label` and a `Tooltip` | Gate 3 (axe `button-name`) |
| C3.4 | `destructive` is only for the confirming action of an irreversible operation (`AlertDialogAction`) or the trigger inside a Danger zone | Review |
| C3.5 | Other variants: `secondary` or `outline` for secondary actions, `ghost` for toolbar, row and icon actions, `link` for inline text actions. `raised` and `raised-outline` only where `/dev/design-system` documents them. | Review |
| C3.6 | In footers and action bars the primary action goes last (rightmost), Cancel (`ghost`) before it. Labels are verbs: "Create agent", not "OK". | Review |

### 3.4 Overlays (C4)

Pick the overlay by the job:

| Job | Use | Never |
|---|---|---|
| Confirm a destructive or irreversible action | `AlertDialog` with a verb button (`variant="destructive"`) | `Dialog`; a toast-only undo for something irreversible |
| A form task: create, edit, rename, move, share, invite, upgrade or paywall | `ResponsiveDialog`. It becomes a bottom sheet below `md`. | Plain `Dialog`, `Popover` |
| Inspect or edit details while keeping the list visible | `SplitView` aside (§4.6) | A modal |
| One small anchored control (a single field or a short choice) | `Popover`, `DropdownMenu`, `Select`, `Command` | A dialog |
| Extra info on hover | `Tooltip` (labels), `HoverCard` (previews). Never the only route to an action or essential information. | |
| Full-screen viewing of one artifact: deck, DAG, image, video | Full-bleed `Dialog` (P8). The only sanctioned plain `Dialog`. | |
| Success of an async action the user started | Toast (C7.1) | A modal |

| ID | Rule | Check |
|---|---|---|
| C4.1 | Destructive or irreversible confirmations MUST use `AlertDialog`. Workspace and account deletion MUST require typing the name. | Review (all 9 confirms in production already comply) |
| C4.2 | Forms MUST use `ResponsiveDialog`. Plain `Dialog` is allowed only for P8 viewers, each marked with an exception comment (§7.7). | Gate 1 + Gate 3 (D9, mobile) |
| C4.3 | One modal level at most. The only allowed stack is an `AlertDialog` (for example "Discard changes?") opened from a `ResponsiveDialog`. | Review |
| C4.4 | Feature code MUST NOT render `MobileSheet` (or the stock copy's `Drawer`) directly. `ResponsiveDialog` and `SplitView` choose the sheet. | Gate 1 (C1.1 bans `vaul`) + review |
| C4.5 | A form dialog holds one section of at most 7 fields and no live preview. Widths: `sm:max-w-md` (1–3 fields), `sm:max-w-lg` (4–7 fields, the default), `sm:max-w-2xl` (a table or picker grid inside the dialog). Two or more sections, more than 7 fields, or a live preview make it a P6 page. | Gate 1 (T9.1) + review |

### 3.5 Status (C5)

| ID | Rule | Check |
|---|---|---|
| C5.1 | The state of an agent, run, task, step, schedule or service MUST be a `StatusPill`, mapped as in the table below | Review + Gate 1 (T1 stops hand-coloured states) |
| C5.2 | MUST NOT show state by colour alone. A pill always has a label. A bare dot or bar (timeline, uptime chart) needs an adjacent label or tooltip. | Gate 3 (axe) + review |
| C5.3 | `Badge` is for static attributes, never live state | Review |

| Domain state | Tone |
|---|---|
| queued, scheduled, waiting for trigger | `scheduled` |
| running, thinking, in progress, syncing | `running` |
| succeeded, completed, operational, connected | `done` |
| failed, errored, outage, disconnected | `failed` |
| needs approval, needs input, degraded, expiring | `attention` |
| new, unread, just published | `new` |
| paused, disabled, archived, cancelled, unknown | `neutral` |

### 3.6 Loading, empty, error and running states (C6)

Every region backed by data implements all four states.

| ID | Rule | Check |
|---|---|---|
| C6.1 | Zero data MUST render `EmptyState`, and any `data-testid` ending in `-empty` sits on it. Two kinds: **first run** (title, one-line description, at most one action, usually the same as the page's primary action) and **no results** (title "No matches", a "Clear filters" action, compact). Until `EmptyState size="sm"` ships (§9), compact means `className="border-none bg-transparent py-6"`. This is the one sanctioned `className` override on a library component. | Gate 1 + Gate 3 (D6) |
| C6.2 | Loading MUST use a `Skeleton` shaped like the final content. Spinners appear only inside a `Button` or an inline action. No full-page spinners. | Review |
| C6.3 | A failed region MUST show `ErrorState` with retry. Field errors are inline `text-destructive text-xs`. A partial failure inside a working page is a Callout with the `failed` tone. Never use a toast alone for an error the user must act on. | Review |
| C6.4 | Agent work in progress MUST show `StatusPill tone="running"` plus streamed or step progress, MUST NOT block the page, and MUST offer cancel when the work can be cancelled | Review |

### 3.7 Feedback (C7)

| ID | Rule | Check |
|---|---|---|
| C7.1 | Toasts (`sonner`) confirm completed async actions the user started. One line. Include Undo for reversible destructive actions. | Review |
| C7.2 | Persistent notices (pending account deletion, quota warnings, degraded service) use the Callout recipe `rounded-(--callout-radius) border border-callout-border bg-callout px-(--callout-px) py-(--callout-py)`, with `StatusPill` or status text for tone. Never a raw-palette banner. | Gate 1 (T1) + review |
| C7.3 | An agent action with external effect (send, delete, spend credits, post) MUST go through the existing action-approval card before it runs | Review |

### 3.8 Headings and text (C8)

| ID | Rule | Check |
|---|---|---|
| C8.1 | Exactly one `h1` per screen, rendered by `PageHeader`. P1 and P8 have no `h1`: their header title is `text-sm font-semibold`. Never write `<h1>` in feature code. | Gate 1 + Gate 3 (D2) |
| C8.2 | `h2` is a `PageSection` title. `h3` is a card or subsection title. Markdown rendered inside messages or generated blocks demotes `h1`/`h2` to `h3`/`h4`. | Review |
| C8.3 | Sentence case for titles, labels and buttons. Buttons are verbs. | Review |

---

## 4. Layout system (L)

This is the layer that didn't exist before. It defines how a whole screen is put together.

### 4.1 App shell (L1)

```
Desktop (≥ md)                                             Mobile (< md)
┌────────────┬───────────────────────────┬────────────┐    ┌────────────────────────┐
│ Sidebar    │ Main pane                 │ Aside      │    │ ≡  Main pane           │
│ bg-sidebar │ = one Page (pattern root) │ optional   │    │ = one Page             │
│ w-64       │                           │ SplitView  │    │ asides collapse        │
│ border-r   │                           │ (§4.6)     │    │ (sheet or stack)       │
│ edge, z-30 │                           │            │    │ sidebar → drawer, z-40 │
└────────────┴───────────────────────────┴────────────┘    └────────────────────────┘
```

- **L1.1** Every routed screen renders exactly one `Page` as its main pane root. `Page` sets `data-pattern`. (Gate 3, D1)
- **L1.2** The body never scrolls inside the app shell. Each pane owns its own scroll (L4).

### 4.2 Width tiers (L2)

| Tier | `Page width=` | Max width | Use |
|---|---|---|---|
| narrow | `"narrow"` | `max-w-2xl` (42 rem) | P6 forms, single-column settings content |
| reading | `"reading"` | `max-w-3xl` (48 rem) | P1 message column and composer, long-form generated documents |
| default | `"default"` | `max-w-5xl` (64 rem) | P3 record detail, P4 run detail, P5 settings with section nav |
| wide | `"wide"` | `max-w-7xl` (80 rem) | P2 collections, dashboards |
| full | `"full"` | none | P8 viewers, canvases, DAGs |

Standalone P7 cards use `max-w-sm`. Tables wider than their tier scroll horizontally inside the table container, never the page.

### 4.3 Gutters and rhythm (L3)

`Page` owns these values. Feature code doesn't set them.

| | Mobile | ≥ md |
|---|---|---|
| Horizontal gutter | `px-4` | `px-8` |
| Top / bottom | `pt-6` / `pb-12` | `pt-8` / `pb-12` |
| Between page blocks (header, toolbar, sections) | `gap-8` | `gap-8` |
| Inside a section | `gap-4` | `gap-4` |
| Between related controls | `gap-2` | `gap-2` |
| Card grid gap | `gap-4` | `gap-4` |

### 4.4 Scroll ownership (L4)

- **L4.1** One vertical scroll container per pane (`min-h-0 flex-1 overflow-y-auto`). Never nest vertical scrollers in the same pane. Code blocks and wide tables may scroll horizontally.
- **L4.2** Pane headers (ChatHeader, a sticky `PageToolbar`, table headers) are `sticky top-0 z-10` inside their pane, never `fixed`.

### 4.5 Responsive behaviour (L5)

- **L5.1** Viewport breakpoints only decide shell-level changes. Below `md`, the sidebar becomes a drawer, asides collapse, `ResponsiveDialog` becomes a sheet and multi-column settings stack.
- **L5.2** Inside resizable panes (chat column, artifact panel, side panels), use named container queries (`@container/chat`, `@[46rem]/chat:`), not viewport breakpoints. The existing chat column already does this.
- **L5.3** Touch targets come from the `size` prop, never from custom heights. On touch screens v2 already raises `--button-h` to 36 px and `--button-h-sm` to 32 px (§1.2). Below `md`, don't use `xs` or `icon-xs` (24 px or less) for anything a finger must hit, and give icon-only buttons `icon-md` (36 px) or larger.

### 4.6 Split views (L6)

- **L6.1** At most one aside per screen. It's always on the right, and it's built with `SplitView`.
- **L6.2** Aside widths: `sm` = `w-80` (metadata rail), `md` = `w-96` (inspector or step detail), `half` = 50% (artifact panel, builder preview).
- **L6.3** Below `lg`, the aside collapses by its declared mode:
  - `stack`: renders below the main content (metadata).
  - `sheet`: opens as a side `Sheet`, or a full-height mobile sheet below `md` (inspector, artifact).
  - `hide`: decorative only.

### 4.7 Pattern components (L7)

These live in `src/components/patterns/`. They compose existing library components and own every layout value in §4.2–4.6. They're thin by design. If they don't exist yet, create them from the reference below in a separate PR (CODEOWNERS routes it to design). Never make local copies.

| Component | Props | Renders |
|---|---|---|
| `Page` | `pattern` (`conversation`, `collection`, `record`, `run`, `settings`, `create`, `standalone`, `viewer`), `width` (§4.2) | `<main data-pattern data-width>` with scroll, background, gutters, width and `gap-8`. `conversation` and `viewer` get no gutters or gap. |
| `PageHeader` | `title`, `description?`, `leading?` (Avatar or tile), `status?` (StatusPill), `meta?` (text items), `actions?` (≤ 1 primary + ≤ 2 secondary + overflow `DropdownMenu`), `back?` (`{to, label}`) or `breadcrumb?`, `tabs?` (`Tabs variant="line"`) | `data-slot="page-header"`, the only `h1` |
| `PageToolbar` | `search?`, `filters?`, `scope?` (`Tabs variant="pill"`), `view?` (`Tabs variant="default"`), `actions?`, `sticky?` | `data-slot="page-toolbar"`, `flex flex-wrap items-center gap-2`. When `sticky`: `sticky top-0 z-10 bg-background py-2`. Replaced by the bulk toolbar while rows are selected. |
| `PageSection` | `title?`, `description?`, `actions?`, `children` | `<section data-slot="page-section" class="flex flex-col gap-4">`; `h2` is `text-base font-semibold text-foreground`, description `text-sm text-muted` |
| `SplitView` | `main`, `aside`, `asideWidth` (`sm`, `md`, `half`), `collapse` (`stack`, `sheet`, `hide`), `asideOpen?`, `onAsideOpenChange?` | ≥ lg: flex row; aside is `shrink-0 border-l border-edge bg-surface overflow-y-auto`. Below lg: per `collapse`. |
| `ActionBar` | `children` (buttons, primary last) | `data-slot="action-bar"`, `sticky bottom-0 z-10 flex justify-end gap-2 border-t border-border bg-background pt-3 pb-[calc(0.75rem+var(--sab))]` |

Reference implementation for the two components every pattern uses:

```tsx
// src/components/patterns/page.tsx
import type { ReactNode } from 'react';
import { cn } from '@/lib/utils'; // the same class-merge helper src/components/ui uses

const WIDTH = {
  narrow: 'max-w-2xl', reading: 'max-w-3xl', default: 'max-w-5xl', wide: 'max-w-7xl', full: 'max-w-none',
} as const;
export type Pattern =
  | 'conversation' | 'collection' | 'record' | 'run'
  | 'settings' | 'create' | 'standalone' | 'viewer';

export function Page({ pattern, width = 'default', children }: {
  pattern: Pattern; width?: keyof typeof WIDTH; children: ReactNode;
}) {
  if (pattern === 'conversation' || pattern === 'viewer') {
    return (
      <main data-pattern={pattern} data-width="full" className="relative flex min-h-0 flex-1 flex-col bg-background">
        {children}
      </main>
    );
  }
  return (
    <main data-pattern={pattern} data-width={width} className="flex min-h-0 flex-1 flex-col overflow-y-auto bg-background">
      <div className={cn('mx-auto flex w-full flex-col gap-8 px-4 pt-6 pb-12 md:px-8 md:pt-8', WIDTH[width])}>
        {children}
      </div>
    </main>
  );
}
```

```tsx
// src/components/patterns/page-header.tsx
import type { ReactNode } from 'react';
import { Link } from '@tanstack/react-router'; // the app's router
import { ChevronLeft } from 'lucide-react';    // the app's icon set
import { Button } from '@/components/ui/button';

export function PageHeader({ title, description, leading, status, meta, actions, back, breadcrumb, tabs }: {
  title: ReactNode; description?: ReactNode; leading?: ReactNode;
  status?: ReactNode; meta?: ReactNode; actions?: ReactNode;
  back?: { to: string; label: string }; breadcrumb?: ReactNode; tabs?: ReactNode;
}) {
  return (
    <header data-slot="page-header" className="flex flex-col gap-4">
      {back ? (
        <Button variant="ghost" size="sm" asChild className="w-fit">
          <Link to={back.to}><ChevronLeft aria-hidden />{back.label}</Link>
        </Button>
      ) : breadcrumb}
      <div className="flex flex-wrap items-start gap-x-4 gap-y-3">
        {leading && <div className="shrink-0">{leading}</div>}
        <div className="flex min-w-0 flex-1 flex-col gap-1">
          <div className="flex min-w-0 items-center gap-2">
            {/* eslint-disable-next-line no-restricted-syntax -- C8.1: PageHeader renders the one sanctioned h1 */}
            <h1 className="truncate font-sans text-(length:--page-title-size) leading-(--page-title-leading) font-(--page-title-weight) text-foreground">
              {title}
            </h1>
            {status}
          </div>
          {description && <p className="text-sm text-muted">{description}</p>}
          {meta && <div className="flex flex-wrap items-center gap-x-3 gap-y-1 text-xs text-muted">{meta}</div>}
        </div>
        {actions && <div className="flex shrink-0 items-center gap-2">{actions}</div>}
      </div>
      {tabs}
    </header>
  );
}
```

---

## 5. Screen patterns (P1–P8)

These eight patterns are canon. Every routed screen is one of them, or a registered one-off (§5.9).

### 5.0 Choosing a pattern

Ask these questions top to bottom and take the first "yes". Creao is chat-first, so P1 comes first.

```
Q1. Is the main thing a conversation with an agent or with people?           → P1 Conversation
Q2. Is it full-screen viewing of a single artifact (deck, DAG, media)?       → P8 Immersive viewer
Q3. Is it outside the signed-in shell (auth, install, share, status)?        → P7 Standalone
Q4. Is it one execution over time (steps, logs, a run)?                      → P4 Run & trace
Q5. Is it one item: its identity and its sub-views?                          → P3 Record detail
Q6. Is it configuration of a workspace, org, integration or account?         → P5 Settings
Q7. Is it creating something with ≥ 2 sections, > 7 fields or a preview?     → P6 Create flow
    (Smaller create and edit forms are a ResponsiveDialog over the current
    screen, not a screen of their own: C4.5)
Q8. Is it many items to find, compare or open?                               → P2 Collection
None of these → stop. Open a design review (§5.10).
```

Initial route registry. It's classified from the production route table and should be confirmed in rollout week 1.

| Pattern | Routes |
|---|---|
| P1 | `/chat`, `/rooms/$roomId`, `/direct/$userId` |
| P2 | `/apps`, `/discover`, `/discover/skills`, `/discover/workers`, `/discover/mcp-servers`, `/skills`, `/files`, `/crm`, `/rooms`, `/rooms/tasks`, `/feed`, `/views`, `/memory`, `/devices`, `/integrations`, `/team-pulse` |
| P3 | `/agents/$appId`, `/apps/$appId`, `/crm/$accountId`, `/discover/$slug`, `/discover/workers/$entryId`, `/discover/mcp-servers/$entryId`, `/views/$viewId` |
| P4 | `/apps/$appId/trace`, `/agent-triggers/runs/$runId`, plus the room run panel and task DAG/Gantt panels |
| P5 | `/workspaces/$workspaceId/settings`, `/organization`, `/settings/integrations/slack` |
| P6 | `/apps/new`, `/add-app`, `/onboarding`, the create-agent flow |
| P7 | `/login`, `/signup`, `/forgot-password`, `/reset-password`, `/install/$appId`, `/share/$token`, `/s/$token`, `/status`, `/download` |
| P8 | Deck preview, workflow DAG viewer, `/image`, `/video` viewers |
| One-off (§5.9) | `/pricing`, `/rewards`, `/affiliates`, `/voice`, `/harness`, `/widgets`; `/dev/*` is internal and exempt |

### 5.1 P1 Conversation

**Use for:** talking to an agent, a room or a person. **Not for:** browsing threads (that's P2).

```
┌ ChatHeader: sticky z-10, bg-chat-header-surface, backdrop-blur, border-b ──────────────────────┐
│ [agent avatar] Agent name ▾   Thread title       [trace] [memory] [⋯]                          │
├─────────────────────────────────────────────────────────────────┬──────────────────────────────┤
│        ┌─ message column: max-w-3xl mx-auto px-4 ──────────┐    │ Aside: artifact panel        │
│        │ user message                                      │    │ SplitView asideWidth="half"  │
│        │ agent message (text-base)                         │    │ collapse="sheet"             │
│        │ ┌ generated block (.jr-block), full column ─────┐ │    │ bg-surface                   │
│        │ └───────────────────────────────────────────────┘ │    │                              │
│        │ ┌ action-approval card ─────────────────────────┐ │    │                              │
│        │ └───────────────────────────────────────────────┘ │    │                              │
│        └───────────────────────────────────────────────────┘    │                              │
│        ┌─ ComposerShell: same max-w-3xl · docked bottom ───┐    │                              │
│        │ [queued tray · mention tray], same width          │    │                              │
│        │ [ input …                  ]  [attach]  [send]    │    │                              │
│        │ rounded-(--composer-radius) on shell and trays    │    │                              │
│        └───────────────────────────────────────────────────┘    │                              │
└─────────────────────────────────────────────────────────────────┴──────────────────────────────┘
```

| ID | MUST |
|---|---|
| P1.1 | `Page pattern="conversation"`. ChatHeader is the only header, with no `PageHeader` and no `h1`. |
| P1.2 | The message column and the composer share the reading width (`max-w-3xl`) and the same horizontal gutter |
| P1.3 | Anything attached to the composer (queued messages, mentions, attachments, slash suggestions) sits inside the composer's width and uses `rounded-(--composer-radius)` |
| P1.4 | The message list is the only vertical scroller. The composer is docked at the bottom with safe-area padding (`--sab`, `--keyboard-height`). |
| P1.5 | Artifacts open in the `SplitView` aside (`half`) at ≥ lg and as a full-height sheet below lg. Never in a modal. |
| P1.6 | Generated UI follows §6. Approvals use the action-approval card (C7.3). |
| P1.7 | Zero state is the composer plus suggestion chips, not an `EmptyState` card. Its test id is `chat-zero-state`, so C6.1 doesn't apply. |
| P1.8 | States: thread loading uses message-shaped skeletons. Agent running shows `StatusPill tone="running"` in the header plus streaming (C6.4). A failed turn shows an inline `ErrorState` in place of the message, with retry. |
| P1.9 | Layout inside the column responds to `@container/chat`, never to viewport breakpoints (L5.2) |

### 5.2 P2 Collection

**Use for:** finding, comparing or opening items such as agents, apps, skills, files, accounts or rooms.

```
┌ Page pattern="collection" width="wide" ──────────────────────────────────────┐
│ PageHeader  Apps                                       [Import] [New app]    │
│             Agent Apps in this workspace (text-sm text-muted)                │
│ PageToolbar [Search…] [Owner ▾] [Status ▾]  (All|Mine|Team)  [Grid|List]     │
│ ┌ Card ─────────┐ ┌ Card ─────────┐ ┌ Card ─────────┐   ← identity-first     │
│ │ tile  Name  ⋯ │ │               │ │               │     → card grid        │
│ │ meta · status │ │               │ │               │                        │
│ └───────────────┘ └───────────────┘ └───────────────┘                        │
│   — or — Table: Name | Owner | Status | Last run | Runs | ⋯   ← compare      │
│ [Load more]                                                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

| ID | MUST |
|---|---|
| P2.1 | `Page pattern="collection" width="wide"`, then `PageHeader` (title, description, ≤ 1 primary action), then `PageToolbar`, then content, then paging |
| P2.2 | **Grid or table.** Use a card grid when items are recognised by identity or preview and show ≤ 3 attributes (agents, apps, skills, templates). Use a `Table` when people compare or sort on ≥ 3 attributes (accounts, runs, files, members). Offer a Grid/List switch (`Tabs variant="default"`) only when both are genuinely useful. |
| P2.3 | Grid: `grid gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4`. Each card has one primary click target (the whole card links to the record), secondary actions in a `⋯` `DropdownMenu` at the top right, and no visible buttons beyond one. |
| P2.4 | Table: identity first (avatar or tile plus name), state as `StatusPill`, numbers right-aligned with `tabular-nums`, row actions as a ghost icon button with `DropdownMenu` at the row end, header `sticky top-0 z-10`. Below `md`, tables with > 3 columns become stacked rows: identity, 2 key fields, status. |
| P2.5 | Selecting rows swaps `PageToolbar` for the bulk toolbar in the same position. Nothing shifts. |
| P2.6 | States: first-run `EmptyState` with the header's primary action. No results: compact `EmptyState` with "Clear filters". Loading: 6 skeleton cards or 8 skeleton rows. |
| P2.7 | Filters and search are reflected in the URL (`?q=&status=`) |

### 5.3 P3 Record detail

**Use for:** one agent, app, account, skill, worker or view, together with its sub-views.

```
┌ Page pattern="record" width="default" ────────────────────────────────────────┐
│ ← Apps                                                                        │
│ [tile] Weekly digest  (running)                    [Run now] [Edit] [⋯]       │
│        Owner Dana · Updated 2h ago · v12  (text-xs text-muted)                │
│ Overview | Runs | Memory | Settings        (Tabs variant="line", URL-synced)  │
├───────────────────────────────────────────────────────┬───────────────────────┤
│ PageSection "Recent runs"   …                         │ Aside (≥ lg, w-80)    │
│ PageSection "Connected tools" …                       │ metadata, members     │
└───────────────────────────────────────────────────────┴───────────────────────┘
```

| ID | MUST |
|---|---|
| P3.1 | `PageHeader` with `back`, `leading` (avatar or tile), `title`, `status` (StatusPill), `meta` and `tabs` |
| P3.2 | Tabs are URL-addressable (`?tab=runs`). At most 6 tabs. Order: Overview, activity (Runs), content tabs, Settings last. |
| P3.3 | Record settings live in a Settings tab laid out like P5. Edits of ≤ 3 fields may use `ResponsiveDialog`. |
| P3.4 | Destructive record actions live in the `⋯` menu and confirm via `AlertDialog` (C4.1) |
| P3.5 | Optional metadata aside: `SplitView asideWidth="sm" collapse="stack"` |
| P3.6 | Run lists inside a record use the P2 table rules. Opening a run goes to P4. |

### 5.4 P4 Run and trace

**Use for:** one execution over time, such as an agent run, a trigger run, a task tree or a trace.

```
┌ Page pattern="run" width="default" (DAG/Gantt views: width="full") ───────────┐
│ ← Weekly digest                                                               │
│ Run #482  (running)    Started 10:42 · 1m 12s (tabular-nums)    [Cancel run]  │
├──────────────────────────────┬────────────────────────────────────────────────┤
│ Step list                    │ Step detail                                    │
│ (done)    Fetch emails   2s  │ Input / Output / Logs                          │
│ (running) Summarise          │ ┌ font-mono text-xs bg-surface-alt ─────────┐  │
│ (scheduled) Send digest      │ │ rounded-(--card-radius), collapsible      │  │
│                              │ └───────────────────────────────────────────┘  │
└──────────────────────────────┴────────────────────────────────────────────────┘
```

| ID | MUST |
|---|---|
| P4.1 | The header shows the run identity, a `StatusPill`, start time and duration (`tabular-nums`). Cancel and Retry are header actions, and Cancel confirms through `AlertDialog`. |
| P4.2 | Steps are a list. Each step has a status tone and label (C5.2) and a duration. |
| P4.3 | `SplitView` (step list, then step detail with `asideWidth="md"`). Below lg, tapping a step opens its detail full screen with a back control. |
| P4.4 | Logs and payloads use `font-mono text-xs` on `bg-surface-alt`. Long output collapses after 20 lines with "Show all". |
| P4.5 | On load, a failed run auto-selects its first failed step |
| P4.6 | Live updates never shift layout. Durations tick in place, and new steps append at the end. |

### 5.5 P5 Settings

**Use for:** configuring a workspace, an organization, an integration or the account.

```
┌ Page pattern="settings" width="default" ──────────────────────────────────────┐
│ PageHeader  Workspace settings                                                │
├────────────────┬──────────────────────────────────────────────────────────────┤
│ General        │ PageSection "General"                                        │
│ Members        │  Name                                   [ Acme            ]  │
│ Billing        │  Shown to members (text-xs text-muted)                       │
│ Integrations   │  Default model                          [ Select ▾        ]  │
│ Danger zone    │ PageSection "Danger zone"  (Callout, failed tone)            │
│ (w-48, ≥ md)   │  Delete workspace                       [Delete workspace]   │
└────────────────┴──────────────────────────────────────────────────────────────┘
```

| ID | MUST |
|---|---|
| P5.1 | Section navigation: a vertical list (`w-48`) at ≥ md. Below md, `Tabs variant="line"` or a `Select`. |
| P5.2 | Each setting row has the label (`text-sm font-medium`) and description (`text-xs text-muted`) on the left and the control on the right at ≥ sm, stacked below sm |
| P5.3 | Saving: toggles and selects autosave and confirm with a toast. Text fields save per section through an `ActionBar` that appears when the section has unsaved changes. |
| P5.4 | Danger zone is last and uses the Callout recipe with the failed tone. Destructive actions go through `AlertDialog`, with type-to-confirm for workspace or account deletion (C4.1). |

### 5.6 P6 Create flow

**Use for:** creating something with ≥ 2 sections, more than 7 fields or a live preview (an agent, an app, onboarding).

```
Form variant: Page width="narrow"            Builder variant: Page width="full"
                                             SplitView asideWidth="half"
┌─────────────────────────────────────┐      ┌──────────────────────────┬────────────────────────┐
│ ← Agents                            │      │ P1 conversation column   │ live preview (aside)   │
│ PageHeader   New agent              │      │ (chat with the builder)  │ collapse="sheet"       │
│ PageSection  Basics (≤ 7 fields)    │      │                          │                        │
│ PageSection  Tools                  │      │                          │                        │
│ ActionBar    [Cancel] [Create agent]│      │ composer                 │                        │
└─────────────────────────────────────┘      └──────────────────────────┴────────────────────────┘
```

| ID | MUST |
|---|---|
| P6.1 | Single column. At most 7 fields per `PageSection`. One `ActionBar` with Cancel (`ghost`) and the primary verb last. |
| P6.2 | Validate inline on blur, and summarise errors at the top on submit. Never disable submit without explaining why. |
| P6.3 | Use multi-step only when the steps are independent. Show progress with `Progress` and a step label. Back never loses input. |
| P6.4 | Chat-assisted builders: P1 rules for the chat side, plus a `SplitView` preview aside (`half`, `collapse="sheet"`) |
| P6.5 | A create form with one section of ≤ 7 fields and no preview is a `ResponsiveDialog` (C4.5), not a P6 page |

### 5.7 P7 Standalone

**Use for:** screens outside the signed-in shell, such as login, signup, password reset, install, share links, status and download.

```
┌ Page pattern="standalone" (no sidebar) ─────────┐
│                  [logo]                         │
│        ┌ Card max-w-sm ─────────────────┐       │
│        │ PageHeader (title, description)│       │
│        │ form or message                │       │
│        │ [Primary, full width]          │       │
│        └────────────────────────────────┘       │
│        footer links (text-xs text-muted)        │
└─────────────────────────────────────────────────┘
```

| ID | MUST |
|---|---|
| P7.1 | Same tokens and four modes as the app. `.auth-legacy-theme` is retired. |
| P7.2 | Status shown publicly (for example `/status`) uses `StatusPill` tones and semantic tokens, never its own palette |
| P7.3 | Third-party sign-in buttons follow the provider's brand rules. They're the only allowed `white`/`black` and live under `src/assets/brands/`. |

### 5.8 P8 Immersive viewer

**Use for:** full-screen viewing of one artifact, such as a slide deck, workflow DAG, image or video.

| ID | MUST |
|---|---|
| P8.1 | A full-bleed `Dialog` or a full route with `Page pattern="viewer"`. This is the only sanctioned plain `Dialog`, and each one carries a `C4.2` exception comment. |
| P8.2 | A top bar with the title (`text-sm font-semibold`) and close. A bottom bar with paging and controls. Esc closes. |
| P8.3 | Controls that auto-hide must stay reachable from the keyboard, and they reappear on focus |

### 5.9 One-offs

Promotional and experimental pages may use their own layout, on three conditions:
1. Token rules T1–T9 still apply in full.
2. The route is listed in the one-off row of the registry (§5.0).
3. The route file is owned by Product Design in CODEOWNERS.

A one-off that a second feature wants to copy becomes a pattern proposal (§5.10).

### 5.10 New features and new patterns

To lay out a new feature (or a coding agent's task):
1. Split the feature into screens and overlays. Classify each screen with §5.0 and each overlay with the C4 table.
2. For every data region, list its loading, empty (first run and no results), error and running states (C6).
3. Build with the pattern components. Add the first screen of each pattern in the feature to `design/exemplars.json` so Gate 3 covers it.
4. If no pattern fits, open a design review with the closest pattern and the specific gap. If the review accepts it, the new pattern is added here as P9+ with its own anatomy, MUST rules and an exemplar. Nobody ships an unregistered layout.

**Worked example (illustrative).** A feature for scheduling agent automations:
- "All schedules" is P2, as a table: name, agent, next run, last result, owner.
- "Schedule detail" is P3, with Overview, Runs and Settings tabs.
- A single run is P4.
- "New schedule" is one section of 4 fields with no preview, so it's a `ResponsiveDialog` at `sm:max-w-lg` (C4.5), not a P6 page.
- "Pause schedule" is reversible, so it's a toast with Undo. "Delete schedule" goes through `AlertDialog`.
- States: no schedules shows a first-run `EmptyState` with "New schedule". A failed run shows `StatusPill failed`, and P4 auto-selects the failed step.

---

## 6. Agent-generated UI (GU)

Creao's own agents render UI inside conversations: json-render specs drawn from a component catalog, shown in `.jr-block` containers. That UI is a product surface, held to the same system. These rules govern the runtime agent's output and the catalog. Coding agents are governed by §2–5.

| ID | Rule | Check |
|---|---|---|
| GU1 | Generated UI MUST render only catalog components, and every catalog entry MUST wrap a `src/components/ui` primitive. The stock shadcn copy is removed from the bundle. | Gate 2 (`C1.2` slot count) |
| GU2 | `--jr-*` variables MUST alias semantic tokens (mapping below). Delete the `.dark .jr-block` override block, because aliased tokens already adapt. | Gate 2 (T1.6) |
| GU3 | MUST NOT restyle library components inside generated UI with descendant selectors (`.jr-block [data-slot=card] {…}`). If generated UI needs a denser look, the library adds a `density="compact"` variant (§9). The existing selectors are frozen and counted. | Gate 2 (`GU3` count) |
| GU4 | Blocks span the conversation column (reading width) with `rounded-(--card-radius) border border-edge`. A block taller than 60vh collapses behind "Show more" or offers "Open in panel" (P1.5). | Gate 3 (generated-UI fixture) |
| GU5 | Blocks contain no page-level structure: no `h1`/`h2`, no `PageHeader`, no navigation. Block headings are `h3` (`text-base font-semibold`) or `h4` (`text-sm font-semibold`). | Gate 3 (D2) |
| GU6 | At most one primary `Button` per block. Any action with external effect or destructive outcome goes through the action-approval card or an `AlertDialog` before it runs (C7.3). The model can't one-click destroy. | Gate 3 (C3.1 count) + review |
| GU7 | Charts use `var(--color-lane-1…8)` for series in order, with ≤ 8 series. Status colours appear only for status. Axes and grid use `--color-border` and `--color-muted`. No hex. | Gate 1 (T1.7) |
| GU8 | Catalog prop schemas expose enums only: `variant`, `tone`, `size`, `lane`. They have no `className`, `style` or free-form colour props. Unknown props are stripped and logged as `jr.schema_violation`. The model physically can't emit off-system styling. | Catalog schema unit test + runtime log |

GU2 mapping:

| Variable | Value |
|---|---|
| `--jr-container-bg` | `var(--color-background)` (the block sits on the conversation, bordered, with no fill) |
| `--jr-card-bg` | `var(--color-card)` |
| `--jr-card-border` | `var(--color-edge)` |
| `--jr-card-shadow` | `none` |
| `--jr-ink` | `var(--color-foreground)` |
| `--jr-muted` | `var(--color-muted)` |
| `--jr-accent` | `var(--color-destructive)`. Its only use is the destructive `Badge` inside blocks. |
| `--jr-pill-bg` / `--jr-pill-fg` | `var(--color-primary)` / `var(--color-primary-foreground)` |
| `--jr-positive` | `var(--color-success)` |

---

## 7. Enforcement

### 7.1 Four gates

| Gate | Runs | Blocks merge when | Catches |
|---|---|---|---|
| **1. Lint:** `creao/classes`, `creao/classes-advisory`, restricted imports and syntax | Editor; pre-commit on changed files; CI | Any error in a changed file | Token misuse, raw `h1`/`button`, the wrong overlay, primitive imports, hex in code, hand-rolled empty states |
| **2. Build scan and ratchet:** `design/scan.mjs` (class rules, slot count, structural CSS checks), token contract test | CI after `vite build` | Any metric rises above `design/baseline.json`, or the token contract test fails | Everything that reaches the bundle: dynamic class strings, global CSS, vendor theming, generated-UI CSS, duplicate libraries, including copies pulled in from `node_modules` |
| **3. Render matrix:** Playwright across 4 modes × exemplar routes | CI on PRs touching `src/**/*.tsx` or CSS; nightly on all exemplars | Any DOM assertion, contrast or snapshot fails | Wrong page structure, mode-specific breakage, off-scale computed styles, overlays on mobile, contrast |
| **4. Review:** CODEOWNERS and the PR template | Every PR | A design owner hasn't approved owned paths | Pattern choice, new primitives, exceptions, one-offs |

Gates 1 and 2 are deterministic and fast. They do most of the work and are the ones a coding agent can run on itself. Gate 3 catches what only shows up when the page renders. Gate 4 covers judgement calls.

### 7.2 Commands

```jsonc
// package.json → scripts
"design:lint":   "git diff --name-only --diff-filter=ACMR $(git merge-base origin/main HEAD) -- 'src/*.ts' 'src/*.tsx' | xargs -r eslint --max-warnings=0",
"design:scan":   "node design/scan.mjs dist/assets",
"design:render": "playwright test e2e/design",
"design:check":  "npm run design:lint && npm run build && npm run design:scan"
```

- `design:lint` lints only the files this branch changed, including uncommitted edits to tracked files. A brand-new file is included once it's staged (`git add`). `xargs -r` skips ESLint when nothing changed; without it, ESLint would lint the whole repo.
- `--max-warnings=0` applies to those files only, so advisory rules (`T4.2`, `T5.2`, `T7.2`) block new code from day one without failing on legacy files.

### 7.3 Gate 1: lint

**Rule registry.** `design/rules.json` is the single source of truth, shared by lint and the scan.

```json
{
  "version": 1,
  "scope": {
    "feature": ["src/**/*.{ts,tsx}"],
    "exempt": ["src/components/ui/**", "src/styles/**", "src/assets/brands/**",
               "src/routes/dev/**", "**/*.stories.tsx", "**/*.test.tsx"]
  },
  "classRules": [
    { "id": "T1.1", "on": "utility", "pattern": "^(bg|text|border(-[trblxyse])?|ring|ring-offset|outline|fill|stroke|divide|from|via|to|shadow|decoration|caret|accent|placeholder)-(red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose|slate|gray|zinc|neutral|stone)-\\d{2,3}(/\\S+)?$", "message": "Raw Tailwind palette colour. Use a semantic token (guideline 2.1)." },
    { "id": "T1.2", "on": "utility", "pattern": "^(bg|text|border(-[trblxyse])?|ring|outline|fill|stroke|divide|from|via|to)-grey-\\d+(/\\S+)?$", "message": "grey-* is not theme-safe. Use a semantic surface/text/border token (guideline 2.1)." },
    { "id": "T1.3", "on": "utility", "pattern": "-\\[(#|rgba?\\(|hsla?\\(|oklch\\(|oklab\\(|color:)", "message": "Arbitrary colour value. Use a semantic token (guideline 2.1)." },
    { "id": "T1.4", "on": "utility", "pattern": "-ox-", "message": "ox-* is a frozen legacy palette. Use the semantic equivalent (guideline 2.1)." },
    { "id": "T1.5", "on": "utility", "pattern": "^(bg|text|border|fill|stroke)-(white|black)(/\\S+)?$", "message": "Use a foreground/background token pair, not white/black (guideline 2.1)." },
    { "id": "T2.1", "on": "class", "pattern": "(^|:)dark:(\\S+:)?(bg|text|border(-[trblxyse])?|ring|outline|fill|stroke|divide|from|via|to)-(?!(\\d|px$|xs$|sm$|base$|lg$|xl$|[2-9]xl$|left$|center$|right$|justify$|start$|end$|wrap$|nowrap$|balance$|pretty$|ellipsis$|clip$|none$|solid$|dashed$|dotted$|double$|hidden$|x$|y$|x-|y-|offset-\\d|inset$|cover$|contain$|fixed$|linear|radial|conic|gradient))", "message": "Manual dark-mode colour. Semantic tokens already adapt; fix the token instead (guideline 2.2)." },
    { "id": "T3.1", "on": "utility", "pattern": "^text-\\[[\\d.]", "message": "Arbitrary font size. Use the type scale (guideline 2.3)." },
    { "id": "T3.2", "on": "utility", "pattern": "^font-\\[", "message": "Arbitrary font weight or family (guideline 2.3)." },
    { "id": "T3.3", "on": "utility", "pattern": "^(leading|tracking)-\\[", "message": "Arbitrary line-height or letter-spacing (guideline 2.3)." },
    { "id": "T4.1", "on": "utility", "pattern": "^rounded(-[a-z]{1,2})?-\\[", "message": "Arbitrary radius. Use a radius token (guideline 2.4)." },
    { "id": "T5.1", "on": "utility", "pattern": "^shadow-\\[", "message": "Hand-written shadow. Use a shadow token (guideline 2.5)." },
    { "id": "T6.1", "on": "utility", "pattern": "^z-(?!(0|10|20|auto)$)(\\d+|\\[[^\\]]+\\]|\\([^)]+\\))$", "message": "Feature code may only use z-0, z-10, z-20 or z-auto (guideline 2.6)." },
    { "id": "T7.1", "on": "utility", "pattern": "^(p|px|py|pt|pr|pb|pl|ps|pe|m|mx|my|mt|mr|mb|ml|ms|me|gap|gap-x|gap-y|space-x|space-y|inset|inset-x|inset-y|top|right|bottom|left|translate-x|translate-y)-\\[", "allow": "var\\(--(sat|sab|sal|sar|sat-content|keyboard-height|top-chrome-h)", "message": "Arbitrary spacing. Use the spacing scale (guideline 2.7)." },
    { "id": "T8.1", "on": "utility", "pattern": "^(duration|ease|delay|animate)-\\[", "message": "Custom motion. Use duration-150/200/300 with ease-out or ease-in-out (guideline 2.8)." },
    { "id": "T9.1", "on": "utility", "pattern": "^max-w-\\[", "message": "Arbitrary width. Use a width tier (guideline 4.2)." },
    { "id": "C2.3", "on": "class", "pattern": "\\[data-slot=", "message": "Restyling a library component's internals from outside. Request a variant instead (guideline 3.2)." },
    { "id": "T5.2", "level": "warn", "on": "utility", "pattern": "^shadow(-(2xs|xs|sm|md|lg|xl|2xl))?$", "message": "Generic Tailwind shadow. Use shadow-card / shadow-card-hover / shadow-composer (guideline 2.5)." },
    { "id": "T4.2", "level": "warn", "on": "utility", "pattern": "^rounded(-(t|r|b|l|s|e|tl|tr|br|bl|ss|se|es|ee))?(-(xs|sm|md|lg|xl|2xl|3xl|4xl))?$", "message": "Generic radius does not adapt between classic and v2. Use a component radius token (guideline 2.4)." },
    { "id": "T7.2", "level": "warn", "on": "utility", "pattern": "^(p|px|py|pt|pr|pb|pl|ps|pe|m|mx|my|mt|mr|mb|ml|ms|me|gap|gap-x|gap-y|space-x|space-y)-(5|7|9|10|11|14)$", "message": "Off-rhythm spacing step. Use 1, 1.5, 2, 3, 4, 6, 8, 12 (guideline 2.7)." }
  ]
}
```

**Shared matcher.** `design/match.mjs`:

```js
import { readFileSync } from 'node:fs';

export const registry = JSON.parse(readFileSync(new URL('./rules.json', import.meta.url), 'utf8'));
const compiled = registry.classRules.map((r) => ({
  ...r, re: new RegExp(r.pattern), allowRe: r.allow ? new RegExp(r.allow) : null,
}));

// "md:hover:-mt-[3px]!" -> { variants: ["md", "hover"], utility: "mt-[3px]" }
export function splitClass(cls) {
  const parts = []; let depth = 0; let cur = '';
  for (const ch of cls) {
    if (ch === '[' || ch === '(') depth++;
    if (ch === ']' || ch === ')') depth--;
    if (ch === ':' && depth === 0) { parts.push(cur); cur = ''; } else cur += ch;
  }
  parts.push(cur);
  const utility = parts.at(-1).replace(/^!|!$/g, '').replace(/^-/, '');
  return { variants: parts.slice(0, -1), utility };
}

export function violations(cls) {
  const { utility } = splitClass(cls);
  return compiled.filter((r) => {
    const subject = r.on === 'class' ? cls : utility;
    return r.re.test(subject) && !(r.allowRe && r.allowRe.test(subject));
  });
}
```

**ESLint plugin.** `design/eslint-plugin.mjs`. Every word of a string in `className`, `class` or a `cn()`/`clsx()`/`cva()` call is checked. Any other string is checked only when every word in it looks like a class, and then only the words with a `-`, `:` or `[`. That catches tone maps such as `{ outage: "text-[#c0392b]" }` and leaves UI copy such as "Totals are rounded to the nearest credit" alone.

```js
import { violations } from './match.mjs';

const CLASS_ATTRS = new Set(['className', 'class']);
const CLASS_FNS = new Set(['cn', 'clsx', 'cx', 'cva', 'tv', 'twMerge']);
const CLASS_WORD = /^[a-z0-9!@*[(-]\S*[a-z0-9\])%!]$/;            // starts and ends like a Tailwind class

function inClassContext(node) {                                       // className="…" or inside cn(…)
  for (let p = node.parent; p; p = p.parent) {
    if (p.type === 'JSXAttribute') return CLASS_ATTRS.has(p.name.name);
    if (p.type === 'CallExpression') return CLASS_FNS.has(p.callee.name);
  }
  return false;
}

const makeRule = (advisory) => ({
  meta: { type: 'problem', docs: { description: 'Creao token rules (guideline section 2)' }, schema: [] },
  create(context) {
    const check = (node, text) => {
      const words = text.split(/\s+/).filter(Boolean);
      const strict = inClassContext(node);
      if (!strict && !words.every((w) => CLASS_WORD.test(w))) return;  // prose, labels, log lines
      for (const cls of words) {
        if (!strict && !/[-:[]/.test(cls)) continue;                   // bare words outside class context
        for (const r of violations(cls)) {
          if ((r.level === 'warn') !== advisory) continue;
          context.report({ node, message: `[${r.id}] ${cls} - ${r.message}` });
        }
      }
    };
    return {
      Literal(node) { if (typeof node.value === 'string') check(node, node.value); },
      TemplateElement(node) { check(node, node.value.cooked ?? ''); },
    };
  },
});

export default { rules: { classes: makeRule(false), 'classes-advisory': makeRule(true) } };
```

**ESLint config.** Add this block to `eslint.config.mjs`:

```js
import tseslint from 'typescript-eslint';
import comments from '@eslint-community/eslint-plugin-eslint-comments';
import creao from './design/eslint-plugin.mjs';
import { registry } from './design/match.mjs';

export default [
  // ...existing config
  {
    files: registry.scope.feature,
    ignores: registry.scope.exempt,
    languageOptions: { parser: tseslint.parser },
    plugins: { creao, '@eslint-community/eslint-comments': comments },
    rules: {
      'creao/classes': 'error',
      'creao/classes-advisory': 'warn',
      '@eslint-community/eslint-comments/require-description': 'error', // exceptions need a reason (7.7)
      'no-restricted-imports': ['error', { patterns: [{
        group: ['@radix-ui/*', 'vaul', 'cmdk'],
        message: '[C1.1] Import the wrapper from src/components/ui, never the primitive package.',
      }] }],
      'no-restricted-syntax': ['error',
        { selector: "JSXOpeningElement[name.name='h1']",
          message: '[C8.1] Page titles come from <PageHeader>. Do not render <h1> directly.' },
        { selector: "JSXOpeningElement[name.name='button'] > JSXAttribute[name.name='className']",
          message: '[C2.1] Styled <button>. Use <Button> (asChild for custom elements).' },
        { selector: "JSXOpeningElement[name.name='Dialog']",
          message: '[C4.2] Forms use <ResponsiveDialog>; confirmations use <AlertDialog>. <Dialog> is for P8 viewers only.' },
        { selector: "JSXOpeningElement[name.name!='EmptyState'] > JSXAttribute[name.name='data-testid'][value.value=/-empty$/]",
          message: '[C6.1] Zero-data states render <EmptyState>.' },
        { selector: 'Literal[value=/^#(?:[0-9a-fA-F]{3,4}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$/]',
          message: '[T1.7] Hex colour in code. Use a token (CSS class, or var(--color-...) for SVG/canvas).' },
      ],
    },
  },
];
```

What a developer or agent sees. This is real output, from running the plugin on code rebuilt from production (see the audit):

```
  7:25  error  [T4.1] rounded-[6px] - Arbitrary radius. Use a radius token (guideline 2.4)       creao/classes
 12:10  error  [C8.1] Page titles come from <PageHeader>. Do not render <h1> directly            no-restricted-syntax
 27:5   error  [C4.2] Forms use <ResponsiveDialog>; confirmations use <AlertDialog>. ...           no-restricted-syntax
 36:21  error  [T1.3] text-[#c0392b] - Arbitrary colour value. Use a semantic token (guideline 2.1)  creao/classes
```

### 7.4 Gate 2: build scan and token contract

The **build scan** (`design/scan.mjs`) reads `dist/assets`. It takes every class Tailwind generated, keeps only those the shipped JS actually references, counts each rule from `rules.json`, adds three structural checks, and compares the result with `design/baseline.json`.

The three structural checks:
- `C1.2` (slot count): extra definitions of the same `data-slot` value in the shipped JS. A second component library raises it by one for every part it defines again. It reads the build, not `src/`, so it also catches a copy that arrives inside an npm package. Some duplicates are legitimate: `ResponsiveDialog` defines its parts once for the desktop dialog and once for the mobile sheet. So the count is ratcheted rather than required to be zero.
- `GU3`: descendant `[data-slot=…]` selectors.
- `T1.6`: literal colours in non-theme custom properties.

Any increase fails CI. `--update-baseline` can only lower values.

```js
// design/scan.mjs — usage: node design/scan.mjs <distAssetsDir> [--update-baseline]
import { readFileSync, readdirSync, writeFileSync, existsSync } from 'node:fs';
import { join } from 'node:path';
import { registry, violations } from './match.mjs';

const dir = process.argv[2] ?? 'dist/assets';
const update = process.argv.includes('--update-baseline');
const files = readdirSync(dir);
const read = (f) => readFileSync(join(dir, f), 'utf8');
const css = files.filter((f) => f.endsWith('.css')).map(read).join('\n');
const js = files.filter((f) => f.endsWith('.js')).map(read)
  .filter((t) => !t.trimStart().toLowerCase().startsWith('<!doctype')).join('\n');

const unescape = (s) => s.replace(/\\(.)/g, '$1');
const cssClasses = new Set([...css.matchAll(/\.((?:[A-Za-z0-9_-]|\\.)+)/g)].map((m) => unescape(m[1])));
const jsTokens = new Set(js.split(/[\s"'`]+/));
const live = [...cssClasses].filter((c) => jsTokens.has(c));                       // referenced by shipped code
const phantom = [...cssClasses].filter((c) => c.includes('[') && !jsTokens.has(c)); // generated from docs/comments

const counts = Object.fromEntries(registry.classRules.map((r) => [r.id, 0]));
for (const cls of live) for (const r of violations(cls)) counts[r.id]++;

const slotN = {};                                                                   // definitions per data-slot value
for (const m of js.matchAll(/"data-slot":\s*"([a-z0-9-]+)"/g)) slotN[m[1]] = (slotN[m[1]] ?? 0) + 1;
counts['C1.2'] = Object.values(slotN).reduce((sum, n) => sum + n - 1, 0);           // extra definitions
const selectors = [...css.matchAll(/([^{}]+)\{/g)].map((m) => m[1]);
counts.GU3 = selectors.filter((s) => /\s\[data-slot=/.test(s)).length;
const themeSel = /^(:root|:host|\.dark|:root\.dark|:root\[data-design|\.auth-legacy-theme|\.dark \.auth-legacy-theme)/;
counts['T1.6'] = 0;
for (const m of css.matchAll(/([^{}]+)\{([^{}]*)\}/g)) {
  const sel = m[1].trim();
  if (themeSel.test(sel) || sel.startsWith('@')) continue;
  counts['T1.6'] += [...m[2].matchAll(/--(?!tw-)[\w-]+:\s*(#[0-9a-f]{3,8}|rgba?\(|oklch\(|oklab\()/gi)].length;
}
counts.phantomClasses = phantom.length;

const baselinePath = new URL('./baseline.json', import.meta.url);
const baseline = existsSync(baselinePath) ? JSON.parse(readFileSync(baselinePath, 'utf8')) : null;
const slotsPath = new URL('./slots.json', import.meta.url);                         // per-slot counts, written with the baseline
const slotBase = existsSync(slotsPath) ? JSON.parse(readFileSync(slotsPath, 'utf8')) : {};
const rows = Object.keys(counts).map((id) => ({ id, now: counts[id], base: baseline?.[id] ?? null }));
for (const r of rows) console.log(`${r.id.padEnd(15)} ${String(r.now).padStart(5)}  base ${String(r.base ?? '-').padStart(5)}${r.base !== null && r.now > r.base ? '  <-- REGRESSION' : ''}`);

if (update) {
  writeFileSync(baselinePath, JSON.stringify(Object.fromEntries(
    rows.map((r) => [r.id, r.base === null ? r.now : Math.min(r.base, r.now)])), null, 2) + '\n');
  writeFileSync(slotsPath, JSON.stringify(Object.fromEntries(
    Object.entries(slotN).map(([s, n]) => [s, Math.min(slotBase[s] ?? n, n)])), null, 2) + '\n');
} else if (rows.some((r) => r.base !== null && r.now > r.base)) {
  const again = Object.keys(slotN).filter((s) => slotN[s] > (slotBase[s] ?? 1));
  if (counts['C1.2'] > baseline['C1.2'] && again.length) console.error('C1.2 primitive defined again:', again.join(' '));
  console.error('FAIL: design metrics increased. Fix, or file an exception (guideline 7.7).');
  process.exit(1);
}
```

**Starting baseline.** `design/baseline.json` holds the values measured on production on 2026-11-09:

```json
{ "T1.1": 370, "T1.2": 264, "T1.3": 11, "T1.4": 61, "T1.5": 47, "T2.1": 489,
  "T3.1": 44, "T3.2": 4, "T3.3": 36, "T4.1": 23, "T5.1": 47, "T6.1": 22,
  "T7.1": 105, "T8.1": 23, "T9.1": 74, "C2.3": 13, "T5.2": 15, "T4.2": 39,
  "T7.2": 63, "C1.2": 81, "GU3": 47, "T1.6": 20, "phantomClasses": 193 }
```

These are build-wide counts and include library internals. That's deliberate: the ratchet tolerates today's debt and refuses new debt.

`--update-baseline` also writes `design/slots.json`, the definition count per `data-slot`. That lets a failing run name the primitive that was defined again, instead of listing every existing duplicate.

`C1.2` is 81 today. Deleting the stock copy (F-01) takes it to 17. Those are legitimate repeats inside Creao's own library, such as `ResponsiveDialog`'s desktop and mobile branches. After that PR, run `--update-baseline` so the lower value is locked in.

**Token contract test.** Owned by the library owner and run in the same CI job. It's a unit test over the theme CSS that asserts:
- (a) each `grey-*` ramp is monotonic in luminance in all four modes;
- (b) every `--color-*` defined for classic light is also defined for classic dark, v2 light and v2 dark, or is listed as mode-invariant;
- (c) every `--jr-*` value is a `var(…)`.

It keeps the token defects found in the audit (F-17) from coming back.

### 7.5 Gate 3: render matrix

`design/exemplars.json` lists one or more routes per pattern from a seeded test workspace, for example `{ "route": "/apps", "pattern": "collection" }`. The suite runs each one in all four modes.

```ts
// e2e/design/render.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';
import exemplarJson from '../../design/exemplars.json';

type Exemplar = { route: string; pattern: string; dialogTrigger?: string };
const exemplars = exemplarJson as Exemplar[];
const MODES = [['classic', 'light'], ['classic', 'dark'], ['v2', 'light'], ['v2', 'dark']] as const;

for (const { route, pattern } of exemplars) for (const [design, theme] of MODES) {
  test(`${pattern} ${route} [${design}/${theme}]`, async ({ page }) => {
    await page.addInitScript(([d, t]) => {
      // The same keys the app reads at boot (verified in the production bundle).
      if (d === 'v2') localStorage.setItem('creao.designSystem', 'v2');
      else localStorage.removeItem('creao.designSystem');
      localStorage.setItem('creao-ui-theme', t);
    }, [design, theme]);
    await page.goto(route);
    await page.waitForLoadState('networkidle');

    // D1 exactly one pattern root, and it is the declared one
    await expect(page.locator('[data-pattern]')).toHaveCount(1);
    await expect(page.locator('[data-pattern]')).toHaveAttribute('data-pattern', pattern);

    // D2 one h1, inside PageHeader (P1 and P8 have none)
    const none = pattern === 'conversation' || pattern === 'viewer';
    await expect(page.locator('h1')).toHaveCount(none ? 0 : 1);
    if (!none) await expect(page.locator('[data-slot=page-header] h1')).toHaveCount(1);

    // D3-D5 computed font-size, radius and z-index stay on the token scales of this mode
    // D10 every Button is exactly the height its size prop promises (C3.2)
    const off = await page.evaluate(() => {
      const root = getComputedStyle(document.documentElement);
      const px = (v: string) => parseFloat(v) * (v.trim().endsWith('rem') ? 16 : 1);
      const tok = (n: string) => px(root.getPropertyValue(n));
      const buttonH: Record<string, number> = {
        default: tok('--button-h'), sm: tok('--button-h-sm'), xs: tok('--button-h-xs'), lg: tok('--button-h-lg'),
        icon: 32, 'icon-xs': 24, 'icon-sm': 32, 'icon-md': 36, 'icon-lg': 40, // fixed in the library today (§1.2)
      };
      const sizes = new Set([...['xs', 'sm', 'base', 'lg', 'xl', '2xl', '3xl', '4xl', '5xl', 'body-lg'].map((k) => tok(`--text-${k}`)),
        tok('--page-title-size'), tok('--status-text')]);
      const radii = new Set([0, ...['xs', 'sm', 'md', 'lg', 'xl', '2xl', '3xl'].map((k) => tok(`--radius-${k}`)),
        ...['card', 'chip', 'badge', 'composer', 'callout', 'pill', 'segmented', 'segmented-item', 'count'].map((k) => tok(`--${k}-radius`))]);
      const layers = new Set(['auto', '0', '10', '20', '30', '40', '50']);
      const bad: string[][] = [];
      for (const el of document.querySelectorAll('[data-pattern] *:not(.prose *)')) {
        const cs = getComputedStyle(el);
        const hasText = [...el.childNodes].some((n) => n.nodeType === 3 && n.textContent!.trim());
        if (hasText && !sizes.has(parseFloat(cs.fontSize))) bad.push(['font-size', cs.fontSize, el.outerHTML.slice(0, 100)]);
        const r = cs.borderTopLeftRadius;
        if (!r.endsWith('%') && parseFloat(r) > 0 && parseFloat(r) < 9999 && !radii.has(parseFloat(r))) bad.push(['radius', r, el.outerHTML.slice(0, 100)]);
        if (!layers.has(cs.zIndex)) bad.push(['z-index', cs.zIndex, el.outerHTML.slice(0, 100)]);
        const want = el.matches('[data-slot=button]:not([data-variant=link])') ? buttonH[el.getAttribute('data-size') ?? 'default'] : undefined;
        if (want && el.getClientRects().length && Math.abs(el.getBoundingClientRect().height - want) > 0.5)
          bad.push(['button-height', cs.height, el.outerHTML.slice(0, 100)]); // skips hidden buttons
      }
      return bad.slice(0, 20);
    });
    expect(off, JSON.stringify(off, null, 1)).toEqual([]);

    // D6 zero-data regions are EmptyState
    for (const el of await page.locator('[data-testid$="-empty"]').all())
      await expect(el).toHaveAttribute('data-slot', 'empty-state');

    // D7 contrast and accessible names (C3.3)
    const axe = await new AxeBuilder({ page }).withRules(['color-contrast', 'button-name']).analyze();
    expect(axe.violations).toEqual([]);

    // D8 one primary per region (C3.1, GU6)
    const regions = '[data-slot=page-header], [data-slot=page-toolbar], [data-slot=action-bar], [data-slot=card], '
      + '[data-slot=responsive-dialog-footer], .jr-block';
    for (const region of await page.locator(regions).all())
      expect(await region.locator('[data-slot=button][data-variant=default]').count()).toBeLessThanOrEqual(1);

    // Visual baseline per mode
    await expect(page).toHaveScreenshot(`${pattern}${route.replaceAll('/', '_')}-${design}-${theme}.png`, { fullPage: true, maxDiffPixelRatio: 0.01 });
  });
}

// D9 forms open as ResponsiveDialog on phones (C4.2). Exemplars with a "dialogTrigger" get this check.
test.describe('mobile overlays', () => {
  test.use({ viewport: { width: 390, height: 844 } });
  for (const ex of exemplars.filter((e) => e.dialogTrigger)) {
    test(`${ex.route} dialog`, async ({ page }) => {
      await page.goto(ex.route);
      await page.locator(ex.dialogTrigger).click();
      await expect(page.locator('[role=dialog]')).toHaveAttribute('data-slot',
        /^(responsive-dialog-content|mobile-sheet-content|alert-dialog-content)$/);
    });
  }
});
```

The exemplar set must include a conversation fixture that renders one block of every catalog component (GU4–GU6). That's how generated UI enters the matrix.

### 7.6 Gate 4: review

`CODEOWNERS`:

```
src/components/ui/**            @creao/design-system
src/components/patterns/**      @creao/product-design
src/styles/**                   @creao/design-system
design/**                       @creao/product-design
src/routes/pricing* src/routes/rewards* src/routes/affiliates*   @creao/product-design
```

PR template, Design section (required for any PR touching `src/**/*.tsx` or CSS):

```markdown
### Design
- Pattern(s): P_ (route → pattern) · new route added to design/exemplars.json? yes/no
- States covered: loading / first-run empty / no-results / error / running
- Overlays: (job → component per C4)
- New primitives or pattern components: none | <link to design review>
- Exceptions: none | <rule ID, reason, ticket>
- Checked in: classic light · classic dark · v2 light · v2 dark
```

### 7.7 Exceptions

- Use a standard disable comment with a rule ID, a reason and a ticket. The `require-description` rule makes the description mandatory:
  `// eslint-disable-next-line no-restricted-syntax -- C4.2 P8 viewer: deck preview is full-bleed (DS-142)`
- CI prints every exception with its rule ID. More than 3 new exceptions in one PR needs Product Design approval (Gate 4).
- **Coding agents MUST NOT add exceptions.** If a rule blocks the task, the agent stops and reports the rule ID and why. A person decides.
- A rule that collects repeated, legitimate exceptions is a signal to change the rule. Product Design reviews the exception list monthly.

### 7.8 Agent contract

Add this to the repo's agent instruction file (`AGENTS.md`, `CLAUDE.md` or equivalent):

```markdown
## UI work (required)
Before writing or changing UI, read design-system-guideline.md §0, §2 and the pattern you need in §5.
1. Name the pattern (P1–P8) for every screen you touch, in your plan and in the PR. No fit → stop and ask.
2. Build page structure with src/components/patterns and everything inside with src/components/ui.
   Never import @radix-ui/*, vaul or cmdk. Never create a new primitive.
3. Use only the tokens in §2. Never write [arbitrary] values, raw palette colours, grey-*, ox-*,
   white/black, hex, or dark: colour overrides. Watch the token traps table in §2.1.
4. Every data region gets loading / empty / error / running states (C6).
5. Before finishing, run `npm run design:check` and fix every error and warning in the files you changed.
6. Never add eslint-disable for creao/*, no-restricted-syntax or no-restricted-imports. If a rule blocks you,
   stop and report the rule ID.
7. Fill in the PR template's Design section.
```

### 7.9 Rollout and the ratchet

| When | What | Exit criterion |
|---|---|---|
| Week 1 | Land `design/` (rules, matcher, plugin, scan). Generate `baseline.json` and `slots.json` from main's current build: `node design/scan.mjs dist/assets --update-baseline` with no baseline file present. The numbers in §7.4 are the 2026-11-09 sample. Lint blocks changed files. The scan ratchet blocks increases, including new duplicate primitives (`C1.2`). Agent contract added. Route registry confirmed. | CI green on main with the gates on |
| Week 2 | Build the pattern components and move one exemplar per pattern onto them. Gate 3 runs on the exemplars. Document them on `/dev/design-system`. | 8 exemplars × 4 modes green |
| Weeks 3–4 | Fix the S1 (critical) audit findings: generated UI on tokens (F-02), stock library removed (F-01), mode-dependent sizes (F-03). Then form dialogs (F-06), page titles (F-04) and empty states (F-05). Advisory rules become errors. | F-01 to F-06 closed; `C1.2` down from 81 to 17 |
| Ongoing | Every sprint lowers the baseline. Target: the colour metrics (T1.1, T1.2, T2.1) down 25% per quarter. 100% of new routes ship with `data-pattern`. | Baseline trends down; zero unregistered layouts |

### 7.10 Verification status

| Part | Status |
|---|---|
| `rules.json`, `match.mjs`, `scan.mjs` (including the `C1.2` slot count), ratchet | **Run.** The code blocks in this document were extracted from this file and run against the production build fetched 2026-11-09 (last re-run 2026-11-10, after the v1.1 changes). They reproduce the baseline above exactly. A simulated PR adding `text-[13.25px]` and `bg-[#123456]` failed with exit code 1 on T1.3 and T3.1. |
| `eslint-plugin.mjs`, the `no-restricted-*` rules and the full §7.3 config | **Run.** The config was extracted from this file and used unchanged: ESLint 9.39, `typescript-eslint` 8 and `eslint-comments` 4, on `.tsx`. Code rebuilt from production class strings gives 23 errors, all correctly attributed. A guideline-compliant file gives 0. A probe file shows where the plugin draws the line. All four values in the `/status` tone map (F-07) are caught, and so are `z-1`, `z-[15]`, `hover:z-30` and `z-(--top)`. UI copy such as "Totals are rounded to the nearest credit" and "fix z-index bug in sidebar" passes. The `Page` and `PageHeader` reference code in §4.7 gives 0. Removing the reason from PageHeader's exception comment fails `require-description`, and removing the comment fails C8.1. |
| Render matrix (§7.5), token contract test | Reference implementations. They need the repo and a seeded test workspace, so they haven't been run yet. The storage keys, `data-size` values and button heights they rely on were checked against the production bundle. |

---

## 8. Decisions for the CTO

Each decision has a default that applies until the CTO decides otherwise.

1. **Target design mode.** Recommendation: make `v2` the target. It has the complete status vocabulary, component dimension tokens and a calmer neutral palette. Freeze classic (no classic-only additions) and set a sunset date. *Default until decided:* both modes are supported and every screen passes all four combinations (T2.2).
2. **Remove the stock shadcn copy** and point the generative-UI catalog at `src/components/ui` (C1.2, GU1). The catalog first needs Creao versions of the four stock-only parts it renders (§9). *Default:* the `C1.2` ratchet blocks any new duplicate from week 1. When the copy goes, the baseline drops from 81 to 17.
3. **Retire `ox-*` and `.auth-legacy-theme`.** *Default:* frozen. No new usages (T1.4, P7.1).
4. **Approve the library requests in §9.** *Default:* the interim recipes in C6.1 and C7.2 apply.
5. **Fund a seeded test workspace for Gate 3.** *Default:* Gates 1, 2 and 4 run without it.

---

## 9. Library requests

These go to the owner of `src/components/ui` and the theme CSS. They add variants and fix defects. None of them redesigns the library.

| Request | Why | Interim |
|---|---|---|
| `EmptyState size="sm"` (no border or fill, `py-6`) | "No results" states inside lists are hand-rolled because the only EmptyState is a large dashed card | `className="border-none bg-transparent py-6"` (C6.1) |
| A `Callout` component with `tone` (`neutral`, `attention`, `failed`, `done`, `running`) | Only a class recipe exists. The stock `Alert` lives in the duplicate library. | The C7.2 recipe |
| `CardHeader`, `CardTitle` and `CardDescription` in `src/components/ui`, plus `Callout` (above) to replace `Alert` | Generated UI renders these four parts from the stock copy: the `.jr-block` CSS styles `[data-slot=card-title]`, `card-header`, `card-description` and `alert`, which only the stock copy defines. They must exist in Creao's library before the copy can be deleted (F-01). | None. Blocks F-01. |
| Put icon-only button sizes on the mode tokens | `icon` … `icon-lg` are fixed `size-6`…`size-10` while text buttons follow `--button-h`, so in v2 an `icon-md` button (36 px) stands 8 px taller than the default button beside it | On desktop, beside default-size text buttons, use `icon-sm` (32 px). It's 4 px off in both modes, instead of exact in classic and 8 px off in v2. Touch layouts follow L5.3. |
| Export the retry card as `ErrorState` | Has no `data-slot` and no clear export name | Use it under its current name |
| `density="compact"` on Card, Table, Badge and Tabs | Generated UI restyles these with 47 descendant selectors (GU3) | None. The existing selectors are frozen by the ratchet. |
| Move `z-[60]`…`z-[300]` into the T6 scale | 18 distinct z levels (0–300) in production; overlays at z-50 can end up underneath | None |
| Token fixes (audit F-17): a monotonic grey ramp, complete dark overrides (grey-0…950, `action-strong-hover`), `--jr-*` aliases (GU2), one name per value (`muted`/`muted-foreground`, the 6 aliases of `#fefdfb`) | These defects are why there are 489 manual `dark:` patches | T1.2 and T2.1 keep feature code off the broken parts |
| Document `raised` and `raised-outline` on `/dev/design-system` | Variants with no stated purpose | Don't use them (C3.5) |
