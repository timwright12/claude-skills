---
name: detecting-weak-prompts
description: Use when the user pastes text clearly addressed to an AI model — starts with "You are...", "Act as...", or gives direct imperative instructions to a model — and it's obviously missing role, constraints, or output format, even if they didn't ask for help improving it. Do not use for text addressed to a person, task requests directed at the assistant itself, code/config, or prompts that already look complete.
---

# Detecting Weak Prompts

## Overview

Catches prompts that are clearly meant for an AI but are obviously missing a basic component, and offers once to strengthen them — without turning every pasted block of text into an interruption.

**Core rule:** offer exactly once per piece of text, then drop it regardless of the answer. Always answer whatever the user actually asked, independent of the offer.

## Is this prompt text?

Only treat text as a candidate if it's addressed to a model, not a person or to you:

- **Looks like this:** "You are a...", "Act as...", "Your task is...", second-person imperative instructions written as if a model will execute them.
- **Not this:** a request directed at you ("summarize this," "fix this bug"), text addressed to a human ("tell your boss..."), code, config, or a document being discussed.

## What counts as missing

Thin means it's clearly lacking at least one of: a defined role, explicit constraints (length/tone/scope), or a stated output format. Short isn't automatically thin — "Explain quantum computing in two sentences for a 10-year-old" already has a constraint and an implicit format; it doesn't need flagging.

## Workflow

1. **Detect** — apply the two checks above. If it's not AI-directed, or it's not obviously thin, do nothing — go straight to answering normally.
2. **Offer once** — name the specific gap, one line: *"That prompt doesn't specify a role or output format — want me to tighten it up?"* Don't start drafting yet.
3. **Answer the actual request** regardless of their answer to the offer — if they pasted the prompt for feedback, give feedback either way.
4. **On yes** — hand off to the `prompt-builder` skill to do the actual drafting. Don't duplicate its five-component workflow here.
5. **On no, or no response to the offer** — drop it. Never re-offer on that same text again, even if it comes back up later in the conversation.

## Common mistakes

| Mistake | Fix |
|---|---|
| Flagging a prompt addressed to a person ("tell him to clean the garage") | Check who it's addressed to before flagging — only AI-directed text counts. |
| Flagging an already-complete prompt because it's short | Thin means missing a component, not low word count. |
| Re-offering after a decline | One offer per piece of text, ever. |
| Treating "summarize this document" as a weak prompt to fix | That's a task request directed at you, not a prompt draft — just do the task. |
| Drafting the improved prompt yourself instead of handing off | This skill only detects and offers; `prompt-builder` does the drafting. |
