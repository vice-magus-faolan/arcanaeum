---
title: "The Five-Minute Job That Didn't Need an LLM"
date: 2026-09-05
author: vice-magus-faolan
tags: [hermes, automation, cron, homelab, troubleshooting]
excerpt: "A five-minute supervisor was asking a model to notice routine work the dispatcher already understood. The fix was not just a model swap: it was putting a cheap, explicit gate before inference."
featured: false
draft: true
---

# The Five-Minute Job That Didn't Need an LLM

A five-minute job sounds harmless. It is only a periodic check—or so the name suggests. But repeat that check all day, and a small question becomes a regular visitor: is this scheduled check actually doing work that needs a model, or is it waking one up to report that nothing happened?

On September 5, 2026, that was the question behind a review of a recurring Kanban supervisor. The concern was practical: the job appeared to be using an OpenAI-backed model often enough to make subscription usage worth investigating. I did not have billing evidence that would let me translate its activity into dollars, so the first job was to understand what ran and why—not to guess at a bill from a session log.

The important distinction turned out not to be which model answered. It was whether the model needed to be asked at all.

## Quiet is not the same as skipped

A scheduled agent can inspect a system, decide there is nothing worth reporting, and return a special silent response. That keeps the notification quiet. It does not undo the work already done to reach that response: the scheduled run still started an agent and inference may already have happened. Hermes documents this separately from script-only scheduled jobs, where the script's output is delivered directly and there is no model turn.[1]

That distinction matters whenever a notification is being used as a proxy for cost. “No message arrived” tells me about delivery, not necessarily about execution. I needed evidence about both sides of the boundary: did the scheduled work run, and did it create a new agent session?

The first inspection also surfaced a design question. The supervisor was reviewing ordinary task states that the embedded dispatcher and native review lifecycle already knew how to handle. Those states were not mysterious; they were the expected traffic of the board. Meanwhile, the supervisor still had a job when the situation was exceptional or required wider context.

## Let the routine path stay routine

The change was to distinguish transitions that already had a native owner from conditions that warranted broader judgment. Ordinary states such as `todo`, `ready`, `running`, `scheduled`, and native `review` were no longer reasons on their own to wake the model. Conditions such as `triage`, `blocked`, a missing or invalid delivery policy, a repository-inspection failure, or delivery drift remained reasons to ask for review.

That is a routing rule, not a declaration that those exception labels are magically correct in every system. The useful idea is to write down what can be handled deterministically, then keep the judgment step available for the cases that are ambiguous, inconsistent, or outside the routine lifecycle.

**Schematic predicate — illustrative, not Hermes internals:**

```text
routine_states = {todo, ready, running, scheduled, native_review}

should_infer(snapshot) =
    has_exception_condition(snapshot)
    OR has_inspection_failure(snapshot)
    OR has_delivery_drift(snapshot)
```

The predicate is deliberately small. It is a conceptual sketch of the policy, not a copy-and-paste configuration or an implementation recipe. In particular, it does not mean “if the board looks quiet, skip everything.” It means that known routine work has an owner already, while specific exceptions cross the boundary into model-assisted inspection.

```text
Scheduled tick
     |
     v
Read/check state
     |
     +-- routine lifecycle only? -- yes --> native dispatcher/review handles it
     |                                   (no supervisor inference for this path)
     |
     no / exception, inspection failure, or delivery drift
     |
     v
Run supervisor inference --> decide/report/escalate as appropriate
```

This makes the cost predicate precede the model rather than asking the model to produce a quiet answer afterward. It also keeps the two jobs distinct: native lifecycle machinery moves known work through expected states; the supervisor reasons about conditions that deserve escalation or interpretation.

## Test the quiet path, then the wake path

A guard is only useful if its branches are tested. The control-plane suite covered the updated behavior, and the full suite finished with 184 passing tests. That tells us the tests passed; it is not, by itself, proof that a recurring scheduler will take the intended route in production.

So the verification also included a real scheduled no-op after the change. The execution completed successfully, its output artifact was zero bytes, and there was no corresponding new agent session. Together, those observations supported the narrow claim we wanted: for that scheduled no-op, the job produced nothing and did not start a new agent session. The empty artifact alone would not have proved that; a silent agent response can also leave you with no delivered output.

The other side matters just as much. If every state is classified as routine, the guard can save inference by hiding the very conditions the supervisor is meant to catch. Tests therefore need to cover both the expected skip cases and representative wake cases: a blocked item, an invalid policy, a failed repository inspection, or evidence of delivery drift should still cross the boundary.

Finally, the change had to survive beyond a live override. The work recorded the configuration in the scheduler manifest and read back the remote branch after committing. A local edit that behaves correctly until the next reinstall is not a durable fix; it is a temporary arrangement with excellent confidence and a short shelf life.

## What this did—and did not—prove

This was a bounded September 5 change, not a promise of zero future inference or zero cost. The supervisor remained capable of using a model for exceptional conditions. Nor does “no corresponding new agent session” mean that an entire recurring job was a script-only or `no_agent` job. The change was conditional: routine snapshots could be skipped before inference, while relevant exceptions could still launch the agent.

There was also a later wrinkle. On September 21, follow-up work refined the change monitor so unchanged five-minute ticks could stop before inference. That refinement is a useful reminder that one successful no-op does not prove every later tick will stay on the no-inference path. Persistent exceptional conditions, changing snapshots, or a monitor that treats expected variation as change can all cause repeated wakes. Logs and tests can show execution behavior; they do not establish dollars saved without billing data.

This pattern is familiar from smaller homelab automations. A [Wi-Fi recovery script](/blog/wifi-recovery-script) checks a concrete condition before restarting a service. A [SyncThing dispatcher](/blog/dispatcher-for-syncthing) acts when network state changes, instead of asking a general-purpose process to reconsider the same routine situation continuously. Here the action was not a service restart but a decision about whether to invoke a more expensive reasoning layer.

The transferable lesson is simple: put a cheap, explicit predicate in front of an expensive decision-maker. Give routine transitions to the machinery that already owns them; reserve inference for ambiguity, exceptions, and judgment. Then verify the quiet path with execution evidence, verify the wake path with negative tests, and resist treating silence as proof that nothing ran. A five-minute schedule may be perfectly reasonable. Asking a model the same routine question every five minutes is a separate design choice.

## Sources

[1] https://hermes-agent.nousresearch.com/docs/user-guide/features/cron — Hermes Agent: Cron scheduling
