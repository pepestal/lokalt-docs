# Frontend Design & Code Review — Signal

**Scope:** `signal_frontend/` (Next.js 16 App Router, React 19, Tailwind 4 + CSS-variable theme engine, Zustand).
**Method:** Reviewed from source — every route, every shared component, the theme engine, and both chart stacks. Token drift quantified with repo-wide greps (counts are cited inline).
**Reviewed against first principles / the target "Scandinavian high-end fintech" aesthetic. `UI_RULES.md` and the other `src/docs/*` rulebooks were deliberately not read and did not inform any judgment here.**

> **Inspection limitation.** I could not run the dev server against live rendering. The app fetches everything from an external FastAPI backend (`signal_frontend/src/config.ts` → hardcoded `http://65.109.143.130:8000`), the default `API_KEY` is empty, and there's no browser-automation tool in this environment. Nearly every page is a client component, so SSR HTML wouldn't reflect the real render. **All visual findings are derived from code** (exact tokens, class names, computed fallbacks). Where I say "renders as X," it's traced through the CSS/JS, not observed on screen. Loading/empty/error states were read from source and are noted per-page. A follow-up visual QA pass at 380 / 768 / 1440px is still worth doing.

---

## 1. Executive summary — the ten that matter

1. **The large portfolio chart is silently un-themed and disagrees with the mini chart.** `PortfolioLargeChart` reads CSS variables that the theme engine never sets (`--chart-main-color`, `--main-text-color`, `--table-border-color`, `--chart-secondary-color`), so it always falls back to hardcoded hex: an **emerald** line + **violet** index. The dashboard mini chart uses the real tokens → a **blue** line + slate index. Same portfolio, two different color languages. (`signal_frontend/src/components/PortfolioLargeChart.tsx:47-54`)
2. **Six CSS variables are referenced but never defined anywhere** — `--text-main`, `--table-hover`, `--table-header-bg`, `--flash-positive`, `--flash-negative`, `--radius-outer`. They resolve to nothing. Real consequence: the live price-flash **row background** in Holdings never appears, and notification-card hovers/tints are dead.
3. **The "Motorer → Shadow" tab (`EngineStats`) is the single biggest aesthetic outlier** — 7 hardcoded palette hues (emerald/purple/blue/indigo icons, `emerald-400`/`red-400` numbers), raw `text-3xl`/`text-sm`/`text-xs`, and a Recharts chart hardwired to `#333`/`#666`/`#888`/`#1f2937`/`#374151` that ignores the theme entirely (broken in light mode).
4. **Three different charting idioms** live in the app: `lightweight-charts` (portfolio), `recharts` (engines), and hand-rolled SVG (mini chart, R-histogram, underwater plot). Axes, gridlines, tooltips and line weights don't match across pages. Standardize on the SVG idiom + one library.
5. **Accent-hue sprawl.** Beyond the justified mono + blue + green/red, the app also renders amber, rose (`#f43f5e`), violet (`#8b5cf6`), purple, indigo, and a stray warm cream (`#f4f1e8`) — roughly the "seven simultaneous accents" the target aesthetic explicitly rejects.
6. **`font-black` (900) is a no-op.** Inter is loaded at `400;500;600;700` only (`ThemeScript.tsx:227`), yet `font-black` is used 20× (plus the hero). It renders identically to `font-bold`. The type scale effectively has 4 weights, not the 5–6 the code implies.
7. **Radius drift.** A clean token trio exists (`--radius-bento` 24 / `--radius-inner` 12 / `--radius-buttons` 9999), but components also scatter `rounded-lg` (×22), `rounded-md`, `rounded-xl`, `rounded-2xl`, `rounded-sm` for tooltips, modals, inputs and pills — none of which map to the token scale.
8. **The genuinely excellent core is `SignalDataGrid` + the token-driven data surfaces** (`HoldingsTable`, `ScreenerTable`, `ScreenerGrid`, `SignalScore`, `DrawdownPanel`, `RHistogram`). These are disciplined, tabular-figure-correct, and fully theme-reactive. They should survive any redesign untouched.
9. **The runtime "Appearance engine" is enormous and works against the brief.** Vibrancy, material grain, six background textures, ambient bleed, X-ray mode, per-element text-scale sliders, saved themes — a maximalist customization surface layered under a product whose target feeling is *restraint*. The default theme is tasteful; almost none of the knobs move it toward the target and several move it away.
10. **`.text-bento-body` is used 6× but is not a defined class** — those texts silently fall back to inherited body size, so several notification titles/empty-states render at the wrong size.

