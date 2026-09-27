---
name: phantombuster-salesnav-viewers-rotation
description: "Run the Sales Navigator Profile Viewers Export Phantom (script 7495268271828736, org phantombuster) while rotating only among saved identities with an active Sales Navigator subscription, and install a Claude scheduled task (cron) so rotation + launch happen unattended on the chosen cadence. Use whenever the user asks to run, schedule, reschedule, or reconfigure this profile-viewers extractor, to change/check its identity-rotation frequency, or to set up unattended Sales Navigator profile-viewer extraction across multiple accounts."
---

## What this skill manages

- **Agent**: not yet created. Before first use, create it with `agents_save` (`script: "Sales Navigator Profile Viewers Export.js"`, `org: "phantombuster"`), then record the returned agent id at the top of this file for future runs.
- **Script**: id `7495268271828736`, "Sales Navigator Profile Viewers Export" — extracts the users who have viewed your LinkedIn profile.
- **Behavior arguments** (default — confirm with the user before first run):
  - `numberOfProfilesPerLaunch`: 100 (PhantomBuster's own default if left empty)
- **Purpose**: pull "who viewed my profile" leads automatically, cycling only through identities that actually have Sales Navigator access — a Basic/free LinkedIn identity cannot run this Phantom at all.
- **Scheduling**: one or more Claude scheduled tasks (via the `scheduled-tasks` MCP server) that re-invoke this skill in run mode at each rotation slot. PhantomBuster's own `repeatedLaunchTimes` is deliberately NOT used — see "Why Claude cron, not PhantomBuster's" below.

This script requires a Sales Navigator session and does not natively support multi-identity rotation. Rotation must be performed by this skill: filter to eligible identities, pick the next one, call `agents_save` to point the agent at it, then launch.

## Two modes this skill runs in

At every invocation, decide which mode you are in before anything else:

- **Setup mode** — the memory file `/areas/phantombuster-salesnav-viewers-rotation.md` does not exist yet, OR the user explicitly asks to configure/reconfigure the schedule, change the frequency, or add/remove/refresh the scheduled task(s). Go to **Setup procedure**.
- **Run mode** — the memory file exists and the user (or a scheduled-task firing) just wants one round executed. Go to **Run procedure**. Scheduled-task firings arrive as a prompt like "Invoke the phantombuster-salesnav-viewers-rotation skill to run one round"; treat that as run mode.

## Identity eligibility filter (required, same rule as the Sales Nav Alert Extractor skill)

1. Call `identities_search` (type `linkedin`) to get all saved identities.
2. **Keep only identities whose `subscriptionTitles` array contains an entry mentioning "Sales Nav"** (e.g. `"Sales Navigator Core"`, `"Sales Navigator Advanced"`, `"Sales Navigator Advanced Plus"` — match on the substring `"Sales Nav"`, case-insensitive, rather than an exact string, since PhantomBuster's plan labels vary).
3. Sort the eligible list by `id` for a stable rotation order. This filtered list — not the full identity list — is what "N" refers to everywhere below.
4. **If zero identities qualify**, do not create/launch the agent and do not install any scheduled tasks. Tell the user that none of their saved LinkedIn identities currently have a Sales Navigator subscription, so this Phantom has nothing to run as. Suggest they save a Sales Navigator account as an identity first.

## Frequency modes (user-selectable)

Stored current setting: **not yet chosen — ask the user** (options below), then update this line.

1. **Weekly** — one identity per week. Rotate on a fixed weekday (default Monday, 9:00). Advance the rotation index by 1 each week.
2. **Daily** — a different identity each day. Rotate once per day (default 9:00). Advance the rotation index by 1 each day.
3. **Several times/day (9:00–18:00, evenly distributed)** — with N *eligible* identities, compute N launch slots evenly spaced across the 9-hour window: `slot[i] = 9:00 + i * (9/N) hours` for `i = 0..N-1`. Each slot uses the next eligible identity in rotation order. With N=1 there is a single 9:00 slot.

## Setup procedure

Run this at first invocation, and any time the user asks to reconfigure.

1. Run the eligibility filter above. If N = 0, stop here (see filter step 4) — do not create any scheduled tasks.
2. If the user hasn't already picked one, ask which frequency mode they want (weekly, daily, or several times/day). For weekly, also confirm the weekday and hour; for daily, confirm the hour. Confirm `numberOfProfilesPerLaunch` at the same time.
3. Compute the cron expression(s) using the eligible N:
   - **Weekly**: one entry, e.g. `0 9 * * 1` (Mon 09:00).
   - **Daily**: one entry, e.g. `0 9 * * *` (every day 09:00).
   - **Several times/day**: N entries, one per computed slot. For N=3 that's `0 9 * * *`, `0 12 * * *`, `0 15 * * *`; for N=4 it's `0 9 * * *`, `15 11 * * *`, `30 13 * * *`, `45 15 * * *` (round each slot to the nearest whole minute). Use one scheduled task per slot rather than a single task that "figures out which slot it is" — that way a missed or paused slot doesn't cascade into the others, and each task's name makes its role legible in `list_scheduled_tasks`.
4. Load the scheduled-tasks tools with a single ToolSearch call:
   `select:mcp__scheduled-tasks__list_scheduled_tasks,mcp__scheduled-tasks__create_scheduled_task,mcp__scheduled-tasks__update_scheduled_task,mcp__scheduled-tasks__delete_scheduled_task`
5. Call `list_scheduled_tasks` and find any prior tasks from this skill by title prefix `Sales Nav viewers rotation`. If the new configuration produces a different set of cron entries than what's already installed, delete the stale tasks via `delete_scheduled_task` before creating the new ones — never leave orphan schedules behind or the phantom will fire more often than intended.
6. For each cron entry, call `create_scheduled_task` with:
   - **title**: `Sales Nav viewers rotation — <mode> [slot i+1/N]` (omit the slot part for weekly/daily).
   - **schedule/cron**: the expression from step 3.
   - **prompt**: `Invoke the phantombuster-salesnav-viewers-rotation skill in run mode to launch one rotation round.`
   Record the returned task id.
7. Create or update `/areas/phantombuster-salesnav-viewers-rotation.md` with: the chosen frequency mode, the scheduled-task ids and their cron expressions, the current eligible identity count N and the ids of those eligible identities (so run mode can detect eligibility drift, not just count drift), the last-used rotation index (start at -1 so the first fired run advances to 0), the confirmed behavior arguments, and today's date. Update the "Stored current setting" line at the top of this skill file to reflect the chosen mode.
8. Ask the user whether to trigger one round now to confirm the pipeline works end-to-end. If yes, fall through to the Run procedure. Otherwise, tell them the first scheduled firing time.

## Run procedure

Run this at each firing (whether triggered by a scheduled task or manually).

1. Run the eligibility filter above to get the current eligible identity list and its size N. Re-run this every time — a previously-eligible identity may have downgraded, and a new one may have been added.
2. If N = 0, stop and notify the user (see filter step 4). Also list scheduled tasks with prefix `Sales Nav viewers rotation` and warn the user that they're still installed but will keep firing into a no-op — offer to delete them.
3. Read `/areas/phantombuster-salesnav-viewers-rotation.md` for the last-used rotation index, stored frequency setting, stored N, and stored eligible-id list. If the current eligible-id set differs from what's stored (identities added, removed, upgraded, or downgraded), the slot math for "several times/day" is now wrong and the weekly/daily rotation index may point at a stale identity — stop, tell the user what changed, and offer to re-enter Setup mode. For weekly/daily, if the user says "just continue", proceed with `next_index = (last_index + 1) mod N` against the new eligible list.
4. Determine whose turn it is, using only the eligible list:
   - Weekly/Daily: `next_index = (last_index + 1) mod N`.
   - Several times/day: figure out which of today's N slots the current time falls in, and use that slot's identity directly (this is robust to a missed firing — the next slot still picks the correct identity).
5. Call `agents_save` with the agent's id and `argument: {"numberOfProfilesPerLaunch": <current setting>, "identityId": "<next eligible identity id>"}` to switch the account.
6. Call `agents_launch` with the agent's id to run it immediately.
7. Update `/areas/phantombuster-salesnav-viewers-rotation.md` with the new last-used index/identity and timestamp.

## Why Claude cron, not PhantomBuster's

PhantomBuster's own recurring scheduler (`repeatedLaunchTimes`) can fire the agent unattended on a cadence, but it always launches with whatever identity is currently saved on the agent — it cannot re-check eligibility or swap identities between firings. So it works only for the trivial single-eligible-identity case, and silently reuses one account the moment a second eligible identity exists. Worse, if the saved identity later loses its Sales Nav subscription, every native firing will fail because the phantom needs Sales Navigator access.

A Claude scheduled task invokes this skill's Run procedure at each slot, which re-runs the eligibility filter and swaps identities *before* launching. That's why Setup installs Claude cron entries rather than setting `repeatedLaunchTimes` on the PhantomBuster agent. Leave `repeatedLaunchTimes` empty on the agent — if both schedulers are armed, the phantom fires twice per slot.

## Changing settings

- **Number of viewer profiles per launch** (`numberOfProfilesPerLaunch`): update the value used in step 5 of the Run procedure (and note the new default here). No schedule change needed.
- **Frequency mode, weekday, or hour**: re-enter Setup mode. It will delete the old scheduled task(s) and create new ones so the schedule and this file never drift apart.
- **Eligible identities added, removed, upgraded, or downgraded** (any change to the eligible set): re-enter Setup mode so the number of scheduled tasks matches the new eligible N and the stored eligible-id list reflects reality.
- **Stop rotation entirely**: `list_scheduled_tasks` filtered to titles starting with `Sales Nav viewers rotation`, then `delete_scheduled_task` on each id. The agent itself stays in place and can still be launched manually.
