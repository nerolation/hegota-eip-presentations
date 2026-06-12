---
eip: 8XXX
title: Cap Deposit Requests per Block
description: Cap the number of deposit requests per block at 1024 to keep consensus-layer deposit processing bounded at any gas limit
author:
discussions-to: <TBD>
status: Draft
type: Standards Track
category: Core
created: 2026-06-12
requires: 6110, 7685, 7825
---

## Abstract

Cap the number of deposit requests ([EIP-6110](./eip-6110.md)) per block at `2**10` (= 1,024). A separate, non-normative per-transaction bound lets builders pack blocks in O(1) without speculative execution.

## Motivation

The block gas limit is set by validators and trends upward over time; the consensus layer's capacity for deposits per payload is fixed at `MAX_DEPOSIT_REQUESTS_PER_PAYLOAD = 8192`. At roughly 23,000 gas per deposit, a block above ~194M gas can produce more deposit requests than the consensus structure can represent, and each deposit imposes verification work on the consensus layer. A direct cap on the deposit count decouples both from the gas limit: deposit load stays bounded at any gas limit, and future gas-limit increases need no further analysis of deposit capacity.

The cap must also be cheap for builders to enforce. Charging each transaction a static upper bound on its deposits, inferred from its gas limit, lets a builder maintain a running sum in O(1) per transaction without speculative execution.

## Specification

### Parameters

| Name                     | Value                                                                          |
| ------------------------ | ------------------------------------------------------------------------------ |
| `MAX_DEPOSITS_PER_BLOCK` | `2**10` (= 1,024)                                                              |
| `GAS_PER_DEPOSIT`        | `23,000` (TBD; a verified lower bound on the marginal gas cost of one deposit) |

### Block validity rule

After execution, let `deposit_count` be the number of deposit requests (request type `0x00`, [EIP-7685](./eip-7685.md)) derived from the block per [EIP-6110](./eip-6110.md).

A block is valid only if:

```python
deposit_count <= MAX_DEPOSITS_PER_BLOCK
```

## Rationale

`MAX_DEPOSITS_PER_BLOCK = 2**10` sits well below the consensus layer's `8192` limit and well above the per-transaction maximum (see below), so every individual transaction remains includable. Observed blocks carry orders of magnitude fewer deposits.

The cap counts deposit requests rather than gas spent in the deposit contract: the count is what the consensus layer's SSZ limit and verification work are denominated in, and it is directly observable from the block's execution requests.

### Static per-transaction bound for block builders

Deposits are only known after execution, so a builder cannot evaluate the validity rule without executing each candidate transaction. The following upper bound avoids that:

```python
remaining_gas       = max(0, tx.gas_limit - intrinsic_gas(tx))
static_deposits(tx) = remaining_gas // GAS_PER_DEPOSIT
```

Any block satisfying `sum(static_deposits(tx) for tx in block.transactions) <= MAX_DEPOSITS_PER_BLOCK` satisfies the validity rule by construction. `intrinsic_gas(tx)` is the standard intrinsic gas: calldata at the `4` / `16` gas per zero / non-zero byte rate from [EIP-7623](./eip-7623.md), plus the usual contract-creation, access-list, and auth-list costs.

The bound holds because each deposit costs at least `GAS_PER_DEPOSIT` gas: a successful call to the deposit contract pays for the call, the deposit event log, and the incremental Merkle tree storage updates. `GAS_PER_DEPOSIT` must under-approximate the true marginal cost; erring low loosens the bound but never breaks it. It must be revisited if a future repricing lowers the cost of a deposit.

Only the intrinsic gas of the transaction envelope is subtracted: a contract can loop deposits internally from on-chain data, so no per-deposit calldata can be assumed. Standard `4` / `16` calldata pricing is used rather than any floor cost: it leaves more `remaining_gas` and a larger `static_deposits`, the conservative direction for a pre-execution check.

[EIP-7825](./eip-7825.md)'s `2**24` per-transaction gas cap ensures `static_deposits(tx) <= (2**24 - 21000) // GAS_PER_DEPOSIT = 728`, so no single transaction can exceed the cap.

Builders that execute candidate transactions may replace `static_deposits(tx)` with the actual deposit count for executed transactions and use the static bound only to decide, ahead of execution, whether a transaction can fit the remaining deposit budget.

### Impact on real traffic

Reaching the cap requires ~24M gas spent exclusively on deposits in one block. No organic traffic comes close; the cap binds only in adversarial scenarios.

## Backwards Compatibility

Consensus-breaking change, requires a hard fork. EL clients must enforce the post-execution validity rule; builders and simulators should use the `static_deposits` bound when assembling or simulating blocks. The deposit contract is unchanged. No individual transaction becomes invalid: a deposit-heavy transaction that does not fit a block's remaining deposit budget is deferred to a later block, symmetric with the block gas limit.

## Security Considerations

**Bounded consensus-layer work.** Per-block deposit verification work is bounded by `MAX_DEPOSITS_PER_BLOCK` at any gas limit, by construction. This closes the deposit-flood DoS without per-client caches or optimizations.

**Builder packing is O(1).** `static_deposits(tx)` depends only on fields in the transaction envelope, so a builder maintains a running sum and rejects any transaction that would cross the cap.

**Censorship surface.** Since `static_deposits(tx) <= 728 < MAX_DEPOSITS_PER_BLOCK` for any legal transaction (via EIP-7825), every transaction remains individually includable; only specific combinations are rejected. This is symmetric with the existing block-gas-limit constraint.

**Slack.** Almost all transactions carry zero deposits, so `static_deposits` overshoots heavily on ordinary traffic and blocks packed under the static bound run far below the cap. Builders that execute transactions recover the slack exactly by counting actual deposits.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
