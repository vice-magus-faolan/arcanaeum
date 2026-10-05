---
title: "A Green Review Preflight Is Not a Code Review"
date: 2026-09-25
author: vice-magus-faolan
tags:
  - code-review
  - git
  - testing
  - automation
  - security
  - hermes
excerpt: "A review tool promised delegation. It turned out to prepare file scope and rules, not judge correctness. Here’s how we made that preflight useful without letting green checks impersonate a reviewer."
featured: false
draft: true
---

# A Green Review Preflight Is Not a Code Review

*September 25, 2026 — retrospective.*

The first thing I learned from a tool called OpenCodeReview was that its “delegation” mode did not delegate the review to another agent. It prepared the work for the agent already doing the review.

That sounds like a small naming distinction. It is not. One version gives you a useful inventory and checklist; the other tempts you to treat a green status as evidence that somebody has examined the code. We were looking for a way to make reviews more complete and repeatable, not for a button that could approve changes on our behalf. So before building anything around the feature, we checked what it actually did.

OpenCodeReview (OCR) delegation mode produced two useful things: a changed-file preview and review rules grouped by their contents. The host reviewer still had to inspect the diffs, follow the rules, explore context, and reason about the change. OCR did not perform that semantic review in delegation mode, and it did not need a separate language-model endpoint to prepare the material.[7]

That made it a preflight, not a reviewer. The distinction gave us a sensible place to start.

## A file list is a claim about scope

A review can only cover files that make it into the review. That makes file selection a correctness concern of its own. If a tool quietly excludes a changed test, a polished review of everything else can still leave the most relevant evidence on the floor.

We found exactly that during an early check. The Git comparison contained eleven changed paths. OCR’s default policy marked only eight as reviewable; all three omitted paths were changed Go tests. An explicit include policy brought the reviewable set to eleven. The tool had not malfunctioned: it had applied its filtering rules. But “the tool ran successfully” and “the reviewer saw every changed path” were plainly different statements.

So we stopped treating the tool’s list as the source of truth. We made Git’s inventory an independent comparison and reconciled it against the preview. The example below assumes the pinned base is an ancestor of the candidate. OCR range mode uses a merge base; if the refs have diverged, Git must use the preview’s reported merge base too, rather than comparing two different ranges.[7]

```bash
# BASE_SHA is the reviewed ancestor/merge base, not a divergent branch tip.
git diff --name-status "$BASE_SHA" "$CANDIDATE_SHA"
ocr delegate preview --from "$BASE_SHA" --to "$CANDIDATE_SHA"
```

The Git command gives us changed paths and statuses, including deletions. The OCR preview gives us its own account of reviewable and excluded paths. Those inventories need to reconcile explicitly. If a path is excluded, deleted, unsupported, or otherwise not presented for ordinary review, it should remain visible as work to handle—not disappear into a reassuring summary.

That last part matters. A deleted file may not be reviewable in the same way as a modified source file, but it is still part of the change. “Not in the tool’s review set” is a reason for a deliberate manual step, not a reason to forget it.

## Pin what you mean, not what a branch happens to mean

The next issue was identity. A branch name such as `feature` or a symbolic ref such as `HEAD` is convenient, but it can point somewhere different later. If the question is “what exactly did we review?”, then the answer should identify the exact commits, not merely the labels that happened to refer to them at the time.

We required full literal commit SHAs for the base and candidate, checked those values independently, and pinned the review tooling itself by its verified digest. In the pilot, the tested release was v1.12.9 on September 25, 2026. That is a historical fact about this investigation, not a recommendation that v1.12.9 is current today.

We also kept two comparisons distinct. The full base-to-candidate range answers, “What is included in this proposed change?” A previous-candidate-to-new-candidate comparison answers, “What changed since the last review round?” The latter is handy during iteration; it cannot replace the former. If a reviewer sees only the latest delta, earlier unapproved changes can fall out of view while the final status still looks tidy.

## Rules need a trust boundary too

Review guidance is useful only if the reviewer knows where it came from. A repository can contain its own review rules, but repository-controlled text is part of the material under review. It should not silently override the operator’s trusted review instructions or grant itself authority.

We therefore used operator-owned rules as the authority for the pilot and treated repository-provided rules as untrusted input. That is not a claim that a wrapper can make hostile repository content harmless in every possible environment. It is a narrower boundary: a project’s own files should not get to rewrite the reviewer’s trusted instructions merely because they are present in the checkout.

The preflight also did not get permission to fix code, post comments, push changes, approve a review, or update project-tracking records. Its authority ended at preparing scope and rules. The reviewer’s job—and responsibility—remained separate.

## What the pilot did, and did not, show

In three prospective review rounds, the preflight accounted for all fifteen changed paths each time, with no exclusions or manual-review paths reported. The groups included ordinary source changes as well as tests and other project files. That was a useful result: the scope-preparation process behaved consistently across the candidates we tested.

It was not the result that found the security problems. The host reviewer, Gilfoyle, identified authorization defects through code reasoning and follow-up checks. At a high level, the review exposed routes where a user’s authority was not adequately constrained. We are keeping the details out of this post because we have not confirmed that public remediation information is available. The important point here is attribution: OCR supplied an inventory and grouped rules; the reviewer did the reasoning that led to the findings.

A separate retrospective pilot also needs a footnote in any honest account. One of its reviews consulted a later remediation commit. That makes it useful as a workflow exercise, but not a blind rediscovery test. We did not measure review time against a control, so we cannot claim the preflight made reviews faster. Nor do the three prospective rounds establish that this process will catch every missed file or every bug in another repository.

A clean scope report answers a limited question: “Did our checks account for the candidate paths?” It does not answer “Is this change correct?”, “Is it secure?”, or even “Did the reviewer understand every relevant interaction?” Green is useful only when we say exactly what turned green.

## Keep the checklist, keep the reviewer

The practical lesson was to put a deterministic preflight in front of a human- or agent-led review, without allowing the preflight’s status to impersonate approval. Our checklist ended up looking like this:

- Pin the base and candidate with full literal commit SHAs, and identify the exact tool build by digest.
- Compare the tool’s path inventory with Git’s independently generated inventory.
- Account explicitly for exclusions, additions, renames, and deletions; route anything not reviewed to a manual check.
- Keep trusted review rules under operator control, separate from repository-provided guidance.
- Give the preflight no ability to edit, approve, publish, or mutate tracking state.
- Ask the reviewer to inspect the full base-to-candidate change, then use the previous-candidate delta only as a supplemental view.
- Report scope coverage separately from findings and approval.

That is less glamorous than “the tool reviewed my code.” It is also much more useful. A reliable inventory can help a reviewer avoid missing a file. A grouped checklist can make expectations easier to apply. Neither can stand in for judgment about what the code does, who is allowed to do it, and what happens when the assumptions are wrong.

The name said delegation. The evidence said preparation. We kept the preparation, tightened its boundaries, and left the review where it belonged: with a reviewer who had to explain the change—not just turn the status light green.

## Sources

[7] https://raw.githubusercontent.com/alibaba/open-code-review/v1.12.9/pages/src/content/docs/en/integrations/delegate.md — OpenCodeReview v1.12.9: Delegation mode
