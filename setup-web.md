# Claude Code Agentic Workflow — Replication Guide for Websites

> **What this is:** A battle-tested Claude Code workflow system, adapted from
> a production mobile setup to the web. It makes Claude design-first,
> test-first, self-correcting, and self-documenting — without the user having
> to ask each time. It is written to be installed into **any website
> codebase**, whatever the stack: React (Vite / Next.js / Remix), or pure
> HTML + CSS + JavaScript, with or without TypeScript.
>
> **Who this is for:** You, Claude, working in a target web repo. Your job is
> NOT to copy files verbatim. Your job is to **curate this system for the
> site you are in** — its framework, its build tooling, its test framework,
> its design system, its deploy pipeline. Every template below uses
> `{{PLACEHOLDERS}}` you resolve during Phase 0, and every phase says what to
> adapt and what to keep.
>
> **Time to set up:** roughly one focused session. Do the phases in order —
> each builds on the previous one. Confirm the Phase 0 findings with the user
> before writing any files.

---

## The Philosophy (read this first)

The system has five layers, each with a distinct job. Don't blur them:

| Layer | Location | Job | Analogy |
| --- | --- | --- | --- |
| **CLAUDE.md** | repo root | Compact entry-point index: overview, critical rules, quick-reference table, commands. Links out to docs — never duplicates them. | Table of contents |
| **Rules** | `.claude/rules/*.md` | Contextual guidance auto-applied by file path (design-system constraints, TDD decision scorecard, "offer a mockup first"). Rules *guide judgment*. | Style guide |
| **Skills** | `.claude/skills/*/SKILL.md` | Repeatable multi-step procedures invoked as `/name` (scaffold a page, generate a mockup, gated deploy). Skills *encode procedures*. | Runbooks |
| **Hooks** | `.claude/settings.json` + `.claude/hooks/*.sh` | Deterministic enforcement the model can't forget: auto-format on save, block protected files, verify on Stop, re-inject rules after compaction. Hooks *enforce mechanically*. | CI on your desk |
| **Agents** | `.claude/agents/*.md` | Delegated audits with focused prompts (clean-code sweep, UI review, perf audit, a11y/SEO audit, architecture planning). Agents *review at scale*. | Specialist reviewers |

Two workflow principles sit on top:

1. **Design-first, at real viewport widths.** Non-trivial UI is never
   implemented cold. First a standalone HTML mockup is generated — the page
   or component rendered inside **viewport frames**: a browser frame
   (browser chrome with traffic-light dots, tab, and URL bar) at the desktop
   baseline (1440px), plus a **phone frame at 390px** for the mobile
   breakpoint — styled with a shared `docs/_theme.css` that mirrors the
   site's real design tokens, opened in the browser, and approved by the
   user. Only then is it implemented, and optionally verified against the
   mockup with a `design-match` skill. This kills the "build it, hate it,
   rebuild it" loop. Every mockup shows **desktop AND mobile together** —
   the web is responsive by definition, and a design approved at one width
   only is a design half-approved.

   The mockup layer is a **ladder**, climbed only as far as the work needs:

   - **Static mockup** (`/html-mockup`) — one page or section, its states
     and breakpoints side by side.
   - **Design variants** (`/design-variants`) — when the *direction* is
     undecided: research inspiration (web + design galleries), then one
     showcase page with 3–5 live, labeled variants in mini browser frames;
     the user picks before any site code.
   - **Interactive mockup** (`/interactive-mockup`) — when the work is a
     *flow* (signup, checkout, booking, multi-step form): a clickable
     simulator with a control panel (step navigation + state/edge-case
     toggles) beside the framed page; becomes the implementation spec.
   - **Design match** (`/design-match`) — after implementation, real
     browser screenshots vs. the approved mockup until they agree.

2. **Test-gated, judgment-based TDD.** A scorecard (not a blanket mandate)
   decides when TDD applies: 2+ signals (has logic / has a contract / can
   break silently / touches shared code / non-trivial) → write failing tests
   first. Pure style/layout/copy changes skip it. Tests gate every deploy; a
   Stop hook nudges (never writes) missing tests.

The meta-rule: **the user should never have to ask for any of this.** Rules
auto-apply, hooks auto-run, skills auto-trigger from their descriptions.

---

## Phase 0 — Discover the Site (do this before writing anything)

Explore the repo and fill in this table. Then **show it to the user and
confirm** before generating files — wrong assumptions here poison everything
downstream.

