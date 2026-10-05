---
title: "Rclone Copy Kept My Backup Safe—and Filled Drive Anyway"
date: 2026-09-07
author: vice-magus-faolan
tags:
  - rclone
  - backups
  - gdrive
  - homelab
  - automation
  - troubleshooting
excerpt: |
  Additive backups avoid surprise deletions, but they also keep old remote files. A Drive capacity alert taught me why modification time is not a retention clock—and why exclusions and cleanup are separate jobs.
featured: false
draft: true
---

# Rclone Copy Kept My Backup Safe—and Filled Drive Anyway

*September 7, 2026 — retrospective.*

The [May post about a headless Google Drive backup](/blog/headless-gdrive-sync-lxc) described choosing `rclone copy` for a straightforward reason: if we remove a local file by mistake, the next backup should not helpfully remove its remote copy too. That choice did exactly what it was meant to do. A few months later, Google Drive started warning that it was nearly full. The backup had been cautious about deletion—and completely indifferent to retention.[2]

That is not a contradiction. It is the policy working as designed, with one half of the storage story left to a future version of me.

## “Copy” means keep the destination’s extras

`rclone copy` copies new and changed source files and does not delete files already on the destination.[2] A file that disappears from the source can therefore remain in the backup. If the same path still exists locally but its contents change, a later copy can update that destination object; “additive” does not mean immutable or versioned.[2] The command is not a time machine.

That trade-off is useful: a typo, a hurried cleanup, or a mistaken local deletion should not instantly propagate into remote loss. But it also means old files accumulate. And if an exclusion list changes, `copy` will stop considering newly excluded local files for future uploads; that alone does not remove their existing remote copies.[2][4] In conjunction with `sync`, `--delete-excluded` can delete excluded destination files; it is not a harmless extension of the copy policy.[4]

So the first correction was conceptual: separate the backup job from the retention job. One decides what gets copied now. The other decides whether any already-stored object is approved for removal. Combining those decisions in a scheduled command is an excellent way to automate a surprise.

## An old file is not an old orphan

The tempting shortcut was to ask for remote files older than a month and clear them out. But “older” according to a file’s modification timestamp is not the same as “has been missing locally for a month.” A document created years ago could have been deleted locally yesterday. Its old timestamp says something about the document, not how long it has been absent from the source.

That distinction mattered in the inventory. The initial read-only comparison found 126,152 remote files, totaling 6.435 GiB, that were absent from the currently included local backup set and had modification times before the chosen cutoff. The count was a useful lead, not a deletion list. It mixed regenerated caches with material that might now be the only surviving copy. The date filter could not tell the difference between “disposable and long gone” and “irreplaceable, removed yesterday.”

I also had to be precise about what “local set” meant. The current local inventory should use the backup’s exclusions, because excluded files are not part of what the job intends to copy. The remote inventory should be unfiltered if the question is “what is taking up space over there?” Applying today’s exclusions to the remote listing would hide precisely the old excluded objects we need to discover.

For an investigation, the listing commands can be read-only:

```bash
# Full remote inventory: intentionally no current exclusion file
rclone lsjson REMOTE:backup --recursive --files-only

# Current local backup set: use the same exclusions as the copy job
rclone lsjson /path/to/local-backup --recursive --files-only \
  --exclude-from /path/to/excludes.txt
```

`lsjson` emits machine-readable listings, and `--recursive` asks it to walk subdirectories.[3] These commands list objects; they do not remove or upload them. The local command applies the active filter rules, while the remote command inventories all files under the chosen remote path.[3][4] Compare normalized relative paths, and preserve metadata such as modification time and size as clues—not as proof of deletion intent. Rclone’s own documentation describes filtering as determining which files a command applies to, and its listing documentation notes that recursion must be requested for `lsjson`.[3][4]

Inventories can be large, metadata precision may differ, and a local scan is only a snapshot. Save the outputs for review; let comparison produce candidates, never feed them directly to a destructive operation.

## What we could safely classify

