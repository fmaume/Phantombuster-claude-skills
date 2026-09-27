<h1 align="center">PhantomBuster Claude Skills</h1>

<p align="center">
  <strong>Claude Code / Claude Desktop skills for running PhantomBuster LinkedIn & Sales Navigator Phantoms </strong>
</p>

<p align="center">
  <a href="#available-skills">Skills</a> &bull;
  <a href="#install">Install</a> &bull;
  <a href="#how-the-rotation-works">How rotation works</a> &bull;
  <a href="#prerequisites">Prerequisites</a>
</p>

A set of Claude skills to get more out of [PhantomBuster](https://phantombuster.com/?referral=fmc-f8VgvF4). 
Don't know about PhantomBuster yet? Check out my in-depth [Phantombuster Review](https://www.tetriz.io/blog/phantombuster-review-a-versatile-automation-tool/).

## Available skills

| Name | Description | Author |
|------|-------------|--------|
| [`phantombuster-linkedin-rotation`](skills/phantombuster-linkedin-rotation/SKILL.md) | Run the LinkedIn Auto Invitation Accepter Phantom (script 2885) while rotating which saved LinkedIn identity it uses, and install a Claude scheduled task (cron) so rotation + launch happen unattended on the chosen cadence. Use to accept pending connection requests across multiple LinkedIn accounts. | [Fabian Maume](https://github.com/fmaume) |
| [`phantombuster-linkedin-withdraw-rotation`](skills/phantombuster-linkedin-withdraw-rotation/SKILL.md) | Run the LinkedIn Auto Invitation Withdrawer Phantom (script 3672) while rotating identities and installing a Claude cron schedule. Cleans up stale/unaccepted outgoing connection requests unattended across every saved LinkedIn account. | [Fabian Maume](https://github.com/fmaume) |
| [`phantombuster-salesnav-alert-rotation`](skills/phantombuster-salesnav-alert-rotation/SKILL.md) | Run the Sales Navigator Alert Extractor Phantom (script 8380235547652117) while rotating only among identities with an active Sales Navigator subscription, and installing a Claude cron schedule. Pulls lead or account alerts unattended across multiple Sales Nav accounts. | [Fabian Maume](https://github.com/fmaume) |
| [`phantombuster-salesnav-viewers-rotation`](skills/phantombuster-salesnav-viewers-rotation/SKILL.md) | Run the Sales Navigator Profile Viewers Export Phantom (script 7495268271828736) while rotating only among Sales Nav-eligible identities, and installing a Claude cron schedule. Pulls "who viewed my profile" leads unattended across multiple Sales Nav accounts. | [Fabian Maume](https://github.com/fmaume) |

## Install

Drop the skill folder under your Claude skills directory. For Claude Code on Windows:

```
C:\Users\<you>\.claude\skills\<skill-name>\SKILL.md
```

For Claude Desktop / other clients, use whichever local skills path that client reads.

Or clone the whole repo and symlink the four folders in.

## How the rotation works

Some skills allow you to rotate LinkedIn credentials on specific Phantoms. That allows you to save on Phantom slots and get the most out of your subscription.

Those skills follow the same shape:

- **Setup mode** runs the first time you invoke the skill (or when you ask to reconfigure). It asks which cadence you want, counts your saved LinkedIn identities via `identities_search`, computes the cron expression(s), and installs one Claude scheduled task per rotation slot via the `scheduled-tasks` MCP server. It also clears any conflicting `repeatedLaunchTimes` on the PhantomBuster agent so the Phantom doesn't fire twice per slot.
- **Run mode** is what each scheduled-task firing invokes. It re-reads the identity list, figures out whose turn it is (round-robin for weekly/daily, slot-of-day for several-times/day), calls `agents_save` to point the agent at that identity, then `agents_launch` to run it.
- **Memory file** at `/areas/<skill-name>.md` stores the last-used rotation index, the chosen frequency, and the installed scheduled-task ids so both modes stay in sync.

The two Sales Navigator skills add an **eligibility filter** on top: they only rotate through identities whose `subscriptionTitles` include a Sales Navigator plan, and re-check that on every firing so a downgraded account is dropped from the rotation automatically.

## Prerequisites

- A PhantomBuster account with the `phantombuster` MCP server connected to Claude (`identities_search`, `agents_save`, `agents_launch`).
- The `scheduled-tasks` MCP server enabled in Claude (used to install the cron entries).
- One or more saved LinkedIn identities in PhantomBuster (the two Sales Nav skills additionally need each identity to have a Sales Navigator subscription).
