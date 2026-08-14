# CCIP-028

## Preamble

| CCIP Number   | 028                                            |
| ------------- | ---------------------------------------------- |
| Title         | MiamiCoin Fair Redemption with sBTC Rewards    |
| Author(s)     | Rapha (github.com/Rapha-btc)                   |
| Consideration | Governance, Economic, Technical                |
| Type          | Standard                                       |
| Status        | Draft                                          |
| Created       | 2026-08-13                                     |
| License       | BSD-2-Clause                                   |
| Supplements   | CCIP-026, CCIP-027                             |

## Introduction

CCIP-027 re-entered the MiamiCoin treasury into Proof of Transfer under PoX-5.
Rewards now arrive as sBTC, while the burn-to-exit mechanism established by
CCIP-026 pays in STX. Until those two are connected, sBTC accumulates in the
rewards treasury and no MIA is retired.

This CCIP proposes to swap the sBTC rewards to STX on-chain at a price committed
before settlement, route that STX through a fair order book that lets holders
name their own exit price, and burn every MIA the book acquires.

Nothing about how a holder exits changes. Offers are posted at or below par, the
cheapest offers fill first, offers may be cancelled at any time, and sellers are
paid in STX.

Two things do change. The redemption ratio is corrected to account for the MIA
already burned, and the below-par spread is burned rather than retained.

## Problem Statement

### The rewards cannot reach the redemption mechanism

CCIP-027 moved 10,241,497.066794 STX from `ccd002-treasury-mia-mining-v3` into
`ccd014-pox5-staking-mia` and staked it from reward cycle 141 for 96 cycles,
unlocking at cycle 237. The treasury is intact and earning.

The reward path is already permissionless:

```
pox-5 -> ccd014-pox5-staking-mia -> forward-rewards -> ccd002-treasury-mia-rewards-v3
```

`forward-rewards` takes no arguments, may be called by anyone, and sweeps the
full sBTC balance. CCIP-027 additionally called `set-allowed` for `sbtc-token` on
the rewards treasury, with the stated intent that a future proposal would
withdraw it. This is that proposal.

What is missing is the conversion from sBTC to the STX the redemption ratio is
denominated in.

### The redemption ratio is stale

`ccd013-burn-to-exit-mia` pays 1710, that is 1,710 STX per 1,000,000 MIA. That
was correct when it was set. Every exit round since has burned MIA, shrinking
supply while the treasury is untouched, so the true ratio has risen and holders
redeeming today receive less than their claim is worth.

Measured on-chain on 2026-08-13:

| Input                          | Value                     |
| ------------------------------ | ------------------------- |
| MIA v1 supply                  | 260,130,901 MIA           |
| MIA v2 supply                  | 4,754,074,853.491373 MIA  |
| Combined supply                | 5,014,205,754 MIA         |
| Treasury, now in ccd014        | 10,241,497.066794 STX     |
| Ratio by the DAO's formula     | 2042                      |
| Ratio currently paid           | 1710                      |

19.4% more than holders are being paid. The figure moves with every burn, so what
should be ratified is the formula, not the number above.

The ratio cannot be corrected in place. `ccd013.initialize-redemption` sets
`redemptions-enabled` permanently and the contract exposes no setter, so 1710 is
final there. Its treasury reader also still points at
`ccd002-treasury-mia-mining-v3`, which CCIP-027 emptied into
`ccd014-pox5-staking-mia`; it reads 0 STX and would revert on
`ERR_GETTING_REDEMPTION_BALANCE`. Refreshing par therefore requires a replacement
redemption extension, not a configuration change.

## Specification

### Overview

```
  pox-5 rewards (sBTC)
          |
          v
  ccd014-pox5-staking-mia
          |  forward-rewards      (permissionless, already deployed)
          v
  ccd002-treasury-mia-rewards-v3
          |  withdraw-ft          (authorised by this proposal)
          v
  swap extension                  (new)
          |  open-rfq -> fix-price -> fulfill
          v
        STX
          |
          v
  redemption contract  <------  fair order book, STX-denominated (new)
          |                            ^
          |  redeem at par             |  holders post offers at or below par
          v                            |
      MIA burned  ---------------------
```

### 1. Correct the redemption ratio

A replacement redemption extension, reading the treasury from
`ccd014-pox5-staking-mia` (locked plus unlocked) so the existing formula returns a
correct figure without a hand-entered number. `ccd013` is left untouched and
remains readable for historical redemptions.

Par must stay fixed while offers are resting, or sellers cannot price against it.
But it should not stay fixed forever, because each round of burns raises what the
remaining holders are owed. The natural cadence is therefore **once per round**:
refresh par when a cycle's rewards have been fully consumed and the book is
between runs, then hold it steady for the next round.

Whether that refresh is automatic or a governance action is left open below.

### 2. Swap extension

A new DAO extension that holds sBTC and converts it to STX through a
request-for-quote desk. Required properties:

- **Permissionless trigger, no arguments.** Following the pattern of
  `forward-rewards`: the budget is the balance, and no individual needs to be
  available for rewards to reach holders.
- **Hard-coded destination.** The resulting STX can only be sent to the
  redemption contract. A caller cannot redirect it.
