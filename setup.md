# Claude Code Agentic Workflow — Replication Guide for Mobile Apps

> **What this is:** A battle-tested Claude Code workflow system, extracted from
> a production mobile app (GrowthX, React Native + Expo) where it made Claude
> design-first, test-first, self-correcting, and self-documenting — without
> the user having to ask each time. It is written to be installed into **any
> mobile app codebase**, whatever the stack: React Native / Expo, Flutter,
> native iOS (Swift/SwiftUI), native Android (Kotlin/Compose), or Kotlin
> Multiplatform.
>
> **Who this is for:** You, Claude, working in a target mobile repo. Your job
> is NOT to copy files verbatim. Your job is to **curate this system for the
> app you are in** — its framework, its build tooling, its test framework, its
> design system, its release pipeline. Every template below uses
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
| **Skills** | `.claude/skills/*/SKILL.md` | Repeatable multi-step procedures invoked as `/name` (scaffold a screen, generate a mockup, gated release). Skills *encode procedures*. | Runbooks |
| **Hooks** | `.claude/settings.json` + `.claude/hooks/*.sh` | Deterministic enforcement the model can't forget: auto-format on save, block protected files, verify on Stop, re-inject rules after compaction. Hooks *enforce mechanically*. | CI on your desk |
| **Agents** | `.claude/agents/*.md` | Delegated audits with focused prompts (clean-code sweep, UI review, perf audit, architecture planning). Agents *review at scale*. | Specialist reviewers |

Two workflow principles sit on top:

1. **Design-first, in an iPhone frame.** Non-trivial UI is never implemented
   cold. First a standalone HTML mockup is generated — the screen rendered
   inside an **iPhone frame** (390px-wide viewport, rounded corners, Dynamic
   Island/notch, home indicator) — styled with a shared `docs/_theme.css`
   that mirrors the app's real design tokens, opened in the browser, and
   approved by the user. Only then is it implemented, and optionally verified
   against the mockup with a `design-match` skill. This kills the "build it,
   hate it, rebuild it" loop. Every mockup is offered in the iPhone frame —
   one consistent, phone-real preview medium regardless of stack.

   The mockup layer is a **ladder**, climbed only as far as the work needs:

   - **Static mockup** (`/html-mockup`) — one screen, its states side by side.
   - **Design variants** (`/design-variants`) — when the *direction* is
     undecided: research inspiration (Mobbin + web), then one showcase page
     with 3–5 live, labeled variants in mini phone frames; the user picks
     before any app code.
   - **Interactive mockup** (`/interactive-mockup`) — when the work is a
     *flow*: a clickable simulator with a control panel (step navigation +
     state/edge-case toggles) inside the iPhone frame; becomes the
     implementation spec.
   - **Design match** (`/design-match`) — after implementation, screenshots
     vs. the approved mockup until they agree.

2. **Test-gated, judgment-based TDD.** A scorecard (not a blanket mandate)
   decides when TDD applies: 2+ signals (has logic / has a contract / can
   break silently / touches shared code / non-trivial) → write failing tests
   first. Pure style/layout/copy changes skip it. Tests gate every release; a
   Stop hook nudges (never writes) missing tests.

The meta-rule: **the user should never have to ask for any of this.** Rules
auto-apply, hooks auto-run, skills auto-trigger from their descriptions.

---

## Phase 0 — Discover the App (do this before writing anything)

Explore the repo and fill in this table. Then **show it to the user and
confirm** before generating files — wrong assumptions here poison everything
downstream.

