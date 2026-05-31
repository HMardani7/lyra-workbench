# Invo Reader Session Expiry Fix

## Quick Summary

The Invo reader stops working after ~6 days because Invo expires browser sessions server-side. The reader only refreshes JWT tokens, not session cookies.

**Fix:** Add a session keepalive that reloads the full Flutter app every 3 hours, keeping cookies alive.

## Files

| File | Read This |
|---|---|
| `01-issue-analysis.md` | Root cause, failure sequence, why restarts don't help |
| `02-fix-plan.md` | Exact code changes with line references and snippets |

## Implementation Instructions

1. Read `01-issue-analysis.md` to understand the problem
2. Read `02-fix-plan.md` for the implementation plan
3. Modify **only two files** in `/workspace/repos/invo-reader`:
   - `src/watcher.js` — add keepalive function + 401 detection (~40 lines)
   - `src/status.js` — add new status fields (~3 lines)
4. Test: deploy to VPS and verify `/status` shows `last_session_keepalive_at`
5. Monitor: after 7+ days, session should still be alive

## Target Repo

`/workspace/repos/invo-reader`
