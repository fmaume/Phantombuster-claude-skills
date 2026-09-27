---
name: phantombuster-linkedin-rotation
description: "Run the LinkedIn Auto Invitation Accepter Phantom (script 2885, org phantombuster) while rotating which saved LinkedIn identity it uses, and install a Claude scheduled task (cron) so rotation + launch happen unattended on the chosen cadence. Use whenever the user asks to run, schedule, reschedule, or reconfigure this connection-request auto-accepter, to change/check its identity-rotation frequency, or to set up unattended invitation-accepting across multiple LinkedIn accounts."
---

## What this skill manages

- **Agent**: not yet created. Before first use, create it with `agents_save` (`script: "LinkedIn Auto Invitation Accepter.js"`, `org: "phantombuster"`), then record the returned agent id at the top of this file for future runs.
- **Script**: id `2885`, "LinkedIn Auto Invitation Accepter" — accepts pending connection requests you've received.
- **Behavior argument**: `numberOfProfilesToAdd`: 10 (invitations accepted per launch) — change via `agents_save` if the user asks.
- **Purpose**: accept pending LinkedIn connection requests automatically, cycling through every saved LinkedIn identity in the account so no single account gets all the automation load.
- **Scheduling**: one or more Claude scheduled tasks (via the `scheduled-tasks` MCP server) that re-invoke this skill in run mode at each rotation slot. PhantomBuster's own `repeatedLaunchTimes` is deliberately NOT used — see "Why Claude cron, not PhantomBuster's" below.

This Phantom's script does **not** natively support multi-identity rotation (no `identities` array in its argument schema, unlike some other PhantomBuster scripts). Rotation must be performed by this skill: pick the next identity, call `agents_save` to point the agent at it, then launch.

## Two modes this skill runs in

At every invocation, decide which mode you are in before anything else:

- **Setup mode** — the memory file `/areas/phantombuster-linkedin-rotation.md` does not exist yet, OR the user explicitly asks to configure/reconfigure the schedule, change the frequency, or add/remove/refresh the scheduled task(s). Go to **Setup procedure**.
- **Run mode** — the memory file exists and the user (or a scheduled-task firing) just wants one round executed. Go to **Run procedure**. Scheduled-task firings arrive as a prompt like "Invoke the phantombuster-linkedin-rotation skill to run one round"; treat that as run mode.

## Frequency modes (user-selectable)

Stored current setting: **not yet chosen — ask the user** (options below), then update this line.

1. **Weekly** — one identity per week. Rotate on a fixed weekday (default Monday, 9:00). Advance the rotation index by 1 each week.
2. **Daily** — a different identity each day. Rotate once per day (default 9:00). Advance the rotation index by 1 each day.
3. **Several times/day (9:00–18:00, evenly distributed)** — with N saved identities, compute N launch slots evenly spaced across the 9-hour window: `slot[i] = 9:00 + i * (9/N) hours` for `i = 0..N-1`. Each slot uses the next identity in rotation order. With N=1 there is a single 9:00 slot.

## Setup procedure

Run this at first invocation, and any time the user asks to reconfigure.

1. If the user hasn't already picked one, ask which frequency mode they want (weekly, daily, or several times/day). For weekly, also confirm the weekday and hour; for daily, confirm the hour.
2. Call `identities_search` (type `linkedin`) to get the current count N of saved identities. Sort by `id` for a stable rotation order and record N — the memory file needs it so run mode can detect drift.
3. Compute the cron expression(s):
   - **Weekly**: one entry, e.g. `0 9 * * 1` (Mon 09:00).
   - **Daily**: one entry, e.g. `0 9 * * *` (every day 09:00).
   - **Several times/day**: N entries, one per computed slot. For N=3 that's `0 9 * * *`, `0 12 * * *`, `0 15 * * *`; for N=4 it's `0 9 * * *`, `15 11 * * *`, `30 13 * * *`, `45 15 * * *` (round each slot to the nearest whole minute). Use one scheduled task per slot rather than a single task that "figures out which slot it is" — that way a missed or paused slot doesn't cascade into the others, and each task's name makes its role legible in `list_scheduled_tasks`.
