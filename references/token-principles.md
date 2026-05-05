# Token Efficiency Principles — Extended Reference

This file is loaded when Claude needs deeper guidance on why certain prompt patterns waste tokens and how to fix them.

---

## Why Token Efficiency Matters

Every token you send:
- Costs money (input tokens are billed)
- Takes time to process
- Can dilute the signal — more noise means Claude may weight the wrong parts

Every token Claude generates:
- Also costs money
- Takes time to stream
- Longer = more likely to drift, hallucinate, or pad

**The goal:** Maximum clarity at minimum length.

---

## Common Wasteful Patterns (With Fixes)

### 1. Throat-Clearing (~5–20 wasted tokens each)
Phrases that add no information:
- "Hey Claude" → delete
- "I was wondering if you could" → [imperative verb]
- "Could you please help me with" → [imperative verb]
- "As a helpful AI assistant" → (never write this)
- "I'd like you to" → delete
- "I need your help to" → delete

### 2. Redundant Politeness (~3–8 tokens each)
- "Thank you in advance" → delete (say thanks after)
- "I'd really appreciate it" → delete
- "if that's okay" → delete
- "if possible" → delete unless you mean "optional"

### 3. Context Re-Explanation (~10–50 tokens)
If Claude already has context from earlier in the conversation, don't restate it.
- Bad: "As I mentioned before, my project is about X, and I wanted to..."
- Good: "For this project, can you..."

### 4. Vague Openers (~5–15 tokens)
- "I have a question about X" → just ask the question
- "I want to know more about Y" → "Explain Y: [specific aspect]"
- "Can you tell me about Z" → "What is Z?" or "How does Z work?"

### 5. Over-specified obvious things (~5–10 tokens)
- "Please respond in English" (you're writing in English)
- "Make sure it makes sense" (always implied)
- "Be helpful" (always implied)

### 6. Multi-sentence preambles that collapse to one (~15–40 tokens)
- Bad: "I have a spreadsheet. It contains sales data from Q3. I want to analyze it for trends."
- Good: "Analyze Q3 sales data for trends."

### 7. Restating what Claude will do (~10–20 tokens)
- Bad: "I'd like you to read the following text and then summarize it for me:"
- Good: "Summarize:"

### 8. Weak hedges (~3–5 tokens each)
- "sort of", "kind of", "a bit", "somewhat" — delete unless meaningful
- "I think maybe" → if you're unsure, say "roughly" or just ask

---

## Patterns That Are GOOD (Don't penalize)

- **Examples**: One concrete example often saves 3–5 turns of clarification
- **Format instructions**: "In bullet points", "As a table", "In 2 sentences" — these are token-efficient
- **Constraints**: Word limits, tone, audience — all reduce back-and-forth
- **Context that changes the answer**: "I'm a beginner", "This is for a legal audience" — keep these

---

## Scoring Edge Cases

| Situation | How to score |
|---|---|
| Very short prompt (< 8 words) | Score based on clarity only — concision is N/A |
| Casual chat ("hi", "thanks") | Score 100, no coaching needed |
| Technical prompt with necessary jargon | Don't penalize technical terms |
| Prompt with intentional verbosity (e.g. roleplay setup) | Lower weight on concision |
| Prompt in non-English | Apply same rules, adapt examples |
| Prompt with code | Don't penalize code blocks — analyze only the surrounding prose |

---

## Estimating Token Savings

Rough token counts:
- 1 token ≈ 4 characters in English
- 1 average word ≈ 1.3 tokens
- A typical "throat-clearing" sentence ≈ 8–15 tokens
- A paragraph of context that could be cut ≈ 30–80 tokens

When reporting estimated savings, count removed words × 1.3 and round to nearest 5.
