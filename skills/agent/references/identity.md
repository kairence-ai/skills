# Identity and money - everything readable from one address

Read-only: no private key, no signature, no transaction. One input, `TOKEN`; every other
address here is a protocol singleton, identical for every agent.

Use **viem** for this. It takes human-readable signatures, encodes and decodes for you, and
batches every read into a single `multicall` - so the whole report below is two network calls
instead of twenty, with no rate-limit dance. If you have no Node at all, read
`references/raw-rpc.md` for the zero-dependency path.

```bash
npm install viem@2.55.16   # pinned; already present if you installed the journal's turbo-sdk
```

## Address book (Base mainnet)

| Contract | Address | What it answers about you |
|---|---|---|
| AgentRegistry | `0xf6df07b5a8E39F90672859736b11418641F587BE` | the birth record: your Safe, vault, metadata, opening FDV |
| KDiem (kDIEM) | `0xf8B22f75b7Ee248fF723650f43C98B253e7dfb60` | liquid kDIEM balance |
| YDiem (yDIEM) | `0x5D3Bf05a4F234557F78ED784f888E56af6397C84` | yDIEM shares (ERC-4626, NOT kDIEM) |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | liquid USDC balance |
| AgentTreasury | `0x3e4c8aa29A5516A291c4efF1764Bd1eeF07Aa080` | your permalocked kDIEM book |
| TokenBuyer | `0x3a064D0545d191ABA6d33215Ca5093B8643B10c6` | the buyback pots earmarked for your token |
| TokenBurner | `0x4A2Ff46B5b7940D0111A8a158EE638358522adb9` | how much of your token was burned |
| AgentPoolReader | `0xF7FCA4a8011e7FAfAb519c825a1C82aab70e85AD` | the kDIEM sitting in your own band |

Your own addresses - Safe, VeniceVault, fee recipient - are NOT in this table. You read them
from the registry, keyed by your token.

## What each number means

- **Your Safe** (`agent(TOKEN)`) is your wallet: the agent IS its Safe. Liquid money lives
  there - kDIEM, USDC, yDIEM shares, your own token.
- **Your VeniceVault** (`vaultOf(TOKEN)`) is your Venice staking identity. `stakedDiem()` is
  the total, split into `pool()` (staked against your own band), `treasury()` (mirrors your
  permalocked book) and `bought()` (delivered from Market purchases you made). Staked DIEM buys
  you inference capacity; it is not liquid, and the vault never holds kDIEM - these are DIEM at
  Venice.
- **Your permalocked book** is `balanceOf(VAULT)` on AgentTreasury - keyed by your VAULT, not
  your token. This kDIEM backs you forever and is never spendable: read it as weight, not money.
- **Supply and burns**: at birth supply is exactly 1,000,000,000, and `totalSupply()` plus
  `totalBurn(TOKEN)` always sum back to it.
- **Buyback pots** are earmarked to buy and burn YOUR token. Informational: the protocol spends
  them, you do not.
- **yDIEM balances are SHARES**, not kDIEM - price them with `convertToAssets(shares)`.

## Full self-report (tested)

Two multicalls; the second keys off the addresses the first returned.

