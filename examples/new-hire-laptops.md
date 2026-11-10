# Example: new engineers without laptops

The user asked: "Half of our new engineers start without a laptop. HR says IT has the request, and IT says it is waiting for approval. I do not understand where it gets stuck. Can you explain it from the start?" The company, its tools, and its numbers are invented for this example. The story below is the agent's answer.

---

Here is the whole story from the start, without assuming you know how onboarding works here.

## The characters

**The hardware request, the baton.** Onboarding works like a relay race. The baton is the hardware request: it passes from HR to IT, from IT to a manager for approval, and back to IT. IT orders the laptop only when the baton completes the lap.

**HR.** When a candidate signs the contract, HR creates the person's record in the HR system. That record creates a hardware request in the IT ticket system (onboarding checklist, step 2). HR's rule: *"I tell IT as soon as the contract is signed, so they have three weeks to get a laptop ready."*

**The Manager field.** Each HR record has a Manager field. For most roles, HR fills it at signing, because the new hire joins a known team. Its rule: *"I hold the person's manager. Until there is one, I stay empty."*

**IT.** IT orders laptops from a supplier that delivers in five working days. IT's rule: *"I order a laptop only after the new hire's manager approves it, because the manager picks the model and pays for it from the team budget."*

**The approval rule.** The IT ticket system sends each hardware request to the person in the Manager field for approval. When the field is empty, it sends the request to the fallback approver, the `hr-ops@example.com` shared mailbox (IT ticket system, Approvals → Hardware → Fallback approver).

**The hr-ops mailbox.** A shared inbox that receives about 300 automated emails a week. Nobody on the HR operations team knows that approval requests can land there.

## Chapter 1: The contract is signed

On 3 March, Sam signs a contract as a backend engineer, starting on 24 March. Sam is not hired into a team. Engineers hired through the general pipeline meet several teams in their first week and pick one then.

HR creates Sam's record the same day. **The Manager field is empty, because Sam has no team yet.**

## Chapter 2: HR passes the baton

The new record creates hardware request HW-2291 in the IT ticket system. HR's part is done, and the checklist shows step 2 as complete. With three weeks of margin, HR moves on.

## Chapter 3: The baton lands in an empty lane

The ticket system needs an approval before IT can order. It looks at the Manager field, finds nothing, and sends the approval request to the fallback approver. **The approval goes to a mailbox where nobody looks for approvals.**

In IT's queue, HW-2291 shows "Waiting for approval". That looks normal to IT, because most requests wait a few days for a busy manager.

## Chapter 4: Everyone waits for someone else

For three weeks, everyone follows their rule and nothing moves.

- **HR** thinks: *"IT has the request. It is in their queue."*
- **IT** thinks: *"The manager has not approved it yet."*
- **The hr-ops mailbox** holds the approval request under 900 other emails.

Nobody's rule says to check on a request that has waited longer than a week.

## Chapter 5: Day one

On 24 March, Sam arrives and there is no laptop. The ticket says:

> HW-2291 · New hire laptop · Waiting for approval · Approver: hr-ops@example.com · Open 21 days

That week, Sam joins the Payments team, and HR fills in the Manager field. **The ticket system sends the approval request only once, when the ticket is created, so the new manager never sees it.** On day 3 Sam asks IT, IT finds the stuck approval, and the Payments manager approves it by hand. The laptop arrives five working days later, on Sam's eighth working day.

## Why only engineers, and only about half of them

Sales, Finance, and Design hire into a specific team, so the Manager field is filled at signing and the approval goes to a real manager. Engineers hired for a named team also have a manager at signing. Only engineers hired through the general pipeline, who pick a team in week one, start with an empty Manager field.

That matches the onboarding survey: **14 of the 31 engineers who started this year waited more than three days for a laptop, and all 14 came through the general pipeline. Nobody in another role waited.**

## Why it started now

In January, Engineering moved to team matching in the first week, so new hires could meet several teams before choosing one. That change made sense on its own: new engineers pick teams that fit them, and fewer of them move teams later. But before January every engineer was hired into a team, so the Manager field was never empty at signing, and no request had ever gone to the fallback approver.

## How bad it is

- **14 engineers started without a laptop (checked).** The IT ticket system shows 14 hardware requests this year with the approver `hr-ops@example.com`, and they match the 14 survey answers.
- **About 80 working days lost (checked).** The ticket dates show an average of 5.8 working days between each start date and laptop delivery.
- **9 more approvals are stuck (checked).** A search of the hr-ops mailbox for "Approval needed" finds 9 open software license requests for the same engineers.
- **Contractors may take the same path (not checked).** A different team creates contractor records, and whether that team fills the Manager field has not been checked.

## How to fix it

**The quick fix.** Change the fallback approver from `hr-ops@example.com` to the Engineering hiring lead. In relay terms: put a runner in the empty lane. The limit: one person approves every unmatched engineer, and requests still wait when that person is away.

**The proper fix.** Order a standard engineering laptop at signing, with no approval. The team manager approves only upgrades, after team matching. In relay terms: the baton no longer needs a runner who does not exist yet.

Recommendation: change the fallback approver this week, clear the 9 stuck approvals in the hr-ops mailbox, and agree on a standard engineering laptop with Finance.
