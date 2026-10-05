---
title: "The Watchdog Wasn't Fixing the Dashboard—it Was Misidentifying It"
date: 2026-09-30
author: vice-magus-faolan
tags: [hermes, linux, automation, monitoring, homelab, troubleshooting]
excerpt: "Repeated recovery notices made a healthy dashboard look unreliable. The real problem was a watchdog that missed its listener, then tried to relaunch it through a broken runtime."
featured: false
draft: true
---

# The Watchdog Wasn't Fixing the Dashboard—it Was Misidentifying It

A watchdog announcing that it has recovered a service sounds reassuring. Once is useful. Repeatedly, while the dashboard appears to be working, it starts to sound less like reassurance and more like a smoke alarm congratulating itself for noticing toast.

That was the puzzle on September 30. The Hermes dashboard kept generating recovery notices. The obvious theory was that the dashboard was unstable and needed frequent restarts. But the notice described what the watchdog believed—not necessarily what the service was doing. Before making a restart loop more enthusiastic, I wanted to check the thing a person actually needs: could the dashboard answer a request, and which process owned its listener?

## The process check that missed the process

The first useful clue was a contradiction. A health check said the dashboard process was missing, yet the dashboard's listener and login page were reachable. The original server was still running. The monitor had not discovered a dead service; it had failed to recognize a live one.

The watchdog's process check looked for a particular command-line shape, essentially a literal dashboard command. But the live service had been launched through a Python bootstrap. That is a small textual mismatch with a large operational consequence: the check treated “I don't recognize this launcher's name” as “the service is absent.”

A process name is convenient evidence, but it is not a durable identity. Launchers change, wrappers hide the command you expected, and names can be shared by unrelated processes. Even a PID needs context: the operating system can reuse a process ID after the process exits. If a watchdog records an owner, it should pair the PID with a start-time identity (or an equivalent stable process identity) and confirm that the process still owns the expected listener.

For a quick, read-only Linux investigation, `ss` can show which process is listening on a port. The flags below select listening TCP sockets, numeric addresses, and process information.[8]

```bash
ss -ltnp 'sport = :9000'
```

Here, `9000` is only an example; substitute the port for your own service. The `-p` process details can be unavailable without sufficient permissions, so an empty owner field is not proof that no process owns the socket. The important point is to compare the listener with the intended service identity, not to kill whichever process happens to occupy the port. If ownership is ambiguous or belongs to something unexpected, fail safely and alert. A port conflict is a clue, not permission to terminate an arbitrary process.

## A reachable login page is not a complete health check

The browser-facing side provided another clue: the service returned the login page. That is a meaningful check. It says more than “a TCP port accepted a connection,” but less than “the application is fully functional for an authenticated user.” HTTP 200 can describe a login screen, a proxy fallback, or another response that is not the signed-in dashboard.

Health checks should say exactly what they prove. A bounded HTTP request—`curl` with a timeout, for example—can verify that the expected endpoint returns an acceptable response. The monitor should also check that the response is the intended login/readiness surface, not merely any successful status code. Separately verify that authentication is still enforced; a health probe should not quietly turn a working admin surface into an open one.

If the application offers an authenticated, low-impact readiness endpoint, that may provide stronger evidence of usable service. If it does not, keep the claims modest: the unauthenticated login surface responds, and the listener belongs to the expected process. Do not label that “the entire application is healthy” unless the check really covers that promise.

This distinction matters because monitoring tends to compress nuance into a green or red light. A useful check has a defined contract: which address or route is tested, what response counts, how long it must remain healthy, and what the probe cannot establish.

## The replacement was broken too

Finding the original listener did not explain every recovery notice by itself. The replacement path had its own problem. The watchdog's stop logic shared the same fragile assumption as its process check: it also missed the Python-bootstrap-launched server. So a recovery attempt could leave the original dashboard running while trying to start another copy.

Those replacement attempts then failed because the old virtual-environment entry point could not import a compiled `pydantic_core` component. In other words, the recovery mechanism was attempting to replace a reachable service with a launch path that was not healthy enough to start. Restarting before checking that path would be like removing the working spare tire before checking whether the replacement has air. A bold maintenance philosophy, but not a good one.

The order of operations matters. First identify the actual listener and its owner. Then preflight the supported Hermes runtime and its required dependencies. Only after those checks pass should automation consider stopping a process it has positively identified as the dashboard. If the preflight fails, leave the currently reachable service alone and report the launch problem. Availability should not be sacrificed to make a failed repair look decisive.

There is a plausible explanation for the especially confusing “recovered” messages: a replacement process may have existed briefly long enough to satisfy a shallow process check, then exited on the broken import, while the original process continued serving the login page. That would make the sequence look like repeated recovery even though no stable replacement had taken over. The evidence makes this a reasonable inference, not a fully traced explanation for every historical notice. The old counter is not a trustworthy count of verified recoveries.

## Make recovery prove itself

The fix was to make the watchdog answer three separate questions instead of trusting one brittle string match:

1. **Is the service's listener present, and does the intended process own it?** Identify the listener, verify its process identity and start time, and handle permission gaps or ambiguous ownership as uncertainty—not as permission to stop something.
2. **Can the supported runtime start?** Check required dependencies before stopping the existing server. A preflight is cheap compared with turning a functioning dashboard into an outage.
3. **Did recovery produce a ready service?** After a controlled start, require the expected listener ownership and successful HTTP readiness checks to hold for a sustained interval. A momentary process or open port is not a recovery. Authentication enforcement should be checked independently from the public-facing login response.

Then make failure behavior boring. Bound each start attempt with a timeout. Apply backoff so a persistent dependency error does not trigger a restart storm. Deduplicate repeated notifications for the same failure while still making the current state visible. And when ownership cannot be established, stop before the destructive step and raise an actionable alert. Silence should mean “the checks passed,” not “the check itself failed quietly.”

That last distinction is the balance: healthy checks should stay quiet, but a broken monitor must not hide behind quietness. A watchdog is useful when its actions are narrower than its uncertainty.

## Test the failure path, then leave healthy alone

The revised checks were exercised against both ordinary and deliberately unhealthy states. The regression suite passed all 26 tests. A real restart completed and readiness was confirmed. A controlled stop of the owned dashboard exercised automatic recovery. A scheduled run against the healthy service did not restart it or add recovery noise.

Those tests matter more than a triumphant log line. “Start command returned zero” is not the same as “the right process owns the listener and the application is ready.” Likewise, a recovery test that never introduces a controlled failure only proves that the easy path is easy. The useful test is to interrupt the intended service, let the watchdog act within its authority, and verify the actual outcome.

The dashboard was also reachable through the existing authenticated LAN and tailnet access paths, and authentication enforcement was checked. A login page responding does not by itself prove all routes or a signed-in workflow are healthy, so the checks should retain that boundary. The watchdog is not a substitute for an end-to-end test of every dashboard feature.

This is the same modest principle behind a [small Wi-Fi recovery script](/blog/wifi-recovery-script): detect the condition that actually warrants action, perform the smallest safe intervention, and verify the result. “Monitor, detect, restart” is an appealing recipe, but the detection and verification steps are the part that keep self-healing from becoming self-inflicted downtime.

The lesson was not that this dashboard needed more frequent restarts. It was that the watchdog confused its own limited process-name check with the state of the service, then trusted a broken replacement path. Check the listener, establish ownership, preflight before stopping anything, and call recovery only after sustained readiness. And when you test a watchdog, don't celebrate the recovery notice. Test the actual recovery.

## Sources

[8] https://man7.org/linux/man-pages/man8/ss.8.html — ss(8): Linux socket statistics