4. Confirm the agent's `repeatedLaunchTimes` is empty (call `agents_save` with `null`/`[]` if anything is set). If both PhantomBuster's native scheduler and Claude cron are armed, the phantom fires twice per slot and burns invitation quota.
5. Load the scheduled-tasks tools with a single ToolSearch call:
   `select:mcp__scheduled-tasks__list_scheduled_tasks,mcp__scheduled-tasks__create_scheduled_task,mcp__scheduled-tasks__update_scheduled_task,mcp__scheduled-tasks__delete_scheduled_task`
6. Call `list_scheduled_tasks` and find any prior tasks from this skill by title prefix `LinkedIn accepter rotation`. If the new configuration produces a different set of cron entries than what's already installed, delete the stale tasks via `delete_scheduled_task` before creating the new ones — never leave orphan schedules behind or the phantom will fire more often than intended.
7. For each cron entry, call `create_scheduled_task` with:
   - **title**: `LinkedIn accepter rotation — <mode> [slot i+1/N]` (omit the slot part for weekly/daily).
   - **schedule/cron**: the expression from step 3.
   - **prompt**: `Invoke the phantombuster-linkedin-rotation skill in run mode to launch one rotation round.`
   Record the returned task id.
8. Create or update `/areas/phantombuster-linkedin-rotation.md` with: the chosen frequency mode, the scheduled-task ids and their cron expressions, the identity count N used to compute slots, the last-used rotation index (start at -1 so the first fired run advances to 0), and today's date. Update the "Stored current setting" line at the top of this skill file to reflect the chosen mode.
9. Ask the user whether to trigger one round now to confirm the pipeline works end-to-end. If yes, fall through to the Run procedure. Otherwise, tell them the first scheduled firing time.

## Run procedure

Run this at each firing (whether triggered by a scheduled task or manually).

1. Call `identities_search` (type `linkedin`) to get the current, up-to-date list of saved identities. Sort by `id` for a stable rotation order.
2. Read `/areas/phantombuster-linkedin-rotation.md` for the last-used rotation index, stored frequency setting, and stored N. If the current live count of identities differs from the stored N in "several times/day" mode, the slot math is now wrong — stop, tell the user, and offer to re-enter Setup mode so the number of scheduled tasks matches the number of identities.
3. Determine whose turn it is:
   - Weekly/Daily: `next_index = (last_index + 1) mod N`.
   - Several times/day: figure out which of today's N slots the current time falls in, and use that slot's identity directly (this is robust to a missed firing — the next slot still picks the correct identity).
4. Call `agents_save` with the agent's id and `argument: {"numberOfProfilesToAdd": <current setting>, "identityId": "<next identity id>"}` to switch the account.
5. Call `agents_launch` with the agent's id to run it immediately.
6. Update `/areas/phantombuster-linkedin-rotation.md` with the new last-used index/identity and timestamp.

## Why Claude cron, not PhantomBuster's

PhantomBuster's own recurring scheduler (`repeatedLaunchTimes`) can fire the agent unattended on a cadence, but it always launches with whatever identity is currently saved on the agent — it cannot swap identities between firings. So it works only for the trivial single-identity case, and silently reuses one account the moment you add a second.

A Claude scheduled task invokes this skill's Run procedure at each slot, which does the identity swap *before* launching. That's why Setup installs Claude cron entries rather than setting `repeatedLaunchTimes` on the PhantomBuster agent. Leave `repeatedLaunchTimes` empty on the agent — if both schedulers are armed, the phantom fires twice per slot.

## Changing settings

- **Invitations-per-launch** (`numberOfProfilesToAdd`): update the value used in step 4 of the Run procedure (and note the new default here). No schedule change needed.
- **Frequency mode, weekday, or hour**: re-enter Setup mode. It will delete the old scheduled task(s) and create new ones so the schedule and this file never drift apart.
- **Identities added or removed** (several-times/day mode only): re-enter Setup mode so the number of scheduled tasks matches the new N. Weekly/daily modes don't need reconfiguration — the rotation index just advances through the new list.
- **Stop rotation entirely**: `list_scheduled_tasks` filtered to titles starting with `LinkedIn accepter rotation`, then `delete_scheduled_task` on each id. The agent itself stays in place and can still be launched manually. Also confirm `repeatedLaunchTimes` is still empty on the agent so it doesn't quietly resume firing.
