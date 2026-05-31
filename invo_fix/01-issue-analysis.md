# Invo Reader Session Expiry — Issue Analysis

## Status
**Status:** Diagnosed, fix planned  
**Date:** 2026-05-31  
**Repo:** `/workspace/repos/invo-reader`  
**Fix plan:** `02-fix-plan.md` (this folder)

## Symptom

After ~6 days of continuous operation, the Invo reader stops processing new notifications and enters a restart loop:

```
last_error: "API not responding — forcing restart"
last_detail_extracted_at: <4+ hours ago, never recovers>
```

The reader remains alive (browser up, on the notifications page, logged_in=true) but all API calls return zero results. It force-restarts the browser every ~3.3 minutes, but the restart never fixes the problem.

## Root Cause

**Invo expires browser sessions server-side after ~6 days.** The reader only refreshes the JWT access token (every 8 minutes) via the `/auth/refresh_token` endpoint, but it does NOT keep the underlying browser session cookies alive.

### How Auth Works in the Reader

```
┌─────────────────────────────────────────────────────────┐
│                  Browser Session                         │
│  storage_state.json contains:                           │
│  - Session cookies (HttpOnly, set by Invo server)       │
│  - These are what Invo uses to validate the session     │
│  - Expire server-side after ~6 days of inactivity       │
├─────────────────────────────────────────────────────────┤
│                  JWT Access Token                        │
│  - Captured from Authorization header on requests       │
│  - Refreshed every 8 min via /auth/refresh_token        │
│  - This refresh ONLY works while session is valid       │
│  - Once session expires → refresh also fails (401)      │
└─────────────────────────────────────────────────────────┘
```

### Failure Sequence

```
Day 1-5: Everything works
  ├─ Poll every 20s → API returns items
  ├─ Token refreshed every 8 min → works
  └─ Session cookies still valid

Day ~6: Session expires server-side
  ├─ storage_state.json cookies become invalid
  ├─ Token refresh fails silently (401)
  ├─ Next poll: fetchNotificationsDirectly() → empty
  ├─ Retry with token refresh → still empty
  ├─ After 10 empty polls (~3.3 min):
  │     throw "API not responding — forcing restart"
  ├─ Browser restarts with same expired cookies
  └─ Loop repeats forever
```

### Why Restart Doesn't Fix It

`restartBrowser()` in watcher.js (line 713):
1. Closes browser
2. Launches new browser → loads `storage_state.json` from disk
3. Calls `ensureLoggedIn()` → checks URL, sees dashboard → thinks logged in
4. BUT the cookies are expired → API calls still fail
5. Polling resumes with expired session → same failure

### Why the Current Token Refresh Doesn't Help

The `refreshAuthToken()` function (line 255) has two methods:

1. **Method 1 (direct API):** Calls `POST /auth/refresh_token` with the captured refresh token. This only works if the session is still valid. When session expires → 401.
2. **Method 2 (page navigation):** Navigates to `config.invo.url` (homepage) → wait 2s → navigate to `/notifications` → wait 5s → check for new token. This reuses the same expired cookies → Flutter app loads but API calls still fail.

**Neither method forces a full re-login or refreshes the session cookies.**

## Impact

- No new Invo signals forwarded to Aria during the outage
- Missed copy-trading opportunities
- Requires manual intervention (re-login via noVNC) to fix
- Happens predictably every ~6 days

## Detection

The `/status` endpoint already shows:
- `last_error: "API not responding — forcing restart"` — the symptom
- `last_detail_extracted_at` — stops updating ~6 days after last manual login

But there's no early warning. The reader silently fails for ~3.3 minutes before the first restart, and then loops forever.