| Variable | How to find it | Examples across mobile stacks |
| --- | --- | --- |
| `{{APP_NAME}}` | manifest, README | GrowthX |
| `{{STACK}}` | deps, project files | React Native + Expo / Flutter / SwiftUI + Xcode / Compose + Gradle / KMP |
| `{{SRC_EXTS}}` | file census | `.ts .tsx` / `.dart` / `.swift` / `.kt` |
| `{{RUN_CMD}}` | scripts, README | `pnpm ios` / `flutter run` / `xcodebuild` + simulator / `./gradlew installDebug` |
| `{{TEST_CMD}}` + `{{TEST_ONE_CMD}}` | scripts, CI config | `pnpm test` + `pnpm test -- <file>` / `flutter test` + `flutter test <file>` / `xcodebuild test` / `./gradlew test` |
| `{{STATIC_CHECK_CMD}}` | typechecker or compiler | `tsc --noEmit` / `dart analyze` / `xcodebuild build` / `./gradlew compileDebugKotlin` |
| `{{LINT_CMD}}` / `{{FORMAT_CMD}}` | configs | `eslint --fix` + `prettier --write` / `dart fix` + `dart format` / SwiftLint + swift-format / ktlint + ktfmt |
| `{{DESIGN_TOKENS}}` | theme files, tailwind config, design-system package | NativeWind theme vars + 390px scale baseline / ThemeData + ColorScheme / Asset catalog colors + type styles / MaterialTheme tokens |
| `{{STYLE_RULES}}` | what's forbidden vs. allowed in UI code | no hardcoded colors, no arbitrary px, sanctioned scaling helper for icon sizes |
| `{{MODULE_PATTERN}}` | src layout | `modules/<feature>/{screen,components,hooks,context,api}` + shared `lib/` / `lib/features/<feature>/` / feature folders or Swift packages / Gradle feature modules |
| `{{NAV_PATTERN}}` | how screens are wired | expo-router file routes / go_router / NavigationStack / NavHost |
| `{{API_PATTERN}}` | how HTTP is done | class services in `lib/api/` with a shared client singleton / repository + dio / URLSession service layer / Retrofit + repository |
| `{{E2E_TOOL}}` | deps, config dirs | Maestro (`.maestro/flows/`) / Detox / XCUITest / Espresso / **none** |
| `{{PROTECTED_FILES}}` | secrets and build/signing configs that must not be silently edited | `.env`, `eas.json`, `app.config.ts`, `google-services.json`, `GoogleService-Info.plist`, keystores, provisioning profiles, `Info.plist` release keys |
| `{{RELEASE_PIPELINE}}` | scripts, CI, fastlane | gated `pnpm deploy` → EAS build/submit / fastlane lanes / Xcode Cloud / Play Console via Gradle + fastlane; OTA channel if any (EAS Update / CodePush / Shorebird) |
| `{{CI}}` | `.github/workflows/`, Bitrise, etc. | lint + tests on PR / none yet |

Also note:

- **Test file convention** — co-located `__tests__/` dirs? `test/` mirror tree
  (Flutter)? Separate test target (XCTest)? `src/test/` source set (Android)?
  Match what exists; if nothing exists, propose the stack's idiomatic default.
- **Which directories carry logic** (hooks/services/api/context, blocs/
  providers/usecases, view models/repositories) vs. which are thin (route
  files, pure presentational views) — this drives the TDD scorecard and the
  suggest-tests hook.
- **Existing docs** — fold them into the docs structure rather than duplicate.
- **Naming and structure conventions** already in force (casing, prefixes,
  import ordering, error-handling idiom) — these become Critical Rules.

Two special cases to raise with the user before proceeding:

- **No design system** (ad-hoc colors and sizes scattered through views): the
  design-first workflow needs tokens to be meaningful. Offer to extract a
  token set (colors, type scale, spacing, radii) from the existing UI first —
  it becomes both the app's theme layer and the mockup theme in Phase 1.
- **No test framework wired up**: setting one up is a prerequisite, not part
  of this system. Propose the stack default (Jest + RNTL / flutter_test /
  XCTest / JUnit + Turbine), seed tests for the most load-bearing logic, then
  install the TDD rule so coverage grows with every change.

---

## Phase 1 — Docs System + Shared Mockup Theme

### 1a. Directory structure

```text
docs/
  _theme.css          # Shared dark/light CSS for ALL HTML docs and mockups (see 1b)
  _theme.md           # Cheat-sheet: skeleton, component classes, tokens
  architecture.md     # Tech stack, directory map, data flow, patterns
  documentation-guide.md  # How the doc system works + feature-doc template
  SETUP.md            # Dev environment setup (prerequisites, simulators, build)
  workflows/          # Step-by-step how-tos (add a screen, add an API, release runbook)
  guides/             # Reference guides (scaling, signing, store submission, OTA, perf)
  features/           # One .md per shipped feature (template below)
  mockups/<feature>/  # iPhone-framed HTML design mockups, organized by feature
  research/           # Audits, deep dives, proposals
  implementation/     # Plans for upcoming work
  testing/            # Testing strategy
  metrics/            # Dashboards, audit logs
```