---

## 2. App map

### Routes (`signal_frontend/src/app/`)
| Route | File | Purpose | Notes |
|---|---|---|---|
| `/` | `page.tsx` → `components/DashboardView.tsx` | Dashboard (hero value + mini chart, KPIs, top/bottom, watchlist, notifications, alerts, top picks) | 8-col hero + 4-col KPI bento grid |
| `/portfolio` | `portfolio/page.tsx` | Tabs: Innehav (`Overview` → `HoldingsTable`) / Statistik (`Statistics`) | |
| `/transactions` | `transactions/page.tsx` (1088 lines) | Ledger + CSV import (`CSVImportModal`) | Largest page file |
| `/statistik` | `statistik/page.tsx` | Tabs: Portfölj / Affärer / Marknaden (placeholder) / System | `PortfolioPerformancePanel`, `DrawdownPanel`, `AllocationSnapshot`, `AffarsStatistik`, `BigNumbers` |
| `/signaler` | `signaler/page.tsx` | Tabs: Signaler (`SignalsView`) / Bevakningslista (`WatchlistView`) | Merged Screener + Recommendations + Watchlist |
| `/engines` | `engines/page.tsx` | Tabs: Översikt (`EngineExplanations`) / Shadow (`EngineStats`) / Analys (`EngineAnalys`) / Detaljer (`EngineDetails`) | |
| `/settings` | `settings/page.tsx` | Tabs: Kontrollrum (`ControlRoom`) / Utseende (`Appearance`, 1013 lines) | |
| `/notiser` | `notiser/page.tsx` → `NotisCenterList` | Notification center (Inkorg/Olästa/Arkiv) | |
| `/screener`, `/watchlist`, `/recommendations` | `*/page.tsx` | **Redirect stubs** → `/signaler` | Good consolidation; keep |

Consistent page shell everywhere: `grid grid-cols-12 gap-[var(--grid-gap)] … p-[var(--page-gutter)] animate-cascade` + `<PageHeader>` with a `<GlobalSubmenu>` as children. This structural discipline is a real strength.

### Component tree (selected)
- **Layout/nav:** `Sidebar` (desktop), `MobileBottomNav` + `MobileMoreMenu` (≤768px), `NotificationDrawer`, `PageHeader`, `ui/GlobalSubmenu`.
- **Primitives:** `ui/BentoBox` (`cn` helper lives here), `ui/SignalScore`, `ui/SignalDataGrid/*`, `ui/BentoTooltip`, `ui/StatTip`, `ui/RangeSlider`, `ui/ColorPicker`, `ui/ToggleButton`, `ui/IconButton`.
- **Charts:** `PortfolioLargeChart` (lightweight-charts), `PortfolioMiniChart` (SVG), `EngineStats` (recharts), `statistik/*` (SVG).
- **Theme engine:** `store/useAppearanceStore.ts` (468 lines, single source of truth for the profile), `ThemeScript.tsx` (blocking anti-FOUC writer, stringified into `<head>`), `ThemeSync`, `TextureEngine`, `XRayBadge`/`XRayTooltip`.

### Token inventory (the counts that make drift undeniable)

**Grays.** The default dark palette is actually *restrained*: 8 slate steps (`#020617`, `#0f172a`, `#1e293b`, `#475569`, `#64748b`, `#94a3b8`, `#cbd5e1`, `#f8fafc`) in `useAppearanceStore.ts`. But components add **~13 more off-token grays** by hand: `#333`, `#666`, `#888`, `#9ca3af`, `#374151`, `#1f2937`, `#0d1117`, `#30363d` (EngineStats/EngineDetails), `gray-300`, `gray-400`, `slate-100/700/800/900` (GlobalSubmenu), and the cream `#f4f1e8`. **Effective distinct grays ≈ 20.**

