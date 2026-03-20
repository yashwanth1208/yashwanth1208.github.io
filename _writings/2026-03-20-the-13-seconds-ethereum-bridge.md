---
layout: post
title: "The 13 Seconds Ethereum Bridge"
date: 2026-03-20
tags: [bridges, cross-chain, interoperability, web3, blockchain]
in_archive: false
---

<details>
<summary><strong>Table of Contents</strong></summary>

<ul>
<li><a href="#why-bridges-wait-so-long-today">Why Bridges Wait So Long Today</a></li>
<li><a href="#how-ethereum-finality-actually-works">How Ethereum Finality Actually Works</a></li>
<li><a href="#enter-fcr">Enter FCR</a></li>
<li><a href="#who-benefits-and-how">Who Benefits and How</a></li>
<li><a href="#the-implementation-ask-is-almost-nothing">The Implementation Ask is Almost Nothing</a></li>
<li><a href="#the-tradeoffs-worth-knowing">The Tradeoffs Worth Knowing</a></li>
<li><a href="#so-where-does-this-leave-us">So Where Does This Leave Us</a></li>
</ul>

</details>
I spent a non-trivial amount of engineering hours building a bridge that makes users wait 13 minutes. Not because I wanted to. Because Ethereum finality takes 13 minutes (~12.8 minutes to be precise) and acting before it means risking minted tokens backed by nothing.

