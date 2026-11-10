---
name: story-walkthrough
description: Explain a bug, defect, incident, race condition, or confusing system behavior as a story told from the beginning, for a reader who does not know the code. Use this whenever the user asks to be walked through an issue step by step, asks for it "like a story", says "start from the beginning", "explain it from scratch", "I'm not familiar with this", "I don't get what's happening", or asks how a failure happens after an investigation, even if they do not use the word "story".
---

# Story Walkthrough

The user is asking to understand an issue they have not lived inside. They do not know the components, the vocabulary, or the order in which things happen. A list of findings or a diff summary does not help them. A story does: it introduces each actor once, then shows what each one does, in the order it happens, until the failure follows naturally.

Investigate first. The story is only as good as the facts behind it, so read the code, logs, tickets, and history before writing. If the investigation already happened earlier in the conversation, reuse it rather than repeating it.

## Shape of the answer

Open with one line that sets the frame, for example "Here is the whole story from the start, without assuming you know the code." Then use these sections. Skip a section that has nothing real to say rather than padding it.

### The characters

Introduce every actor the story needs, and nothing else: components, classes, messages, timers, data. Give each one a bold name and one or two plain sentences on what it is and what it does in this story.

- **Name each character precisely, and link it to the code.** Write `price_cache_key()` with a `file:line` link, not "the key builder". A loose label makes the reader stop and ask what it means. If a character spans several pieces of code (a method plus the framework hooks around it), say so here.
- **Pick one physical metaphor for the abstract mechanism and keep it for the whole story.** For example, message headers become "notes stapled to an envelope", a cache becomes "a whiteboard", a lock becomes "a key". Introduce the metaphor next to the real name ("the price cache, a whiteboard next to each web server") so the reader can map it back to the code. Do not switch metaphors partway, and do not stack two.
- Give the rule each character follows, phrased as the character's own thought in italics: *"Today is whatever date my clock shows."* Bugs usually come from two reasonable rules colliding, and this makes the collision visible later.

### Chapters

Tell what happens in time order, one chapter per step: `## Chapter 1: A shopper looks at prices before midnight`, `## Chapter 2: The new prices arrive`, and so on. Short titles that say what happens.

- Keep each chapter short: a few sentences or a few bullets.
- Bold the key detail of the chapter, the one fact the reader must not miss ("**the import wipes the wrong corner of the whiteboard**").
- When two things happen at the same time, give that its own chapter and show both sides one after the other.
- The failure chapter quotes the exact error message as a blockquote, and says which code catches it and what it logs.

### Why the symptom looks the way it does

After the failure, explain the observed pattern: why only the first few hours, why only on some servers, why only after a restart. Tie it back to what the ticket or the logs actually showed. This is the check that the story is right, so keep it.

### When the issue is a risk, not a failure

Sometimes the story is about something that could go wrong: a review comment, a latent bug, a "what if". Nothing has failed, so there is no log line or ticket symptom to anchor the story. Tell it differently, because a reader who sees a quoted error will assume it happened.

- **Say early that it has not happened.** State in the opening line or the first chapter that this is a possible failure found by reading code, not one seen in logs or tests.
- **Spell out the conditions.** The failure chapter lists exactly what must be true for the failure to occur, as numbered conditions ("1. the operation calls another service after the scope closed, and 2. no new scope is open around that call"). Then name what would go wrong: the assert, the exception type, the line. Do not quote an error message that nobody has seen. If you name an exception, say it comes from reading the code.
- **Replace "why the symptom looks this way" with "why nothing hits it today".** Walk through each real caller or flow and show which condition it fails to meet, with links. This section carries the weight of the story, because the reader has to decide whether the risk is live.
- **Hunt for the callers you have not checked.** Writing the "how bad it is" list for a risk is where unchecked paths show up: a call made outside the usual place, a different transport, a background task. List each one as not checked rather than leaving it out, and say what evidence there is (for example, a clean test run that exercised it without tracing it).
- **Compare costs in the fix section.** A fix for a risk can bring back a real failure. If it does, say that plainly: the fix trades a risk that nothing hits for a failure that happens.

### Why it started now

If the behavior is a regression, say what changed, when, and why that change made sense on its own. Name the PR or commit in words first, with the ID in parentheses. The reader needs to know that the earlier change was not careless, only incomplete.

### How bad it is

List the real impact: data, numbers, alerts, log noise. Separate what is confirmed from what is inferred, because the reader will repeat these claims to others.

- Mark each claim as checked (and how: a test, a log query, a code path read end to end) or as not checked.
- Do not write "the data is fine" unless something verified it. A plausible claim that turns out wrong costs more trust than an honest "not checked yet".

### How to fix it

Give the options from smallest to largest (for example a quick fix and a proper fix), what each one changes in story terms ("give both characters the same calendar"), and its limits. End with a short recommendation.

## Style

- Plain words. Define every term the first time it appears, or leave it out.
- Keep code identifiers in backticks next to the metaphor, so the story stays traceable to the code.
- Prefer short paragraphs and bullets over long blocks.
- Follow the user's writing-style rules from their CLAUDE.md or AGENTS.md files.

## After the story

Expect follow-up questions about a character ("So X is the handler?"). Answer by pinning the character to exact code: list what runs, in order, with `file:line` links, and say which part does the thing that matters. If the question shows the original story named something loosely, fix the naming in your answer rather than defending it.

## Example

`references/example-stale-prices.md` holds a full story written in this style for a stale-cache bug, plus notes on what it got wrong. Read it before writing your first story in a session, to calibrate length and tone.