| Variable | How to find it | Examples across web stacks |
| --- | --- | --- |
| `{{SITE_NAME}}` | package.json, README, `<title>` | indiraIVF |
| `{{STACK}}` | deps, project files | React + Vite / Next.js / Remix / pure HTML + CSS + JS (no build) / HTML + a bundler |
| `{{SRC_EXTS}}` | file census | `.ts .tsx` / `.js .jsx` / `.html .css .js` |
| `{{RUN_CMD}}` | scripts, README | `npm run dev` (Vite/Next) / `npx serve .` or `python3 -m http.server` / Live Server |
| `{{BUILD_CMD}}` | scripts | `npm run build` / **none** (static files are the build) |
| `{{TEST_CMD}}` + `{{TEST_ONE_CMD}}` | scripts, CI config | `npm test` + `npx vitest run <file>` / `npx jest <file>` / **none yet** |
| `{{STATIC_CHECK_CMD}}` | typechecker or validators | `tsc --noEmit` / `npx html-validate '**/*.html'` + `npx stylelint '**/*.css'` |
| `{{LINT_CMD}}` / `{{FORMAT_CMD}}` | configs | `eslint --fix` + `prettier --write` / prettier alone for pure HTML/CSS/JS |
| `{{DESIGN_TOKENS}}` | theme files, tailwind config, `:root` CSS vars | Tailwind theme / CSS custom properties in `styles/tokens.css` / styled-components theme / **none yet** |
| `{{STYLE_RULES}}` | what's forbidden vs. allowed in UI code | tokens only — no hardcoded hex, no arbitrary px outside the spacing scale; rem for type; sanctioned breakpoints only |
| `{{MODULE_PATTERN}}` | src layout | `src/features/<feature>/{components,hooks,api}` + shared `src/lib/` / `components/` + `pages/` / per-page folders with `index.html` + `style.css` + `script.js` + shared `assets/` |
| `{{ROUTING_PATTERN}}` | how pages are wired | react-router routes / Next.js file routes / plain `.html` files + `<a>` links |
| `{{API_PATTERN}}` | how HTTP is done | service modules in `src/api/` with a shared fetch/axios client / Next server actions or route handlers / a single `api.js` with fetch wrappers / **static site, no API** |
| `{{E2E_TOOL}}` | deps, config dirs | Playwright (`e2e/` or `tests/`) / Cypress (`cypress/`) / **none** |
| `{{PROTECTED_FILES}}` | secrets and deploy configs that must not be silently edited | `.env*`, `vercel.json`, `netlify.toml`, `firebase.json`, `wrangler.toml`, `CNAME`, `robots.txt`, `.github/workflows/*`, analytics/GTM snippets |
| `{{DEPLOY_PIPELINE}}` | scripts, CI, host config | gated `npm run deploy` → Vercel / Netlify / Cloudflare Pages / GitHub Pages / rsync or S3 upload; preview deploys on PR if any |
| `{{BROWSER_TARGETS}}` | browserslist, analytics, user base | last 2 Chrome/Safari/Firefox/Edge + iOS Safari / must support older Android browsers (know your audience) |
| `{{CI}}` | `.github/workflows/`, etc. | lint + tests on PR / host's auto-build / none yet |

Also note:

- **Test file convention** — co-located `*.test.ts(x)` next to source?
  `__tests__/` dirs? A separate `tests/` tree? Match what exists; if nothing
  exists, propose the stack's idiomatic default (Vitest co-located for
  Vite/React; a plain `tests/` folder with Vitest or Jest for vanilla JS).
- **Which directories carry logic** (hooks, api/services, state/context,
  utils, form validation) vs. which are thin (route files, purely
  presentational components, static pages) — this drives the TDD scorecard
  and the suggest-tests hook.
- **Rendering mode and its constraints** — static site, SPA, SSR, or SSG?
  This decides where SEO lives (real `<meta>` in HTML vs. a head-management
  layer), what "a page" means, and what the deploy artifact is.
- **Existing docs** — fold them into the docs structure rather than duplicate.
- **Naming and structure conventions** already in force (casing, file naming,
  import ordering, CSS methodology — BEM/utility/modules) — these become
  Critical Rules.

Special cases to raise with the user before proceeding:

- **Empty repo / no codebase to analyze**: Phase 0 has nothing to discover,
  so **ask the user which existing website to replicate the design system
  from** (their current live site, a competitor, or a reference site they
  admire). Then extract the token set from that site — fetch its pages and
  read the rendered CSS: colors, font families and type scale, spacing
  rhythm, radii, shadows, breakpoints, header/footer patterns. Confirm the
  extracted tokens with the user (show them as a small HTML swatch page
  using the Phase 1 theme skeleton) before writing `docs/_theme.css` or any
  code — these tokens become the design system the whole workflow enforces.
  Also confirm the stack choice (React vs. pure HTML/CSS/JS) at the same
  time, since it fills most of the remaining Phase 0 table.
- **No design system** (ad-hoc colors and sizes scattered through CSS): the
  design-first workflow needs tokens to be meaningful. Offer to extract a
  token set (colors, type scale, spacing, radii, breakpoints) from the
  existing UI into `:root` CSS custom properties (or the Tailwind theme)
  first — it becomes both the site's theme layer and the mockup theme in
  Phase 1. For a **brand-new empty repo**, define the tokens WITH the user
  as step one — everything downstream consumes them.
- **No test framework wired up**: setting one up is a prerequisite, not part
  of this system. Propose the stack default (Vitest + React Testing Library
  for React; Vitest for vanilla JS modules), seed tests for the most
  load-bearing logic, then install the TDD rule so coverage grows with every
  change. For a purely static content site with near-zero JS logic, say so —
  the TDD layer shrinks to "any JS module with logic gets a test" and the
  E2E layer (Playwright smoke of key pages) does the heavy lifting.
- **Pure HTML/CSS/JS with no build step**: that's fine — this system does
  not require a bundler. `{{STATIC_CHECK_CMD}}` becomes validators
  (html-validate, stylelint), `{{BUILD_CMD}}` disappears, and the deploy gate
  runs checks against the files themselves.

---

## Phase 1 — Docs System + Shared Mockup Theme

### 1a. Directory structure