```js
import {createPublicClient, http, formatUnits, parseAbi} from 'viem';
import {base} from 'viem/chains';

const TOKEN = '0x...the address your human gave you...';

const REG      = '0xf6df07b5a8E39F90672859736b11418641F587BE';
const KDIEM    = '0xf8B22f75b7Ee248fF723650f43C98B253e7dfb60';
const YDIEM    = '0x5D3Bf05a4F234557F78ED784f888E56af6397C84';
const USDC     = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
const TREASURY = '0x3e4c8aa29A5516A291c4efF1764Bd1eeF07Aa080';
const BUYER    = '0x3a064D0545d191ABA6d33215Ca5093B8643B10c6';
const BURNER   = '0x4A2Ff46B5b7940D0111A8a158EE638358522adb9';
const READER   = '0xF7FCA4a8011e7FAfAb519c825a1C82aab70e85AD';

const abi = parseAbi([
  'function isAgent(address) view returns (bool)',
  'function agent(address) view returns (address)',
  'function vaultOf(address) view returns (address)',
  'function feeRecipientOf(address) view returns (address)',
  'function agentMetadataURI(address) view returns (string)',
  'function openingFdvOf(address) view returns (uint256)',
  'function symbol() view returns (string)',
  'function name() view returns (string)',
  'function human() view returns (address)',
  'function totalSupply() view returns (uint256)',
  'function balanceOf(address) view returns (uint256)',
  'function stakedDiem() view returns (uint256)',
  'function pool() view returns (uint256)',
  'function treasury() view returns (uint256)',
  'function bought() view returns (uint256)',
  'function totalBurn(address) view returns (uint256)',
  'function poolKdiem(address) view returns (uint256)',
  'function spendableKdiem(address) view returns (uint256)',
  'function lockedKdiem(address) view returns (uint256)',
  'function spendableUsdc(address) view returns (uint256)',
  'function lockedUsdc(address) view returns (uint256)',
]);

const client = createPublicClient({chain: base, transport: http('https://mainnet.base.org')});
const read = (contracts) => client.multicall({allowFailure: false, contracts});

// Round 1: who you are. Everything after this keys off your Safe and your vault.
const [registered, safe, vault, feeTo, ticker, name, human] = await read([
  {address: REG, abi, functionName: 'isAgent', args: [TOKEN]},
  {address: REG, abi, functionName: 'agent', args: [TOKEN]},
  {address: REG, abi, functionName: 'vaultOf', args: [TOKEN]},
  {address: REG, abi, functionName: 'feeRecipientOf', args: [TOKEN]},
  {address: TOKEN, abi, functionName: 'symbol'},
  {address: TOKEN, abi, functionName: 'name'},
  {address: TOKEN, abi, functionName: 'human'},
]);
if (!registered) throw new Error(`${TOKEN} is not a registered Kairence agent - ask your human`);

// Round 2: everything else, in ONE call.
const rows = [
  ['metadata URI',            {address: REG,      abi, functionName: 'agentMetadataURI', args: [TOKEN]}, 'str'],
  ['opening FDV (USD)',       {address: REG,      abi, functionName: 'openingFdvOf',     args: [TOKEN]}, 18],
  ['total supply',            {address: TOKEN,    abi, functionName: 'totalSupply'},                     18],
  ['burned so far',           {address: BURNER,   abi, functionName: 'totalBurn',        args: [TOKEN]}, 18],
  ['kDIEM in your band',      {address: READER,   abi, functionName: 'poolKdiem',        args: [TOKEN]}, 18],
  ['kDIEM in Safe',           {address: KDIEM,    abi, functionName: 'balanceOf',        args: [safe]},  18],
  ['USDC in Safe',            {address: USDC,     abi, functionName: 'balanceOf',        args: [safe]},   6],
  ['yDIEM shares in Safe',    {address: YDIEM,    abi, functionName: 'balanceOf',        args: [safe]},  18],
  ['own token in Safe',       {address: TOKEN,    abi, functionName: 'balanceOf',        args: [safe]},  18],
  ['staked DIEM',             {address: vault,    abi, functionName: 'stakedDiem'},                      18],
  ['  pool bucket',           {address: vault,    abi, functionName: 'pool'},                            18],
  ['  treasury bucket',       {address: vault,    abi, functionName: 'treasury'},                        18],
  ['  bought bucket',         {address: vault,    abi, functionName: 'bought'},                          18],
  ['permalocked book',        {address: TREASURY, abi, functionName: 'balanceOf',        args: [vault]}, 18],
  ['buyback kDIEM spendable', {address: BUYER,    abi, functionName: 'spendableKdiem',   args: [TOKEN]}, 18],
  ['buyback kDIEM locked',    {address: BUYER,    abi, functionName: 'lockedKdiem',      args: [TOKEN]}, 18],
  ['buyback USDC spendable',  {address: BUYER,    abi, functionName: 'spendableUsdc',    args: [TOKEN]},  6],
  ['buyback USDC locked',     {address: BUYER,    abi, functionName: 'lockedUsdc',       args: [TOKEN]},  6],
];
const values = await read(rows.map(([, call]) => call));
const gas = await client.getBalance({address: safe});

console.log(`You are ${ticker} (${name}), token ${TOKEN}`);
for (const [label, value] of [['human', human], ['Safe', safe], ['VeniceVault', vault], ['fee recipient', feeTo]]) {
  console.log(label.padEnd(26), value);
}
rows.forEach(([label, , decimals], i) => {
  console.log(label.padEnd(26), decimals === 'str' ? values[i] : formatUnits(values[i], decimals));
});
console.log('ETH for gas'.padEnd(26), formatUnits(gas, 18));
```

## One number, not the whole report

Most questions want a single read. `readContract` is the whole call:

```js
const staked = await client.readContract({
  address: vault, abi, functionName: 'stakedDiem',
});
console.log(formatUnits(staked, 18), 'DIEM staked');
```

## Known-good check

Before your own launch exists, run the report against the first agent:
`TOKEN = 0xca18A528Ea897040f715edC92e6e4572780c5ca1`. On 2026-08-18 it printed ticker
`KAI (Kairence)`, opening FDV 100000, supply 973,078,878.38 with 26,921,121.62 burned (summing
to exactly 1B) and 31.813 staked DIEM. Exact numbers drift daily; the shape should match.

If a call reverts or returns empty, you are on the wrong chain or the wrong address - do not
retry blindly.
