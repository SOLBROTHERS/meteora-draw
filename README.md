# Meteora campaign: grand-draw verifier

The Meteora liquidity campaign on
[campaign.ecosystemcall.com/meteora](https://campaign.ecosystemcall.com/meteora)
(29 Oct – 5 Nov 2026, close at 14:00 UTC) ends with one ticket-weighted grand
draw. This repository holds what anyone needs to reproduce the winner. It stays
empty until the close; the files listed below are added after the draw.

## The verifier

The campaign page's final-draw section is the verifier. It reads
`/meteora/draw-entries.json`, fetches the committed block from Solana mainnet
and recomputes the winner in your browser. The computation, step by step:

1. **Seed.** Before the close a mainnet slot is committed (`committed_slot`).
   The draw block is the first block produced at or after that slot
   (`block_slot`); the seed is its base58 block hash (`block_hash`). Nobody can
   know that hash before the block exists.
2. **PRNG.** SHA-256 over the UTF-8 bytes of the hash string. The first 4 bytes
   of the digest, read big-endian as an unsigned 32-bit integer, seed a
   mulberry32 generator (`rng_method`: `mulberry32-sha256`).
3. **Pick.** One pick per winner, without replacement, over `entries` **in file
   order**: `target = rng() × total tickets`, walk the list subtracting each
   entry's tickets, and the first entry with `target < tickets` wins. Remove it
   and repeat for the next winner.

mulberry32, for reference:

```js
function mulberry32(seed) {
  let a = seed >>> 0;
  return function () {
    a = (a + 0x6d2b79f5) >>> 0;
    let t = a;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

## Files (added at the close)

- `draw-entries.json`: the entries file, identical to the one the page serves.
- `draw-record.json`: the draw as recorded (seed, winner, the frozen entries it
  ran over, time).
- `flagged-wallets.json`: every wallet flagged under Terms §7 (one person, one
  entry) during the campaign, as a JSON array of addresses. The ones still in
  the pool at the close are in `excluded` and shown greyed at the bottom of the
  pool on the page, which links this file from a count line.

## `draw-entries.json`

| Key | Meaning |
|---|---|
| `committed_slot` | The mainnet slot committed before the close |
| `block_slot` | The draw block: the first produced block at or after `committed_slot` |
| `block_hash` | That block's base58 hash, which is the seed |
| `campaign_end_utc` | The close, `2026-11-05T14:00:00Z` |
| `draw_time_utc` | When the draw was recorded |
| `winners` | Number of winners |
| `prize_usd_each` | Prize per winner, in USD |
| `rng_method` | `mulberry32-sha256` |
| `total_wallets`, `total_tickets` | Totals over `entries` |
| `entries` | `[{"wallet": "<base58>", "tickets": N}]`: the frozen pool, sorted by tickets (highest first), then by wallet. The order is part of the draw. |
| `excluded` | Addresses removed from the pool under Terms §7, sorted |

A wallet is in `entries` when its last liquidity snapshot before the close shows
active liquidity and it holds at least one unspent ticket earned before the
close. Excluded wallets and the team's test wallets are never entries.