```text
docs/
  _theme.css          # Shared dark/light CSS for ALL HTML docs and mockups (see 1b)
  _theme.md           # Cheat-sheet: skeleton, component classes, tokens
  architecture.md     # Tech stack, directory map, data flow, patterns
  documentation-guide.md  # How the doc system works + feature-doc template
  SETUP.md            # Dev environment setup (prerequisites, local server, build)
  workflows/          # Step-by-step how-tos (add a page, add an API call, deploy runbook)
  guides/             # Reference guides (responsive strategy, SEO, a11y, perf budgets, hosting)
  features/           # One .md per shipped feature/page (template below)
  mockups/<feature>/  # Viewport-framed HTML design mockups, organized by feature
  research/           # Audits, deep dives, proposals
  implementation/     # Plans for upcoming work
  testing/            # Testing strategy
  metrics/            # Lighthouse/CWV reports, audit logs
```

### 1b. `docs/_theme.css` — the keystone of design-first

One shared stylesheet used by every HTML doc and every mockup. Build its
tokens from the **site's real design system** so mockups genuinely preview
the site:

- **Tokens as CSS vars:** surfaces (`--bg`, `--card`, `--glass`), text
  (`--fg`, `--muted`), borders (`--hairline`), the site's accent palette, a
  3-stop gradient, the full type scale (`--t-xs`…`--t-6xl`), the site's
  actual font stacks, radii, shadows, and the **breakpoint values** as
  documented constants. Dark default + light counterparts that swap via
  `prefers-color-scheme` plus a manual `data-theme` override with a
  `.theme-toggle` button. Pull the values from `{{DESIGN_TOKENS}}` — the
  Tailwind config or the site's `:root` variables — so a hex in the mockup
  is the same hex the site ships. (If the site itself uses CSS custom
  properties, the mockup theme can import or literally copy that file — one
  source of truth.)
- **Doc components:** page `.wrap`, `.hero` (title + meta sidebar),
  `.eyebrow`, numbered `.sec-head`, `.grid.cols-2|3|4`, `.card`,
  `.pill.ok|warn|info`, `.table-wrap`, `.flow .step` (auto-arrow flow
  diagrams), `.phases` (roadmap columns), `.callout.warn|danger|ok`,
  `.compare .col`, syntax-highlighted `pre`.
- **The viewport-frame components** — used by every mockup (see Phase 4):
  - `.browser-frame` — a desktop browser chrome: rounded top bar with
    traffic-light dots, a tab, and a URL bar showing the page's real path;
    content area at the **desktop design baseline (1440px, scaled to fit
    with `transform: scale()` when needed)**, hairline border,
    `overflow: hidden`, site `--bg` background.
  - `.phone-frame` — a 390px-wide phone body (rounded corners, notch, home
    indicator) for the **mobile breakpoint** of the same design.
  - Optionally `.tablet-frame` at 768px when a design changes meaningfully
    at tablet width.
  Define these once in the theme so every mockup gets identical frames. If
  the site's own design baseline differs (e.g. 1280px desktop), match the
  site's baseline.
- Write `docs/_theme.md` alongside: HTML skeleton + a component/token
  cheat-sheet table, so future sessions compose docs without re-reading the
  CSS.

**The one rule with an exception:** explainer/research/plan HTML docs MUST
use the theme components (no one-off `<style>` blocks). UI **mockups** link
the theme for tokens and the frames but get custom CSS inside the page —
pixel-level mockup control needs it.

### 1c. Feature-doc template (in `documentation-guide.md`)

Every shipped feature or page gets `docs/features/<kebab-name>.md` with
grep-searchable tags at the top:

```markdown
# Feature: <Name>

@features <feature-name>
@pages </route1> </route2>
@modules <module1> <module2>
@services <service1>

## Summary
One paragraph: what and why.

## Code Index
Tables of Pages/Routes / Components / State & logic (hooks, stores, JS
modules) / API calls → file → purpose.

## Data Flow, SEO/meta notes, Decisions, Changelog
```

The `@tags` make `grep -r "@pages /booking" docs/` a working feature-lookup
tool.

---

## Phase 2 — CLAUDE.md (the entry point)

Structure it exactly like this (compactness is the feature — link, don't
inline):

1. **One-paragraph site summary** — stack, rendering mode (static/SPA/SSR),
   styling approach, architecture in 3 lines.
2. **Documentation index** — table linking every doc, plus the `docs/` tree.
3. **Project structure tables** — pages/routes, feature modules, shared
   libraries, shared components, types/models (or JS modules). One line per
   entry: path → purpose.