**Accent hues.** Justified: blue `#3b82f6` (the one accent), emerald `#10b981` (positive), red `#ef4444` (negative). **Unjustified additions in code:** amber `#fbbf24`/`#f59e0b`, rose `#f43f5e` (a second red), violet `#8b5cf6`, purple `#a855f7`, indigo, `emerald-400`/`red-400` (off-token shades of the semantic pair), cream `#f4f1e8`. Grep of raw palette classes in `.tsx`: red ×9, emerald ×7, amber ×6, slate ×5, blue ×5, gray ×4, purple ×2, indigo ×1.

**Font sizes.** 12 tokenized classes (`--text-fluid-hero … --text-page-header`) — a real, well-structured scale. Undermined by **43 raw Tailwind size usages** (`text-3xl`, `text-sm`, `text-xs`, `text-[10px]`, `text-[11px]`, `text-[0.85em]`) and **6 uses of the undefined `.text-bento-body`**.

**Weights.** 5 requested (`black`/`bold`/`semibold`/`medium`/`normal`; `font-bold` alone ×358) but only 4 loaded → **`font-black` == `font-bold`** visually.

**Radii.** Token trio + `rounded-lg`×22, `rounded-md`×4, `rounded-xl`×3, `rounded-2xl`×2, `rounded-sm`×1, plus undefined `rounded-[var(--radius-outer)]`×1. **≈6 competing radii.**

**Shadows.** One token (`--shadow-bento`) + `shadow-sm`/`lg`/`xl`/`2xl`/`inner` + bespoke glows (`shadow-[0_0_10px_rgba(16,185,129,0.2)]`, `0_0_15px_rgba(16,185,129,0.3)]`) + three selection-glow classes.

**Charting:** 3 idioms (lightweight-charts, recharts, hand SVG).

---

## 3. Design verdict (ranked)

**What already lands — protect this:**
1. The **default dark theme** is exactly on-brief: near-monochrome slate, a single blue accent, green/red reserved strictly for signal direction. Calm and expensive-feeling.
2. **`SignalDataGrid`** is a serious, correct table engine — subgrid alignment, tabular figures, `+/−`/`%`/`SEK` suffix styling, sticky detached headers, right-aligned numeric anchors. This is the "data-forward, chrome recedes" ideal.
3. **Typography plumbing** — tabular-nums everywhere numbers live, a genuine 12-step token scale, disciplined uppercase-tracked labels.
4. **Structural consistency** of page shells, `PageHeader` glassmorphism, and the segmented submenu.
5. **Route consolidation** (screener/watchlist/recommendations → redirect stubs) shows real editorial subtraction.

**What breaks the feeling — ranked worst first:**
1. **Chart incoherence** (§1.1, §1.3, §1.4) — three visual languages plus a mis-themed hero chart. Charts are supposed to be the heroes; here they're the least consistent surface.
2. **The Appearance engine's maximalism** — glows, grain, textures, vibrancy, ambient bleed. Even off by default, its *existence* is the opposite of the target ethos and it's where a lot of code weight and rogue color lives.
3. **`EngineStats`/`EngineDetails`/`EngineExplanations` palette sprawl** — colored icon soup (emerald/purple/blue/indigo/amber) reads as crypto-dashboard, not Nordic fintech.
4. **Off-token grays and radii** accumulating in tooltips, modals, notification cards.
5. **The segmented submenu's active pill** — a warm cream (`#f4f1e8`) gradient with inset highlight + drop shadow. Decorative, off-palette, and the only warm color in an otherwise cool system.

Net: **the foundation is a high-end fintech product; the analytics leaves and the customization layer drag it toward a maximalist dashboard.** The gap is closable by subtraction, not redesign.

---

## 4. Per-page critique

