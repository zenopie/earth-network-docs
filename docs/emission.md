---
sidebar_position: 3
---

# Emission

Earth issues up to **4 ERTH a second, for ever**: 126,144,000 ERTH a year. The
rate is fixed in the software. It does not halve, and no vote can change it.

Because the rate is fixed while the supply grows, inflation falls on its own.
Nobody has to maintain a schedule.

## The four pillars

The 4 ERTH a second is split into four pillars of 1 ERTH a second each. Capital
decides two of them and people decide two. In each pair, one pays individuals
by rule and one is a fund that voters point.

| Pillar | Who decides | Who receives |
| --- | --- | --- |
| Staking rewards | Capital | Stakers, in proportion to bonded ERTH |
| ANML buyback and burn | People | Every ANML holder, through a permanent buyer |
| Caretaker fund | People, one vote each | Whatever registered humans vote for |
| Groundworks fund | Capital, by bonded stake | Whatever stakers vote for |

A fund that nobody has voted on issues nothing. Emission that nobody chose is
never created, so before the first human registers the Caretaker fund is silent.

### Staking rewards

Standard proof of stake. Delegate ERTH to a validator and earn a share of 1 ERTH
a second in proportion to your stake, less the validator's commission. Unbonding
takes **21 days**.

### ANML buyback and burn

Every second the chain issues 1 ERTH, uses it to buy ANML on its own exchange,
and destroys the ANML. Registered humans each claim 1 ANML a day and can sell
it, and the chain is always there to buy, funded by a quarter of all issuance.

### The two funds

The **Caretaker fund** and the **Groundworks fund** each pay 1 ERTH a second to
the options their voters choose. You split your vote across options by
percentage and can change it at any time. Rewards follow your current split
continuously.

The funds differ in who chooses and who may add options:

- **Caretaker** is one vote per registered human. Anyone can add an option for a
  small fee, because holdings buy no extra votes here.
- **Groundworks** is weighted by bonded stake. Only governance can add an option.
  If anyone could, every staker's best move would be to list their own address
  and vote for it, and the fund would just be a second staking reward. Registered
  humans can vote to **remove** a Groundworks option on their own, with no stake
  vote, so that they can end a bad option and not merely object to it.

The Groundworks fund includes an **emergency fund** option that pays the chain's
community pool. Stake pointed at it builds a reserve before anyone knows what it
will be for. Governance decides how the pool is spent.

The first Caretaker option is the **registration reward**. It pays each new
registrant, and their referrer if they have one, from a pool funded at genesis
and topped up by the Caretaker votes pointed at it.

## Genesis supply

The chain started with **2,522,880,000 ERTH**, twenty years of issuance at
4 ERTH a second. It was split into four quarters of **630,720,000 ERTH**:

| Quarter | Where it went |
| --- | --- |
| 1 | The ANML/ERTH pool on the exchange |
| 2 | The ERTH that liquidity auction bidders receive |
| 3 | The ERTH paired with what those bidders pay, to open a second pool |
| 4 | The registration reward pool |

Genesis also placed **21,000 ERTH** in three ordinary accounts: 1,000 to bond the
first validator and 10,000 each to two others. No other individual received an
allocation, and there are no founder, team or investor tokens.

## Protocol-owned liquidity retires in five years

The chain owns its two pools at the start, and burns them down on a straight line
over **five years**, each from the day it opens. A market needs active
management, and a liquidity provider's incentives are not a token holder's. So
the protocol opens the market and then steps back, and independent providers
take its place. Their share of LP rewards grows every year.

The ANML/ERTH pool burns both assets as it retires. The auction pool burns only
its ERTH. The other asset stays in the pool, where it buys ERTH from the market
over time.

## Supply over time

Two of the four quarters sit in these pools, so five years of retirement burns
1,261,440,000 ERTH while issuance adds 630,720,000. **Supply falls by about
630 million ERTH over the first five years**, then grows at the fixed rate.

| Point | Supply, before fee burns | Yearly issuance as a share of supply |
| --- | --- | --- |
| Genesis | 2.52 billion | Supply shrinks about 5% a year while pools retire |
| Year 5 | 1.89 billion | 6.7% |
| Year 20 | 3.78 billion | 3.3% |

These assume all four pillars are voted on from the start and both pools open at
launch. Real supply is lower, because of the two burns below.

## Burns

Two mechanisms destroy ERTH continuously:

- **Half of every swap fee.** The fee is 0.3% a hop, charged in ERTH. Half stays
  with liquidity providers and half is burned.
- **Half of every transaction fee.** The other half goes to validators and their
  delegators. When a fee does not split evenly, the extra unit is burned.

So activity shrinks supply while issuance grows it at a fixed rate.

The paid half of fees is the only part of validator income that grows with use:
staking rewards are a fixed 1 ERTH a second whatever happens. That matters most
for registration, because verifying a passport proof costs validators real CPU
time.