- **A reclaim path.** RFQ settlement is two-phase: a quote is opened and fixed,
  then fulfilled in a later transaction. If no market maker fulfils, the
  extension must recover its sBTC.

A quoted desk is specified rather than an automated market maker because the
price is committed before it settles, leaving no pool state for a sandwich to
exploit. A recurring swap of predictable size on a predictable schedule against a
public pool is an invitation to be front-run, which is the same failure the fair
order book was built to remove from the redemption side. Solving it in one place
and reintroducing it in the other would be inconsistent.

The desk maintains an on-chain allowlist with a proposal and cooldown before
confirmation. The swap extension's contract principal is the client. Counterparty
diligence applies to the entity behind the DAO, not to the contract.

### 3. Fair order book, all MIA burned

An STX-denominated order book, adapted from the implementation currently in
production, with one behavioural change: the below-par spread is burned rather
than retained.

Preserved as proven: the sorted offer book, insertion-sort placement, partial
fills, cheapest-first settlement, cancellation at any time, and the cap that
prevents a settler from profiting above par.

The accounting consequence of burning the spread is that a fill retires a claim
worth `ratio * amount` on the treasury while paying only the seller's discounted
ask. Because payment comes from staking yield and never debits the STX treasury,
the treasury `T` is unchanged and only supply `S` shrinks:

```
ratio_before = T / S           ratio_after = T / (S - burned)
```

The full retired claim therefore accrues to holders who did not sell. The
discount does not create that surplus; it determines how much MIA is retired per
STX of yield, which is why the book fills cheapest first.

### 4. Related Work

The mechanism described in section 3 has been running in production against
`ccd013-burn-to-exit-mia` since cycle 138, which provides operating evidence
rather than an untested design. Lifetime figures as of 2026-08-13:

| Metric                       | Value             |
| ---------------------------- | ----------------- |
| MIA cleared out of the queue | 42,903,974.60 MIA |
| STX paid to sellers          | 62,167.14 STX     |
| Offers currently resting     | 50                |
| MIA currently on the book    | 49,769,714.87 MIA |

The most recent settlement, on 2026-08-11, cleared 16,239,028 MIA in a single
transaction, paid three sellers at exactly the prices they had posted, and left
the redemption treasury at zero. The settler took no profit, being capped at par
by construction.

That implementation retains the spread rather than burning it. Section 3 changes
this.

## Disclosure

The request-for-quote desk the author intends to propose for section 2 is built
and operated by the author. It received a Stacks Endowment grant, and CityCoins
volume routed through it would assist in bootstrapping a market maker on that
desk. The author benefits if this route is chosen.

The design does not depend on that specific venue. Any venue committing a price
before settlement satisfies the requirement, and the author is willing to
implement an alternative if the community prefers one. This is disclosed so the
recommendation may be weighed accordingly, not to pre-empt the choice.

## Rationale

**Why not denominate the order book in sBTC?** Par is not a market price. It is
treasury arithmetic: a known quantity of STX behind a known supply of MIA,
verifiable by anyone. Nothing backs MIA in sBTC, so an sBTC par would have to be
manufactured from an external price rather than derived from the treasury.
Holders would redeem against an estimate instead of against the backing, which
changes what the guarantee means rather than merely its denomination.

**Why not distribute sBTC directly to redeemers?** The same problem relocated. It
requires a MIA/sBTC price at distribution time, which must come from an oracle,
and an oracle on a known schedule carries the same manipulation surface as a swap
on a known schedule.

**Why burn the spread rather than seed liquidity?** Seeding a MIA/sBTC pool would
create a second venue for MIA and has genuine value. It also requires the DAO to
hold and manage a position, decide who supplies the paired side, and accept
impermanent loss. Burning requires no ongoing decision and distributes the
benefit to every holder in proportion rather than to whoever participates. If the
community prefers the liquidity route, it should be proposed separately and voted
on its own merits.

## Open Questions

The following are deliberately left to the community rather than decided here.

1. **Par refresh.** Recompute between rounds, as proposed, or set once and
   re-ratify by vote when the community chooses? If between rounds, should the
   extension do it automatically or should each refresh be a proposal?
2. **Spread.** Burn it as proposed, or continue capturing it to seed a MIA/sBTC
   pool?
3. **Swap cadence.** Per reward cycle, or accumulate and swap on a size
   threshold? Larger, less frequent swaps price better but leave sBTC idle
   longer.
4. **Venue.** Ratify a named desk, or specify the required property (a price
   committed before settlement) and let the implementation follow?

## Backwards Compatibility

`ccd013-burn-to-exit-mia` and the existing order book remain deployed and
readable. Historical redemptions are unaffected.

Offers currently resting are unaffected by this proposal and may be cancelled by
their owners at any time. Migration of resting offers to a new book, should one
be deployed, must be explicit and opt-in rather than automatic.

## Activation

Execution through `ccd001-direct-execute` with 3 of 5 approver signals, following
the pattern established by CCIP-027: the proposal enables extensions, configures
them, and authorises the treasury withdrawal, but does not perform the swap or
the settlement itself. Those remain permissionless calls made separately, so
execution timing does not depend on approver availability.

## Reference Implementation

To follow, pending community feedback on the open questions above. Contract work
is deliberately deferred so the mechanism can be debated before it is built.