### Dashboard (`/`) — `components/DashboardView.tsx`
- **Hierarchy is right:** the hero portfolio value (`text-fluid-hero`) wins, mini chart supports it, KPIs sit beside. Good first-glance read.
- **Change:** the mini chart (blue, tokenized) and the full chart on `/portfolio` (emerald, hardcoded — §5 C1) must match. Pick blue via tokens.
- **Change:** "Signalstyrka" is drawn **twice** with two different bar treatments — a raw `<div>` meter here (`DashboardView.tsx:247`) and the `SignalScore variant="meter"` component elsewhere. Use `SignalScore` here too.
- **Remove:** nothing structural; the card set is coherent.

### Portfolio → Innehav — `Overview.tsx` (1160 lines) + `HoldingsTable.tsx`
- `HoldingsTable` is a model surface — fully tokenized, tabular, semantic color only. Keep as-is.
- **Bug:** live price-flash row background is dead (`--flash-positive/-negative` undefined — §5 A2). The text-color flash works; the row wash doesn't.
- **Change:** `Overview.tsx` at 1160 lines is doing too much (fetch/poll orchestration, chart modal, dropdowns, SEK toggle). Extract data-fetching into a hook; it'll de-risk the file and make the hero chart swap trivial.

### Portfolio → Statistik & `/statistik`
- `DrawdownPanel`, `RHistogram`, `AllocationSnapshot`, `BigNumbers` are all clean SVG/token surfaces — **the reference for how every chart should look.**
- **Change:** `BigNumbers` uses `text-page-header` as a KPI number size (`BigNumbers.tsx:31`) — semantically it's a stat, not a page title. Add a dedicated `--text-stat` token so KPI numbers don't borrow the page-title size.
- "Marknaden" is an honest "coming soon" placeholder — fine, keep honest.

### Signaler (`/signaler`) — `SignalsView.tsx` + `ScreenerTable`/`ScreenerGrid`
- Strong page. Filter chips are token-driven; grid and table share the score/color libs.
- **Loading = spinner** (`Loader2`, tinted green `text-[var(--text-positive)]/50`), not a skeleton. For a data grid, a skeleton table reads calmer and more premium (§6 C).
- **Change:** the green tint on the spinner is arbitrary; make it `--text-secondary`.

### Motorer (`/engines`) — **the problem child**
- **`EngineStats` (Shadow tab):** rewrite the palette. Hardcoded `text-emerald-500/-400`, `text-purple-500`, `text-blue-500/-400`, `text-red-400`, `text-gray-400` (`EngineStats.tsx:419-469, 628-650`) → replace with `--text-positive/--text-negative/--text-secondary` and drop the multicolor icon set. The Recharts chart (`:544-566`) uses `#333/#666/#888` and a `#1f2937` tooltip that ignore the theme → re-skin to tokens or port to the SVG idiom.
- **`EngineDetails`:** `#0d1117`/`#30363d` "GitHub" code panes, `text-indigo-400`, `text-amber-500`, `text-gray-300/400`, green glow shadows (`:147, 174, 226-227`). Consolidate to `--bg-inner`/`--table-border`/`--text-*`.
- **`EngineExplanations`:** `text-amber-500`, `text-purple-500` icon accents (`:41, 50, 102`). These carry no semantic meaning — make them `--icon-color`.
- **Hierarchy note:** three stat cards each with a differently-colored icon compete for attention; none should win over the equity curve. Neutralize the icons, let the chart lead.

### Inställningar (`/settings`) — `ControlRoom`, `Appearance` (1013 lines)
- `Appearance` is impressively engineered but is the maximalism engine (§7). Not a per-page visual bug; a product-scope question.
- **Change:** if the brief is honored, most of `Appearance`'s effects tab (grain, textures, vibrancy, ambient bleed, shadow spread/blur sliders) should be cut, shrinking a 1013-line file dramatically.

