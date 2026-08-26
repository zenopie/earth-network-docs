---
sidebar_position: 3
---

# Emission

Earth issues **4 ERTH per second, permanently**. This is a fixed rate. It is not
a halving schedule and not a governance parameter.

The rate is constant. The supply that it adds to increases. The inflation
*rate* therefore decreases without intervention: approximately 5% in year one,
and below 2.5% by year twenty. There is no schedule to maintain.

## The four pillars

The chain divides the 4 ERTH/sec into **four equal pillars of 1 ERTH/sec**. Two
pillars use personhood as the weight and two use capital. Two pay individuals
and two pay destinations that voters select.

| Pillar | Weight | Recipient |
| --- | --- | --- |
| Staking rewards | Capital | Individuals, in proportion to stake |
| ANML buyback-and-burn | Personhood | All users, through ANML destruction |
| Caretaker fund | Personhood | Options selected at one vote for each human |
| Groundworks fund | Capital | Options selected by stake |

### Staking rewards

This is standard proof-of-stake. Delegate ERTH to a validator. You then earn a
part of 1 ERTH/sec in proportion to your stake, less the commission of that
validator.

Unbonding takes **21 days**.

### ANML buyback-and-burn

Each second, the chain mints 1 ERTH, buys ANML with it on the internal exchange,
and destroys the ANML.

This gives ANML a price floor. A registered human claims 1 ANML each day and can
sell it. The chain is a permanent buyer, funded by one quarter of all issuance.

### The two funds

The **Caretaker fund** and the **Groundworks fund** each direct 1 ERTH/sec to
options that holders select.

The two funds differ in who selects. The Caretaker fund uses **one vote for each
human**. Holdings have no effect. The Groundworks fund uses **bonded stake**.

Any user can add an option to either fund for a small fee. Divide your vote
between options by percentage. You can change the division at any time. Rewards
accrue continuously to the current selection.

The Groundworks fund includes an **emergency fund** option. This option pays the
community pool of the chain. Stake that points at it adds ERTH to the pool at
each block. Governance then decides the use of the pool. Stakers can therefore
build a reserve before they know its purpose.

## The pre-mine

The chain started with **2,522,880,000 ERTH**. This is exactly twenty years at
the rate of 4 ERTH/sec, minted at genesis.

The pre-mine is *additive*. It is not a substitute for issuance. The chain
continues to issue in addition to it. At year twenty the total is approximately
5.05 billion, not 2.52 billion.

The pre-mine was divided into three equal parts:

- One third started the ANML/ERTH pool on the exchange.
- One third pays the bidders in the liquidity auction.
- One third is paired with the amount that those bidders raise, to start a second
  pool.

All of the pre-mine went to liquidity. **No person received an allocation.**

## Protocol-owned liquidity is temporary

The chain owns this liquidity at the start. It **retires the liquidity to zero
over ten years**.

A market requires active management. The incentives of a liquidity provider
differ from the incentives of an ERTH holder. The protocol therefore starts the
market and then withdraws from it. Providers who manage the position replace the
protocol. The share of rewards for those providers increases each year.

The ANML/ERTH position burns both assets as it retires. The auction pool burns
only its ERTH. The other asset stays in the pool, which then uses it to buy ERTH
from the market over time.

## Burning

Two mechanisms destroy ERTH continuously:

- **Half of each swap fee.** The fee is 0.3% for each hop, charged in ERTH. Half
  stays with the liquidity providers. The chain destroys the other half.
- **Half of each transaction fee.** Gas uses the same division. The chain
  destroys half and pays half to validators and their delegators through the
  normal staking payout. If the division is not even, the chain burns the extra
  unit.

Activity therefore reduces supply while issuance increases it at a constant rate.

The paid half is the only part of validator revenue that responds to network
use. The staking pillar is a fixed 1 ERTH/sec at all levels of activity. This is
most important for registration. Verification of a passport proof uses
significant CPU time, and validators pay that cost at each registration.
