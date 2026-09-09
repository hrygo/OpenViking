# Trusted Mode Identity Batch Registration

## Purpose

Trusted-mode requests normally use `X-OpenViking-Account` and
`X-OpenViking-User` only as request identity. This feature lets an upstream
opt in to registering those identities so existing account and user management
APIs can list them, without adding registry I/O to the data-plane request.

## Request contract

Only a trusted data-plane request with this header opts in:

```http
X-OpenViking-Register-Identity: true
```

The value is strictly `true` or `false`; absent and `false` leave existing
behavior unchanged. Invalid values return `INVALID_ARGUMENT`. Admin routes
(`/api/v1/admin` and its subpaths) never auto-register identities. Other auth
plugins do not inspect this header.

For example:

```bash
curl -X POST http://localhost:1933/api/v1/sessions \
  -H 'X-API-Key: <root-key>' \
  -H 'X-OpenViking-Account: acme' \
  -H 'X-OpenViking-User: alice' \
  -H 'X-OpenViking-Register-Identity: true' \
  -H 'Content-Type: application/json' \
  -d '{"session_id":"session-001"}'
```

## Runtime behavior

`TrustedAuthPlugin` keeps a bounded per-instance pending batch and LRU of
already-flushed identities. Request handling only performs in-memory dedupe;
it does not read registry files, acquire a registry lock, create a task, or
write storage. The total `pending + in-flight` backlog is capped. When full, a
new identity is dropped from automatic registration but the business request
continues unchanged.

One `asyncio.Task` per server instance performs the flush. The first run waits
for `interval + random(0, interval)` to avoid multi-instance startup bursts;
subsequent runs use the fixed interval. Shutdown cancels the loop and makes one
best-effort final flush. A process crash can therefore lose identities queued
only during the final interval; a later opt-in request queues them again.

The server configuration is:

```json
{
  "server": {
    "trusted_identity_flush_interval_seconds": 300,
    "trusted_identity_pending_max_size": 10000,
    "trusted_identity_known_max_size": 100000
  }
}
```

All values must be positive. The default five-minute flush has its first
execution randomly delayed to 5–10 minutes after startup.

## Registry merge and scope

The batch merge reads `accounts.json` once and one `users.json` for each
affected account. It takes the existing exact path locks, re-reads under lock,
and writes a file only when that file has a real addition. Concurrent instances
therefore converge without duplicate errors or overwriting existing user keys
and roles. New automatic users have no API key and are stored as:

```json
{
  "role": "user",
  "identity_source": "trusted",
  "created_at": "<ISO-8601 timestamp>"
}
```

Existing user records are never modified. This feature does not read or write
`groups.json`, create groups, or assign group membership. Deletion conflicts
and tombstones are intentionally outside this first version.

Ordinary account and user management mutations follow the same lock-scoped
read/merge/write rule. A stale instance changes only its target account or user
and preserves identities written by another instance.

There is no trusted-mode registry watcher. Account/user management reads
(including settings detail reads) instead perform an on-demand registry
signature check. An unchanged registry returns from the local snapshot; a
changed signature reloads the registry before the response. Account reads stat
only `accounts.json`; user reads stat `accounts.json` and that account's
`users.json`. This keeps data-plane requests and idle service instances free
from periodic storage polling.