### Notiser (`/notiser`) + `NotisCenterList`, `NotificationBento`, `NotificationDrawer`
- **Bugs:** `--text-main`, `--table-hover`, `--table-header-bg` are undefined (§5 A1). Result: unread vs read notification cards have **no background differentiation** (the `color-mix` on an undefined var collapses to transparent) and hover backgrounds don't fire. Titles use undefined `.text-bento-body`.
- **Change:** category→icon-color mapping (amber=Signaler, blue=Påminnelser) is duplicated inline in `NotisCenterList.tsx:33-37` and `NotificationBento.tsx:55-57`. Extract one `categoryMeta()` helper, and move the two category hues into tokens (`--cat-signal`, `--cat-reminder`) rather than raw `amber-500`/`blue-500`.

### Transactions (`/transactions`) — 1088 lines
- Not deep-read line-by-line (size), but it shares the same token vocabulary as Holdings. Flag for the same treatment: verify no raw palette classes crept in, and consider extracting the CSV-import flow (already partly in `CSVImportModal`).

---

## 5. Findings by severity

Paths are workspace-relative. Line numbers cite the anchor.

### Critical

**C1 — Large portfolio chart never reads the theme; disagrees with the mini chart.**
`signal_frontend/src/components/PortfolioLargeChart.tsx:47-54` reads `--chart-main-color`, `--main-text-color`, `--table-border-color`, `--chart-secondary-color`, `--chart-buy-color`, `--chart-sell-color`. **None are set by `ThemeScript.tsx`** (which sets `--graph-line-portfolio`, `--graph-line-index`, `--table-border`, `--text-secondary`). Every read falls back to hardcoded hex → line = emerald `#10b981`, index = violet `#8b5cf6`, grid = `#1e293b`. Meanwhile `PortfolioMiniChart.tsx:105-107` uses `graphLinePortfolio` (blue `#3b82f6`) and `graphLineIndex` (slate). **Why it matters:** same data, two color languages; the hero chart is also frozen against theme changes and light mode. **Fix:** read the variables that actually exist — `--graph-line-portfolio`, `--graph-line-index`, `--text-secondary`, `--table-border` — or set the `--chart-*` names in `ThemeScript`. Then delete the emerald/violet fallbacks.

**C2 — Candlestick + index colors are off-token even after C1.**
Same file `:166-176` hardwires candles to emerald `#10b981` / rose `#f43f5e`; `:38, 50` default the index to violet `#8b5cf6`. Rose ≠ the app's `--text-negative` (`#ef4444`); violet is a hue that exists nowhere else. **Fix:** candles → `--text-positive`/`--text-negative`; index → `--graph-line-index`.

### High

**A1 — Undefined CSS variables in the notification system.**
`--text-main` (used in 3 files), `--table-hover` (4 files), `--table-header-bg` (2 files) are never defined by `ThemeScript.tsx`/`globals.css`. E.g. `NotificationBento.tsx:48-50` builds card backgrounds from `color-mix(... var(--table-header-bg) ...)` → invalid → transparent → **read and unread look identical**; `NotisCenterList.tsx:137,159,169` hover to `var(--table-hover)` → **dead hover**. `notiser/page.tsx:66,97` also hover to `--table-hover`. **Fix:** map to real tokens (`--text-primary`, `--bg-hover`, `--bg-signal-table-header`) or define the aliases once.

**A2 — Live price-flash row background is dead.**
`HoldingsTable.tsx:584-586` sets row `backgroundColor: "var(--flash-positive)"` / `"var(--flash-negative)"` with no fallback; neither var is defined. The green/red **row wash** on a price tick never renders (the cell text-color flash at `:423` still works, which masks the bug). **Fix:** define `--flash-positive/-negative` (e.g. `color-mix(in srgb, var(--text-positive) 12%, transparent)`) in the theme.

