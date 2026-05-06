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

## PART 5 — DASHBOARD (HTML artifact)

**Trigger:** User says "show dashboard", "prompt dashboard", "my stats", "show my performance", "how am I doing", or similar.

### Step 0 — Ask the user which mode they want (ALWAYS do this first)

Before building anything, ask:

```
📊 Dashboard ready — which mode do you want?

**A) Live dashboard** — opens as a persistent artifact that updates as you chat. Instant to load, always current. (Recommended)

**B) On-demand snapshot** — generated fresh when you ask. Shows full session history but takes 60–90 seconds to build since it re-scans every prompt from scratch.

Reply A or B.
```

Then wait for their reply before doing anything else.

---

### Mode A — Live dashboard

**How it works:**
- User says "show dashboard" → Claude renders the live dashboard widget (pure HTML, no JS libraries)
- User says "update dashboard" → Claude scans the session, builds a SESSION array of all prompts with scores and issues, then re-renders the widget with that data hardcoded in
- Dashboard renders instantly — no re-scanning at render time, data is already baked in
- No manual logging — Claude does all the work on "update dashboard"

**When user says "update dashboard":**
1. Scan every user message this session (skip casual chat)
2. For each prompt: score it, identify top issue, count words × 1.3 = tokens
3. Build a SESSION array with all entries
4. Render the widget below with SESSION hardcoded

**Widget template — use this EXACTLY, only swap SESSION data:**

