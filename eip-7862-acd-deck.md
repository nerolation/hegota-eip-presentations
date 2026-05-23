---
marp: true
theme: default
paginate: true
size: 16:9
---

# EIP-7862: Delayed State Root

**Get state root computation off the builder's critical path.**

ACD · Toni Wahrstätter
Co-authors: Charlie Noyes, Dan Robinson, Justin Drake

---

## Why It Matters

State root computation sits on the **builder's critical path**:

- Each MEV candidate ordering produces a different post-state root, computed serially after that candidate's execution.
- SR is expensive and a meaningful share of per-candidate builder cost.
- BAL (EIP-7928) parallelizes SR but block builders / provers don't profit from it.

-> Under ePBS (EIP-7732), execution extends later into the slot, leaving builders with less time for building.

-> Under BALs (EIP-7928), validation can be parallelized, building not.

---

## The Change

> Header `n` carries the post-state root of block `n-1`.

- No new header fields. Only `state_root` semantics change.
- Pre-state of `n` ≡ post-state of `n-1`.

```python
class Header:
    ...
    state_root: Root  # post-state of block (n-1)
```

Small spec: track `last_computed_state_root`, validate against it, compute SR for the *next* block.

---

## What It Unlocks

- **For builders and provers**: no per-candidate SR during the MEV auction. The bid commits to a pre-state root that is already known at slot start.
- **For attesters**: SR leaves the critical path. BAL already gave parallelism; So, real improvement.
- **EL analog of Gloas's CL deferral**: Gloas already defers payload-derived CL state to the child slot. 7862 does the same shift for the EL `state_root`; the Gloas bid still commits to it today.

--- 

EIP: `EIPS/eip-7862.md` · Magicians: `t/22559`
