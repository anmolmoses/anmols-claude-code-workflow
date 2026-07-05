 # Anmol's Claude Code Workflow for Mobile Apps

  A production-tested agentic workflow that makes Claude Code **design-first, test-first, self-correcting, and self-documenting** in
  any mobile codebase — React Native / Expo, Flutter, native iOS (SwiftUI), native Android (Compose), or KMP.

  ## The problem this solves

  Claude Code out of the box is a great pair programmer with amnesia. Every session it forgets your design system, skips tests unless
  asked, builds UI you have to see in the app before you can reject it, and re-learns your architecture from scratch. You end up
  being the enforcement layer.

  This workflow moves enforcement into the repo itself, so **the user never has to ask**:

  - **Design-first, in an iPhone frame.** Non-trivial UI starts as an HTML mockup rendered inside a realistic iPhone frame, using a
  shared `_theme.css` built from your app's *real* design tokens. You approve the design in a browser in minutes, not in a rebuilt
  app in hours.
  - **Judgment-based TDD.** A 5-signal scorecard decides automatically when tests come first (logic, contracts, silent-breakage risk,
  shared code, non-triviality). Style tweaks skip it; business logic never does.
  - **Hooks that never forget.** Auto-format on save, protected-file guards, typecheck + tests on every task completion, missing-test
  nudges, and rule re-injection after context compaction.

  ## The five layers

  | Layer | Lives in | Job |
  | --- | --- | --- |
  | **CLAUDE.md** | repo root | Compact index: rules, structure, quick reference. Links out, never duplicates. |
  | **Rules** | `.claude/rules/` | Judgment guidance auto-applied by file path — design-system allowlists, the TDD scorecard, "offer
  a mockup first" |
  | **Skills** | `.claude/skills/` | Repeatable procedures as `/commands` — mockups, scaffolding, reviews, gated releases |
  | **Hooks** | `.claude/settings.json` | Deterministic enforcement the model can't forget — format, guard, verify, remind |
  | **Agents** | `.claude/agents/` | Delegated auditors — clean-code sweep, UI review, perf audit, architecture planning |

  ## The mockup ladder

  Every design decision climbs only as far as it needs, always in an iPhone frame:

  1. **`/html-mockup`** — one screen, its states side by side, in the browser
  2. **`/design-variants`** — direction undecided? Research (Mobbin + web) → one showcase page with 3–5 live animated variants → you
  pick before any app code
  3. **`/interactive-mockup`** — building a flow? A clickable simulator with step navigation and state toggles (auth, outcomes,
  loading/error/empty) — this becomes the implementation spec
  4. **`/design-match`** — after implementation, simulator screenshots vs. the approved mockup until they agree

  A Stop hook even notices when a static mockup was just created and offers the interactive upgrade at exactly the right moment.

  ## How to use it

  1. Copy [`setup.md`](setup.md) into the root of your mobile app repo.
  2. Open Claude Code there and say:

     > Read setup.md and install this workflow, curated for this app. Start with Phase 0 and confirm your findings with me before
  writing any files.

  3. Claude discovers your stack (test runner, design tokens, lint/format commands, release pipeline), confirms the findings with
  you, then builds each layer adapted to your app — the templates use `{{PLACEHOLDERS}}`, not copy-paste.
  4. Run the Phase 8 verification checklist at the end to prove every hook, rule, and skill actually fires.

  One focused session, start to finish.

  ## What gets installed

  ```
  your-app/
  ├── CLAUDE.md                    # Entry-point index Claude reads every session
  ├── .claude/
  │   ├── settings.json            # Hooks: format-on-save, protected files, verify-on-stop
  │   ├── rules/                   # Design-system allowlist, TDD scorecard, mockup-first
  │   ├── skills/                  # /html-mockup, /design-variants, /interactive-mockup,
  │   │                            # /design-match, /new-screen, /new-feature, /ui-review, …
  │   ├── agents/                  # clean-code, code-architect, ui-reviewer, perf-auditor, …
  │   └── hooks/                   # suggest-tests.sh, suggest-interactive-mockup.sh
  └── docs/
      ├── _theme.css               # Your design tokens as CSS — powers every mockup
      ├── mockups/<feature>/       # iPhone-framed HTML mockups, by feature
      ├── features/                # One doc per shipped feature, grep-taggable
      └── workflows/               # Release runbook, how-tos
  ```

  ## FAQ

  **Does this only work for React Native?**
  No — `setup.md` is stack-agnostic across mobile. Phase 0 maps every command (test, lint, format, static check, release) to your
  stack's equivalents, and the Curation Notes cover RN, Flutter, native iOS, native Android, and KMP specifically.

  **My app has no tests at all.**
  Setting up the runner is a stated prerequisite — the guide tells Claude to propose the stack's idiomatic default, seed tests for
  the most load-bearing logic, then install the TDD rule so coverage grows with every change instead of demanding a big-bang
  backfill.

  **Won't the Stop hooks slow every task down?**
  They're budgeted: fail-fast test runs with trimmed output, a <1s git-diff clean-code check, and guidance to drop slow compilers
  from the hook and let CI carry the full suite. Everything fails open except the protected-file guard.

  **Why are mockups HTML and not Figma?**
  HTML is what Claude can generate, animate, and iterate on in seconds, with zero tooling, using your exact tokens. Figma references
  still work — the `/design-match` skill compares against them too.

  ---

  Built by [@anmolmoses](https://github.com/anmolmoses).PRs and adaptations welcome.