```html
<style>
*{box-sizing:border-box;margin:0;padding:0;font-family:var(--font-sans)}
.w{padding:1rem 0}
.kg{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:10px;margin-bottom:1.25rem}
.kc{background:var(--color-background-secondary);border-radius:var(--border-radius-md);padding:.875rem}
.kl{font-size:11px;color:var(--color-text-secondary);margin-bottom:5px}
.kv{font-size:20px;font-weight:500}
.ks{font-size:11px;color:var(--color-text-secondary);margin-top:3px}
.card{background:var(--color-background-primary);border:0.5px solid var(--color-border-tertiary);border-radius:var(--border-radius-lg);padding:1rem;margin-bottom:10px}
.cl{font-size:12px;font-weight:500;color:var(--color-text-secondary);margin-bottom:10px}
.row{display:flex;justify-content:space-between;padding:6px 0;border-bottom:0.5px solid var(--color-border-tertiary);font-size:12px;align-items:center}
.row:last-child{border-bottom:none}
.ir{display:flex;gap:8px;padding:7px 0;border-bottom:0.5px solid var(--color-border-tertiary);align-items:flex-start}
.ir:last-child{border-bottom:none}
.bdg{font-size:10px;font-weight:500;padding:2px 6px;border-radius:var(--border-radius-md);flex-shrink:0;margin-top:1px}
.dr{margin-bottom:9px}
.dh{display:flex;justify-content:space-between;align-items:center;margin-bottom:3px}
.bt{background:var(--color-background-secondary);border-radius:4px;height:5px}
.bf{height:5px;border-radius:4px}
</style>

<div class="w">
  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1.25rem">
    <div>
      <div style="font-size:16px;font-weight:500;color:var(--color-text-primary)">Prompt coach — live</div>
      <div style="font-size:12px;color:var(--color-text-secondary);margin-top:3px" id="sub"></div>
    </div>
    <button onclick="clearAll()" style="font-size:12px;padding:5px 12px;cursor:pointer;background:transparent;border:0.5px solid var(--color-border-tertiary);border-radius:var(--border-radius-md);color:var(--color-text-secondary);font-family:var(--font-sans)">Clear</button>
  </div>
  <div class="kg" id="kpis"></div>
  <div class="card"><div class="cl">Score history</div><div id="hist"></div></div>
  <div class="card"><div class="cl">Top issues</div><div id="iss"></div></div>
  <div class="card"><div class="cl">PE scorecard</div><div id="dims"></div></div>
</div>

<script>
// ── REPLACE THIS ARRAY WITH REAL SESSION DATA ──
const SESSION=[
  {n:1,preview:"first 5 words of prompt",score:55,issue:"throat-clearing"},
  {n:2,preview:"next prompt preview",score:78,issue:"vague scope"},
  // one entry per user prompt this session
];
// ── END SESSION DATA ──

const RC=[["#FCEBEB","#791F1F"],["#FAEEDA","#633806"],["#EAF3DE","#27500A"],["#E6F1FB","#0C447C"]];
const FX={"vague scope":"Add output format + what to change","throat-clearing":"Delete everything before your first verb","no format hint":"Add 'in 3 bullets' or 'as a table'","missing context":"Add who it's for","no imperative verb":"Start with Explain / Write / List","preamble":"Collapse to one imperative sentence"};
const DIMS=["clarity","concision","context","structure","specificity"];
const DIM_ISSUES={"clarity":["no imperative verb","missing context"],"concision":["throat-clearing","preamble","caps"],"context":["redundant","over-injection"],"structure":["no format hint"],"specificity":["vague scope","vague"]};
const sc=s=>s>=75?"#3B6D11":s>=50?"#854F0B":"#A32D2D";
const slbl=s=>s>=75?"Good":s>=50?"Needs work":"Critical";
let log=SESSION;

async function init(){
  try{await window.storage.set("pc_log_v3",JSON.stringify(log));}catch(e){}
  render();
}

async function clearAll(){
  if(!confirm("Clear all data?"))return;
  log=[];
  try{await window.storage.delete("pc_log_v3");}catch(e){}
  render();
}

function render(){
  const N=log.length;
  const avg=N?Math.round(log.reduce((a,p)=>a+p.score,0)/N):0;
  document.getElementById("sub").textContent=`${N} prompts · avg ${avg}/100 · updated now`;

  const topEntry=Object.entries(log.filter(p=>p.issue&&p.issue!=="none").reduce((m,p)=>{m[p.issue]=(m[p.issue]||0)+1;return m;},{})).sort((a,b)=>b[1]-a[1])[0];
  document.getElementById("kpis").innerHTML=[
    {l:"Avg score",v:`${avg}/100`,c:sc(avg),s:slbl(avg)},
    {l:"Prompts tracked",v:N,c:"var(--color-text-primary)",s:`${log.filter(p=>p.issue&&p.issue!=="none").length} with issues`},
    {l:"Top issue",v:topEntry?topEntry[0]:"none",c:topEntry?"#854F0B":"var(--color-text-secondary)",s:topEntry?`${topEntry[1]}× this session`:"—"},
  ].map(k=>`<div class="kc"><div class="kl">${k.l}</div><div class="kv" style="color:${k.c};font-size:${String(k.v).length>9?"14px":"20px"}">${k.v}</div><div class="ks">${k.s}</div></div>`).join("");

  document.getElementById("hist").innerHTML=N?log.slice(-10).reverse().map(p=>`
  <div class="row">
    <span style="color:var(--color-text-secondary);max-width:62%;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">P${p.n} — ${p.preview}</span>
    <div style="display:flex;gap:8px;align-items:center;flex-shrink:0">
      ${p.issue&&p.issue!=="none"?`<span style="font-size:10px;color:var(--color-text-secondary)">${p.issue}</span>`:""}
      <span style="font-weight:500;color:${sc(p.score)}">${p.score}/100</span>
    </div>
  </div>`).join(""):`<div style="font-size:12px;color:var(--color-text-secondary);padding:1rem 0;text-align:center">No prompts yet</div>`;

  const im={};
  log.filter(p=>p.issue&&p.issue!=="none").forEach(p=>{if(!im[p.issue])im[p.issue]={c:0,ex:[]};im[p.issue].c++;if(im[p.issue].ex.length<1)im[p.issue].ex.push(p.preview);});
  const top=Object.entries(im).sort((a,b)=>b[1].c-a[1].c).slice(0,4);
  document.getElementById("iss").innerHTML=top.map(([nm,d],i)=>`
  <div class="ir">
    <span class="bdg" style="background:${RC[Math.min(i,3)][0]};color:${RC[Math.min(i,3)][1]}">#${i+1}</span>
    <div style="min-width:0;flex:1">
      <div style="font-size:13px;color:var(--color-text-primary)">${nm} <span style="font-size:11px;color:var(--color-text-secondary)">${d.c}×</span></div>
      <div style="font-size:11px;color:var(--color-text-secondary);margin-top:2px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">"${d.ex[0]}"</div>
      ${FX[nm]?`<div style="font-size:11px;color:var(--color-text-secondary);background:var(--color-background-secondary);padding:4px 8px;border-radius:var(--border-radius-md);margin-top:4px">${FX[nm]}</div>`:""}
    </div>
  </div>`).join("");

  document.getElementById("dims").innerHTML=DIMS.map(d=>{
    const related=DIM_ISSUES[d]||[];
    const flagged=log.filter(p=>related.some(r=>p.issue&&p.issue.toLowerCase().includes(r))).length;
    const pts=N?Math.round(20-(flagged/N)*20):20;
    const c=pts>=16?"#3B6D11":pts>=10?"#854F0B":"#A32D2D";
    const bg=pts>=16?"#EAF3DE":pts>=10?"#FAEEDA":"#FCEBEB";
    return `<div class="dr"><div class="dh"><span style="font-size:12px;color:var(--color-text-primary)">${d[0].toUpperCase()+d.slice(1)}</span><div style="display:flex;gap:6px;align-items:center"><span style="font-size:11px;color:var(--color-text-secondary)">${pts}/20</span><span style="font-size:10px;padding:1px 6px;border-radius:var(--border-radius-md);background:${bg};color:${c}">${pts>=16?"Good":pts>=10?"Needs work":"Critical"}</span></div></div><div class="bt"><div class="bf" style="width:${Math.round(pts/20*100)}%;background:${c}"></div></div></div>`;
  }).join("");
}

init();
</script>
```

