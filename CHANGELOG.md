# Changelog

All notable changes to `prompt-coach` are documented here.

---

## v1.4.0 — Cost tracking + session comparison + improvement plan

### Added
- **$ cost on every coaching line** — shows `~$0.0001 saveable` alongside token count using Sonnet/Haiku/Opus rates
- **Monthly projection** — if session waste exceeds $0.50/month equivalent, shows projection
- **Weekly improvement plan** (`weekly plan` trigger) — 3-focus personalized plan ranked by $ impact, with concrete fix and daily drill per issue
- **Session comparison tab** on dashboard — side-by-side vs last session with delta arrows on every KPI
- **Prev session dotted line** on score trend chart when prior session data is available

---

## v1.3.0 — Token-usage-tracker reference integration

### Added
- **Context window health line** — appears after turn 8 or 50k tokens, color-coded at 70%/90%/97% thresholds
- **Context window KPI card** on dashboard — mini progress bar showing estimated % of 200k window consumed
- **File / paste cost estimator** — estimates token cost of large pastes, advises on quoting only relevant sections
- **Session health tracking** — SESSION_LOG now tracks correction_count, over_injection_count, cumulative_tokens, context_window_pct

---

## v1.2.0 — Session health + rework prevention

### Added
- **`⚠️ rework risk` flag** — ambiguous prompts that would likely cause Claude to build the wrong thing (costs 2–3x to fix)
- **`⚠️ start fresh` flag** — long threads with repeated corrections
- **`⚠️ over-injection` flag** — prompts pasting more content than needed
- **6 rework-prevention tips** — done criteria, file naming, plan mode, batching, context re-use
- **5 session health tips** — thread length, over-injection, plan-before-complex-tasks
- **Prompt audit** (`audit my prompting`) — 5-dimension audit with Session Health as new dimension
- SESSION_LOG expanded with flags array per prompt

---

## v1.1.0 — Anthropic PE framework alignment

### Changed
- Scoring rubric updated to match Anthropic's official 5 PE dimensions: Clarity, Concision, Context, Structure, Specificity
- Issue tags now use PE dimension format: `[concision: throat-clearing]` instead of generic labels
- Token principles reference updated with Anthropic-sourced anti-patterns

### Added
- 12 PE principles with dimension labels
- Score bands with PE terminology

---

## v1.0.0 — Initial release

### Features
- Per-prompt coaching line with score, token count, waste %, diff, and issue tag
- On-demand rewrite via `?` trigger
- Micro-tip rotation (8 tips) for scores < 65
- Session tracking (SESSION_LOG)
- Interactive dashboard with score trend, token bars, issues breakdown, PE scorecard
- ✓ format for scores ≥ 85, full coaching line for scores < 85
- Casual chat detection (skip coaching on hi/thanks/ok)
