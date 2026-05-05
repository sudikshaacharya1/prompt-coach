# prompt-coach

A Claude skill that silently analyzes every prompt you send, scores it for token efficiency, shows you exactly what to cut, and trains you to write better prompts over time.

Runs automatically in the background on every message. No commands needed.

---

## Live session dashboard

Type `show dashboard` at any point and get a full interactive breakdown of your session:

- **Overview** — score trend chart, token used vs optimal bars, top issues with expandable fix tips
- **vs last session** — side-by-side comparison with ↑↓ delta arrows on every metric
- **Improvement plan** — your 3 personalized focus areas ranked by $ impact, each with a concrete drill
- **PE scorecard** — Anthropic's 5 dimensions scored out of 20, progress bars, best and worst prompt

The dashboard populates with real data from your session — not placeholder numbers. Copy the summary at the end and paste it into your next session to enable the "vs last session" comparison.

---

## The problem this solves

Most people waste 30–70% of their tokens on prompts — not because they're asking for too much, but because of how they ask. Throat-clearing, hedging, vague scope, missing format hints. These patterns repeat across every session, cost real money at API scale, and cause Claude to misunderstand and produce work that needs redoing (which costs 2–3x more).

There's no feedback loop. You send a prompt, you get a response, you never know if you could have gotten the same response for half the tokens.

`prompt-coach` closes that loop.

---

## How it works

After every response, Claude appends a single coaching line:

```
📊 55/100 · 49 tokens (73% waste · ~$0.0001 saveable) · [concision: throat-clearing] · ~~Hey Claude, I was wondering if you could maybe help me~~ → Write
💡 Throat-clearing costs tokens: "I was wondering if you could maybe" adds 7 words, 0 information — delete it.
```

That's it. One line. Doesn't interrupt the answer. Teaches you the pattern, shows the exact fix, tells you what it cost.

---

## Features

### Per-prompt coaching (automatic, every message)
- **Score /100** across Anthropic's 5 PE dimensions: Clarity, Concision, Context, Structure, Specificity
- **Exact token count** — calculated as word count × 1.3 (standard estimate)
- **Waste %** and **$ cost** — how much you used vs how much you needed (Sonnet: $3/1M tokens)
- **Strikethrough diff** — the single biggest fix shown inline: `~~bad phrase~~ → fix`
- **PE dimension tag** — tells you which principle you violated: `[concision: throat-clearing]`
- **Micro-tip** — one rotating lesson when score < 65, with before/after example

### Warning flags
| Flag | Meaning |
|---|---|
| `⚠️ rework risk` | Prompt is ambiguous enough that Claude will likely misunderstand and need redoing (costs 2–3x) |
| `⚠️ start fresh` | Thread is long with repeated corrections — cheaper to start a new session |
| `⚠️ over-injection` | You've pasted more than needed — quote the relevant section instead |

### Context window health (shows after turn 8)
```
🪟 Context: 42k used · 21% of window · healthy
🪟 Context: 178k used · 89% of window · ⚠️ filling up — consider starting fresh soon
```
Tracks cumulative session tokens. Warns at 70%, 90%, and 97% of the 200k window.

### On-demand rewrite (`?` trigger)
Reply `?` after any coaching line to get the full rewrite:
```
Rewrite:
> Write a short professional project update email to my boss.

What changed & why:
~~Hey Claude, I was wondering if you could maybe help me~~ removed — throat-clearing, 8 words, 0 info
~~I'd really appreciate it if you could keep it professional~~ removed — "professional" already stated

Tokens: 49 → 13 · saved 36 tokens (73%)
PE principles applied: Imperative verbs, Remove throat-clearing, Scope your ask
```

### File / paste cost estimator
Paste a large block or ask "how many tokens is this?" and get:
```
📎 Paste cost: ~840 tokens · 0.4% of your remaining context window
Tip: Large — quote only the relevant section (~20 lines) to save ~760 tokens
```

### Prompt audit (`audit my prompting`)
Scores your session habits across 5 dimensions with real data from your session:
- Context — right amount, not too much or little
- Clarity — unambiguous asks
- Efficiency — no filler or hedging
- Rework Risk — prompts likely to cause misalignment
- Session Health — thread length, over-injection, re-correction loops

### Weekly improvement plan (`weekly plan`)
Generates a personalized 3-focus plan based on your actual recurring issues:
```
Focus #1 — vague scope · saves ~$0.003/month if fixed
You did this 6x this session — e.g. "what features have used"
Fix: Add what to change + desired output format
Drill: Before sending, ask "what format do I want back?" and add it
```

---

## Scoring rubric

Scores each prompt across 5 dimensions (20 pts each = 100 max):

| Dimension | What it checks | Common red flags |
|---|---|---|
| **Clarity** | Clear imperative verb, unambiguous intent | "Can you...?", "I need help with...", missing verb |
| **Concision** | No filler, hedging, or throat-clearing | "I was wondering if", "maybe", "just wanted to", "please" |
| **Context** | Useful context only — nothing redundant | Re-explaining prior conversation, unnecessary backstory |
| **Structure** | Format hint or example when needed | No format on multi-step asks, no example when output format matters |
| **Specificity** | Audience, depth, length, or format stated | "Make it better", "Write something about X", "Explain X" |

Score bands: 85–100 excellent · 65–84 good · 40–64 moderate · 0–39 high waste

---

## Installation

### Claude Code (personal, works across all projects)
```bash
# Download and unzip
unzip prompt-coach-claude-code.zip -d ~/.claude/skills/

# Verify
ls ~/.claude/skills/prompt-coach/
# Should show: SKILL.md  references/
```

Restart Claude Code. The skill loads automatically.

### Claude.ai (via Projects)
1. Go to **Projects → New Project**
2. Paste the contents of `SKILL.md` into the Project Instructions box
3. Every conversation inside that project runs the coach automatically

### Manual (any Claude conversation)
Paste the `SKILL.md` content at the start of any conversation.

---

## Prompt engineering principles reference

The skill scores against [Anthropic's official prompt engineering framework](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview). The `references/token-principles.md` file contains the full extended reference with common wasteful patterns, fixes, and scoring edge cases.

---

## What it doesn't do

- Doesn't modify your prompts automatically — it shows you what to change, you decide
- Doesn't store data between sessions (Claude has no persistent memory by default)
- Doesn't work on system prompts or Claude's own responses — only your messages
- Doesn't penalize casual chat — "thanks", "ok", "yes" are skipped entirely

---

## Background

Built iteratively over a single conversation using Claude. Informed by:
- [Anthropic's prompt engineering documentation](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)
- [MindStudio: 5 Claude Code skills that cut token costs 70%](https://www.mindstudio.ai/blog/5-claude-code-skills-cut-token-costs-70-percent-benchmarked)
- [Stop Wasting Tokens: A Developer's Guide to Claude Code Cleanup](https://naqeebali-shamsi.medium.com/stop-wasting-tokens-a-developers-guide-to-claude-code-cleanup-de842f6403e5)
- [mcpmarket.com token-usage-tracker skill](https://mcpmarket.com/tools/skills/token-usage-tracker)

---

## Files

```
prompt-coach/
├── SKILL.md                          # Main skill file — install this
├── README.md                         # This file
├── CHANGELOG.md                      # Version history
└── references/
    └── token-principles.md           # Extended PE principles reference
```

---

## License

MIT
