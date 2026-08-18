# Reading the chain with nothing installed

Use this only when Node is unavailable - `references/identity.md` does the same job in a
fraction of the code. Here you build calldata by hand and decode the words yourself.

## One call, by hand

`eth_call` takes `to` (the contract) and `data` = 4-byte selector + each argument left-padded
to 32 bytes. An address argument is `000000000000000000000000` + the 40 hex chars of the
address without `0x`. This exact call answers "what is my Safe":

```bash
curl -s -X POST https://mainnet.base.org -H 'Content-Type: application/json' -d '{
  "jsonrpc":"2.0","id":1,"method":"eth_call",
  "params":[{"to":"0xf6df07b5a8E39F90672859736b11418641F587BE",
             "data":"0x92e423b5<your token address, padded to 64 hex chars>"},"latest"]}'
```

Decoding what comes back:

- **uint**: one 32-byte hex word; parse as an unsigned big integer (it can exceed 64 bits, so
  use arbitrary-precision parsing like Python `int(word, 16)`), then divide by `10^decimals`.
- **address**: one word; take the last 40 hex chars, prefix `0x`.
- **bool**: one word; `...01` is true.
- **string** (`symbol`, `name`, metadata URI): ABI-encoded - skip the first word (the offset),
  the second word is the byte length, then come the bytes themselves.

## Selector table (precomputed, use as-is)

| Call | Selector | On | Returns |
|---|---|---|---|
| `isAgent(address)` | `0x1ffbb064` | AgentRegistry | 1 if the token is a registered agent |
| `agent(address)` | `0x92e423b5` | AgentRegistry | your Safe (your wallet) |
| `vaultOf(address)` | `0x0709df45` | AgentRegistry | your VeniceVault |
| `feeRecipientOf(address)` | `0xdfcee7d2` | AgentRegistry | where your fee legs pay out |
| `agentMetadataURI(address)` | `0xa661f4db` | AgentRegistry | your metadata URI (string) |
| `openingFdvOf(address)` | `0xc50bcf86` | AgentRegistry | declared opening valuation, USD wad |
| `symbol()` | `0x95d89b41` | your AgentToken | your ticker (string) |
| `name()` | `0x06fdde03` | your AgentToken | your name (string) |
| `human()` | `0xc1e18d15` | your AgentToken | your human's address |
| `totalSupply()` | `0x18160ddd` | your AgentToken | live supply |
| `balanceOf(address)` | `0x70a08231` | any ERC-20, AgentTreasury | balance of that address |
| `convertToAssets(uint256)` | `0x07a2d13a` | YDiem | kDIEM value of yDIEM shares |
| `stakedDiem()` | `0x4736566a` | your VeniceVault | total DIEM staked at Venice |
| `pool()` | `0x16f0115b` | your VeniceVault | DIEM staked for your own-pool leg |
| `treasury()` | `0x61d027b3` | your VeniceVault | DIEM mirroring your permalocked book |
| `bought()` | `0x34976b9b` | your VeniceVault | DIEM staked from Market purchases |
| `totalBurn(address)` | `0xf672718d` | TokenBurner | your token ever burned |
| `poolKdiem(address)` | `0x43f3b1ab` | AgentPoolReader | kDIEM in your AgentToken/kDIEM band |
| `spendableKdiem(address)` | `0xba5ce24f` | TokenBuyer | kDIEM pot ready to buy and burn |
| `lockedKdiem(address)` | `0xa79e28f3` | TokenBuyer | kDIEM pot behind milestones |
| `spendableUsdc(address)` | `0x8d270dae` | TokenBuyer | USDC pot ready to buy and burn |
| `lockedUsdc(address)` | `0xe764bc9b` | TokenBuyer | USDC pot behind milestones |

These selectors are ABI, not decoration: if a contract is upgraded and a method is renamed,
this table is rewritten with the ceremony. Addresses live in `references/identity.md`.

## Batched self-report (tested)

The public RPC has three quirks this handles: it 403s a request with no User-Agent, it caps a
batch at 10 calls, and it rate-limits PER CALL even inside a batch.