4. **Critical Rules** — numbered, ~15 max, each one sentence of rule + one
   of detail. The universal web shapes, instantiated with Phase 0 findings:
   - API calls only through `{{API_PATTERN}}` — never raw `fetch` scattered
     through components/pages.
   - Import/dependency discipline: `{{path aliases, no deep relative paths,
     no barrel imports — whatever the repo enforces; for no-build sites:
     script-tag order / ES-module conventions}}`.
   - Naming conventions: `{{casing, prefixes, file naming, CSS class
     methodology (BEM/utility/modules)}}`.
   - **Design tokens only** — no hardcoded colors, no arbitrary pixel values
     outside the scale; `{{STYLE_RULES}}`; type in `rem`, spacing from the
     scale, only sanctioned breakpoints.
   - **Responsive discipline** — mobile-first CSS; every page correct at
     390 / 768 / 1440; no fixed widths that break small screens; no
     horizontal scroll ever.
   - **Semantic HTML + accessibility** — landmarks, one `h1` per page,
     labeled form controls, alt text, focus-visible states, keyboard
     operability. A11y violations are errors, not polish.
   - **SEO/meta hygiene** — every page has a unique `<title>` and meta
     description, canonical URL, and OG tags via `{{the head-management
     layer or plain HTML}}`.
   - Structure conventions: `{{component function style, internals
     ordering, ~100-line limit before extracting logic to a hook/module}}`.
   - Route/page entries are thin delegates; business logic lives in
     `{{MODULE_PATTERN}}` modules, which are self-contained.
   - Error-handling idiom: `{{what surfaces to users (toasts/inline
     messages/error boundaries), what goes to logging/error reporting}}`.
   - State/persistence idioms: `{{context/store pattern, localStorage
     discipline, URL-as-state for shareable views — as applicable}}`.
   - **Performance budget** — `{{e.g. LCP < 2.5s, CLS < 0.1, INP < 200ms;
     images sized + lazy-loaded + modern formats; JS budget per page}}`.
   - **Tests + checks gate every deploy — never ship from an unverified
     tree.** Name the blocking checks vs. advisory ones explicitly, and the
     exact commands that must not be bypassed (no raw host deploys, no
     skipped test flags).
5. **Observability table** — error reporting, analytics, session replay,
   uptime: where each lives and the one-liner to use it.
6. **Claude Code Tooling section** — tables of the skills, agents, rules,
   and hooks you're about to create in Phases 3–6, so every future session
   knows they exist. *This is what makes the system self-triggering.*
7. **Quick Reference table** — "How do I X?" → link/answer. 15–20 rows
   (add a page, add an endpoint call, run one test, run E2E, deploy, where
   auth state lives, how meta tags work, where the tokens live, how to run
   Lighthouse…).
