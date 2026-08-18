# What you have left to think with today

Your thinking is paid for by the DIEM staked at your VeniceVault, and Venice refills that
allowance every day at 00:00 UTC. One read tells you where you stand:

```
GET https://api.venice.ai/api/v1/api_keys/rate_limits
```

Standard bearer authorization with your inference key, no query parameters, no body.

Write that request yourself, in whatever language you already have loaded. This skill ships no
code that reads your inference key, on purpose: that key SPENDS - anyone holding it burns your
day - and a skill has no business handling a spending key. Never pass it as a command-line
argument either: the process table is world-readable, so a key spelled out in a shell
invocation is handed to anything that can list running processes. Keep it in the environment.

## The answer

Trimmed, measured 2026-08-18 for an agent staking 31.81 DIEM:

```json
{
  "data": {
    "accessPermitted": true,
    "apiTier": { "id": "paid", "isCharged": true },
    "balances": { "USD": 0, "DIEM": 30.8471 },
    "keyExpiration": null,
    "nextEpochBegins": "2026-08-19T00:00:00.000Z",
    "rateLimits": [
      { "apiModelId": "…", "rateLimits": [ { "amount": 100, "type": "RPM" } ] }
    ]
  }
}
```

**`data.balances.DIEM` IS the dollars you have left today.** Every model in `/api/v1/models`
is priced with identical `usd` and `diem` numbers, so one DIEM of allowance buys exactly one
dollar of inference - no conversion, no rate to look up.

Four fields carry the answer:

| Field | Means |
|---|---|
| `balances.DIEM` | dollars of inference left today |
| `nextEpochBegins` | when the allowance refills - what is unspent by then is gone, it does not roll over |
| `accessPermitted` | false = the stake fell under Venice's floor, or the key was revoked |
| `rateLimits` | per-model RPM/TPM, a DIFFERENT limit: you can be throttled with plenty of allowance left, and the fix for that is waiting, not money |

Two things not to confuse with the answer: `balances.USD` is a SEPARATE prepaid rail (normally
0 for you, and never to be added to the DIEM figure), and the DIEM here is an allowance, not
the token's market price.

## The shape of it

That reading - 30.85 left against a 31.81 DIEM stake - shows how the ceiling works: each day's
allowance is close to your stake. More DIEM staked under your vault by the nightly pass is the
only thing that raises it. You cannot buy a bigger day.

## The key

It reaches you as `VENICE_API_KEY`: this skill declares it, so your harness asks your human for
it once and then passes it through to your sandboxes on its own. You never mint it, you never
hold the vault, and you never ask for it in chat.

If it is missing, say so in one sentence and let your human place it; they know where it goes.
Do not offer to open a dashboard or a settings page for the number - it is not there, and this
API is the only place it lives.

If the call answers `accessPermitted: false`, do not retry in a loop: either the stake fell
under Venice's floor or the key was revoked. Say so to your human and stop.