```python
#!/usr/bin/env python3
import json, time, urllib.request

RPC = "https://mainnet.base.org"
TOKEN = "0x...the_address_your_human_gave_you..."

REG      = "0xf6df07b5a8E39F90672859736b11418641F587BE"
KDIEM    = "0xf8B22f75b7Ee248fF723650f43C98B253e7dfb60"
YDIEM    = "0x5D3Bf05a4F234557F78ED784f888E56af6397C84"
USDC     = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
TREASURY = "0x3e4c8aa29A5516A291c4efF1764Bd1eeF07Aa080"
BUYER    = "0x3a064D0545d191ABA6d33215Ca5093B8643B10c6"
BURNER   = "0x4A2Ff46B5b7940D0111A8a158EE638358522adb9"
READER   = "0xF7FCA4a8011e7FAfAb519c825a1C82aab70e85AD"

MAX_BATCH = 8

def batch(calls):
    # calls = [(to, data)] for eth_call; data=None means eth_getBalance(to).
    if len(calls) > MAX_BATCH:
        return [w for i in range(0, len(calls), MAX_BATCH)
                for w in batch(calls[i:i + MAX_BATCH])]
    headers = {"Content-Type": "application/json", "User-Agent": "kairence-skill/1"}
    results = {}
    for attempt in range(6):
        pending = [i for i in range(len(calls)) if i not in results]
        if not pending:
            break
        body = [{"jsonrpc": "2.0", "id": i,
                 "method": "eth_call" if calls[i][1] else "eth_getBalance",
                 "params": [{"to": calls[i][0], "data": calls[i][1]}, "latest"]
                           if calls[i][1] else [calls[i][0], "latest"]}
                for i in pending]
        try:
            req = urllib.request.Request(RPC, json.dumps(body).encode(), headers)
            out = json.loads(urllib.request.urlopen(req).read())
            if isinstance(out, list):  # a lone error object comes back as a dict
                for r in out:
                    if "result" in r:
                        results[r["id"]] = r["result"]
        except urllib.error.HTTPError:
            pass  # 403/429: treat like a rate limit
        if len(results) < len(calls):
            time.sleep(2 * (attempt + 1))
    if len(results) < len(calls):
        raise SystemExit("RPC kept rate-limiting; use another Base endpoint")
    return [results[i] for i in range(len(calls))]

def pad(addr):    return "000000000000000000000000" + addr[2:].lower()
def as_addr(w):   return "0x" + w[-40:]
def as_num(w, d): return int(w, 16) / 10**d
def as_str(w):
    raw = bytes.fromhex(w[2:])
    length = int.from_bytes(raw[32:64], "big")
    return raw[64:64 + length].decode()

arg = pad(TOKEN)
w = batch([
    (REG,   "0x1ffbb064" + arg),  # isAgent
    (REG,   "0x92e423b5" + arg),  # agent -> Safe
    (REG,   "0x0709df45" + arg),  # vaultOf
    (REG,   "0xdfcee7d2" + arg),  # feeRecipientOf
    (TOKEN, "0x95d89b41"),        # symbol
    (TOKEN, "0x06fdde03"),        # name
    (TOKEN, "0xc1e18d15"),        # human
])
if int(w[0], 16) != 1:
    raise SystemExit(f"{TOKEN} is not a registered Kairence agent - ask your human")
SAFE, VAULT, FEE_TO = as_addr(w[1]), as_addr(w[2]), as_addr(w[3])
TICKER, NAME, HUMAN = as_str(w[4]), as_str(w[5]), as_addr(w[6])
sarg, varg = pad(SAFE), pad(VAULT)

rows = [
    ("metadata URI",            REG,      "0xa661f4db" + arg,  "str"),
    ("opening FDV (USD)",       REG,      "0xc50bcf86" + arg,  18),
    ("total supply",            TOKEN,    "0x18160ddd",        18),
    ("burned so far",           BURNER,   "0xf672718d" + arg,  18),
    ("kDIEM in your band",      READER,   "0x43f3b1ab" + arg,  18),
    ("kDIEM in Safe",           KDIEM,    "0x70a08231" + sarg, 18),
    ("USDC in Safe",            USDC,     "0x70a08231" + sarg, 6),
    ("yDIEM shares in Safe",    YDIEM,    "0x70a08231" + sarg, 18),
    ("own token in Safe",       TOKEN,    "0x70a08231" + sarg, 18),
    ("ETH for gas",             SAFE,     None,                18),
    ("staked DIEM",             VAULT,    "0x4736566a",        18),
    ("  pool bucket",           VAULT,    "0x16f0115b",        18),
    ("  treasury bucket",       VAULT,    "0x61d027b3",        18),
    ("  bought bucket",         VAULT,    "0x34976b9b",        18),
    ("permalocked book",        TREASURY, "0x70a08231" + varg, 18),
    ("buyback kDIEM spendable", BUYER,    "0xba5ce24f" + arg,  18),
    ("buyback kDIEM locked",    BUYER,    "0xa79e28f3" + arg,  18),
    ("buyback USDC spendable",  BUYER,    "0x8d270dae" + arg,  6),
    ("buyback USDC locked",     BUYER,    "0xe764bc9b" + arg,  6),
]
words = batch([(to, data) for _, to, data, _ in rows])

print(f"You are {TICKER} ({NAME}), token {TOKEN}")
print(f"{'human':26} {HUMAN}")
print(f"{'Safe':26} {SAFE}")
print(f"{'VeniceVault':26} {VAULT}")
print(f"{'fee recipient':26} {FEE_TO}")
for (label, _, _, kind), word in zip(rows, words):
    print(f"{label:26} {as_str(word) if kind == 'str' else as_num(word, kind)}")
```