### 1b. `docs/_theme.css` — the keystone of design-first

One shared stylesheet (~900 lines in the source repo) used by every HTML doc
and every mockup. Build its tokens from the **app's real design system** so
mockups genuinely preview the app:

- **Tokens as CSS vars:** surfaces (`--bg`, `--card`, `--glass`), text
  (`--fg`, `--muted`), borders (`--hairline`), the app's accent palette, a
  3-stop gradient, the full type scale (`--t-xs`…`--t-6xl`), the app's actual
  font stacks, radii, shadows. Dark default + light counterparts that swap via
  `prefers-color-scheme` plus a manual `data-theme` override with a
  `.theme-toggle` button. Pull the values from `{{DESIGN_TOKENS}}` — the
  tailwind config, `ThemeData`, asset catalogs, or `MaterialTheme` — so a hex
  in the mockup is the same hex the app ships.
- **Doc components:** page `.wrap`, `.hero` (title + meta sidebar),
  `.eyebrow`, numbered `.sec-head`, `.grid.cols-2|3|4`, `.card`,
  `.pill.ok|warn|info`, `.table-wrap`, `.flow .step` (auto-arrow flow
  diagrams), `.phases` (roadmap columns), `.callout.warn|danger|ok`,
  `.compare .col`, syntax-highlighted `pre`.
- **The iPhone frame component** — used by every mockup (see Phase 4):
  a `.phone-frame` at **390px width** (iPhone design baseline), tall rounded
  corners (~40px), hairline border, `overflow: hidden`, app `--bg` background,
  a Dynamic Island/notch element at the top, and a home-indicator bar at the
  bottom. Define it once in the theme so every mockup gets the identical
  frame. If the app's own design baseline differs (e.g. 375px), match the
  app's baseline — but it is always an iPhone frame.
- Write `docs/_theme.md` alongside: HTML skeleton + a component/token
  cheat-sheet table, so future sessions compose docs without re-reading the
  CSS.

**The one rule with an exception:** explainer/research/plan HTML docs MUST use
the theme components (no one-off `<style>` blocks). UI **mockups** link the
theme for tokens and the phone frame but get custom CSS inside the screen —
pixel-level mockup control needs it.

### 1c. Feature-doc template (in `documentation-guide.md`)

Every shipped feature gets `docs/features/<kebab-name>.md` with
grep-searchable tags at the top:

```markdown
# Feature: <Name>

@features <feature-name>
@modules <module1> <module2>
@services <service1>

## Summary
One paragraph: what and why.

## Code Index
Tables of Screens / Components / State (hooks, blocs, view models) /
API methods → file → purpose.

## Data Flow, Decisions, Changelog
```

The `@tags` make `grep -r "@modules chat" docs/` a working feature-lookup
tool.

---

## Phase 2 — CLAUDE.md (the entry point)

Structure it exactly like this (compactness is the feature — link, don't
inline):

1. **One-paragraph app summary** — stack, styling approach, architecture in 3
   lines.
2. **Documentation index** — table linking every doc, plus the `docs/` tree.
3. **Project structure tables** — screens/routes, feature modules, shared
   libraries, shared components, types/models. One line per entry: path →
   purpose.
4. **Critical Rules** — numbered, ~15 max, each one sentence of rule + one of
   detail. The universal mobile shapes, instantiated with Phase 0 findings:
   - API calls only through `{{API_PATTERN}}` — never raw HTTP from
     views/components.
   - Import/dependency discipline: `{{path aliases, no deep relative paths,
     no barrel imports — whatever the repo enforces}}`.
   - Naming conventions: `{{casing, prefixes, file naming}}`.
   - **Design tokens only** — no hardcoded colors, no arbitrary pixel values;
     `{{STYLE_RULES}}`, including the sanctioned scaling helper if one exists.
   - Structure conventions: `{{component/view function style, internals
     ordering, ~100-line limit before extracting logic to a hook/view model}}`.
   - Route/screen entries are thin delegates; business logic lives in
     `{{MODULE_PATTERN}}` modules, which are self-contained.
   - **Safe-area discipline** — always use the platform's safe-area API;
     never hardcode notch/status-bar/home-indicator padding.
   - Error-handling idiom: `{{what surfaces to users (alerts/snackbars),
     what goes to logging/crash reporting}}`.
   - State/persistence idioms: `{{context/provider pattern, singleton DB
     pattern, socket connection discipline — as applicable}}`.
   - **Tests gate every release — never ship from an unverified tree.** Name
     the blocking checks vs. advisory ones explicitly, and the exact commands
     that must not be bypassed (no raw store builds, no skipped test flags).