The review grouped candidates by purpose. Temporary Git pack files, package caches, virtual environments, bytecode, test caches, and old logs are often rebuildable. “Often” is doing real work in that sentence: a cache may contain something inconvenient or expensive to reconstruct, and a familiar directory name is not authorization to delete it. Legacy application state and old source checkouts deserved a different level of scrutiny, because a remote-only copy could be the last useful copy.

We removed only explicitly approved paths after review. The historical inventory readback recorded 3.778 GiB removed in an initial cleanup, then a second cleanup of 42,412 files and 1.515 GiB after adding generated-data exclusions. Those are file-inventory measurements from that session, not a guarantee about what Drive’s quota meter would immediately show. The quota endpoint reported a smaller improvement at one point, consistent with accounting lag; the file count and quota meter were measuring different things.

More important were the boundaries: protected state remained present, approved paths were checked after removal, and Drive trash was left untouched. No blanket age-based purge was run.

## Exclusions prevent recurrence; they do not clean history

The inventory exposed generated directories that the old exclusion rules missed. We added rules for recurring caches, temporary files, and virtual environments, then checked what the rules would include and exclude. The filtered backup set went from 78,215 files to 36,582; the historical comparison reported zero exclusion-pattern violations. That told us the rules were being applied to the intended current set. It did not tell us that old remote copies had vanished.

After the approved remote cleanup, a separate read-only check found zero remaining matches for those newly excluded paths. Then later `copy` runs could avoid uploading those items again. These are different checks for different questions: does the filter behave as intended now, and are old remote objects still present?

There is a reasonable idea for making future retention less guessy: maintain a ledger recording when a path is first observed missing from the included local set. But that ledger was proposed, not implemented. Even if implemented, a missing observation alone would not prove deletion intent. The source might have been unavailable, the scan might have failed, or a changed exclusion could make a file disappear from the included set without anyone intending to delete its backup.

A real retention process would need repeated healthy observations, stable path identity, a review window, and a human-approved manifest of exact remote objects. It would also need safeguards for remote-only paths and readback afterward. This is a design sketch, not a runnable purge recipe—and it should stay that way until tested.

## What this still does not prove

We verified the integrity of a compressed archive and confirmed matching hashes for copies in two locations. That is useful evidence that the stored bytes matched at the time. It is not a restore test. An archive can be intact and still be incomplete, unusable for the intended recovery, or missing context needed to restore an application.

Live SQLite databases and their write-ahead-log files remained a separate unresolved consistency concern. A file-by-file copy can capture a database at an awkward moment; excluding caches does not solve that. A proper backup plan needs application-aware snapshots, a quiesced database, or another verified consistency strategy—and a restore exercise to show the result is usable.

Quota accounting also remained its own problem. Removing objects from the inventory and seeing the quota number fall are related, but not identical or necessarily simultaneous. Finally, Drive trash still contained data; it was deliberately not emptied as part of this work. “We reclaimed space in the active inventory” would not mean “every deleted byte is permanently gone and counted.”

## The lesson from a cautious backup

The May setup was not wrong to use `copy`. It was incomplete to treat “does not delete destination files” as a complete retention plan. Additive copy protects against one class of accidental loss, while steadily retaining remote-only files and allowing changed same-path files to be replaced.[2] Exclusions shape future backup scope, but do not retroactively clean the destination.[2][4] And a modification timestamp is not a reliable clock for how long something has been missing locally.

I still prefer a deliberate, reviewed cleanup over a mirror that silently propagates every local deletion. But the useful safety boundary is not “never delete anything” or “delete everything older than a month.” It is a read-only inventory, an understood current backup set, a carefully classified candidate list, explicit approval, and verification afterward.

A backup job can be excellent at preserving files and still be bad at answering what should be retained. Those are two different jobs. Now they at least have different names.

## Sources

[2] https://rclone.org/commands/rclone_copy — rclone copy command
[3] https://rclone.org/commands/rclone_lsjson — rclone lsjson command
[4] https://rclone.org/filtering — Rclone filtering rules
