# Invo Reader — Operations

## Deployment model

Runs on VPS via Docker Compose. Xvfb mode is important because Invo's Flutter Canvas app may fail in pure headless Chromium.

Canonical repo:

```text
/srv/hermes-repos/invo-reader
```

Production container from Lyra memory:

```text
invo-reader-xvfb
```

## Common commands

```bash
npm install
npx playwright install chromium
npm run manual-login
npm run check-session
npm run baseline-seen
npm start
```

Docker production examples:

```bash
docker compose --profile xvfb up -d invo-reader-xvfb
docker compose --profile xvfb run --rm invo-reader-xvfb npm run check-session
docker compose --profile xvfb run --rm invo-reader-xvfb npm run test-notifications-api
```

Manual login/debug:

```bash
docker compose --profile debug up -d invo-reader-debug
# open http://<vps-ip>:6080/vnc.html
```

## Health/status

- `GET /health`
- `GET /status`

Default port is configured via `PORT`, defaulting to `8090`.

## Common failure modes

- Session expiry can cause API not responding/restart loops; recover by noVNC/manual login.
- Browser crash/protocol errors require browser restart.
- Deleting seen/state files can replay old notifications.
- Deleting browser storage requires re-login.

## Safety notes

Do not print `.env`, browser profile/session files, or captured auth data.