5. **Observability table** — crash reporting, analytics, session replay, push:
   where each lives and the one-liner to use it.
6. **Claude Code Tooling section** — tables of the skills, agents, rules, and
   hooks you're about to create in Phases 3–6, so every future session knows
   they exist. *This is what makes the system self-triggering.*
7. **Quick Reference table** — "How do I X?" → link/answer. 15–20 rows
   (add a screen, add an endpoint, run one test, cut a release, push an OTA,
   where auth state lives, how deep links work…).
8. **Commands block** — dev, quality, test, release commands with warnings
   inline (e.g. "ALWAYS use the gated release script, never raw
   `eas build`/`fastlane`/`gradlew bundleRelease` to bypass the gate").

---

## Phase 3 — Rules (`.claude/rules/`)

Three files. Adapt the specifics; keep the shapes.

### 3a. `ui-components.md` — design-system enforcement (auto-applies to UI files)

State ONLY-allowed and NEVER-allowed patterns explicitly — **allowlists beat
prose**. Instantiate with `{{DESIGN_TOKENS}}` and `{{STYLE_RULES}}`:

```markdown
# UI Component Rules (Auto-Applied to {{UI_FILE_GLOB}})

## Color Usage
ONLY these patterns: {{exhaustive list of allowed token references —
theme classes / Theme.of(context) / semantic asset-catalog colors /
MaterialTheme.colorScheme}}
NEVER: {{raw palette values — hex literals, Color(0xFF…), .white/.black,
default framework palette classes}}

## Spacing / Typography
ONLY the design scale: {{spacing scale, type scale}}.
NEVER arbitrary values ({{p-[12px] / EdgeInsets(13.5) / .padding(17) —
whatever the local sin looks like}}).
Exception: {{the sanctioned escape hatch, e.g. a scaling helper for icons}}.

## Component Structure
- {{function/view declaration convention}}
- Internals order: {{e.g. framework hooks → app state → local state →
  handlers → effects → early returns → render}}
- ~100-line max for view logic before extracting a {{hook / bloc / view model}}
- Pre-compute filtering/mapping BEFORE the render/return
- {{conditional-style utility}}, {{platform-branching idiom}}
- Safe-area API always; never hardcoded notch/status-bar padding
```

### 3b. `tdd-workflow.md` — the TDD decision scorecard + cycle

The most important rule file. Keep its structure intact; only the examples and
commands change per stack:

```markdown
# TDD-First Workflow (Smart Auto-Enforced)

Before writing ANY code, evaluate whether the task needs TDD. This decision
must be automatic — the user should never have to tell you.

## Step 0: DECIDE — score these 5 signals; 2+ YES → TDD, else skip:
| Signal | YES | NO |
| Has logic? | conditionals, state transitions, transforms | pure layout, static content |
| Has a contract? | input→output, state holder returns, API shape | renames, import moves |
| Can break silently? | edge cases, async, error paths | color change, copy text |
| Touches shared code? | shared libs/services/state holders | one-off leaf view |
| Is non-trivial? | new function/service/state holder, >10 lines logic | one-liner, prop passthrough |

Explicit APPLIES list ({{this repo's logic dirs — hooks/services/api/context,
blocs/usecases, view models/repositories}}) and DOES-NOT-APPLY list ({{thin
route/screen entries, style-only, type/model-only, config, generated code}}).
When skipping, say: "No TDD — [reason]".

## The Cycle: ANALYZE (identify testable units + test file locations) →
RED (write failing tests FIRST, run {{TEST_ONE_CMD}}, confirm failure) →
GREEN (minimal implementation, all pass; fix the code, not the tests) →
REFACTOR (stay green, no new behavior).

## Test Patterns — concrete copy-paste snippets for THIS stack:
how to mock the {{API_PATTERN}} layer, how to test {{hooks / blocs /
view models}}, how to test components/widgets/views with the stack's testing
library, how to test error paths.
## Running Tests — one file, watch (if available), all — with real commands.
```

Write the test-pattern snippets against the app's *actual* test stack — this
section is what makes future sessions productive instead of flailing at mock
setup.

### 3c. `offer-mockup.md` — the design-first trigger

```markdown
# Offer an iPhone Mockup Before UI Implementation

When asked to build, redesign, or significantly change a screen or component
with non-trivial UI (not a one-liner tweak), ask before coding:
"Want me to generate an HTML mockup first so you can see the design in an
iPhone frame before I implement it? (/html-mockup)"

Offer when: new screen or major component; layout overhaul; UI described
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
auto-triggers it from context; `argument-hint`; `allowed-tools` scoped tight)
plus a stepwise body.

### The design-first suite (highest value — build these first)

Four skills, matching the mockup ladder from the Philosophy section. All of
them present designs inside the iPhone frame.

**`html-mockup`** — the heart of the workflow:

1. Parse arguments; if unclear ask what screen, any reference (Figma /
   screenshot / verbal), and which states to show (empty, loading, error,
   populated).
2. Read `docs/_theme.css` for available tokens and the `.phone-frame`
   component.
3. Generate `docs/mockups/<feature>/<name>.html`:
   - The screen built **inside the iPhone frame** — 390px-wide phone body,
     rounded corners, Dynamic Island/notch, home indicator, status-bar
     spacing, and the app's bottom tab bar if the screen has one.
   - Theme tokens only outside the `<style>` block — the mockup must use the
     same colors/type/spacing the app ships.
   - Realistic content, never lorem ipsum — use domain-plausible data.
   - Multiple states shown as **multiple iPhone frames side by side** (or
     tabs), one frame per state.
   - An annotations section below the frames: design notes on spacing,
     interactions, animations, and edge cases.
4. `open` the file in the browser; ask for changes and iterate in the mockup
   (cheap) rather than in app code (expensive).
5. **After approval, summarize the implementation plan but do NOT implement
   until told** — including how theme vars map to the app's styling layer
   (`{{theme classes / ThemeData fields / asset-catalog names /
   MaterialTheme tokens}}`).

**`design-variants`** — for when the *direction* is undecided (user wants
options, asks for something "cooler", or rejects a first design):

1. Parse the brief: what surface, what it must communicate, any references.
   If a static mockup exists for the surrounding flow, read it — variants
   must fit its palette and tone.
2. **Research in parallel**: a design-inspiration source (Mobbin MCP if
   connected — `search_screens`/`search_flows` with 2–3 phrasings of the
   pattern — else WebSearch for named patterns). Note which apps do it
   memorably and *why*.
3. Distill into **distinct directions, not variations of one idea** — a good
   spread: one literal/functional, one atmospheric/cinematic, one playful,
   one data/technical, one minimal. Each should feel like a different
   designer made it.
4. Build ONE showcase file, `docs/mockups/<feature>/<name>-designs.html`: a
   grid of **mini iPhone frames** (~300–390px, notch, home indicator), one
   per variant, all animating live on load, each labeled (`D1 · NAME` for
   animations, `V1` for static components) with a one-line concept and a
   "why it works" note. Vanilla CSS/JS animation (canvas allowed).
5. **Feasibility card per variant** — how it maps to the app's animation
   tooling (`{{Reanimated/Skia/Lottie | Flutter animations/Rive | Core
   Animation/SwiftUI | Compose animation/Lottie}}`), rough effort, and perf
   risk on low-end devices. The user picks with implementation cost in view;
   never show a variant the stack can't honestly ship without saying so.
6. Open it, summarize each variant in one line, recommend one with reasoning
   — but hold it lightly; users often pick one variant and graft details
   from another. Do NOT implement unprompted; after the pick, summarize the
   CSS→`{{styling layer}}` mapping and offer to fold the winner into the
   static or interactive mockup.

Maintain a **learned taste profile** inside this skill (a short section of
the user's confirmed likes/dislikes — apps they admire, patterns they've
rejected, brand colors/fonts, motion preferences). Apply it silently and
append to it as feedback accumulates — this is what makes round two better
than round one.

**`interactive-mockup`** — for when the work is a *flow*, not a screen.
Upgrades a static mockup (reusing its palette, copy, and screens verbatim —
an upgrade, not a redesign) into a single self-contained simulator at
`docs/mockups/<feature>/<name>-interactive.html`:

1. Enumerate the **state axes to simulate** — from the user or from the
   feature's real state machine in the code: entry state (guest vs.
   logged-in), outcome branches (approved/rejected, success/failure), async
   states (loading/error/empty), data edge cases (long names, zero items),
   business flags (paid/unpaid, feature flag on/off). Only include axes that
   change what's on screen — three good toggles beat eight cosmetic ones.
2. Two-column layout: a **sticky control panel** (segmented toggles per axis,
   a numbered clickable step list with progress marks, prev/next + arrow-key
   navigation, a hint box suggesting what to try) beside a **realistic
   iPhone** (rounded body, side buttons, notch, status bar, home indicator).
   Every screen is an absolutely-positioned layer; exactly one active, with a
   subtle entry animation.
3. Keep the JS honest and small — a plain state object + one `render()`.
   **Model the real flow logic**: if the app skips a step for logged-in
   users, the simulator must too. Pull the step order from the feature's
   actual state machine when it exists — the mockup must not invent a flow
   the code contradicts. In-phone CTAs also advance the flow.
4. Use real copy, real prices, real data shapes — never lorem ipsum.
5. After approval the interactive mockup **is the implementation spec**: each
   layer maps to a screen, each toggle axis to real state; keep it updated if
   the design shifts during implementation — it's the artifact the team
   reviews.

**`design-match`** — post-implementation verification: compare the built
screen against the approved mockup (or Figma/screenshot reference). Take
simulator/emulator screenshots with whatever tooling exists
(`{{Maestro MCP / xcrun simctl io screenshot / adb exec-out screencap /
flutter screenshot}}`) and iterate until the implementation matches the
mockup. Never claim "it matches" from code reasoning alone — screenshot it.

### Scaffolding skills (encode the repo's patterns once)

- **`new-component`** — a UI component with doc comments, types/models, and
  the styling conventions applied; flags for where it lives (shared ui /
  common / inside a feature module).
- **`new-screen`** — the thin navigation entry (`{{route file / go_router
  entry / NavigationStack destination / NavHost composable}}`) + the module
  screen it delegates to + wiring; flags for tab/stack/modal presentation.
- **`new-api`** — an endpoint added to `{{API_PATTERN}}` following the
  existing service/repository shape (+ test per the TDD rule).
- **`new-feature`** — full module scaffold per `{{MODULE_PATTERN}}`: screen,
  components, state holder, API, types, navigation entry, feature doc.

Each scaffolding skill should read one existing exemplar file first and match
it — scaffolds must look indistinguishable from hand-written code.

### Review + guard skills

- **`ui-review`** — reviews `git diff` UI files against the Phase 3a rules
  (tokens, scaling, safe areas, structure); read-only allowed-tools.
- **`platform-check`** — pre-implementation **iOS/Android compatibility
  analysis**: does the API/library/behavior differ per platform (permissions,
  keyboard, back gesture, notch/cutout, haptics, background limits)? Run it
  before building any feature that touches platform capabilities. (For an
  iOS-only or Android-only app, scope it to OS-version compatibility instead.)
- **`ota` / release skill** — the gated release procedure as a skill
  (store build via `{{RELEASE_PIPELINE}}`, and the OTA path if one exists —
  EAS Update / CodePush / Shorebird). Set `disable-model-invocation: true`
  (user-only trigger) and restrict `allowed-tools` to exactly the
  release/verify commands. It must run the verification gate and refuse on
  failure.

---

## Phase 5 — Hooks (`.claude/settings.json` + `.claude/hooks/`)

The enforcement layer. Hooks are POSIX shell regardless of the app's language
— they just call your Phase 0 commands. Template:

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
          { "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && {{TEST_FAST_CMD — fail-fast flag if the runner has one}} 2>&1 | tail -30", "timeout": 120 },
          { "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && FILES=$(git diff --name-only -- {{'*.ext' globs}} 2>/dev/null | head -20) && [ -z \"$FILES\" ] && echo 'No modified source files — skipping clean code sweep.' && exit 0 || { echo 'Clean code sweep needed for:'; echo \"$FILES\"; }", "timeout": 10 },
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/suggest-tests.sh\"", "timeout": 15 },
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/suggest-interactive-mockup.sh\"", "timeout": 15 }
        ]
      }
    ],
    "PostCompact": [
      { "hooks": [ { "type": "command",
          "command": "echo '## Reminders: {{your 4-5 most-violated critical rules, pipe-separated — e.g. theme tokens only | no arbitrary px | API through the service layer | safe areas always}}'" } ] }
    ],
    "Notification": [
      { "hooks": [ { "type": "command",
          "command": "{{OS notification — macOS: osascript -e 'display notification …'; Linux: notify-send …; else drop this hook}}" } ] }
    ]
  }
}
```

Stack-mapping notes:

- **Native stacks:** `{{STATIC_CHECK_CMD}}` is the compiler — an incremental
  `xcodebuild build` or `./gradlew compileDebugKotlin` is the strongest Stop
  signal you have, but it can be slow; if it exceeds the timeout budget, keep
  only lint + tests in the Stop hook and let CI compile.
- **Slow test suites:** scope the Stop-hook run (changed targets, a smoke
  subset, or fail-fast) — the Stop hook needs *fast signal*; CI runs the full
  suite.
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
4. Flags logic files (the `{{logic dirs}}` from Phase 0) that have no test at
   the repo's test-convention location (co-located `__tests__/`, `test/`
   mirror tree, test target, or `src/test/` source set — implement the check
   that matches); skips navigation entries, type/model-only files, generated
   code, and tests themselves.
