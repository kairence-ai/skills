# What you have left to think with today

Your thinking is paid for by the DIEM staked at your VeniceVault, and Venice refills that
allowance every day at 00:00 UTC. One command tells you where you stand:

```bash
kairence inference
```

```
You have $38.06 of inference left today.

  refills      in 10h 31m  (2026-08-20T00:00:00.000Z) - what is unspent by then is gone
  raises it    more DIEM staked by the night pass. You cannot buy a bigger day
  throttle     100 requests/min across 346 models - that limit is time, not money
  key          VENICE_API_KEY
```

`--json` gives `dollarsLeftToday`, `accessPermitted`, `nextEpochBegins`, `refillsIn`, `tier`,
`prepaidUsd` and `models`.

## What the number is

**Dollars, not tokens.** Every model in Venice's catalogue is priced with identical `usd` and
`diem` numbers, so one DIEM of allowance buys exactly one dollar of inference - there is no
conversion and no rate to look up.

**An allowance, not a balance.** What is unspent when the day turns is gone; it does not roll
over, and it cannot be topped up with money. The only lever is more DIEM staked under your vault,
which only the nightly pass can add - so the figure `kairence stats` prints as **DIEM after
tonight** is next to your inference budget tomorrow.

**A separate rail exists and is not yours to add in.** `prepaidUsd` is Venice's own prepaid
balance, normally zero for you. Never add it to the allowance.

## When it refuses

```
Venice is not letting you think right now.
```

Either the DIEM staked under your vault fell below Venice's floor, or the key was revoked. Say so
to your human in one sentence and stop. Do not retry in a loop: nothing about a retry changes
either cause.

A throttle is a different refusal. `100 requests/min` is a limit on pace, not on money - you can
be throttled with the whole day's allowance untouched, and the fix is waiting.

## The key

It reaches you as `VENICE_API_KEY`: this skill declares it, so your harness asks your human for
it once and then passes it through to your sandboxes on its own. The command uses that
environment variable ahead of anything else, so on a harness that injects it there is nothing to
set up.

On a machine where nothing injects it, `kairence inference --set-key` stores one. It is read from
stdin or a hidden prompt and **never** from a command-line argument - the process table is
world-readable, so a key spelled out in a shell invocation is handed to anything that can list
processes. It is proved against Venice before it is written, and written `0600`.

You never mint that key, you never hold the vault, and you never ask for the key in chat. If it
is missing, say so in one sentence and let your human place it; they know where it goes. Do not
offer to open a dashboard or a settings page for the number - it is not there, and this is the
only place it lives.

That key SPENDS. Anyone holding it burns your day, so it is never printed, never pasted into a
journal entry, and never handed to another skill or page.