**B1 — `EngineStats` bypasses the design system wholesale.**
`signal_frontend/src/app/engines/components/EngineStats.tsx`: hardcoded hues at `:419,443,464,469,428,452,637-646,728`; raw type sizes `text-3xl`/`text-sm`/`text-xs` at `:399,428,469,…`; Recharts theme-blind hex `#333/#666/#888` at `:546-548` and tooltip `#1f2937`/`#374151`/`#9ca3af` at `:298-299`. **Why:** breaks in light mode and under any custom theme; is the app's most off-brand surface. **Fix:** tokens for all colors; the 12 `.text-*` classes for all sizes; re-skin or replace Recharts.

**B2 — `font-black` renders as `font-bold`.**
`ThemeScript.tsx:227` loads Inter `400;500;600;700`; `font-black` (900) is used 20×. **Fix:** either load 800/900 in the Google Fonts URL, or replace `font-black` with `font-bold` and accept a 4-weight scale (recommended — see §6).

### Medium

**M1 — `.text-bento-body` is undefined** (used 6×, e.g. `NotisCenterList.tsx:53,65,99`, `NotificationBento.tsx:38,60`). No such utility exists (`globals.css` defines `-h1/-lg/-base/-sm/-xs`). Texts fall back to inherited size. **Fix:** replace with `text-bento-base`/`text-bento-sm`, or define the class.

**M2 — `rounded-[var(--radius-outer)]` undefined** (1×). Falls back to no radius. **Fix:** use `--radius-bento` or `--radius-inner`.

**M3 — Two chart libraries + one SVG idiom.** `lightweight-charts` (`PortfolioLargeChart`) and `recharts` (`EngineStats`) are both shipped; `recharts` (a heavy dep) powers a single tab. **Fix:** the hand-SVG idiom (`RHistogram`, `DrawdownPanel`, `PortfolioMiniChart`) already covers most needs cleanly and is fully themeable; port `EngineStats`' equity curve to it and drop `recharts` (bundle win + visual consistency).

**M4 — GlobalSubmenu active pill is off-palette and decorative.** `ui/GlobalSubmenu.tsx:60,66-67` hardcodes `text-slate-100 dark:text-slate-800` and a `from-slate-700 to-slate-800 dark:from-[#ffffff] dark:to-[#f4f1e8]` gradient with inset highlight + drop shadow. Warm cream `#f4f1e8` is the only warm color in the system. **Fix:** flat `--submenu-active-bg`/`--submenu-active-text` (which already exist) — no gradient, no cream.

**M5 — Duplicated category→color logic** across `NotisCenterList`, `NotificationBento`, (and `DashboardAlerts`). Extract one helper.

**M6 — `EngineStats` uses `eslint-disable react-hooks/exhaustive-deps`** at `:138` and elsewhere to paper over effect deps — fine functionally but a smell worth a `useCallback`.

**M7 — Hardcoded backend URL over HTTP.** `config.ts:1` defaults to `http://65.109.143.130:8000`. On an HTTPS deploy this is mixed content (blocked). Env override exists, but the insecure default is a footgun. **Fix:** default to same-origin/HTTPS or fail loudly if unset.

### Low

- **L1 — Spinner-only loading** on `SignalsView`/`BigNumbers`/`DrawdownPanel`; no skeletons. Premium fintech uses skeletons for tables/cards.
- **L2 — Green-tinted spinner** (`SignalsView.tsx:452`) — arbitrary semantic color on a neutral action.
- **L3 — `signaler/page.tsx`** root grid omits `animate-cascade` that every other page has (minor inconsistency; `SignalsView` animates its own box).
- **L4 — Emoji in UI strings** (`⚠️` in `SignalsView.tsx:456`, `✕` glyph as a close button in `EngineDetails.tsx:170`) instead of the `lucide` icon set used everywhere else.
- **L5 — `contributingEngines` badge** in `SignalScore.tsx:132` uses `opacity-60` on top of a semantic color — verify AA contrast on `strong-sell` red at small size.
- **L6 — `any` types** pervade chart/data props (`PortfolioLargeChart` `transactions?: any[]`, `EngineStats` `data: any`). Not visual, but brittle.

---

## 6. Recommended design system

Sized for **incremental** adoption — it mostly *ratifies* the existing good tokens and deletes the drift. Add these to `:root` in `globals.css` and to the `darkTheme`/`lightTheme` objects in `useAppearanceStore.ts`; then migrate off-token values file-by-file.