---

### Mode B — On-demand snapshot (warn first)

Before building, show this warning:

```
⏱️ On-demand dashboard — this will take 60–90 seconds to generate since I need to re-scan all [N] prompts and hand-draw every chart from scratch. 

Starting now...
```

Then proceed with the full static HTML + SVG build below.

**CRITICAL INSTRUCTIONS for Mode B:**
1. Use the `show_widget` / Visualizer tool — NOT a file artifact
2. Pure static HTML + inline SVG — NO JavaScript, NO external libraries, NO CDN
3. All charts hand-drawn SVG — rectangles for bars, polylines for trend lines
4. Hardcode all values directly into HTML as static text

### Step 1 — Compute these values from conversation history

Scan every user message (skip casual chat like "thanks", "ok", "yes"):
- Count words × 1.3 = tokens_used per prompt
- Score across 5 PE dimensions → overall score
- Identify top dimension violated and sub-issue
- tokens_optimized = lean rewrite word count × 1.3

Then compute session totals:
- avg_score = mean of all scores
- total_used = sum of all tokens_used
- total_optimal = sum of all tokens_optimized
- total_saved = total_used − total_optimal
- waste_pct = round((total_saved / total_used) × 100)
- dollar_wasted = total_saved × 0.000003 (Sonnet rate)
- dollar_monthly = dollar_wasted × 30
- context_pct = round(total_used × 4 / 2000) — rough % of 200k window

### Step 2 — Dashboard layout (pure HTML + SVG, no JS)

Build these sections top to bottom, all visible at once (no tabs — tabs require JS):

**Section 1: Header**
- Title "Prompt coach" + subtitle "[N] prompts · avg [X]/100 · [Y]% waste"

**Section 2: KPI cards (6 cards in 3×2 grid)**
- Avg score (color-coded green/amber/red)
- Prompts analyzed + issues count
- Tokens used / optimal
- $ wasted + monthly projection
- Waste % + delta vs last session
- Context window %

**Section 3: Score trend (SVG polyline)**
- viewBox="0 0 600 120"
- Draw a polyline connecting score points — map score 0-100 to Y axis (0=bottom, 100=top)
- Color dots by score: green ≥75, amber 50-74, red <50
- Draw dotted green line at y=85 (target)
- Label every 5th prompt on X axis

**Section 5: Top issues (3 rows)**
- Ranked list with colored badge (#1 red, #2 amber, #3 green)
- Issue name + count + example prompt + fix tip — all visible, no expand needed

**Section 6: PE scorecard (5 rows)**
- One row per dimension with pts/20, status badge, and progress bar
- All rendered as static HTML divs with inline width styles

**Section 7: Best & worst prompt (2 cards side by side)**

### Color reference (hardcode these hex values):
- Good (≥75): text #3B6D11, bg #EAF3DE
- Needs work (50-74): text #854F0B, bg #FAEEDA  
- Critical (<50): text #A32D2D, bg #FCEBEB
- Token bar used: #B4B2A9
- Token bar optimal: #639922
- Trend line: #378ADD

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