8. **Commands block** — dev, quality, test, deploy commands with warnings
   inline (e.g. "ALWAYS use the gated deploy script, never raw
   `vercel --prod`/`netlify deploy --prod`/direct upload to bypass the
   gate").

---

## Phase 3 — Rules (`.claude/rules/`)

Three files. Adapt the specifics; keep the shapes.

### 3a. `ui-components.md` — design-system enforcement (auto-applies to UI files)

State ONLY-allowed and NEVER-allowed patterns explicitly — **allowlists beat
prose**. Instantiate with `{{DESIGN_TOKENS}}` and `{{STYLE_RULES}}`:

```markdown
# UI Rules (Auto-Applied to {{UI_FILE_GLOB — e.g. src/**/*.tsx, **/*.css, **/*.html}})

## Color Usage
ONLY these patterns: {{exhaustive list of allowed token references —
var(--token) / Tailwind theme classes / theme object keys}}
NEVER: {{raw palette values — hex literals, rgb() one-offs, named CSS
colors, default framework palette classes}}

## Spacing / Typography / Breakpoints
ONLY the design scale: {{spacing scale, type scale in rem}}.
ONLY the sanctioned breakpoints: {{e.g. 640 / 768 / 1024 / 1280}} — never
ad-hoc @media widths.
NEVER arbitrary values ({{p-[13px] / margin: 17px / font-size: 15.5px —
whatever the local sin looks like}}).
Exception: {{the sanctioned escape hatch, if any}}.

## Structure & Semantics
- {{component/function declaration convention}}
- Internals order: {{e.g. framework hooks → app state → local state →
  handlers → effects → early returns → render}}
- ~100-line max for component/page logic before extracting a
  {{hook / module}}
- Pre-compute filtering/mapping BEFORE the render/return
- Semantic elements over div-soup: nav/main/section/article/button —
  a clickable thing is a <button> or <a>, never a div with onClick
- Every interactive element keyboard-reachable with a visible focus state
- Images: explicit width/height (or aspect-ratio), alt text, lazy-load
  below the fold, modern formats
- Mobile-first media queries; test at 390 / 768 / 1440 before calling
  any UI change done
```

### 3b. `tdd-workflow.md` — the TDD decision scorecard + cycle

The most important rule file. Keep its structure intact; only the examples
and commands change per stack:

```markdown
# TDD-First Workflow (Smart Auto-Enforced)

Before writing ANY code, evaluate whether the task needs TDD. This decision
must be automatic — the user should never have to tell you.

## Step 0: DECIDE — score these 5 signals; 2+ YES → TDD, else skip:
| Signal | YES | NO |
| Has logic? | conditionals, state transitions, transforms, validation | pure layout, static content |
| Has a contract? | input→output, hook/store returns, API shape | renames, import moves |
| Can break silently? | edge cases, async, error paths, date/price math | color change, copy text |
| Touches shared code? | shared lib/api/state modules | one-off leaf component |
| Is non-trivial? | new function/service/hook, >10 lines logic | one-liner, prop passthrough |

Explicit APPLIES list ({{this repo's logic dirs — hooks/api/state/utils,
form validation, data transforms}}) and DOES-NOT-APPLY list ({{thin
route/page entries, style-only, markup-only, type-only, config, generated
code}}). When skipping, say: "No TDD — [reason]".

## The Cycle: ANALYZE (identify testable units + test file locations) →
RED (write failing tests FIRST, run {{TEST_ONE_CMD}}, confirm failure) →
GREEN (minimal implementation, all pass; fix the code, not the tests) →
REFACTOR (stay green, no new behavior).

## Test Patterns — concrete copy-paste snippets for THIS stack:
how to mock the {{API_PATTERN}} layer (msw / vi.mock / fetch stub), how to
test {{hooks / stores / plain JS modules}}, how to test components with
{{React Testing Library / DOM assertions on plain JS}}, how to test error
paths, how to test a form's validation.
## Running Tests — one file, watch, all — with real commands.
```

Write the test-pattern snippets against the site's *actual* test stack —
this section is what makes future sessions productive instead of flailing
at mock setup.

### 3c. `offer-mockup.md` — the design-first trigger

```markdown
# Offer a Viewport Mockup Before UI Implementation

When asked to build, redesign, or significantly change a page, section, or
component with non-trivial UI (not a one-liner tweak), ask before coding:
"Want me to generate an HTML mockup first so you can see the design in
desktop and mobile frames before I implement it? (/html-mockup)"

Offer when: new page or major section; layout overhaul; UI described
verbally with no reference; multiple layouts could work.
Route to /design-variants instead when the user wants options, asks for
something "cooler", or has rejected a previous design direction.
Skip when: a Figma link or screenshot was provided to match; small tweak
(color, spacing, copy); user said just implement; user already declined a
mockup this conversation.
```

---

## Phase 4 — Skills (`.claude/skills/<name>/SKILL.md`)

Each skill: YAML frontmatter (`name`; `description` written so the model
auto-triggers it from context; `argument-hint`; `allowed-tools` scoped
tight) plus a stepwise body.

### The design-first suite (highest value — build these first)

Four skills, matching the mockup ladder from the Philosophy section. All of
them present designs inside the viewport frames (desktop browser frame +
390px phone frame).

**`html-mockup`** — the heart of the workflow:

1. Parse arguments; if unclear ask what page/section, any reference (Figma /
   screenshot / verbal), and which states to show (empty, loading, error,
   populated).
2. Read `docs/_theme.css` for available tokens and the `.browser-frame` /
   `.phone-frame` components.
3. Generate `docs/mockups/<feature>/<name>.html`:
   - The page built **inside the browser frame at the desktop baseline**,
     with the site's real header/nav and footer if the page has them, AND
     the same design **inside the phone frame at 390px** — both breakpoints
     on one mockup page. (For a component rather than a page, frame just
     the component region, but still show both widths if it responds.)
   - Theme tokens only outside the `<style>` block — the mockup must use
     the same colors/type/spacing the site ships.
   - Realistic content, never lorem ipsum — use domain-plausible data
     (real service names, plausible prices, real city names).
   - Multiple states shown as **multiple frames side by side** (or tabs),
     one frame per state.
   - An annotations section below the frames: design notes on spacing,
     responsive behavior between breakpoints, interactions, animations,
     hover/focus states, and edge cases.
4. `open` the file in the browser; ask for changes and iterate in the
   mockup (cheap) rather than in site code (expensive).
5. **After approval, summarize the implementation plan but do NOT implement
   until told** — including how theme vars map to the site's styling layer
   (`{{Tailwind classes / CSS custom properties / theme object}}`) and how
   the responsive behavior maps to the sanctioned breakpoints.

**`design-variants`** — for when the *direction* is undecided (user wants
options, asks for something "cooler", or rejects a first design):

1. Parse the brief: what surface, what it must communicate, any references.
   If a static mockup exists for the surrounding page, read it — variants
   must fit its palette and tone.
2. **Research in parallel**: design-inspiration sources (WebSearch for
   named patterns, landing-page galleries, competitor sites in the same
   domain — search 2–3 phrasings of the pattern). Note which sites do it
   memorably and *why*.
3. Distill into **distinct directions, not variations of one idea** — a
   good spread: one literal/functional, one atmospheric/cinematic, one
   playful, one editorial/typographic, one minimal. Each should feel like
   a different designer made it.
4. Build ONE showcase file, `docs/mockups/<feature>/<name>-designs.html`: a
   grid of **mini browser frames** (scaled-down desktop chrome, ~480–640px
   wide), one per variant, all animating live on load, each labeled
   (`D1 · NAME` for animations, `V1` for static components) with a one-line
   concept and a "why it works" note. Vanilla CSS/JS animation (canvas
   allowed). If a variant's mobile treatment is the interesting part, pair
   its mini browser frame with a mini phone frame.
5. **Feasibility card per variant** — how it maps to the site's tooling
   (`{{plain CSS animations / Framer Motion / GSAP / canvas}}`), rough
   effort, and perf/CWV risk (layout shift, main-thread cost, bundle
   weight) on low-end devices and slow connections. The user picks with
   implementation cost in view; never show a variant the stack can't
   honestly ship without saying so.
6. Open it, summarize each variant in one line, recommend one with
   reasoning — but hold it lightly; users often pick one variant and graft
   details from another. Do NOT implement unprompted; after the pick,
   summarize the CSS→`{{styling layer}}` mapping and offer to fold the
   winner into the static or interactive mockup.

Maintain a **learned taste profile** inside this skill (a short section of
the user's confirmed likes/dislikes — sites they admire, patterns they've
rejected, brand colors/fonts, motion preferences). Apply it silently and
append to it as feedback accumulates — this is what makes round two better
than round one.

**`interactive-mockup`** — for when the work is a *flow*, not a page.
Upgrades a static mockup (reusing its palette, copy, and screens verbatim —
an upgrade, not a redesign) into a single self-contained simulator at
`docs/mockups/<feature>/<name>-interactive.html`:

1. Enumerate the **state axes to simulate** — from the user or from the
   feature's real state machine in the code: entry state (guest vs.
   logged-in), outcome branches (form valid/invalid, payment
   success/failure, slot available/booked), async states
   (loading/error/empty), data edge cases (long names, zero results),
   business flags (feature flag on/off), and **viewport (desktop/mobile)
   as a first-class toggle**. Only include axes that change what's on
   screen — three good toggles beat eight cosmetic ones.
