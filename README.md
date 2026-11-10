# Story Walkthrough

Your agent found the root cause. Now someone who was not there has to understand it.

Story Walkthrough is an agent skill that explains a bug, an incident, or any system that keeps getting stuck as a story told from the beginning. The agent introduces each actor once, gives each one the rule it follows, and then tells what happens in time order until the failure makes sense. The reader can be a teammate who is new to the code, a reviewer, a manager, or a support lead.

It works with Claude Code, Codex, and other agents that read `SKILL.md` files.

## Before and after

### A developer problem: customers charged twice

> Customers say we charged them twice on Black Friday. I do not know the payments code. Walk me through what happened, from the beginning.

**Without the skill**, a typical answer:

> **Root cause:** `apiClient` retries on 502/503/504 for all methods, and `POST /payments` is not idempotent. When processor latency exceeded the gateway's 30s upstream timeout, the gateway returned 504 while `create_charge()` was still in flight. The client retry passed the `order.status` check (first charge not yet persisted) and created a second charge.
>
> **Fix:** Restrict retries to idempotent methods, add `Idempotency-Key` support to `/payments`, and persist the key before calling the processor.

Every word is correct. It helps the person who ran the investigation. The support lead, the product manager, and the developer who joined last month stop reading at "not idempotent".

**With the skill**, the same facts as a story (excerpt):

> **The characters**
>
> **The retry.** Every request from the app goes through the shared HTTP client, `apiClient` in `mobile/src/api/client.ts:58`. If a request fails with a 502, 503, or 504, the client sends it again after one second, up to two times. Its rule: *"A 504 means the request did not go through, so sending it again is safe."*
>
> **The API gateway, the waiter.** Think of the backend as a restaurant. The gateway is the waiter: it carries each request from the app to the right service and brings the answer back. The waiter waits at most 30 seconds (`timeout: 30s` in `gateway/routes.yaml:12`). Its rule: *"If the kitchen has not answered in 30 seconds, I go back and say 504 Gateway Timeout."*
>
> **The payment service, the kitchen.** `create_charge()` in `payments/charge.py:41` prepares the dish: it checks the order, asks the bank to charge the card, and records the charge. Its rules: *"I charge an order only if it is not paid yet. Once I start a charge, I finish it, whether anyone is still waiting or not."*
>
> **Chapter 3: The waiter gives up**
>
> At 10:02:30 the gateway stops waiting and answers the app with `504 Gateway Timeout`. It has no way to tell the kitchen to stop, so the payment service keeps waiting for the bank.
>
> From the app's side, the payment failed. **From the kitchen's side, the payment is still going ahead.**
>
> **Chapter 4: The same order, twice**
>
> **The retry** sees the 504 and follows its rule. At 10:02:31 it sends the same `POST /payments` for order 7731 again, and the waiter carries it to the kitchen.
>
> **The kitchen** gets what looks like a new order. It checks order 7731: still not paid, because the first charge is still in the oven and nothing is recorded until the oven is done. **The "already paid" check passes, so the kitchen starts a second charge.**
>
> **Why only on Black Friday, and almost only from the app**
>
> On a normal day the bank answers in about 2 seconds, so the waiter never gives up and the retry never runs. [...] The web checkout does not use `apiClient`. Its `fetch` call in `web/src/checkout/pay.ts:22` does not retry. [...] That matches the support tickets: **212 double charges, all between 10:00 and 13:00, and 209 of them from the app.**

[Read the full story](examples/charged-twice.md): why it started two months ago, which impacts are checked and which are not, and three fixes compared, including one that does not work.

### Not only code: new engineers without laptops

> Half of our new engineers start without a laptop. HR says IT has the request, and IT says it is waiting for approval. I do not understand where it gets stuck. Can you explain it from the start?

**Without the skill**, a typical answer:

> **Root cause:** Hardware approval routing uses the HR record's Manager field, which is empty for general-pipeline engineering hires until team matching. Those requests fall back to `hr-ops@example.com`, which nobody monitors for approvals, and the ticket system does not re-send the approval when the field changes.
>
> **Fix:** Point the fallback approver at the Engineering hiring lead, or remove the approval step for the standard engineering laptop.

Correct again, and HR and IT, who have to agree on the fix, still each believe the other one is holding things up.

**With the skill** (excerpt):

> **The hardware request, the baton.** Onboarding works like a relay race. The baton is the hardware request: it passes from HR to IT, from IT to a manager for approval, and back to IT. IT orders the laptop only when the baton completes the lap.
>
> **The Manager field.** Each HR record has a Manager field. For most roles, HR fills it at signing, because the new hire joins a known team. Its rule: *"I hold the person's manager. Until there is one, I stay empty."*
>
> **IT.** IT orders laptops from a supplier that delivers in five working days. IT's rule: *"I order a laptop only after the new hire's manager approves it, because the manager picks the model and pays for it from the team budget."*
>
> **Chapter 3: The baton lands in an empty lane**
>
> The ticket system needs an approval before IT can order. It looks at the Manager field, finds nothing, and sends the approval request to the fallback approver. **The approval goes to a mailbox where nobody looks for approvals.**
>
> **Chapter 4: Everyone waits for someone else**
>
> For three weeks, everyone follows their rule and nothing moves.
>
> - **HR** thinks: *"IT has the request. It is in their queue."*
> - **IT** thinks: *"The manager has not approved it yet."*
> - **The hr-ops mailbox** holds the approval request under 900 other emails.

[Read the full story](examples/new-hire-laptops.md): why only engineers, why it started in January, and a fix that removes the approval nobody can give.

Both systems and all numbers in these examples are invented.

## What the agent writes

- **The characters.** Every component, person, message, or piece of data the story needs, each linked to the code or the source (a document, a form, a setting), and each given the rule it follows, written as its own thought. Problems usually come from two reasonable rules colliding, and this makes the collision visible.
- **One metaphor.** The abstract mechanism gets one physical picture, such as a waiter and a kitchen, or a relay baton. The story keeps it to the end and always shows it next to the real name.
- **Chapters.** One short chapter per step, in time order, with the key fact in bold and the exact error message or status quoted.
- **Why the symptom looks the way it does.** Why only on busy days, only on some servers, only for some people. This section checks the story against what the logs, tickets, or surveys showed.
- **Why it started now.** The change that triggered it, and why that change made sense on its own.
- **How bad it is.** Each impact marked as checked (and how) or not checked, because the reader will repeat these claims to others.
- **How to fix it.** Options from smallest to largest, each described in story terms, with a recommendation.

The skill also covers risks found in a code review that have not failed yet: it states the exact conditions for the failure and walks through why nothing hits it today.

The skill carries [its own calibration example](skills/story-walkthrough/references/example-stale-prices.md), a stale-cache bug with notes on what the first draft got wrong.

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

The CLI asks which agents to install for. To pick them up front, pass `-a`:

```bash
npx skills add OkosMark/story-walkthrough-skill -a claude-code
```

```bash
npx skills add OkosMark/story-walkthrough-skill -a codex
```

Add `-g` to install for your user instead of the current project.

Or copy the folder by hand into Claude Code's skills folder:

```bash
git clone https://github.com/OkosMark/story-walkthrough-skill.git
cp -r story-walkthrough-skill/skills/story-walkthrough ~/.claude/skills/
```

The skill is a plain `SKILL.md` file in the Agent Skills format, so it works with Claude Code, Codex, and other agents that read that format.

## License

[MIT](LICENSE)
