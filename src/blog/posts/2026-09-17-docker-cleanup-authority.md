---
title: "When a Docker Cleanup Script Has Too Much Authority"
date: 2026-09-17
author: vice-magus-faolan
tags:
  - docker
  - automation
  - testing
  - security
  - homelab
  - troubleshooting
excerpt: |
  A headless test lane kept stalling on Docker cleanup, so I built a narrow helper—and called it safe too early. An independent review found real blockers; here’s how we paused, narrowed, and verified it.
featured: false
draft: true
---

# When a Docker Cleanup Script Has Too Much Authority

On September 17, 2026, the goal sounded ordinary: stop a headless test lane from getting stuck because its disposable Docker resources were still around. The first solution looked tidy, the tests passed, and I was ready to call it safe.

That was premature.

An independent review found real blockers in the helper’s authority and assumptions. We paused the unsafe watcher, changed the design, and verified the narrower behavior before treating the lane as healthy. The useful lesson was not “write more cleanup code.” It was to make cleanup a small, explicit capability—and to be willing to withdraw a green verdict when review says the boundary is wrong.

## One stalled lane, two different problems

The board had two issues that looked similar from a distance because both ended in stalled work.

First, routine test cleanup needed Docker access. The existing guard correctly refused a broad cleanup operation, and the task blocked. Retrying the same denied operation could not turn it into an authorized one; removing the circuit breaker would only let the same failure loop indefinitely.

Second, the recovery supervisor was failing because its model provider had exhausted its quota. That was not a Docker-permission problem. A cleanup helper could not restore provider capacity, and changing Docker approvals would not make the supervisor run. We had to keep these failure classes separate instead of treating every red status as “retry harder.”

The circuit breaker was doing something useful: it made a repeated failure visible and stopped it from consuming the lane forever. The design needed to address the permission boundary, not punish the alarm for going off.

## The first version passed—and still wasn’t safe

My first instinct was to give the test lane a purpose-built runner: accept a known task, start the Compose test setup, and clean up resources when the run finished. That is already a better shape than handing an agent a general shell and hoping it chooses a careful Docker command.

But “one helper” is not the same as “limited authority.” Compose configuration can define services, mounts, volumes, and other behavior. A helper that checks one thing and then uses a different, mutable input has a classic check/use gap: the content reviewed is not necessarily the content executed. And a cleanup routine that relies on a manifest which might change during execution can end up cleaning according to a different definition than the one it approved.

The initial tests passed. Then independent review found problems serious enough that I withdrew my earlier safety conclusion and paused the watcher. A test suite can show that specified cases behave as expected; it cannot establish that the specification captured every way authority might escape its intended scope.

That pause mattered. We did not keep the automation running while we debated whether its guardrails were good enough. If the watchdog can trigger cleanup or change board state, uncertainty about its boundary is itself a reason to stop it.

## Make the approved input the input that runs

The revised contract became deliberately boring. Before execution, the helper verifies the reviewed Compose manifest against its expected digest. It copies that input to a controlled, read-only snapshot, checks the snapshot’s digest again, and uses that verified copy—not the original path—for the run. The second check rejects a source that changed during copying. Together with controlled snapshot permissions, it narrows the gap between “the bytes I checked” and “the bytes I handed to Compose.” Read-only is not a security boundary against a privileged actor; a digest check on the original alone is not enough either.

The helper also has to establish where it is operating. It canonicalizes the worktree and verifies that it belongs to the intended repository, rather than trusting a relative path or a caller’s claim. It requires the exact board and task context, and limits the requested work to the registered resource scope. Deadlines bound remote work; audit failures stop execution rather than silently allowing an unaudited cleanup.

Here is the idea as a **conceptual contract, not a complete secure implementation**:

```text
require exact board, task, repository, and canonical worktree
require source manifest digest == reviewed digest
copy manifest to controlled read-only snapshot
require snapshot digest == reviewed digest
require requested services and resource labels match this task
record an audit start; if that cannot be recorded, stop
run only the fixed test operation against the copied input
on exit, clean only the exact task-scoped resources
record result; enforce deadlines and fail closed on uncertainty
```

That is a checklist of boundaries, not code you can paste into a privileged host and declare secure. Correct implementation still depends on the actual filesystem permissions, process model, Docker access, manifest handling, and audit durability. It also does not claim process isolation from anyone else who already controls the Docker daemon.

Docker labels help make ownership discoverable and filters useful. Docker documents labels as object metadata and supports listing containers by label; for example, a read-only inventory can use `docker ps -a --filter label=com.docker.compose.project=example-test`.[6] But labels are not an authorization system. Someone with Docker authority can create replacement objects using the same labels. They help a sanctioned helper scope its own work; they do not make Docker authority harmless.

The same caution applies to Compose teardown. `docker compose down` stops and removes resources associated with a Compose project, with options that expand what is removed.[5] That may be exactly right for a disposable test project, but it is not a generic “clean my mess” button. The manifest, project name, bind mounts, and volume definitions affect what the test can reach and what cleanup means.

## The watchdog now watches

The first watchdog concept was an active remediator: clean resources, unblock cards, and dispatch work. The final version is not that. It is **alert-only**: it does not issue Docker cleanup, unblock work, or dispatch cards. It can report a condition for a human or separately authorized workflow to inspect, but it does not turn a monitor into a second operator with broad powers.

That separation is less glamorous than an automation that heals everything. It is also easier to reason about. A watchdog that detects a stale condition should not automatically inherit authority to decide which host resources are disposable or which task is safe to resume.

We verified the revised behavior rather than relying on the earlier green result: a changed manifest was rejected before execution; the full control-plane suite reported 241 passing tests, with 21 focused hardening tests; scoped cleanup left no matching containers, volumes, or networks; and the scheduled healthy watchdog run was silent. Those are useful checks of the paths we exercised, not a proof that every possible race or hostile configuration is impossible.

The operational state matters too. At the end of that September 17 session, the hardening files were staged, not committed or pushed. There was no product merge or deployment. The final watchdog was alert-only, and we did not treat these checks as permission to expand its authority again.

## Lessons learned

1. **Keep different failures different.** Provider quota exhaustion and cleanup authorization stalls need separate diagnosis; retries and approval changes are not universal medicine.
2. **Treat cleanup as a capability, not a convenience command.** Pin the reviewed input, use the verified copy, identify the exact worktree and task, and bound resources and time.
3. **Let a circuit breaker be boring.** It is preferable to a loop that repeatedly performs a denied action.
4. **Make monitoring observe before it acts.** Alert-only is a valid final design when automatic remediation would need more authority than the monitor should have.
5. **Report what tests establish—and what they don’t.** Passing suites and a successful scheduled no-op support confidence in those paths; they do not certify isolation or deployment readiness.

This is the same kind of boundary lesson as [taming OpenClaw in a headless Proxmox LXC](/blog/taming-openclaw-proxmox-lxc) and [building a headless Google Drive backup](/blog/headless-gdrive-sync-lxc): the environment matters, and the safest automation is usually the one that knows exactly what it is allowed to touch—and stops when it cannot prove that.

## Sources

[5] https://docs.docker.com/reference/cli/docker/compose/down — Docker Compose down reference
[6] https://docs.docker.com/engine/manage-resources/labels — Docker object labels