2. Two-column layout: a **sticky control panel** (segmented toggles per
   axis, a numbered clickable step list with progress marks, prev/next +
   arrow-key navigation, a hint box suggesting what to try) beside the
   framed page — the browser frame by default, swapping to the phone frame
   when the viewport toggle is on mobile. Every screen/step is an
   absolutely-positioned layer; exactly one active, with a subtle entry
   animation.
3. Keep the JS honest and small — a plain state object + one `render()`.
   **Model the real flow logic**: if the site skips a step for logged-in
   users, the simulator must too. Pull the step order from the feature's
   actual state machine when it exists — the mockup must not invent a flow
   the code contradicts. In-page CTAs also advance the flow.
4. Use real copy, real prices, real data shapes — never lorem ipsum.
5. After approval the interactive mockup **is the implementation spec**:
   each layer maps to a page/step, each toggle axis to real state; keep it
   updated if the design shifts during implementation — it's the artifact
   the team reviews.

**`design-match`** — post-implementation verification: compare the built
page against the approved mockup (or Figma/screenshot reference). Take real
browser screenshots with whatever tooling exists (`{{Playwright screenshot
at 1440 and 390 / Chrome headless --screenshot / an MCP browser tool}}`),
at BOTH breakpoints, and iterate until the implementation matches the
mockup. Never claim "it matches" from code reasoning alone — screenshot it.

### Scaffolding skills (encode the repo's patterns once)

- **`new-component`** — a UI component with doc comments, types (if TS),
  and the styling conventions applied; flags for where it lives (shared ui /
  inside a feature module). For pure HTML/CSS/JS: a documented markup +
  CSS + JS snippet pattern dropped into the right shared location.
- **`new-page`** — the thin routing entry (`{{react-router route / Next
  file route / a new .html file}}`) + the module/page content it delegates
  to + nav wiring + **the page's meta block** (unique title, description,
  canonical, OG tags) + a feature doc stub.
- **`new-api`** — an endpoint call added to `{{API_PATTERN}}` following the
  existing service shape (+ test per the TDD rule). Skip for fully static
  sites.
- **`new-feature`** — full module scaffold per `{{MODULE_PATTERN}}`:
  page/section, components, state/logic module, API calls, types,
  routing + nav entry, feature doc.

Each scaffolding skill should read one existing exemplar file first and
match it — scaffolds must look indistinguishable from hand-written code.

### Review + guard skills

- **`ui-review`** — reviews `git diff` UI files against the Phase 3a rules
  (tokens, breakpoints, semantics, a11y, structure); read-only
  allowed-tools.
- **`browser-check`** — pre-implementation **cross-browser/device
  compatibility analysis**: does the CSS feature/API/behavior differ across
  `{{BROWSER_TARGETS}}` (Safari quirks, iOS Safari viewport units and
  bottom-bar behavior, older-browser support for a CSS feature, touch vs.
  hover, reduced-motion)? Check caniuse-level support before building any
  feature that leans on newer platform capabilities.
- **`seo-a11y-check`** — page-level audit before shipping a new page:
  heading order, landmarks, labels, contrast against the token palette,
  meta/OG completeness, structured data if the domain warrants it
  (e.g. LocalBusiness/MedicalClinic for a clinic site), image alt text.
- **`deploy` skill** — the gated deploy procedure as a skill (verification
  gate → build → deploy via `{{DEPLOY_PIPELINE}}`, plus cache/CDN
  invalidation if applicable). Set `disable-model-invocation: true`
  (user-only trigger) and restrict `allowed-tools` to exactly the
  deploy/verify commands. It must run the verification gate and refuse on
  failure.

---

## Phase 5 — Hooks (`.claude/settings.json` + `.claude/hooks/`)

