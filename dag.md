# DAG Extension (v0.1)

Multi-party state graphs for Blocktrails.

## Abstract

This extension generalizes Blocktrails from a linear chain to a directed acyclic graph (DAG). A linear trail is a DAG where every entry has exactly one parent. This extension adds support for entries with multiple parents, enabling concurrent multi-party state transitions that merge.

The core spec is unchanged. DAG mode is opt-in per trail.

## Motivation

Linear blocktrails work for single-executor contracts: AMMs, escrow, token issuance. One writer, sequential transitions, one UTXO at a time.

Multi-party contracts need concurrency. A trustline between Alice and Bob requires both parties to update state independently, then merge. Payment routing through trust chains requires multiple concurrent branches. Linear trails force artificial serialization where the application is naturally concurrent.

## Design Principle

**A linear trail is a DAG with branching factor 1.** The extension adds one concept — multiple parents — and derives everything else from it.

## Trail Modes

A trail declares its mode at genesis:

**Linear (default, unchanged):**

```json
{
  "mode": "linear",
  "base": "<pubkey>",
  "genesis": "<outpoint>"
}
```

**DAG:**

```json
{
  "mode": "dag",
  "base": "<pubkey>",
  "genesis": "<outpoint>",
  "parties": ["<pubkey>", "<pubkey>"]
}
```

If `mode` is omitted, it defaults to `"linear"`. Existing trails are valid without modification.

## Entry Structure

### Linear (unchanged)

```json
{
  "seq": 5,
  "prev": "a1b2c3..."
}
```

### DAG

```json
{
  "seq": 5,
  "parents": ["a1b2c3...", "f7g8h9..."],
  "party": "<pubkey>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `seq` | integer | Party-local sequence number |
| `parents` | array of hex64 | Hashes of parent entries (1 or more) |
| `party` | pubkey | Author of this entry |

**Compatibility:** `prev` is valid shorthand for `parents: ["<hash>"]`. A linear trail's entries are valid DAG entries with one parent.

## Topology Rules

1. **Genesis** has no parents: `parents: []`
2. **Linear entry** has one parent: `parents: ["<hash>"]`
3. **Merge entry** has multiple parents: `parents: ["<hash>", "<hash>", ...]`
4. **No cycles** — entries form a DAG
5. **Party sub-chains are linear** — each party's entries, filtered by `party` field, form a strictly sequential chain

## Sub-Chain Linearity

Within a DAG trail, each party maintains a linear sub-chain:

```
Alice:  A0 → A1 → A2 → M1 → A3
                        ↑
Bob:    B0 → B1 ────────┘
```

- Alice's entries (A0, A1, A2, A3) are sequential
- Bob's entries (B0, B1) are sequential
- M1 is a merge entry authored by Alice with `parents: [hash(A2), hash(B1)]`

The merge entry belongs to Alice's sub-chain. After a merge, both parties have seen and acknowledged the merged state.

## State Computation

### Current State

The current state is computed by topologically sorting all entries and applying them in order. For entries with no ordering dependency (concurrent branches), any valid topological sort produces the same result — profiles MUST ensure commutativity for concurrent operations, or use merge entries to resolve conflicts.

### Verification

```
verify_dag(base, genesis_outpoint, entries[]) → bool:
  // Build the DAG
  for entry in entries:
    for parent_hash in entry.parents:
      assert parent_hash exists in entries or is genesis

  // Verify each party's sub-chain is linear
  for party in parties:
    sub = entries.filter(e => e.party == party).sort_by(seq)
    for i in 1..sub.length:
      assert sub[i].seq == sub[i-1].seq + 1

  // Verify Bitcoin anchoring (each entry anchored independently or via batch)
  // ... standard Blocktrails Core verification per entry
```

### Selective Verification (RGB-style)

A verifier only needs to traverse the branches relevant to them:

```
verify_relevant(my_pubkey, entries[]) → bool:
  relevant = walk_ancestors(latest_entry_for(my_pubkey), entries)
  verify_dag(base, genesis_outpoint, relevant)
```

This is the key scalability property: in a large multi-party DAG, each party verifies O(depth) entries, not O(total entries).

## Bitcoin Anchoring

### Option A: Shared UTXO (Simple)

All parties share a single UTXO. The trail controller (e.g., an executor) batches DAG entries into linear Bitcoin transitions. The DAG structure lives off-chain; Bitcoin sees a linear spend chain.

This is the simplest approach and works when a trusted executor exists (e.g., a web contract executor).

### Option B: Per-Party UTXOs (Sovereign)

Each party maintains their own UTXO chain. Merge entries reference Bitcoin transactions from multiple parties. Verification requires following multiple spend chains.

This is more complex but removes the need for a shared controller.

### Recommendation

Start with Option A. Most web contracts already have an executor. The DAG is an off-chain data structure; Bitcoin anchors the linear sequence of batched commits.

## Profile Integration

Profiles declare their trail mode:

```json
{
  "profile": "amm.v1",
  "trail": "linear"
}
```

```json
{
  "profile": "trustline.v1",
  "trail": "dag"
}
```

Profiles using `"trail": "linear"` (or omitting the field) work exactly as before. No changes to existing profiles.

## Compatibility

| Scenario | Result |
|----------|--------|
| Existing linear trail | Valid. No changes needed. |
| Linear verifier reads DAG trail | Sees `parents` instead of `prev`. Can reject gracefully or follow `parents[0]`. |
| DAG verifier reads linear trail | Treats `prev` as `parents: [prev]`. Works transparently. |
| New profile uses DAG | Declares `"trail": "dag"`. Only DAG-aware verifiers handle it. |

## Non-Goals

- **Consensus** — DAG does not add consensus. Conflict resolution is profile-specific.
- **Concurrency control** — profiles define what happens when concurrent entries conflict.
- **Privacy** — entries are still explicit. Concealment is a separate concern.

## Relationship to RGB

| Aspect | This Extension | RGB |
|--------|---------------|-----|
| Graph structure | DAG with explicit merge | DAG with concealment |
| Verification | Selective (ancestor walk) | Selective (ownership chain) |
| Anchoring | Shared or per-party UTXO | Per-party UTXO |
| Complexity | Minimal — one new field | Full VM + type system |
| Compatibility | Backward-compatible with linear | Standalone protocol |

## Summary

One field changes: `prev` (string) → `parents` (array). Everything else is additive.

- Linear trails are unchanged
- DAG is opt-in per trail
- Each party's entries remain linear
- Selective verification keeps it scalable
- Bitcoin anchoring stays simple (batched linear commits)

---

*Blocktrails DAG Extension v0.1 — Multi-party state graphs on Bitcoin.*
