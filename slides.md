---
marp: true
title: Block-Level Access Lists in Glamsterdam
author: Toni Wahrstätter
footer: BALs in Glamsterdam · June 2026
theme: gaia
---

<!-- _class: lead invert -->

# Block-Level Access Lists

## EIP-7928, BAL exchange & snap/2, landing in Glamsterdam

# 🧱

---

## EIP-7928: the idea

- Every block ships a **Block Access List**: all accounts and storage slots touched, with their **post-execution values**
- Enforced at the block level, committed via header `block_access_list_hash`, carried in the `ExecutionPayload`, not the block body
- [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) access lists were optional and unenforced

> Unlocks parallel disk IO, parallel tx validation, parallel state-root calc, and executionless state updates.

---

## EIP-7928: what's in it

`address → field → block_access_index → change`

- **Storage** writes and read-only slots
- **Balance, nonce, code** post-tx diffs
- Touched-but-unchanged addresses too, for complete parallel IO
- `block_access_index`: `0` pre-exec, `1…n` txs, `n+1` post-exec / withdrawals

> Post-tx values let nodes reconstruct state without re-executing, and parallelize the txs that touch shared slots.

---

## Getting BALs to peers

- **EIP-8159 (`eth/71`):** new `GetBlockAccessLists` / `BlockAccessLists` messages plus a `block-access-list-hash` header field
- **EIP-8189 (`snap/2`):** drops trie-node healing, adds BAL-based state catch-up

> Engine API serves live BALs; the wire protocol serves historical BALs and peer sync. snap/2 knows its catch-up set upfront: download BALs, apply diffs in order, verify each against its header hash.

---

<style scoped>
section { font-size: 30px; }
h2 { font-size: 44px; }
</style>

## What lands in Glamsterdam

| EIP | What | Layer |
|---|---|---|
| **7928** | Block-Level Access Lists | Core |
| **8159** | `eth/71` BAL exchange | Networking |
| **8189** | `snap/2` BAL-based healing | Networking |

> 7928 defines the data; 8159 and 8189 make it syncable. Together: parallel execution plus faster, simpler sync.

---

<style scoped>
section { font-size: 27px; }
h2 { font-size: 44px; }
</style>

## What's next

- **[EIP-8268](https://eips.ethereum.org/EIPS/eip-8268):** storage roots in BALs, so partially stateful nodes can verify the state root
- **[EIP-8279](https://eips.ethereum.org/EIPS/eip-8279):** BAL byte floor, closing the cold-`SLOAD` bypass (1.55 MB down to 0.89 MB worst-case block)
- **[EIP-8146](https://eips.ethereum.org/EIPS/eip-8146):** BAL sidecars, propagate BALs outside the payload envelope so the EL can prefetch state earlier

---

<!-- _class: lead invert -->

# 🧱

## Thank you

### EIP-7928 · 8159 · 8189 · Glamsterdam
