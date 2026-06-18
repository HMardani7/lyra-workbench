# Invo Reader Session Expiry Fix — Implementation Plan

## Goal

Prevent Invo server-side session expiry by periodically forcing a full app reload (not just API calls), keeping browser session cookies alive indefinitely.

## What This Fix Does

**Problem:** Invo expires browser sessions after ~6 days. The reader only refreshes JWT tokens, not session cookies.

**Change:** Add a periodic "session keepalive" that navigates through the full app (home → notifications) every 3 hours. This simulates real user activity, keeping Invo's server-side session TTL reset.

**Risk prevented:** Multi-hour signal loss requiring manual noVNC re-login every ~6 days.

**Config:** New constant `SESSION_KEEPALIVE_INTERVAL_MS` (3 hours). No new env vars needed.

**Scope:** `src/watcher.js` only — ~40 lines added, 0 file changes elsewhere.

## Files to Modify

### 1. `src/watcher.js` — Add session keepalive

#### Change 1: Add state variables (after line 16-17)

**Location:** After `let isFirstPoll = true;` (line 16), before `const MAX_CONSECUTIVE_ERRORS = 5;` (line 17)

```js
let lastSessionKeepaliveAt = 0;
let lastAuthErrorAt = 0;
const SESSION_KEEPALIVE_INTERVAL_MS = 3 * 60 * 60 * 1000; // Full app reload every 3 hours
const AUTH_ERROR_COOLDOWN_MS = 30 * 60 * 1000; // Only report auth errors once per 30 min
```

#### Change 2: Add `performSessionKeepalive()` function (before `startPolling`, ~line 655)

```js
// ─── Session Keepalive ────────────────────────────────────────────────────────

async function performSessionKeepalive(page) {
  const homeUrl = config.invo.url.replace(/\/$/, '');
  const notifUrl = homeUrl + '/notifications';

  logger.info('Session keepalive: reloading full app to refresh session cookies...');
  status.update({ lastSessionKeepaliveAt: new Date().toISOString() });

  // Step 1: Navigate to home page (forces full Flutter app reload)
  try {
    await page.goto(homeUrl, { waitUntil: 'domcontentloaded', timeout: 30000 });
    await page.waitForTimeout(3000); // Let Flutter initialize
  } catch (err) {
    logger.warn(`Keepalive home navigation error: ${err.message}`);
  }

  // Step 2: Navigate to notifications page
  try {
    await page.goto(notifUrl, { waitUntil: 'domcontentloaded', timeout: 30000 });
    await page.waitForTimeout(5000); // Let Flutter load notifications
  } catch (err) {
    logger.warn(`Keepalive notifications navigation error: ${err.message}`);
  }

  // Step 3: Verify the app actually loaded and API works
  // Wait for the notifications API to respond (via interceptor)
  const startWait = Date.now();
  let apiResponded = false;
  capturedNotifications = [];

  while (Date.now() - startWait < 15000 && !apiResponded) {
    await page.waitForTimeout(1000);
    if (capturedNotifications.length > 0) {
      apiResponded = true;
    }
    // Also check if we have a fresh auth token
    if (capturedAuthToken && Date.now() - tokenCapturedAt < 5000) {
      apiResponded = true; // Token captured = API is working
    }
  }

  if (apiResponded) {
    logger.info(`Session keepalive successful (API responded in ${Date.now() - startWait}ms)`);
    lastSessionKeepaliveAt = Date.now();
    status.update({ 
      lastSessionKeepaliveAt: new Date().toISOString(),
      notificationsApiSeen: true,
    });
    return true;
  }

  logger.warn('Session keepalive: API did not respond within 15s — session may be expired');
  return false;
}
```

#### Change 3: Call keepalive in the polling loop (in `startPolling()`, inside the while loop)

**Location:** After polling loop entry (after `while (running) {` at line 669), before `try { await pollOnce(); }`

```js
  while (running) {
    // ── Session keepalive (every 3 hours) ──
    const page = browser.getPage();
    if (page && Date.now() - lastSessionKeepaliveAt > SESSION_KEEPALIVE_INTERVAL_MS) {
      logger.info(`Session keepalive triggered (last was ${Math.round((Date.now() - lastSessionKeepaliveAt) / 60000)} min ago)`);
      await performSessionKeepalive(page);
      // Reset state after keepalive so we don't carry stale state
      capturedNotifications = [];
      isFirstPoll = false; // Don't re-trigger baseline
    }

    try {
      await pollOnce();
```