Fast Confirmation Rule (FCR) fixes this. 13 seconds. Same security guarantees under normal conditions. No hardfork. I am a little annoyed it did not exist sooner :(.

This blog covers the implications of FCR from a bridge guy perspective.

---

## Why Bridges Wait So Long Today

Bridges that move assets from Ethereum to another chain have one core fear: acting on a transaction that later disappears in a reorg. If a bridge mints tokens on the destination chain and then the source transaction evaporates, you have just created tokens backed by nothing.

So bridges wait for **finality**, the point at which Ethereum's consensus makes a block irreversible. And finality, by design, takes a while.

---

## How Ethereum Finality Actually Works

Ethereum time is split into **slots** (12 seconds each) and **epochs** (32 slots, roughly 6.4 minutes).

Every active validator is assigned to a slot committee within each epoch and attests (votes) exactly once. After one full epoch, every validator has voted.

<img src="/assets/images/fcr_and_bridges/epoch_structure.png" class="blog-diagram" alt="Epoch structure">

But Ethereum's finality mechanism, **Casper FFG**, does not finalize after just one epoch. It needs two full rounds:

- **Round 1 (Justification):** 2/3 of validators vote for a checkpoint. It is now "justified."
- **Round 2 (Finalization):** Next epoch, validators vote again confirming they saw round 1. If 2/3 confirm, the previous checkpoint is finalized.

Two full epochs. Roughly 13 minutes.

**Why two rounds? Doesn't one cover everyone?**

After one epoch, yes, every validator has voted once. But you cannot trust a single round because the network is async. Validators could be on a split network, voting for two different blocks at the same time, without even knowing they are doing it.

Round 2 closes that gap. When a validator votes in epoch 2, they are saying "I saw round 1 and I confirm it." If they had double-voted in epoch 1, they get slashed. So round 2 is not about collecting more voters, it is about **proving round 1 was not a lie**. That is the Byzantine fault tolerance piece.

---

## Enter FCR

FCR takes a completely different approach to the problem.

Instead of running two epochs to prove the vote was honest, it watches attestations arriving in real-time within the current slot and asks: **is the math already settled?**

If 75% of validators have voted for block A within 8 seconds, the remaining 25% cannot overturn a 75% majority. The outcome is already decided, we just have not waited for the ceremony to finish.

<img src="/assets/images/fcr_and_bridges/fcr_attestation_count.png" class="blog-diagram" alt="FCR attestation count">

FCR declares a block "fast-confirmed" the moment overwhelming support makes any other outcome mathematically impossible.

**But doesn't this have the same async network problem?**

Partially, yes. FCR does not fully solve the deception problem, it bets around it. It assumes the network is synchronous, meaning attestations arrive within about 8 seconds. If that holds, the vote count you see is real, not a partitioned artifact.

If synchrony breaks, FCR detects it and falls back to finality automatically. There are two failure modes:

- **Liveness failure (common):** Not enough attestations arrived in time. FCR stalls, resets to the last finalized block, and you wait the usual 13 minutes. No incorrect confirmation is ever issued.
- **Safety failure (extreme):** FCR confirmed a block that later gets reorged. This requires an adversary with over 50% of active validators, well beyond anything FCR's normal threat model covers. Practically, this is the same risk exchanges already take today when using block confirmation counts.

<img src="/assets/images/fcr_and_bridges/fcr_decision_tree.png" class="blog-diagram" alt="FCR decision tree">

---

## Who Benefits and How

FCR is not just a bridge upgrade. Anyone waiting on Ethereum confirmations has something to gain here.

### Cross-Chain Bridges

The most obvious beneficiary. Instead of locking capital for 13 minutes per transfer, bridges act in 13 seconds. Capital recycles ~60x faster. For intent-based bridges where solvers front liquidity and get reimbursed on settlement, this changes the unit economics entirely. Smaller intents become viable. Spreads tighten. And unlike k-deep confirmation which carries no formal safety guarantee, FCR gives you a deterministic guarantee under normal network conditions.

<img src="/assets/images/fcr_and_bridges/bridge_flow.png" class="blog-diagram" alt="Bridge flow comparison">

### Exchanges

Centralized exchanges are arguably the biggest winners here. Every deposit today involves a waiting period before the exchange credits your account. FCR cuts that to seconds.

The secondary benefit is order book liquidity. When assets are stuck in confirmation limbo, they cannot be traded. FCR frees that capital up faster, deepening liquidity and reducing idle time for users. Exchanges also get a stronger safety guarantee compared to their current approach of waiting for a fixed number of blocks, which has no formal safety guarantee at all.

### L2s

L2s read the `safe` head on L1 to process deposits. Today that means waiting for a justified block, roughly one full epoch. With FCR, L2s can tap into L1 liquidity in seconds.

The fallback is clean: if FCR stalls, the L2 falls back to the finalized head automatically. L2 security is unchanged, it is the same model with a faster default path. If an L1 reorg happens under extreme conditions, the L2 reorgs with it, consistent with its existing security model.

### RPC and Infrastructure Providers

No implementation needed. The `safe` block tag already exists. When consensus clients ship FCR, `safe` returns the last fast-confirmed block instead of the last justified block. The API surface does not change at all.

The one thing RPC providers should flag to their customers is that the semantics of `safe` are shifting. If anyone downstream relies on `safe` meaning "justified," they should know it will start meaning "fast-confirmed."

---

## The Tradeoffs Worth Knowing

FCR sits between k-deep confirmation and full finality. Here is an honest breakdown:

| | k-deep | FCR | Finality |
|---|---|---|---|
| Communication model | Synchrony | Synchrony | Asynchrony |
| Adversarial threshold | Unknown | 25% | 33% |
| Economic security | No | No | Yes |
| Latency | k x 12s | ~13 seconds | ~13 minutes |
| Deterministic guarantee | No | Yes | Yes |

Two core assumptions FCR relies on:

**Assumption 1: Network synchrony.** Attestations need to arrive within ~8 seconds. Network partitions or heavy congestion can break this. FCR detects it and falls back to finality.

**Assumption 2: Less than 25% adversarial stake.** Normal finality tolerates 33%. FCR tightens this to 25% because you are calling the result early, before all votes are in. The gap is the safety margin for uncertainty.

For L2 deposits, exchange credits, and bridge transfers, these tradeoffs are very acceptable. For high-value settlements where a safety failure would be catastrophic, full finality is still the right call. Use the right tool for the job.

---

## So Where Does This Leave Us

13 minutes was never a feature. It was a consequence of the security model, and we all just accepted it because there was no better option.

FCR is that better option. It does not compromise security, it reasons about it differently. Under normal conditions, which is almost always, a fast-confirmed block will be finalized. You just know it 13 minutes earlier.

If you are building on Ethereum today, bridges, exchanges, L2s, or any infra that waits on confirmations, FCR should be on your radar for 2026 Q2. The lift is minimal and the payoff for users is immediate.

For more details, timelines, specs, and one-pagers per use case: [fastconfirm.it](https://fastconfirm.it)

---
