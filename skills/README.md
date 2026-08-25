# Hermes Agent Skills

A collection of [Hermes Agent](https://hermes-agent.nousresearch.com) skills built from real production experience - running a small business on a single VPS with scheduled jobs, API integrations, and an AI agent doing the operations work.

Each skill is a `SKILL.md` in [Hermes skill format](https://hermes-agent.nousresearch.com/docs) - frontmatter metadata plus a procedure document the agent loads at task time.

## Skills

### [vps-disk-cleanup](vps-disk-cleanup/SKILL.md)
Safe VPS disk cleanup when the disk is near full. Covers the classic container traps: the `df` vs `du` discrepancy (invisible base image + host swapfile), why deleting base-image files frees nothing on overlayfs, RAM vs swap confusion on shared hosting, and a curated safe-delete list (package caches, npm/npx caches, old whisper models, stale /tmp) that keeps business data and required ML models intact.

### [cron-prompt-template](cron-prompt-template/SKILL.md)
Template for writing cron job prompts that include mandatory verification steps - prevents the classic "agent reports success without evidence" failure in autonomous scheduled runs. Includes a full failure-mode table (missing script paths, silent no_agent exits, iteration-limit timeouts, shared-dependency failures, wrong toolsets) and a session-dump diagnosis procedure.

### [api-readiness-check](api-readiness-check/SKILL.md)
Systematically test all APIs required by cron jobs or services. Produces a status table (working / blocked / broken / missing / pending) with fixes. Includes known quirks for marketplace APIs, ERPNext, Airtable, Google Drive via proxy services, and rate-limit counter patterns.

## Why these exist

All three skills were born from real incidents:

- A disk-full warning from the hosting provider where `df` showed 96% but `du` showed far less - the answer was overlayfs layers and a host-level swapfile, not leftover waste.
- Cron jobs that showed `last_status=ok` for days while doing nothing - the LLM absorbed script failures and reported success.
- API integrations that returned HTTP 200 with errors hidden in the response body.

The common thread: **verify with tools, not intention.** These skills encode that discipline.

## Installation

Copy a skill folder into your Hermes skills directory (`~/.hermes/skills/` or `/opt/data/skills/`), or use `hermes skills install` if your setup supports it. The agent discovers skills by directory name.

## License

MIT - use them freely. Contributions and improvements welcome.

---

*Built by Sam Lee (@samlee-wq) with Hermes Agent.*