#### Change 4: Improve 401 detection in `fetchNotificationsPage` (line 226-231)

**Current code (line 226-231):**
```js
    if (result && !result.success) {
      logger.debug(`Page ${pageNum} failed: status=${result.status || 'unknown'} error=${result.error || ''}`);
      if (result.status === 401) {
        capturedAuthToken = null;
      }
    }
```

**Replace with:**
```js
    if (result && !result.success) {
      const status = result.status || 0;
      logger.debug(`Page ${pageNum} failed: status=${status} error=${result.error || ''}`);
      
      if (status === 401 || status === 403) {
        capturedAuthToken = null;
        
        // Track auth errors with cooldown to avoid spam
        const now = Date.now();
        if (now - lastAuthErrorAt > AUTH_ERROR_COOLDOWN_MS) {
          lastAuthErrorAt = now;
          logger.error(`Auth token rejected (HTTP ${status}). Session may be expired.`);
          status.update({ 
            lastAuthError: `HTTP ${status} on notifications API — session may be expired`,
            lastAuthErrorAt: new Date().toISOString(),
          });
          
          // Try to send Telegram alert
          try {
            await telegram.sendMessage(`⚠️ Invo Reader: Auth token rejected (HTTP ${status}). Session may be expired. Will try session keepalive on next cycle.`);
          } catch { /* ignore */ }
        }
      }
    }
```

### 2. `src/status.js` — Add new status fields (after line 24)

**Location:** Add to the `state` object:

```js
  lastSessionKeepaliveAt: null,
  lastAuthError: null,
  lastAuthErrorAt: null,
```

**Also in the `get()` function** (after line 60):

```js
    last_session_keepalive_at: state.lastSessionKeepaliveAt,
    last_auth_error: state.lastAuthError,
    last_auth_error_at: state.lastAuthErrorAt,
```

## Verification Checklist

- [ ] `SESSION_KEEPALIVE_INTERVAL_MS` constant added
- [ ] `lastSessionKeepaliveAt` and `lastAuthErrorAt` state variables added
- [ ] `performSessionKeepalive()` function:
  - [ ] Navigates to home page
  - [ ] Waits for Flutter to load
  - [ ] Navigates to notifications
  - [ ] Waits for API response (up to 15s)
  - [ ] Returns true/false
- [ ] Called from polling loop every 3 hours
- [ ] 401/403 detection in `fetchNotificationsPage`:
  - [ ] Logs status code
  - [ ] Tracks `lastAuthErrorAt` with cooldown
  - [ ] Sends Telegram alert (once per 30 min)
- [ ] `status.js` updated with new fields
- [ ] Test: Deploy and verify keepalive runs after startup
- [ ] Test: Verify `/status` endpoint shows new fields
- [ ] Test: After 6+ days, session should NOT expire

## Design Decisions

| Decision | Reasoning |
|---|---|
| 3-hour interval | Frequent enough to beat Invo's ~6-day timeout, but not so frequent it looks suspicious or wastes resources |
| Full app navigation vs API-only | API calls alone don't refresh browser session cookies. Full page load through the Flutter app does. |
| 15s API response timeout in keepalive | Flutter app needs time to initialize and make the notifications API call. 15s is generous for a working app but won't hang forever. |
| 30-min auth error cooldown | Avoids spamming Telegram every 20s during an outage. One alert per 30 min is enough. |
| No new env vars | Keepalive interval is a constant, not config. If it needs tuning later, can promote to env var. |
| Keepalive BEFORE pollOnce | Ensures fresh cookies before attempting to fetch notifications. |
| Reset capturedNotifications after keepalive | Prevents stale intercepted data from being processed as new notifications. |

## Files Created

| File | Purpose |
|---|---|
| `src/watcher.js` | Modified (~40 lines added) |
| `src/status.js` | Modified (~3 lines added) |

## Rollback

If the keepalive causes issues (e.g., race conditions, false duplicate detections):
1. Remove the keepalive block from `startPolling()`
2. Remove `performSessionKeepalive()` function
3. Keep the 401 detection improvements (they're purely additive)
