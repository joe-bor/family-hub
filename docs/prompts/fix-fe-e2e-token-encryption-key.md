# Fix FE E2E: Add TOKEN_ENCRYPTION_KEY to Backend Container

## Context

The BE has been merging Google Calendar sync PRs to `main`. These introduced a required `TOKEN_ENCRYPTION_KEY` environment variable (used for AES-256 encryption of OAuth tokens). The BE Docker image now fails to start without it, which breaks the FE CI e2e tests.

## What to Do

Add `TOKEN_ENCRYPTION_KEY` to `docker-compose.e2e.yml` so the BE container starts successfully in CI.

### File: `docker-compose.e2e.yml`

Add this line to the `environment` section:

```
TOKEN_ENCRYPTION_KEY=dGVzdC1lbmNyeXB0aW9uLWtleS0zMi1ieXRlcw==
```

This is a dummy Base64-encoded 32-byte key — it only needs to satisfy the AES-256 config requirement. No real Google OAuth happens in e2e tests.

## Verification

- Start the BE container locally with the updated compose file and confirm it reaches healthy status
- Run `npm run test:e2e` against it to confirm tests pass as before

## Branch & PR

- Branch from `main`, use `fix/e2e-token-encryption-key`
- Single commit is fine — this is a one-line fix