5. If `{{E2E_TOOL}}` exists: maps changed feature dirs → E2E flow/spec
   locations (`modules/events/home` → `.maestro/flows/home`, and so on) and
   nudges a review of those flows. Drop this section otherwise.
6. On findings: prints them to stderr and exits 2 with the instruction:
   **"ASK the user whether to add tests (apply the TDD scorecard). Do NOT add
   tests unprompted."** It suggests; it never writes.

### `.claude/hooks/suggest-interactive-mockup.sh` — the mockup-ladder nudge

Companion Stop hook that surfaces the `/interactive-mockup` upgrade at exactly
the right moment — right after the user has seen a static design. Same
loop-safety contract as suggest-tests:

1. Exits if `stop_hook_active`; exits silently outside a git repo.
2. Collects changed + untracked `docs/mockups/**/*.html` files.
3. Keeps only **static** mockups: skips files whose names mark them as
   already-interactive (`*-interactive`) or variant showcases (`*-designs`,
   `*variants*`), and skips any static mockup that already has an
   `<name>-interactive.html` sibling.
4. **Stamps the mockup-set hash** under `.git/` so the same unchanged set is
   only suggested once.
5. On findings: exits 2 with the instruction to **offer** (via a structured
   question, not plain text) upgrading to an interactive mockup — naming 2–3
   concrete state axes worth simulating *for this specific mockup* — with a
   clear decline option. It never builds unprompted; if the user declines,
   stop normally.

