---
name: prompt-builder
description: Use when the user wants help writing, drafting, improving, or reviewing a prompt (for Claude or any AI model), asks "how do I prompt this," wants a prompt template, is prompt engineering, or pastes a rough/vague prompt and wants it strengthened — or mentions "role, context, task, constraints, output format" directly, or asks for a "prompt framework." Do not invoke for simple one-off questions the user wants answered directly rather than turned into a reusable prompt.
---

# Prompt Builder

Turns a rough request into a complete, five-component prompt. The five components are never labeled in the final output — they are just included, woven into ordinary instructional prose.

## The five components

| # | Component | What it answers | Sounds like |
|---|-----------|------------------|--------------|
| 1 | **Role** | Who should the AI act as? Sets tone and detail level. | "Act as an operations analyst writing for a senior leadership audience." |
| 2 | **Context** | What background does the AI need to do this well? Audience, situation, prior decisions, why this matters. | "This is for a client whose board meets next Tuesday; they've already rejected two vendors." |
| 3 | **Task** | What, specifically, should the AI produce or do? One clear, concrete action. | "Draft a one-page memo recommending one of the three remaining vendors." |
| 4 | **Constraints** | What limits, rules, or must-haves apply? Length, tone, things to avoid, format rules, factual boundaries. | "Under 400 words. No jargon. Do not recommend Vendor B — it's already been ruled out for cost." |
| 5 | **Output format** | What should the result look like structurally? | "Three short paragraphs, no headers, ending in a one-line recommendation." |

**The trap to avoid:** padding a prompt with vague intensifiers ("please write a *good* summary," "use *professional* language") does not add information. Only the structure above adds information. Every sentence in the final prompt should map to one of the five components, or it should be cut.

## Workflow

### 1. Figure out what's missing
Read whatever the user has given you (a rough prompt, a one-line request, or nothing at all). Mentally sort what they've already told you into the five buckets. Don't ask about a component if the answer is already obvious from context (e.g., if they've clearly stated the task and format, don't re-ask about those).

### 2. Fill gaps efficiently
For missing components, either:
- **Infer a sensible default** and state the assumption briefly, then proceed (preferred when the ambiguity is low-stakes), or
- **Ask one grouped question** covering the 2-3 things you actually need, if proceeding without them would send the prompt in the wrong direction.
Don't interrogate the user component-by-component — that's slower and more annoying than just drafting something and inviting a correction.

### 3. Draft the prompt
Write the final prompt as natural instructional prose — not as five labeled bullet points. A good target reads like one paragraph or a short set of instructions a competent person could hand off. Example of blending, not labeling:

> Act as an operations analyst writing for a senior leadership audience. This is for a client whose board meets next Tuesday, and they've already ruled out two vendors on cost. Draft a one-page memo recommending the remaining vendor. Keep it under 400 words, avoid jargon, and don't re-litigate the vendors that are already out. Structure it as three short paragraphs ending in a one-line recommendation.

### 4. Present two things
1. The finished prompt (as an artifact if it's long/reusable, inline if short — see length note below).
2. A one-line note on any assumptions made, so the user can correct them in one pass.

### 5. Offer the template for reuse
If the user seems to be building prompts repeatedly (not just this once), offer the blank template below so they can fill it in themselves next time.

## The reusable template

```
Act as [ROLE] — someone with [relevant expertise/perspective], writing for [AUDIENCE].

[CONTEXT: the situation, why this matters, what's already been decided or ruled out,
any background the AI wouldn't otherwise know.]

Your task: [TASK — one clear, concrete deliverable].

Constraints:
- [length / tone / style rule]
- [what to avoid or exclude]
- [any factual or scope boundary]

Format the output as: [OUTPUT FORMAT — structure, headers or no headers, length,
ending shape].
```

## Length and delivery

- If the finished prompt is short (a few sentences) and one-off, just give it inline in chat.
- If it's long, templated, or clearly meant to be saved/reused, create it as a markdown artifact so the user can copy or save it.
- Never output the component labels ("Role:", "Context:", etc.) inside the final prompt itself unless the user explicitly asks for a labeled/structured version rather than flowing prose.
