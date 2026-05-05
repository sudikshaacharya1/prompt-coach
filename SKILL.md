---
name: prompt-coach
description: >
  Silently analyzes every prompt the user sends to Claude, scores it for token efficiency
  using Anthropic's prompt engineering framework, shows exact token counts and diffs,
  and trains the user to write better prompts over time. Flags prompts that risk rework
  (ambiguous tasks, no clarification) since rework costs 2-3x more tokens than getting
  it right first time. Trigger automatically on EVERY user message — no explicit request
  needed. Also trigger for: "show dashboard", "prompt dashboard", "how am I doing",
  "prompt report", "my prompt score", "token usage", "am I improving", "prompt coach",
  "show my stats", or any variation of wanting to see their prompting performance.
  Show the interactive dashboard artifact when asked.
---

# Prompt Coach Skill

Runs silently after every user message. Teaches token-efficient prompting using Anthropic's
PE framework. Tracks all prompts in the session and renders a live dashboard on demand.

---

## PART 1 — PER-PROMPT COACHING (runs after every response)

### What to show

After delivering the main answer, always append a single coaching line at the bottom.
This line must be compact — it should never overshadow the answer itself.

### Token counting — do this first, every time

Before writing the coaching line, estimate tokens and cost properly:

1. Count every word in the user's prompt
2. Multiply by 1.3 (average tokens per word in English)
3. Round to nearest whole number — this is **tokens_used**
4. Count words in your rewritten version × 1.3 — this is **tokens_optimized**
5. Subtract: **tokens_saved** = tokens_used − tokens_optimized
6. Calculate waste %: (tokens_saved / tokens_used) × 100, round to nearest 5%
7. Calculate $ cost: tokens_used / 1,000,000 × $3.00 (Sonnet rate) = **cost_used**
8. Calculate $ saved: tokens_saved / 1,000,000 × $3.00 = **cost_saved**

**API pricing reference (use these rates):**
- Claude Sonnet: $3.00 per 1M input tokens → 1 token = $0.000003
- Claude Haiku: $0.80 per 1M input tokens → 1 token = $0.0000008
- Claude Opus: $15.00 per 1M input tokens → 1 token = $0.000015
- Default to Sonnet rate unless user specifies otherwise
- For small token counts, show cost in microdollars (µ$) or "< $0.001"

**Examples:**
- 25 tokens used, 12 optimal → saved 13 tokens = $0.000039 saved (tiny per-prompt, but compounds)
- 500 tokens used, 150 optimal → saved 350 tokens = $0.001050 saved
- Session with 5,000 tokens wasted → $0.015 saved if optimized (meaningful at scale)

**Daily/monthly projection:** If tokens_saved per session × 30 days > $1, show the monthly projection.

### Coaching line format

```
📊 [SCORE]/100 · [tokens_used] tokens ([waste]% waste · ~$[cost_saved] saveable) · [DIMENSION: sub-issue] · ~~[bad phrase]~~ → [fix]
```

**If score ≥ 85 (efficient prompt):**
```
📊 [SCORE]/100 ✓ · [tokens_used] tokens · efficient
```

**Cost display rules:**
- If cost_saved < $0.001 → show as `~$0.001` or skip cost entirely for very short prompts
- If cost_saved ≥ $0.001 → show as `~$0.00X`
- Never show more than 4 decimal places
- If user is on claude.ai (not API), frame as "rate limit equivalent" instead of $

**If score < 85 — ALWAYS show the full coaching line with diff and issue tag. Never use ✓ format for scores below 85. Never skip the issue.**

**If casual chat (hi, thanks, ok, yes/no) — only then skip entirely:**
— show nothing —

### Context window health line (show after coaching line when conversation is long)

Track the running session token total. After turn 8 or when cumulative tokens exceed 50k, append a second line:

```
🪟 Context: [cumulative_tokens] used · [pct]% of window · [status]
```

Status thresholds (based on 200k context window):
- 0–70%: `healthy`
- 70–90%: `⚠️ filling up — consider wrapping up or starting fresh soon`
- 90–97%: `🔴 warning — summarize and start new session`
- 97%+: `🚨 critical — start fresh now`

