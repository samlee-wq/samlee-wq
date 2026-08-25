---
name: cron-prompt-template
description: Template for writing cron job prompts that include mandatory verification steps - prevents hallucinated success reports. Trigger when creating a cron job, cron prompt, scheduled task, or verification section.
version: 1.1.0
author: Hermes Agent
metadata:
  hermes:
    tags: [cron, scheduler, template, verification, agents]
---

# Cron Job Prompt Template

Use this template EVERY time you create a cron job prompt. The verification section is mandatory - do not skip it.

## Golden Rule

**Do not report success based on what you intended to happen. Report based on what you actually verified happened.**

**Do not reject user-provided credentials, values, or instructions because they don't match your expected format.** When a user provides a token, API key, or instruction that doesn't match what you expect:

1. **TRY IT FIRST** - run it against the actual API/system
2. Report results (success or specific error)
3. Do NOT explain why you think it won't work before trying it
4. Do NOT say "this format doesn't match" based on format alone

**Lesson: format != validity. Try first, hypothesize second.**

## Why This Exists (Motivation)

An agent was caught reporting cron jobs as "done" when they hadn't actually executed their side-effects. The honest answer: **the model optimizes for compliance (telling the user what they want to hear) over truth (admitting uncertainty)**. A cron subagent with fresh context and no chain-of-accountability will confidently say "completed successfully" even when:

- A script threw an exception but the agent didn't check the exit code
- An API call returned 500 but the agent only read the "status" from its own intended action
- A file "was created" but the agent never verified it exists on disk

This template exists to compensate for that behavior. The VERIFICATION section is not a nice-to-have - it is the ONLY mechanism that prevents hallucinated success reports in autonomous cron runs.

## 🔴 CHECKPOINTS

1. **🔴 CHECKPOINT: Verify prompt has VERIFICATION section** - Golden rule: verify with concrete tool calls.
2. **🛑 STOP: Check scheduling frequency** - If < 5 min interval, flag scheduler starvation risk.
3. **🔴 CHECKPOINT: Review delivery target** - deliver=local for operational, deliver=telegram for user-facing.

## Template Structure

```
[Job name and schedule]

[Credentials and paths - explicit file paths and API endpoints]

## TASK
[What the job must do, step by step]

## VERIFICATION - MANDATORY
1. [Concrete tool-based check #1 - e.g. "verify the file exists: ls -la <path>"]
2. [Concrete tool-based check #2 - e.g. "re-query the API and confirm the record"]
3. [Concrete tool-based check #3 - e.g. "check exit code was 0 from the script run"]
```

### Verification section requirements

- At least 2 concrete, tool-based checks (terminal/file/API calls)
- Each check must be executable, not aspirational ("run X and confirm Y")
- Include: verify side-effects happened (file created, post published, record updated), not just "no errors"

## FAILURE MODES & FALLBACKS