---

## Phase 6 — Agents (`.claude/agents/`)

Frontmatter: `description` (this is how they get picked), `model`, `tools`
(read-only unless the agent applies fixes), `color`. The roster:

| Agent | Tools | Prompt core |
| --- | --- | --- |
| `clean-code.md` | Read, Grep, Glob, Edit, Bash | Sweep `git diff --name-only` files for naming, function size, dead code, duplication. Apply safe behavior-preserving fixes; flag risky ones. |
| `code-architect.md` | Read, Grep, Glob (read-only) | Plans features following the project's architecture *exactly* — embed `{{MODULE_PATTERN}}`, `{{API_PATTERN}}`, and `{{NAV_PATTERN}}` in the prompt so plans come out native to the repo. |
| `ui-reviewer.md` | Read, Grep, Glob (read-only) | Audits changed files against the design-system rules (mirror Phase 3a; violations are errors). |
| `token-auditor.md` | Grep, Glob, Read | Fast whole-codebase scan for hardcoded colors, arbitrary sizes, and design-token violations. Report every instance. |
| `perf-auditor.md` | Read, Grep, Glob, Write, Edit, Bash, WebSearch | Deep file-by-file mobile perf audit with a running audit log; asks before each next file. Goal framed in mobile terms: 60fps scrolling, instant transitions, zero jank, fast cold start, minimal memory. |
| `tdd.md` | Read, Write, Edit, Grep, Glob, Bash | Standalone red-green-refactor sessions; embeds the same Step-0 scorecard as the rule. |