The enforcement layer. Hooks are POSIX shell regardless of the site's
language — they just call your Phase 0 commands. Template:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path // empty') && [ -n \"$FILE\" ] && [ -f \"$FILE\" ] && {{FORMAT_CMD}} \"$FILE\" 2>/dev/null || true" },
          { "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path // empty') && [ -n \"$FILE\" ] && case \"$FILE\" in {{*.ext1|*.ext2}}) {{LINT_CMD}} \"$FILE\" 2>&1 | tail -5 ;; esac || true" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path // empty') && PROTECTED='{{PROTECTED_FILES space-separated}}' && for p in $PROTECTED; do case \"$FILE\" in *\"$p\") echo \"BLOCKED: $FILE is a protected file. Ask before modifying.\" >&2; exit 2 ;; esac; done" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && {{STATIC_CHECK_CMD}} 2>&1 | head -30", "timeout": 60 },
          { "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && {{TEST_FAST_CMD — fail-fast flag if the runner has one, e.g. vitest run --bail 1}} 2>&1 | tail -30", "timeout": 120 },
          { "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && FILES=$(git diff --name-only -- {{'*.ext' globs}} 2>/dev/null | head -20) && [ -z \"$FILES\" ] && echo 'No modified source files — skipping clean code sweep.' && exit 0 || { echo 'Clean code sweep needed for:'; echo \"$FILES\"; }", "timeout": 10 },
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/suggest-tests.sh\"", "timeout": 15 },
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/suggest-interactive-mockup.sh\"", "timeout": 15 }
        ]
      }
    ],
    "PostCompact": [
      { "hooks": [ { "type": "command",
          "command": "echo '## Reminders: {{your 4-5 most-violated critical rules, pipe-separated — e.g. theme tokens only | sanctioned breakpoints only | semantic HTML + a11y | API through the service layer | mobile-first, test at 390}}'" } ] }
    ],
    "Notification": [
      { "hooks": [ { "type": "command",
          "command": "{{OS notification — macOS: osascript -e 'display notification …'; Linux: notify-send …; else drop this hook}}" } ] }
    ]
  }
}
```

Stack-mapping notes:

- **React/TS stacks:** `{{STATIC_CHECK_CMD}}` is `tsc --noEmit` — the
  strongest Stop signal you have. For **pure HTML/CSS/JS**, it's
  `html-validate` + `stylelint` (+ `eslint` for the JS); each is fast, run
  them all.
- **Slow test suites:** scope the Stop-hook run (changed files, a smoke
  subset, or fail-fast) — the Stop hook needs *fast signal*; CI runs the
  full suite. Never put Playwright E2E in the Stop hook — E2E belongs in CI
  and the deploy gate.
- **Every hook fails open** (`|| true`, guarded exits) except the
  protected-file guard and suggest-tests, which exit 2 deliberately to
  block/inject.
- **PostCompact is the cheapest high-leverage hook**: after context
  compaction, re-inject the rules Claude most often forgets in this repo.
- Keep the clean-code sweep as a <1s git-diff check that *reports* files;
  don't spawn a heavy agent on every Stop.

### `.claude/hooks/suggest-tests.sh` — the test-coverage nudge

Write it for the repo, preserving this contract exactly:

1. Reads hook input; **exits if `stop_hook_active`** (loop safety #1).
2. Diffs working tree vs HEAD + untracked files, source extensions only.
3. **Stamps the changeset hash** under `.git/` so an unchanged tree is only
   flagged once (loop safety #2).
4. Flags logic files (the `{{logic dirs}}` from Phase 0) that have no test
   at the repo's test-convention location (co-located `*.test.*`,
   `__tests__/`, or `tests/` mirror — implement the check that matches);
   skips route/page entries, markup-only and style-only files, type-only
   files, generated code, and tests themselves.
5. If `{{E2E_TOOL}}` exists: maps changed pages/features → E2E spec
   locations (`src/features/booking` → `e2e/booking.spec.ts`, and so on)
   and nudges a review of those specs. Drop this section otherwise.
6. On findings: prints them to stderr and exits 2 with the instruction:
   **"ASK the user whether to add tests (apply the TDD scorecard). Do NOT
   add tests unprompted."** It suggests; it never writes.

### `.claude/hooks/suggest-interactive-mockup.sh` — the mockup-ladder nudge

Companion Stop hook that surfaces the `/interactive-mockup` upgrade at
exactly the right moment — right after the user has seen a static design.
Same loop-safety contract as suggest-tests:

1. Exits if `stop_hook_active`; exits silently outside a git repo.
2. Collects changed + untracked `docs/mockups/**/*.html` files.
3. Keeps only **static** mockups: skips files whose names mark them as
   already-interactive (`*-interactive`) or variant showcases (`*-designs`,
   `*variants*`), and skips any static mockup that already has an
   `<name>-interactive.html` sibling.
4. **Stamps the mockup-set hash** under `.git/` so the same unchanged set
   is only suggested once.
5. On findings: exits 2 with the instruction to **offer** (via a structured
   question, not plain text) upgrading to an interactive mockup — naming
   2–3 concrete state axes worth simulating *for this specific mockup* —
   with a clear decline option. It never builds unprompted; if the user
   declines, stop normally.

---

## Phase 6 — Agents (`.claude/agents/`)

Frontmatter: `description` (this is how they get picked), `model`, `tools`
(read-only unless the agent applies fixes), `color`. The roster:

| Agent | Tools | Prompt core |
| --- | --- | --- |
| `clean-code.md` | Read, Grep, Glob, Edit, Bash | Sweep `git diff --name-only` files for naming, function size, dead code, duplication. Apply safe behavior-preserving fixes; flag risky ones. |
| `code-architect.md` | Read, Grep, Glob (read-only) | Plans features following the project's architecture *exactly* — embed `{{MODULE_PATTERN}}`, `{{API_PATTERN}}`, and `{{ROUTING_PATTERN}}` in the prompt so plans come out native to the repo. |
| `ui-reviewer.md` | Read, Grep, Glob (read-only) | Audits changed files against the design-system rules (mirror Phase 3a; violations are errors), including responsive and a11y checks. |
| `token-auditor.md` | Grep, Glob, Read | Fast whole-codebase scan for hardcoded colors, arbitrary sizes, rogue breakpoints, and design-token violations. Report every instance. |
| `a11y-seo-auditor.md` | Read, Grep, Glob (read-only) | Page-by-page audit: heading order, landmarks, labels, alt text, contrast vs. token palette, meta/OG/canonical completeness, structured data. |
| `perf-auditor.md` | Read, Grep, Glob, Write, Edit, Bash, WebSearch | Deep page-by-page web perf audit with a running audit log; asks before each next page. Goal framed in web terms: Core Web Vitals green (LCP, CLS, INP), minimal JS per page, optimized images/fonts, no layout shift, fast on slow 4G. |
| `tdd.md` | Read, Write, Edit, Grep, Glob, Bash | Standalone red-green-refactor sessions; embeds the same Step-0 scorecard as the rule. |

Embed the site's actual conventions *inside each agent prompt* — agents
don't reliably read rule files on their own.

---

## Phase 7 — CI + Deploy Gate

Wire the same checks into three places so they're impossible to miss:

1. **CI** on every PR/push: lint + tests blocking; Playwright E2E on key
   flows if `{{E2E_TOOL}}` exists. Static check + format-check can start
   **advisory** (reported, non-blocking) if the repo has pre-existing debt —
   say so explicitly in CLAUDE.md, and promote to blocking once clean. Add
   a Lighthouse CI budget check when the site stabilizes (start advisory).
2. **A gated deploy script** (`scripts/deploy.sh` or an npm script): runs
   format-check, lint, static check, tests (+ E2E smoke if fast enough);
   **aborts on blocking failures**; only then builds and deploys via
   `{{DEPLOY_PIPELINE}}`, then invalidates CDN cache if applicable.
   CLAUDE.md must say: never run the raw deploy command directly, never
   skip tests. (If the host auto-deploys on push to main, the gate is
   branch protection + CI — say so, and never push to main with failing
   checks.)
3. **A deploy runbook** (`docs/workflows/deploy.md`): branch → verification
   gate → build → **preview deploy** (Vercel/Netlify preview URL, checked
   at desktop + mobile) → production deploy → post-deploy smoke check
   (key pages load, forms submit, analytics firing), with exact commands.
   Document rollback (host's instant rollback / revert + redeploy) as a
   first-class path with its own steps.

---

## Phase 8 — Verify the Installation

Run through this checklist with the user:

- [ ] Edit a source file → it comes back auto-formatted + linted
      (PostToolUse).
- [ ] Try editing a protected file (`.env`, `vercel.json`/`netlify.toml`)
      → blocked with the ask-first message.
- [ ] Finish a task with a deliberate type/validation error → Stop hook
      surfaces it.
- [ ] Change a logic file without a test → suggest-tests asks (once, and
      not again for the same unchanged tree).
- [ ] Ask for a new page with non-trivial UI → Claude offers
      `/html-mockup`; the mockup renders in the browser frame AND the
      390px phone frame, uses `_theme.css` tokens, and opens in the
      browser.
- [ ] Finish a turn after creating a static mockup → the
      interactive-mockup hook fires and Claude offers the upgrade (with
      concrete state axes), once per unchanged mockup set.
- [ ] Ask for "design options" or a "cooler animation" → Claude runs
      `/design-variants` and delivers one showcase page with 3–5 live
      variants in mini browser frames, each with a feasibility card.
- [ ] Ask for a new util/hook/validation function → Claude scores the TDD
      scorecard out loud and writes the failing test first.
- [ ] `/ui-review` after a UI change → violations reported against the
      rules (tokens, breakpoints, semantics, a11y).
- [ ] `/browser-check` before a feature using a newer platform capability
      → cross-browser support analyzed before code.
- [ ] `/seo-a11y-check` on a new page → meta, headings, and a11y verified
      before it ships.
- [ ] CLAUDE.md tooling tables list every skill/agent/rule/hook you
      created.

---

## Curation Notes (how to adapt, not copy)

- **React + Vite** → the reference target: Vitest + React Testing Library,
  ESLint/Prettier, Playwright, deploy to Vercel/Netlify/Cloudflare Pages.
  Port the templates nearly as-is.
- **Next.js** → same testing stack; `{{ROUTING_PATTERN}}` is file routes;
  SEO lives in the Metadata API; `{{STATIC_CHECK_CMD}}` includes
  `next build` in CI (too slow for the Stop hook — keep `tsc --noEmit`
  there); mind server vs. client component boundaries in the rules.
- **Pure HTML/CSS/JS (no build)** → the system still fully applies:
  tokens are `:root` CSS custom properties in a shared stylesheet;
  `{{STATIC_CHECK_CMD}}` is html-validate + stylelint (+ eslint on JS);
  Prettier still formats everything; tests are Vitest on any JS modules
  with logic + Playwright smoke on the pages; `new-page` scaffolds an
  `.html` file with the shared head/meta block, nav include pattern, and
  linked shared CSS/JS; deploy is the gated upload/push to the static
  host. The mockup workflow is *especially* cheap here — the mockup and
  the implementation are the same medium, so approved mockup markup can
  often be lifted directly into the page.
- **No E2E tool yet** → install the rest of the system first; add
  Playwright later and only then wire the E2E section of suggest-tests,
  CI, and the deploy gate.
- **Content-heavy marketing/clinic site** → the TDD layer is small (forms,
  booking logic, any pricing/date math) and the design-first + SEO/a11y +
  perf layers carry the weight; give `seo-a11y-check` and the perf budget
  extra prominence in CLAUDE.md, and add structured data (e.g.
  LocalBusiness/MedicalClinic, FAQ) to the page scaffold.
- **Keep CLAUDE.md under ~400 lines.** When it grows, push detail into
  `docs/` and leave a Quick Reference row. The index is the product.
- **Evolve by feedback:** when the user corrects the same thing twice, it
  becomes a rule; when a procedure repeats twice, it becomes a skill; when
  a rule keeps getting violated anyway, it becomes a hook.