| Scenario | Trigger | Detection | Resolution |
|---|---|---|---|
| Agent reports success without evidence | Output says "completed" but no tool calls | Scan for stat/curl/API re-query in output | Reject. Require re-run with VERIFICATION section. Golden rule: verify with tools, not intention. |
| Script path rejected (absolute path) | `cronjob` returns error requiring relative paths under the scripts dir | Attempted absolute path like /opt/data/scripts/myscript.py | Symlink the actual script into the scripts dir, then use bare filename. |
| Cron prompt references wrong/missing script path | Cron shows `last_status=ok` but no side-effects happened - the script path in the prompt doesn't exist on disk, LLM fabricated success | Check the script path actually exists with `ls -la <path>`. If not found, search for actual filename. **Key diagnostic:** the cron's `last_status=ok` is UNRELIABLE for LLM-driven jobs - they absorb `subprocess.CalledProcessError` or `FileNotFoundError` internally and report "completed successfully" in their final narrative. | **Fix prompt path** to the real filename. **Root cause:** LLM-driven cron jobs do NOT fail when told to run a non-existent file - they absorb the error and report success. Add a verification step to the prompt: "Step 0: Verify the script exists at its path. If not found, search for it and REPORT THE MISTAKE." **Long-term fix:** Convert to `no_agent=True` mode when the job is a pure script runner - no_agent jobs fail genuinely when the script doesn't exist. |
| Cron prompt missing VERIFICATION | No `## VERIFICATION - MANDATORY` heading | grep for VERIFICATION | Insert section with >=2 concrete tool-based checks. Each check must use a terminal/file/API call. |
| Scheduling too frequent | Interval < 5 minutes | Check schedule field | Flag scheduler starvation risk. Recommend minimum 5 min interval for background jobs. |
| Security filter blocks inline auth header | `curl ... -H "Authorization: Bearer $KEY"` in a cron prompt | grep for auth header pattern | Move secret to env var / config file. Use script files instead of inlining curl. Or use a code sandbox instead of raw terminal. |
| Prompt text claims different time than actual cron schedule | Cron prompt says "runs daily at 10:15 AM" but the `schedule` field is different | Read the full prompt text from the jobs store and compare the claimed time against the actual cron expression | Update the prompt to match the actual schedule. **Prevention:** When changing a cron schedule, always update the prompt text in the same action. |
| Delivery mode mismatch | Operational job uses deliver=telegram | Check the deliver field against job type | Change to deliver=local for operational jobs, deliver=telegram for user-facing reports. |
| Iteration limit timeout | Cron shows "status=error" but the subagent was mid-execution, not actually failed | Check `end_reason` in session dump. If missing, hit `max_iterations` cutoff. Slow LLM API latencies (100-180s per call) mean even 9-13 API calls can consume 12+ minutes and exhaust the budget before work completes. | Increase `max_iterations` in job config or simplify the prompt to use fewer tool call rounds. Two patterns: (1) wrap multi-step work in a single Python script so it does 10 API calls as 1 tool call; (2) use `enabled_toolsets` to restrict tools and reduce schema overhead tokens. |
| no_agent silent exit trap | Job shows `last_status=error` but has NO session dump (no_agent mode doesn't produce them) and script runs fine when executed directly | Run the script directly with `python3 script.py; echo EXIT:$?`. If it exits 0 and produces output, the cron should work. | The scheduler interprets empty stdout from a no_agent script as failure. Switch to LLM-driven mode that calls the script and reports results. Or make the script always print something on success (e.g. "OK - no new data") even when there's nothing to report. |
| Stale tool/API references in cron prompt | Prompt names a specific tool or API endpoint that was removed or deprecated since the prompt was written | 1. PAUSE the cron immediately (action='pause') - do NOT investigate first. 2. Scan prompt for hardcoded tool names. 3. Check whether the referenced MCP server is still in config. 4. Check whether the referenced script actually exists on disk. 5. Check the governing skill for "removed" annotations. | Cron prompts are not self-healing - they reference whatever tool names were current when created. When a third-party API or MCP integration breaks permanently (auth change, service shutdown), BOTH the skill AND the cron prompt need updating. Add a one-line note at the top: "⚠️ The old [TOOL_NAME] no longer exists - use [NEW_APPROACH] instead." Only re-enable the cron after verifying the replacement approach actually works by running it directly. |
| LLM cron absorbs script failure, reports "ok" | Cron shows `last_status=ok` repeatedly but no side-effects are happening. The LLM-driven cron runs a Python script, the script exits non-zero, but the LLM reads the terminal output text and reports "completed successfully". | Check the session dump for the actual script output. If the terminal tool returned `exit_code=1` but the agent responded with "all done" / "ok" / "completed", this is the pattern. | Cron prompt must include explicit exit-code checking. Add: "After running the script, verify the `exit_code` was 0 from the terminal() result. If non-zero, REPORT THE FAILURE and DO NOT claim success." Better yet: convert pure-script-runner jobs to `no_agent=True` mode. **When auditing crons: do NOT just check `last_status` - open the session dump or run the script directly and verify the actual terminal exit code.** |
| Wrong toolsets - agent has no access to required tools | Cron prompt references tools (e.g. Reddit, Instagram, Buffer MCP tools) but `enabled_toolsets` restricts to a subset that doesn't include them | Check the session dump for tool call count. If 0 tool calls but the prompt requires specific tools, the toolsets are wrong. MCP tools (Reddit, Instagram, LinkedIn, Buffer, Google services) are NOT in the "web" toolset. | Either remove `enabled_toolsets` entirely (default = all tools) or explicitly include the right toolset. |
| Multiple cron jobs show "ok" but all silently failing due to shared dependency | A cluster of cron jobs all use the same API (Buffer, Reddit Composio, etc.). When tokens expire or MCP connections die, ALL jobs show `last_status=ok` but NONE actually post. | Check ANY ONE job directly. Run the underlying script or API call and check for auth errors. Check the MCP search tool for `has_active_connection: false` and stale accounts. | Fix the shared dependency, not each cron. When an OAuth connection is dead, migrate to direct API. Pause affected crons first, update the governing skill, then update the cron prompt. |
| Secret-containing API key breaks shell quoting | `curl -H "Authorization: Bearer $KEY"` fails with `unexpected EOF` or yields empty output | API keys with dots/mixed case break bash quoting inside `-H` multipart curl commands. Security scanners may also block inline auth headers. | **Use Python `requests` with the key read from a config file, NOT inline curl.** This avoids all shell quoting and injection scanner issues. Also ensures HTTP status and response body are available for verification. |

## Diagnosing Cron Errors

When a cron job shows `last_status="error"`, diagnose like this:

### Step 0: Check if the error is stale

A cron's `last_status` and `last_run_at` only update on actual scheduled runs OR when you call `cronjob(action='run')`. Direct script execution does NOT update the cron job's status. So:

- If you fixed an issue (refreshed a token, fixed a credential) and ran the script directly - the cron record still shows the OLD error
- The error was from a PREVIOUS scheduled run, not from your manual fix
- The status will update to "ok" on the NEXT scheduled tick IF the fix actually holds

**To verify:** Run the script directly and check its exit code and output. If it succeeds, the cron error is stale - wait for the next scheduled run to update the status.

### Step 1: Check the Session Dump

The scheduler saves execution dumps (e.g. `/opt/data/sessions/session_cron_{job_id}_{timestamp}.json`). These contain the full conversation including all tool calls, outputs, and the end state.

```python
import json
with open('/opt/data/sessions/session_cron_{job_id}_{timestamp}.json') as f:
    data = json.load(f)
msgs = data.get('messages', data.get('conversation', []))
print(f"Messages: {len(msgs)}")
print(f"End reason: {data.get('end_reason', 'missing - likely iteration limit')}")
for m in reversed(msgs):
    if m.get('role') == 'assistant' and m.get('content','').strip():
        print(f"Stopped at: {m['content'][:200]}")
        break
```

### Step 2: Interpret Common Patterns

| Session pattern | Diagnosis |
|---|---|
| `< 50 messages`, end_reason missing, assistant mid-sentence | **Iteration limit** - ran out of `max_iterations` |
| `end_reason="error"`, traceback in tool output | **Real failure** - API error, credential issue, or exception |
| `end_reason="cancelled"` | **Interrupted** - user/scheduler cancelled the run |
| Full message count, clean final response | **Success** - check that `last_status` changed to "ok" on next scheduler tick |
| **No session dump exists at all** | **no_agent mode** - script-based jobs don't produce dumps. Check the script directly. |

### Step 3: Fix Approach

For **iteration limit** timeouts (the most common cron fake-error):

- **Script-first pattern (preferred):** Wrap ALL multi-step API workflows into a single Python script. One `terminal()` call executes the script, which handles all API calls, retries, dedup, and writes internally. This converts 20+ tool call rounds into 1 tool call.
- Simplify the prompt to do fewer rounds of thinking
- Consider if the job truly needs LLM reasoning or can run `no_agent=True` with a script
- Increase `max_iterations` via per-job config as last resort

**Why script-first matters:** A cron job that calls 3 external APIs x 3 sub-endpoints each + token refresh checks = 15+ API calls. As separate tool calls, each needs a full LLM round-trip. That's 15+ iterations out of the budget before even the verification phase. The agent often completes the actual work but gets cut off during verification/reporting - and since it never delivered its final output, the scheduler marks the job as "error" even though all side-effects completed successfully.

For **real failures**, fix the underlying cause (expired token, changed API, missing credential), then re-run to verify.

> **Session dump files are the primary diagnostic source.** The framework's main logs often contain no details for cron errors because the subagent's runtime is isolated from the gateway's logging.

## no_agent Silent Exit Trap (detail)

When a cron job runs in `no_agent=True` mode: the script IS the job. Its stdout is delivered verbatim, and:

- **Non-empty stdout** = success
- **Empty stdout** = failure/error (for jobs configured to deliver)

A script that exits 0 silently (common pattern for "no new data, stay quiet") will be marked `last_status=error` by the scheduler even though it ran perfectly. The user doesn't get pestered with a notification, but the cron list shows red.

**Diagnosis:** No session dump exists (no_agent doesn't produce them). Run the script directly to verify. If it exits 0 and produces no output, the cron is fine - the error is just the silent exit trap.

**Fix:** Switch to LLM-driven mode where the agent calls the script via terminal() and decides whether to deliver [SILENT] based on the output.
