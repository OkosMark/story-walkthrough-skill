# Story Walkthrough

An agent skill that explains a bug, incident, race condition, or confusing system behavior as a story told from the beginning, for a reader who does not know the code.

A list of findings helps the person who did the investigation. It does not help a teammate, a reviewer, or a manager who has never opened the code. This skill makes the agent introduce each actor once, then tell what happens in time order, until the failure follows naturally.

## What the agent writes

- **The characters.** Every component, message, timer, or piece of data the story needs, each linked to the code and given the rule it follows, written as its own thought: *"Today is whatever date my clock shows."* Bugs usually come from two reasonable rules colliding.
- **One metaphor.** The abstract mechanism gets one physical picture (a cache is a whiteboard, a header is a note stapled to an envelope), kept for the whole story and always shown next to the real name.
- **Chapters.** One short chapter per step, in time order, with the key fact in bold and the exact error message quoted.
- **Why the symptom looks the way it does.** Why only after midnight, only on some servers, only after a restart. This section checks the story against what the logs or tickets showed.
- **Why it started now.** The change that triggered it, and why that change made sense on its own.
- **How bad it is.** Each impact marked as checked (and how) or not checked.
- **How to fix it.** Options from smallest to largest, each described in story terms, with a recommendation.

The skill also covers risks found in code review that have not failed yet: it states the exact conditions for the failure and walks through why nothing hits it today.

See [the example](skills/story-walkthrough/references/example-stale-prices.md) for a full story about a stale-cache bug, with notes on what the first draft got wrong.

## When it triggers

The agent loads the skill when you ask to be walked through an issue, for example:

- "Walk me through this bug like a story."
- "Start from the beginning, I'm not familiar with this code."
- "I don't get what's happening here. How does the failure happen?"

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add OkosMark/story-walkthrough-skill
```

Or copy the folder by hand:

```bash
git clone https://github.com/OkosMark/story-walkthrough-skill.git
cp -r story-walkthrough-skill/skills/story-walkthrough ~/.claude/skills/
```

The skill is a plain `SKILL.md` file, so it also works with other agents that read the Agent Skills format.

## License

[MIT](LICENSE)