### Type scale (rem @ 16px base) — keep the 12 tokens, fix the weights
| Token | Size | Weight | Use |
|---|---|---|---|
| `--text-fluid-hero` | clamp 2.5→4.5rem | 700 | Portfolio value only |
| `--text-page-header` | 1.5rem (24px) | 600 | Page titles (stop reusing for KPIs) |
| **`--text-stat` (new)** | 1.75rem (28px) | 700 | KPI numbers (BigNumbers, stat cards) |
| `--text-bento-h1` | 1.25rem (20px) | 600 | Card titles |
| `--text-bento-lg` | 1.5rem (24px) | 600 | Emphasis numbers |
| `--text-bento-base` | 1rem (16px) | 500 | Body |
| `--text-bento-sm` | 0.75rem (12px) | 500 | Secondary / labels |
| `--text-bento-xs` | 0.625rem (10px) | 700 | Micro labels (uppercase) |
| table/signal-table tokens | as-is | — | keep |

**Weights: standardize on 4 — 400 / 500 / 600 / 700.** Delete `font-black` (it's already 700 at runtime). Load exactly `400;500;600;700`.

### Spacing — already good; make it law
Base unit **8px**. Keep `--grid-gap: 1.5rem`, `--padding-bento-standard: 1.5rem`, `--padding-bento-hero: 2rem`, `--padding-bento-inner: 1rem`. **Ban** raw `p-3`/`gap-3`/`mb-2` where a token exists; the scale is 4/8/12/16/24/32.

### Palette (dark) — collapse ~20 grays → 6, cap accents at 4
```
--bg-main:        #020617   /* app bg / sidebar */
--bg-bento:       #0f172a   /* cards, gradient end */
--bg-inner:       rgba(2,6,23,0.5)
--surface-line:   rgba(255,255,255,0.06)  /* == --table-border, all hairlines */
--text-primary:   #f8fafc
--text-secondary: #94a3b8
--text-tertiary:  #64748b   /* == today's --text-code/--text-ticker; one step, not three */

--accent:         #3b82f6   /* the ONE accent */
--positive:       #10b981
--negative:       #ef4444
--warning:        #fbbf24   /* the 4th, semantic only (stale/attention) */
```
**Delete from the codebase:** rose `#f43f5e`, violet `#8b5cf6`, purple/indigo icon accents, `emerald-400`/`red-400` shades, cream `#f4f1e8`, and the `#333/#666/#888/#0d1117/#30363d` chart/code grays → map each to the six above. Per-engine chart colors (from the backend registry) are the one legitimate multi-hue exception; keep them **only** inside chart series, never on chrome/icons.

### Radius
```
--radius-bento: 16px   /* consider dropping 24→16: 24px reads slightly soft/consumer; 16 is more "exact" */
--radius-inner: 12px
--radius-control: 8px  /* NEW: inputs, tooltips, small buttons — replaces rounded-lg/md */
--radius-pill: 9999px
```
**Ban** raw `rounded-lg/md/xl/2xl/sm`; every corner maps to one of these four.

### Shadow / elevation
Prefer the `theme-outline` physics (1px `--surface-line`) as the default for a flatter, more "exact" Nordic look; reserve `--shadow-bento` for the hero only. **Delete** all bespoke glow shadows (`shadow-[0_0_10px_rgba(16,185,129,…)]`, selection glows can stay as they're functional focus rings).

### Chart spec (one language, applied to every chart)
- **Grid:** horizontal only, `--surface-line`, 0.5px non-scaling.
- **Axes:** labels `--text-secondary`, `text-bento-xs`, tabular figures, no axis lines (or `--surface-line`).
- **Series:** portfolio = `--accent`, benchmark/index = `--text-tertiary` dashed, positive/negative fills = `--positive/--negative` @ ~18% opacity. Line width 1.5–2px, no dots.
- **Tooltip:** reuse the `BentoTooltip`/`--bg-bento` + `--surface-line` + `--radius-control` panel that the rest of the app already uses — never a bespoke `#1f2937` box.
- **Library:** hand-SVG for spark/histogram/underwater; `lightweight-charts` (themed via the *correct* vars) for the interactive price chart; **remove `recharts`.**

---

## 7. Additions / removals

### Remove (minimalism = subtraction) — in priority order
1. **`recharts`** (dependency) once `EngineStats` is ported to SVG — bundle + consistency win.
2. **The decorative half of the Appearance engine:** material grain (`TextureEngine`), the 6 background textures, vibrancy/glass edges, ambient bleed, shadow spread/blur sliders, per-element text-scale sliders. These are the antithesis of "calm, exact, expensive," add real code weight (`Appearance.tsx` 1013 lines, `useAppearanceStore` 468), and are where rogue color originates. Keep only: light/dark mode, and *maybe* an accent-color picker.
3. **The GlobalSubmenu active-pill gradient** (→ flat token).
4. **`font-black`** (→ `font-bold`).
5. **Emoji glyphs** in UI copy (→ lucide icons).
6. **Duplicated category-color blocks** (→ one helper).
7. Consider removing **X-Ray mode** from the shipped bundle (it's a dev tool) or gating it behind a dev flag.

### Add
1. `--text-stat`, `--radius-control`, `--flash-positive/-negative`, `--text-tertiary` tokens (§6).
2. **Skeleton loaders** for tables/cards (replace spinners).
3. A single **`Chart` wrapper** enforcing the §6 chart spec so no future chart can drift.
4. A tiny **stylelint/eslint rule** (or a grep in CI) banning raw hex, `text-*-\d`, `rounded-(lg|md|xl|2xl|sm)`, and raw `text-(xs|sm|3xl…)` in `src/**` — the token discipline already mostly holds; make regressions impossible.
5. **Data-age indicator**: the app polls a live backend but the UI rarely admits staleness. A subtle "senast uppdaterad …" using `--warning` when stale would fit the fintech-trust ethos (the `BigNumbers` sync-time is the only place that does this today).

---

## 8. Quick wins (< ~1 hour each)

1. **Fix C1** — swap the four `--chart-*` reads in `PortfolioLargeChart.tsx:47-54` to `--graph-line-portfolio` / `--text-secondary` / `--table-border` / `--graph-line-index`; delete emerald/violet fallbacks. Instantly unifies the two charts. *(~20 min)*
2. **Define the 6 missing vars** (`--text-main`→`--text-primary`, `--table-hover`→`--bg-hover`, `--table-header-bg`→`--bg-signal-table-header`, `--flash-positive/-negative`, `--radius-outer`→`--radius-bento`) as aliases in `globals.css`. Fixes A1, A2, M2 in one edit. *(~15 min)*
3. **Replace `.text-bento-body`** (6 occurrences) with `text-bento-base`. *(~10 min)*
4. **Neutralize EngineExplanations/EngineDetails icon colors** — `text-amber-500`/`text-purple-500`/`text-indigo-400` → `text-[var(--icon-color)]`. *(~15 min)*
5. **Candlestick colors** → `--text-positive`/`--text-negative` in `PortfolioLargeChart.tsx:166-176`. *(~10 min)*
6. **GlobalSubmenu pill** → flat `--submenu-active-bg`, drop the cream gradient + inset shadow. *(~15 min)*
7. **`font-black` → `font-bold`** repo-wide (20 sites) — visually identical now, removes a false weight. *(~10 min)*
8. **Spinner de-tint** in `SignalsView.tsx:452` → `--text-secondary`. *(~5 min)*

---

### Closing note
The bones are genuinely good — `SignalDataGrid`, the token scale, tabular figures, the default palette, and the page-shell discipline are exactly what a Nordic fintech product should be built on. The distance to the target aesthetic is almost entirely **subtraction and consolidation**: unify the charts, delete the decorative theme layer, and force the analytics leaves back onto the tokens that the core already respects. None of it requires a redesign.
