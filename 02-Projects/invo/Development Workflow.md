# Invo Reader — Development Workflow

## Before work

Read:

1. `02-Projects/invo/Overview.md`
2. `02-Projects/invo/Architecture.md`
3. `02-Projects/invo/Operations.md`
4. Repo `ARCHITECTURE.md` if modifying code

## Commands

```bash
npm install
npm run check-session
npm run test-parser
npm run test-aria-webhook
npm run test-replay-protection
npm run test-live-coverage
npm run test-baseline-guard
```

## Change discipline

- Do not remove keepalive/token-refresh/browser-restart logic casually.
- Do not remove stable position tracking fields sent to Aria.
- Do not clear seen state except during explicit testing.
- Any Aria webhook contract changes must be coordinated with Aria's `app/invo/*` code.