Embed the app's actual conventions *inside each agent prompt* — agents don't
reliably read rule files on their own.

---

## Phase 7 — CI + Release Gate

Wire the same checks into three places so they're impossible to miss:

1. **CI** on every PR/push: lint + tests blocking. Static check + format-check
   can start **advisory** (reported, non-blocking) if the repo has
   pre-existing debt — say so explicitly in CLAUDE.md, and promote to blocking
   once clean.
2. **A gated release script** (`scripts/deploy.sh`, a fastlane lane, or a
   Gradle task): runs format-check, lint, static check, tests; **aborts on
   blocking failures**; only then builds/submits via `{{RELEASE_PIPELINE}}`.
   CLAUDE.md must say: never run the raw build/submit command directly, never
   skip tests.
3. **A release runbook** (`docs/workflows/release.md`): version branch →
   version/build-number bump → unit gate → E2E gate (`{{E2E_TOOL}}` on
   simulators/devices, both platforms if cross-platform) → build → store
   submit, with exact commands. If the app has an OTA channel (EAS Update /
   CodePush / Shorebird), document it as a **separate path** with its own
   guardrails: verify the tree with tests first, mind the runtime-version
   compatibility, publish per-platform.

---

## Phase 8 — Verify the Installation

Run through this checklist with the user:

- [ ] Edit a source file → it comes back auto-formatted + linted
      (PostToolUse).
- [ ] Try editing a protected file (a keystore config, `.env`) → blocked with
      the ask-first message.
- [ ] Finish a task with a deliberate compile/type error → Stop hook surfaces
      it.
- [ ] Change a logic file without a test → suggest-tests asks (once, and not
      again for the same unchanged tree).
- [ ] Ask for a new screen with non-trivial UI → Claude offers
      `/html-mockup`; the mockup renders in the iPhone frame, uses
      `_theme.css` tokens, and opens in the browser.
- [ ] Finish a turn after creating a static mockup → the interactive-mockup
      hook fires and Claude offers the upgrade (with concrete state axes),
      once per unchanged mockup set.
- [ ] Ask for "design options" or a "cooler animation" → Claude runs
      `/design-variants` and delivers one showcase page with 3–5 live
      variants in mini iPhone frames, each with a feasibility card.
- [ ] Ask for a new util/state-holder function → Claude scores the TDD
      scorecard out loud and writes the failing test first.
- [ ] `/ui-review` after a UI change → violations reported against the rules.
- [ ] `/platform-check` before a platform-touching feature → iOS/Android
      differences analyzed before code.
- [ ] CLAUDE.md tooling tables list every skill/agent/rule/hook you created.

---

## Curation Notes (how to adapt, not copy)

- **React Native / Expo** → closest to the source repo: port the templates
  nearly as-is (Jest + RNTL, ESLint/Prettier, Maestro, EAS + OTA).
- **Flutter** → tests via `flutter_test`/`bloc_test`, format/lint via
  `dart format`/`dart analyze`, tokens from `ThemeData`, E2E via
  `integration_test` or Maestro; mockups stay iPhone-framed HTML.
- **Native iOS** → XCTest, SwiftLint/swift-format, tokens from asset catalogs
  + type styles, XCUITest or Maestro; platform-check becomes OS-version and
  device-class checks.
- **Native Android** → JUnit/Turbine/Compose testing, ktlint/detekt, tokens
  from `MaterialTheme`, Espresso or Maestro; mockups are **still shown in the
  iPhone frame** — it's the preview medium, not a platform statement; add an
  Android-specific annotation note (back gesture, cutout) below the frames
  when relevant.
- **Single-platform app** → keep `platform-check` but scope it to OS-version /
  device-matrix compatibility instead of iOS-vs-Android.
- **No E2E tool yet** → install the rest of the system first; add Maestro
  later (it works across RN, Flutter, and native) and only then wire the E2E
  section of suggest-tests and the release gate.
- **Keep CLAUDE.md under ~400 lines.** When it grows, push detail into
  `docs/` and leave a Quick Reference row. The index is the product.
- **Evolve by feedback:** when the user corrects the same thing twice, it
  becomes a rule; when a procedure repeats twice, it becomes a skill; when a
  rule keeps getting violated anyway, it becomes a hook.