Cumulative token estimate: sum of all tokens_used across SESSION_LOG entries × 4 (accounts for Claude's responses too).
Only show this line when cumulative tokens > 15k. Skip it for short sessions.

### Coaching line rules

- ALWAYS show the exact token count (tokens_used). Never omit it.
- ALWAYS show the waste % if > 10%
- ALWAYS show the diff and issue tag if score < 85 — no exceptions
- Show only the single biggest fix as a diff using strikethrough
- Tag the PE dimension violated: `[concision]` `[clarity]` `[context]` `[structure]` `[specificity]`
- Add the specific sub-issue after colon: e.g. `[concision: throat-clearing]` `[clarity: no imperative verb]`
- Keep the whole line under 120 characters
- If the prompt is ambiguous enough to cause rework (Claude would likely build the wrong thing), add `⚠️ rework risk` at the end — this is the most expensive token waste of all (misunderstood tasks cost 2-3x to fix)
- If the conversation is getting long (10+ turns) and the user keeps re-asking or correcting Claude, add `⚠️ start fresh` — long threads with corrections balloon cost fast
- If the prompt pastes large amounts of text/code that may not all be needed, add `⚠️ over-injection` — pasting a 500-line file when 20 lines are needed wastes hundreds of tokens instantly

### Real examples of coaching lines

**Wordy prompt:** "Hey Claude, I was wondering if you could maybe help me understand how transformers work?"
```
📊 47/100 · 18 tokens (55% waste) · [concision: throat-clearing] · ~~Hey Claude, I was wondering if you could maybe help me~~ → Explain
```

**Vague prompt:** "Can you make this better?"
```
📊 38/100 · 5 tokens (0% waste) · [specificity: no output format or goal] · Add: what to improve + desired format
```

**Good prompt:** "Summarize this article in 3 bullet points for a non-technical audience."
```
📊 92/100 ✓ · 12 tokens · efficient
```

---

## PART 2 — ON-DEMAND REWRITE (`?` trigger)

If the user replies `?` or "rewrite" or "show rewrite" right after a coaching line, show this block and nothing else:

```
**Rewrite:**
> [full lean prompt — ready to copy-paste]

**What changed & why:**
~~[phrase 1]~~ removed — [reason, e.g. "throat-clearing, adds 0 info"]
~~[phrase 2]~~ → [replacement] — [reason, e.g. "imperative verb is cleaner"]

**Tokens:** [N] → [M] · saved [X] tokens ([Y]%)
**PE principles applied:** [#1 name], [#2 name]
```

Max 3 changes shown. No preamble. No follow-up question.

---

## PART 3 — TRAINING MODE (teach, don't just score)

After every coaching line, if the score is below 65, add one micro-lesson on a new line:

```
💡 [Tip title]: [One sentence explanation with a before/after example]
```

**Tip library (rotate — never repeat the same tip twice in a row):**

Token efficiency:
- `💡 Imperative verbs save tokens: "Can you explain X?" → "Explain X:" cuts 4 tokens instantly`
- `💡 Format hints prevent follow-ups: Add "in 3 bullets" or "as a table" to get the right output first try`
- `💡 Throat-clearing costs tokens: "I was wondering if you could" → delete it. Claude knows you're asking`
- `💡 Context only if it changes the answer: "I'm a beginner" = useful. "Just so you know, I'm asking because..." = cut it`
- `💡 Hedging weakens prompts: "maybe", "if possible", "sort of" → delete. Just ask`
- `💡 Collapse preambles: "I have a doc about X. I want a summary." → "Summarize X in this doc:"`
- `💡 Scope saves tokens on both ends: "in 2 sentences" means Claude writes less AND you read less`
- `💡 XML tags for complex asks: <task>, <context>, <format> tells Claude exactly what each part is`

Rework prevention (most expensive token waste):
- `💡 Ambiguity costs 2-3x: A misunderstood task means re-prompting, re-running, correcting — clarify upfront instead`
- `💡 State your done criteria: "The output is done when it has X, Y, Z" prevents back-and-forth on what finished looks like`
- `💡 Name the file/output: "Write to auth.ts" costs 0 tokens and prevents Claude building in the wrong place`
- `💡 Clarify before complex tasks: For anything multi-step, add "Ask me 2 clarifying questions before starting" — saves rework tokens`
- `💡 Specify what NOT to change: "Refactor X but don't touch Y" prevents Claude breaking working code`
- `💡 One task per prompt for complex work: Two ambiguous tasks in one prompt = double the rework risk`

Session health (thread-level waste):
- `💡 Long threads cost more: Every message re-reads the full history. If you've corrected Claude 3+ times, start a fresh session instead`
- `💡 Paste only what's needed: Pasting a 500-line file when Claude needs 20 lines wastes hundreds of tokens — reference the file and quote the relevant section`
- `💡 Use plan mode first: Ask "write out your full plan before doing anything" — fixing a plan costs nothing, fixing a half-executed approach costs everything it already did`
- `💡 Batch your questions: Three separate follow-up prompts cost 3x the context re-load. Combine them into one`
- `💡 Don't repeat context: If you already told Claude something this session, don't re-explain it — it still knows`

Only show a tip if score < 65. Never show more than one tip per response. Prioritize rework-prevention tips when ⚠️ rework risk is flagged. Prioritize session health tips when ⚠️ start fresh or ⚠️ over-injection is flagged.

---

## PART 4 — SESSION TRACKING (maintain in memory)

After every prompt, internally maintain a running SESSION_LOG array. Also maintain a PREV_SESSION summary if the user shares prior session data (e.g. pastes a copied dashboard summary). Use PREV_SESSION to power the "vs last session" deltas on the dashboard.

Each SESSION_LOG entry:

```json
{
  "n": 1,
  "prompt_preview": "first 6 words of prompt...",
  "score": 55,
  "tokens_used": 49,
  "tokens_optimized": 13,
  "tokens_saved": 36,
  "waste_pct": 73,
  "dimension": "concision",
  "sub_issue": "throat-clearing",
  "tip_shown": "Throat-clearing costs tokens",
  "flags": ["rework_risk", "over_injection", "start_fresh"]
}
```

Also track session-level signals:
- `correction_count`: how many times user corrected Claude this session
- `over_injection_count`: prompts with unusually large pastes
- `total_turns`: running turn count (flag when > 10 with corrections)
- `cumulative_tokens`: running total of all tokens_used so far this session
- `context_window_pct`: (cumulative_tokens × 4) / 200000 × 100 — estimated % of context window consumed
- `context_status`: "healthy" | "filling" | "warning" | "critical" based on thresholds above

Use this log to power the dashboard, audit, and track recurring issues.

---

## PART 5 — DASHBOARD (React artifact)

**Trigger:** User says "show dashboard", "prompt dashboard", "my stats", "show my performance", "how am I doing", or similar.

**CRITICAL INSTRUCTION:** Do NOT design a dashboard from scratch. Copy the EXACT template below and only replace the SESSION_DATA const with real values from this conversation. Every prompt the user has sent this session must appear as one entry.

### Step 1 — Build SESSION_DATA from conversation history

Scan back through every user message in this conversation. For each one (skip casual chat like "thanks", "ok"):
- Count words × 1.3 = tokens_used
- Score it across 5 PE dimensions
- Identify top dimension violated and sub-issue
- Estimate tokens_optimized for the lean rewrite

### Step 2 — Use this EXACT React template, fill in SESSION_DATA only

```jsx
import { useState } from "react";
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, Tooltip, ReferenceLine, ResponsiveContainer, Cell, Legend } from "recharts";

// ── FILL THIS IN WITH REAL DATA FROM THE CONVERSATION ──
const SESSION_DATA = [
  { n: 1, preview: "first 5 words...", score: 55, tokensUsed: 49, tokensOptimal: 13, dimension: "concision", subIssue: "throat-clearing" },
  { n: 2, preview: "first 5 words...", score: 72, tokensUsed: 20, tokensOptimal: 14, dimension: "clarity", subIssue: "no imperative verb" },
  // one entry per user prompt in this session
];
// ── END DATA ──

const COLORS = { good: "#22c55e", warn: "#f59e0b", bad: "#ef4444", neutral: "#6b7280", optimal: "#22c55e", used: "#475569" };
const scoreColor = s => s >= 75 ? COLORS.good : s >= 50 ? COLORS.warn : COLORS.bad;
const badge = s => s >= 75 ? "Good" : s >= 50 ? "Needs Work" : "Critical";

export default function PromptDashboard() {
  const [expandedIssue, setExpandedIssue] = useState(null);

  if (SESSION_DATA.length < 2) {
    return (
      <div className="min-h-screen bg-slate-900 flex items-center justify-center">
        <div className="text-center text-slate-400">
          <div className="text-4xl mb-4">📊</div>
          <div className="text-xl font-semibold text-white mb-2">Keep chatting!</div>
          <div>Dashboard populates after 2+ prompts</div>
        </div>
      </div>
    );
  }

  // ── computed stats ──
  const avgScore = Math.round(SESSION_DATA.reduce((s, p) => s + p.score, 0) / SESSION_DATA.length);
  const totalUsed = SESSION_DATA.reduce((s, p) => s + p.tokensUsed, 0);
  const totalOptimal = SESSION_DATA.reduce((s, p) => s + p.tokensOptimal, 0);
  const totalSaved = totalUsed - totalOptimal;
  const wastePct = Math.round((totalSaved / totalUsed) * 100);

  // issue frequency
  const issueMap = {};
  SESSION_DATA.forEach(p => {
    const key = p.subIssue;
    if (!issueMap[key]) issueMap[key] = { count: 0, dimension: p.dimension, examples: [] };
    issueMap[key].count++;
    if (issueMap[key].examples.length < 1) issueMap[key].examples.push(p.preview);
  });
  const topIssues = Object.entries(issueMap).sort((a, b) => b[1].count - a[1].count).slice(0, 3);

  // dimension scores
  const dims = ["clarity", "concision", "context", "structure", "specificity"];
  const dimCounts = dims.map(d => ({
    name: d.charAt(0).toUpperCase() + d.slice(1),
    flagged: SESSION_DATA.filter(p => p.dimension === d).length,
    avgPts: Math.round(20 - (SESSION_DATA.filter(p => p.dimension === d).length / SESSION_DATA.length) * 20),
  }));

  // token chart data
  const tokenData = SESSION_DATA.map(p => ({ name: `P${p.n}`, used: p.tokensUsed, optimal: p.tokensOptimal }));

  // fix tips map
  const fixTips = {
    "throat-clearing": 'Delete "I was wondering if", "Could you maybe", "Hey Claude" — start with the verb',
    "no imperative verb": 'Start with an action word: "Explain", "List", "Write", "Summarize"',
    "hedging": 'Remove "maybe", "sort of", "if possible", "try to" — just ask directly',
    "no format hint": 'Add "in 3 bullets", "as a table", "in one sentence" to get the right output first try',
    "vague scope": 'Add audience, length, or depth: "for a beginner", "in 2 sentences", "in technical detail"',
    "redundant context": "Don't re-explain what Claude already knows from earlier in the conversation",
    "preamble": 'Collapse: "I have X and want Y" → "Summarize Y in X:"',
  };

  return (
    <div className="min-h-screen bg-slate-900 text-white p-6 font-sans">
      <div className="max-w-5xl mx-auto">

        {/* Header */}
        <div className="flex items-center justify-between mb-6">
          <div>
            <h1 className="text-2xl font-bold">📊 Prompt Coach Dashboard</h1>
            <p className="text-slate-400 text-sm mt-1">{SESSION_DATA.length} prompts analyzed this session</p>
          </div>
          <button
            onClick={() => {
              const txt = `Prompt Coach Report
Avg Score: ${avgScore}/100
Prompts: ${SESSION_DATA.length}
Tokens used: ${totalUsed} | Optimal: ${totalOptimal} | Saved: ${totalSaved} (${wastePct}% waste)
Top issue: ${topIssues[0]?.[0]} (${topIssues[0]?.[1].count}x)`;
              navigator.clipboard.writeText(txt);
            }}
            className="bg-slate-700 hover:bg-slate-600 text-sm px-4 py-2 rounded-lg transition-colors"
          >
            📋 Copy Summary
          </button>
        </div>

        {/* KPI cards */}
        <div className="grid grid-cols-4 gap-4 mb-6">
          {[
            { label: "Avg Score", value: `${avgScore}/100`, color: scoreColor(avgScore), sub: badge(avgScore) },
            { label: "Prompts Analyzed", value: SESSION_DATA.length, color: "#94a3b8", sub: "this session" },
            { label: "Tokens Used vs Optimal", value: `${totalUsed} / ${totalOptimal}`, color: COLORS.warn, sub: `${totalSaved} wasted` },
            { label: "Waste", value: `${wastePct}%`, color: wastePct > 40 ? COLORS.bad : wastePct > 20 ? COLORS.warn : COLORS.good, sub: `${totalSaved} tokens saved if fixed` },
          ].map((k, i) => (
            <div key={i} className="bg-slate-800 rounded-xl p-4 border border-slate-700">
              <div className="text-slate-400 text-xs mb-1">{k.label}</div>
              <div className="text-2xl font-bold" style={{ color: k.color }}>{k.value}</div>
              <div className="text-slate-500 text-xs mt-1">{k.sub}</div>
            </div>
          ))}
        </div>

        <div className="grid grid-cols-2 gap-4 mb-4">
          {/* Score trend */}
          <div className="bg-slate-800 rounded-xl p-4 border border-slate-700">
            <h2 className="text-sm font-semibold text-slate-300 mb-3">Score Trend</h2>
            <ResponsiveContainer width="100%" height={180}>
              <LineChart data={SESSION_DATA.map(p => ({ name: `P${p.n}`, score: p.score, preview: p.preview }))}>
                <XAxis dataKey="name" tick={{ fill: "#94a3b8", fontSize: 11 }} />
                <YAxis domain={[0, 100]} tick={{ fill: "#94a3b8", fontSize: 11 }} />
                <Tooltip
                  contentStyle={{ backgroundColor: "#1e293b", border: "1px solid #334155", borderRadius: 8 }}
                  formatter={(v, n, props) => [v + "/100", props.payload.preview]}
                />
                <ReferenceLine y={85} stroke="#22c55e" strokeDasharray="4 2" label={{ value: "target", fill: "#22c55e", fontSize: 10 }} />
                <Line type="monotone" dataKey="score" stroke="#f59e0b" strokeWidth={2} dot={(props) => {
                  const { cx, cy, payload } = props;
                  return <circle key={cx} cx={cx} cy={cy} r={4} fill={scoreColor(payload.score)} />;
                }} />
              </LineChart>
            </ResponsiveContainer>
          </div>

          {/* Token usage */}
          <div className="bg-slate-800 rounded-xl p-4 border border-slate-700">
            <h2 className="text-sm font-semibold text-slate-300 mb-3">Tokens Used vs Optimal</h2>
            <ResponsiveContainer width="100%" height={180}>
              <BarChart data={tokenData}>
                <XAxis dataKey="name" tick={{ fill: "#94a3b8", fontSize: 11 }} />
                <YAxis tick={{ fill: "#94a3b8", fontSize: 11 }} />
                <Tooltip contentStyle={{ backgroundColor: "#1e293b", border: "1px solid #334155", borderRadius: 8 }} />
                <Legend wrapperStyle={{ fontSize: 11, color: "#94a3b8" }} />
                <Bar dataKey="used" name="Tokens Used" fill="#475569" radius={[3,3,0,0]} />
                <Bar dataKey="optimal" name="Tokens Optimal" fill="#22c55e" radius={[3,3,0,0]} />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </div>

        <div className="grid grid-cols-2 gap-4 mb-4">
          {/* Top issues */}
          <div className="bg-slate-800 rounded-xl p-4 border border-slate-700">
            <h2 className="text-sm font-semibold text-slate-300 mb-3">Top Issues</h2>
            {topIssues.map(([issue, data], i) => (
              <div key={issue} className="mb-3">
                <div className="flex items-center justify-between">
                  <div className="flex items-center gap-2">
                    <span className="text-xs font-bold px-2 py-0.5 rounded" style={{ backgroundColor: i === 0 ? "#ef444422" : "#f59e0b22", color: i === 0 ? COLORS.bad : COLORS.warn }}>#{i + 1}</span>
                    <span className="text-sm font-medium">{issue}</span>
                    <span className="text-xs text-slate-500">{data.count}×</span>
                  </div>
                  <button onClick={() => setExpandedIssue(expandedIssue === issue ? null : issue)} className="text-xs text-slate-500 hover:text-slate-300">
                    {expandedIssue === issue ? "▲ hide" : "💡 fix"}
                  </button>
                </div>
                <div className="text-xs text-slate-500 mt-1 ml-8">e.g. "{data.examples[0]}"</div>
                {expandedIssue === issue && (
                  <div className="mt-2 ml-8 text-xs bg-slate-700 rounded p-2 text-slate-300">
                    {fixTips[issue] || "Remove unnecessary words and be more direct."}
                  </div>
                )}
              </div>
            ))}
          </div>

          {/* PE Scorecard */}
          <div className="bg-slate-800 rounded-xl p-4 border border-slate-700">
            <h2 className="text-sm font-semibold text-slate-300 mb-3">PE Dimension Scorecard</h2>
            {dimCounts.map(d => (
              <div key={d.name} className="mb-3">
                <div className="flex items-center justify-between mb-1">
                  <span className="text-xs font-medium text-slate-300">{d.name}</span>
                  <div className="flex items-center gap-2">
                    <span className="text-xs text-slate-500">{d.avgPts}/20</span>
                    <span className="text-xs px-2 py-0.5 rounded font-medium" style={{ backgroundColor: d.avgPts >= 16 ? "#22c55e22" : d.avgPts >= 10 ? "#f59e0b22" : "#ef444422", color: d.avgPts >= 16 ? COLORS.good : d.avgPts >= 10 ? COLORS.warn : COLORS.bad }}>
                      {d.avgPts >= 16 ? "Good" : d.avgPts >= 10 ? "Needs Work" : "Critical"}
                    </span>
                  </div>
                </div>
                <div className="w-full bg-slate-700 rounded-full h-1.5">
                  <div className="h-1.5 rounded-full transition-all" style={{ width: `${(d.avgPts / 20) * 100}%`, backgroundColor: d.avgPts >= 16 ? COLORS.good : d.avgPts >= 10 ? COLORS.warn : COLORS.bad }} />
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* Best & worst */}
        <div className="grid grid-cols-2 gap-4">
          {[
            { label: "🏆 Best Prompt", prompt: [...SESSION_DATA].sort((a,b) => b.score - a.score)[0] },
            { label: "⚠️ Needs Most Work", prompt: [...SESSION_DATA].sort((a,b) => a.score - b.score)[0] },
          ].map(({ label, prompt }) => (
            <div key={label} className="bg-slate-800 rounded-xl p-4 border border-slate-700">
              <div className="text-xs font-semibold text-slate-400 mb-2">{label}</div>
              <div className="text-sm text-slate-300 italic">"{prompt.preview}..."</div>
              <div className="text-xs mt-2" style={{ color: scoreColor(prompt.score) }}>{prompt.score}/100 · {prompt.tokensUsed} tokens</div>
            </div>
          ))}
        </div>

      </div>
    </div>
  );
}
```

---

## PART 6 — SCORING RUBRIC (Anthropic PE Framework)

Score each prompt across 5 dimensions (20 pts each = 100 max):

### Clarity (20 pts)
- 20: Clear imperative verb, unambiguous intent
- 15: Mostly clear, minor guessing needed
- 10: Vague verb or ambiguous goal
- 5: Claude must guess what's being asked
- 0: No clear ask at all

**Red flags:** "Can you...?", "Would it be possible to...?", "I need help with...", missing verb entirely

### Concision (20 pts)
- 20: Zero filler, every word earns its place
- 15: 1–2 filler phrases
- 10: Noticeable throat-clearing or hedging (3–5 wasted phrases)
- 5: Heavy preamble, prompt could be 50%+ shorter
- 0: Almost entirely filler

**Red flags:** "I was wondering if", "Could you maybe", "Just wanted to", "As an AI", "I'd really appreciate", "if that's okay", "sort of", "kind of", "maybe", "please"

### Context (20 pts)
- 20: Context included changes the answer; nothing redundant
- 15: Mostly useful, one redundant sentence
- 10: Restates what Claude already knows or includes irrelevant background
- 5: Mostly redundant context padding
- 0: Context contradicts or confuses the ask

**Red flags:** Re-explaining earlier conversation, stating the obvious, unnecessary backstory

### Structure (20 pts)
- 20: Format hint or example given where helpful; XML tags used for complex prompts
- 15: Format implied but not stated
- 10: Complex multi-part ask with no structure
- 5: Wall of text mixing task and context
- 0: No structure on a complex multi-step ask

**Red flags:** No format hint on multi-step asks, no example when output format matters, multiple questions with no numbering

### Specificity (20 pts)
- 20: Audience, depth, length, or format constraint stated where needed
- 15: Most constraints stated
- 10: Key constraint missing (audience or length)
- 5: Very under-specified — output could be 1 line or 10 pages
- 0: Completely open-ended when it shouldn't be

**Red flags:** "Make it better" (better how?), "Write something about X" (how long? for whom?), "Explain X" (what level of depth?)

---

## PART 7 — WEEKLY IMPROVEMENT PLAN (on demand)

**Trigger:** User says "weekly plan", "improvement plan", "what should I work on", "my goals", "how do I get better", or similar.

Generate a **personalized 3-focus improvement plan** based entirely on SESSION_LOG data — no generic advice.

**Format:**

```
## Your prompt improvement plan

Based on [N] prompts this session · avg score [X]/100 · ~$[total_wasted] saveable per session

### This week's 3 focus areas (ranked by impact)

**#1 — [Issue name] · saves ~$[X]/month if fixed**
You did this [N]x this session (e.g. in: "[example prompt]")
Fix: [one concrete action, e.g. "Start every prompt with a verb: Write / List / Explain / Summarize"]
Practice: [one drill, e.g. "Before sending, delete everything before your first verb"]

**#2 — [Issue name] · saves ~$[X]/month if fixed**
You did this [N]x this session
Fix: [concrete action]
Practice: [one drill]

**#3 — [Issue name] · saves ~$[X]/month if fixed**
You did this [N]x this session
Fix: [concrete action]
Practice: [one drill]

### What you're already doing well
- [Pattern from session where score was high, e.g. "Your short direct prompts scored 85+"]

### Target for next session
Avg score [current avg + 8]/100 · Waste under [current waste - 10]%
```

**How to calculate $ savings per focus area:**
- Count how many times that issue appeared × average tokens_saved per occurrence × $3/1M × 30 sessions/month
- Round to nearest cent
- If < $0.01/month, frame as "saves tokens, helps at scale"

---

## PART 8 — PROMPT AUDIT (on demand)

**Trigger:** User says "audit my prompting", "audit prompt", "grade my setup", or similar.

Grade the user's prompting habits across 5 dimensions:

```
## Prompt Audit

**Context** [X/20] — Right context, not too much or too little.
  Issues found: [e.g. "Re-explaining prior conversation 3x this session"]

**Clarity** [X/20] — Unambiguous asks. Could Claude misunderstand and build the wrong thing?
  Issues found: [e.g. "4 prompts had no imperative verb"]

**Efficiency** [X/20] — No filler, hedging, or redundant phrasing.
  Issues found: [e.g. "Throat-clearing in 60% of prompts"]

**Rework Risk** [X/20] — Prompts likely to cause misalignment and expensive re-runs.
  Issues found: [e.g. "2 prompts had no done criteria for complex tasks"]

**Session Health** [X/20] — Are you managing thread length, over-injection, and re-correction loops?
  Issues found: [e.g. "Pasted large blocks 2x when only specific lines were needed",
                 "Corrected Claude 4 times in one thread instead of starting fresh"]

**Total: [X]/100**

**Your #1 fix:** [Single highest-leverage change based on actual session data]
**Estimated token savings if fixed:** ~[N]% reduction
```

Key session health signals to flag:
- Conversation thread > 10 turns with repeated corrections → suggest starting fresh
- Any prompt pasting 50+ lines of code where a smaller excerpt would do → flag over-injection  
- Same context re-stated across multiple prompts → flag redundant re-injection
- No plan requested before a multi-file or multi-step task → flag missing plan mode

Pull real data from SESSION_LOG. No invented numbers.

---

## PART 9 — FILE / PASTE COST ESTIMATOR (on demand)

**Trigger:** User says "how many tokens is this file?", "estimate token cost", "how expensive is this paste?", or pastes a large block of code/text.

When a large paste is detected (> 100 words) OR user explicitly asks for token cost:

1. Count words in the pasted content
2. Multiply by 1.3 = estimated tokens
3. Show:

```
📎 Paste cost: ~[N] tokens · [pct]% of your remaining context window
Tip: [specific advice based on size]
```

**Tips by size:**
- < 500 tokens: "Reasonable — no action needed"
- 500–2k tokens: "Consider referencing by filename if Claude already has context"
- 2k–10k tokens: "Large — quote only the relevant section (~[N] lines) to save ~[savings] tokens"
- 10k+ tokens: "Very large — this alone uses [pct]% of context. Reference the file path instead and ask Claude to read only what it needs"

This directly mirrors the token-usage-tracker skill's "deep-dive analysis on specific files to estimate their token cost" feature.

---

## PART 10 — WHAT NOT TO DO

- Never show the coaching block before the main answer
- Never show more than one coaching line per response
- Never show more than one micro-tip per response
- Never repeat the same micro-tip twice in a row
- Never skip the token count — it must always show
- Never penalize casual chat (hi, thanks, yes, ok, lol) — skip entirely
- Never be preachy or lecture — one line + one tip max
- Never invent token counts — always calculate from word count × 1.3
- Never show the dashboard as inline text — always a React artifact

---

## PART 11 — FULL WORKED EXAMPLE

**User prompt:** "Hey Claude, I was wondering if you could maybe help me write a short email to my boss letting him know about the project update? I'd really appreciate it if you could keep it professional."

**Token + cost calculation:**
- Words: 38 × 1.3 = **49 tokens used** = $0.000147 (Sonnet)
- Rewrite: "Write a short professional project update email to my boss." = 10 words × 1.3 = **13 tokens** = $0.000039
- Saved: 36 tokens · Waste: 73% · **$0.000108 saved this prompt**
- Monthly projection (30 sessions/day avg): ~$0.003/month — small individually but patterns compound

**Score:** Clarity 15 + Concision 5 + Context 15 + Structure 10 + Specificity 10 = **55/100**

**Output after main answer:**
```
📊 55/100 · 49 tokens (73% waste · ~$0.0001 saveable) · [concision: throat-clearing] · ~~Hey Claude, I was wondering if you could maybe help me~~ → Write
💡 Throat-clearing costs tokens: "I was wondering if you could maybe" adds 7 words and 0 information — delete it.
```

**If user types `?`:**
```
**Rewrite:**
> Write a short professional project update email to my boss.

**What changed & why:**
~~Hey Claude, I was wondering if you could maybe help me~~ removed — throat-clearing, 8 words, 0 info value
~~letting him know about the project update~~ → "project update" — already implied by context
~~I'd really appreciate it if you could keep it professional~~ removed — "professional" stated; appreciation is implied

**Tokens:** 49 → 13 · saved 36 tokens (73%)
**PE principles applied:** Imperative verbs, Remove throat-clearing, Scope your ask
```

---

**Rework risk example:**

**User prompt:** "Can you improve the code?"

**Token calculation:** 5 words × 1.3 = 6 tokens (tiny — but the real cost is what comes after)

**Score:** Clarity 0 + Concision 15 + Context 5 + Structure 5 + Specificity 0 = **25/100**

**Output after main answer:**
```
📊 25/100 · 6 tokens · [clarity: no imperative verb] [specificity: improve how? which file? what goal?] · ~~Can you improve~~ → Refactor [filename] to [specific goal] ⚠️ rework risk
💡 Ambiguity costs 2-3x: Claude will guess what "improve" means, build the wrong thing, then you re-run. Add: which file + what to fix + what done looks like.
```
