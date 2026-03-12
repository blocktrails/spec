# Blocktrails (v0.2)

Trust through time — Nostr-native state anchoring on Bitcoin.

## Abstract

This specification defines Blocktrails, a Nostr-native method for binding off-chain state to Bitcoin's UTXO graph. State is committed by tweaking a base public key; advancing state requires spending to a newly tweaked key. The result is a linear, Bitcoin-ordered sequence of commitments inheriting Bitcoin's security guarantees. Full verification requires only block headers and minimal proof data — no full node needed.

## Definition

A **Blocktrail** is a sequence of P2TR key-path outputs where each output key is derived by cumulatively tweaking the base key. Each state transition adds a new tweak, and spending the output advances the trail to the next committed state.

## Key Representation

Blocktrails operate on full secp256k1 private keys and public keys internally, as used by Nostr. State transitions are defined as scalar additions to the private key. When committing state to Bitcoin, the corresponding public key is encoded as a Taproot output key by taking its x-coordinate. Verification compares x-coordinates only.

This preserves:

- **Scalar arithmetic**: `privkey + t` just works
- **Vanity generation**: grind keys without parity concerns
- **Nostr compatibility**: same key model
- **Composability**: standard secp256k1 tooling

Parity normalization is an output encoding concern, not a state or semantic concern.

Key encodings such as `nsec`, `npub`, or compressed public keys are out of scope and may be used freely by applications.

## Mechanism

Blocktrails use **chained tweaking**: each state transition adds a scalar tweak to the previous key, forming a cryptographic chain that mirrors the spend chain.

For genesis state `s₀`:

1. Compute `t₀ = scalar(P_base, s₀)`
2. Private key: `d₀ = d_base + t₀`
3. Public key: `P₀ = P_base + t₀·G`

For transition to state `sᵢ`:

1. Compute `tᵢ = scalar(Pᵢ₋₁, sᵢ)` — tweak depends on current pubkey
2. Private key: `dᵢ = dᵢ₋₁ + tᵢ`
3. Public key: `Pᵢ = Pᵢ₋₁ + tᵢ·G`

Note: Unlike simple additive tweaking, BIP-341 tweaks are **pubkey-dependent**. The same state produces different tweaks at different positions in the chain.

**BIP-341 Tagged Hash:**

```
tagged_hash(tag, msg):
  tag_hash = sha256(tag)
  return sha256(tag_hash || tag_hash || msg)
```

**Scalar function (BIP-341 TapTweak):**

```
scalar(P, s):
  h = sha256(serialize(s))                        // 32 bytes
  t = tagged_hash("TapTweak", x_only(P) || h)     // BIP-341
  t = int(t, big-endian) mod n                    // n = secp256k1 group order
  if t == 0: reject state as invalid              // probability ~2^-256
  return t
```

Where `x_only(P)` extracts the 32-byte x-coordinate from a compressed public key.

The tweak `t` MUST be in the range [1, n-1]. Implementations MUST reject states where `t = 0`. This is an application-layer rule — Bitcoin accepts the output regardless.

**Benefits of BIP-341 derivation:**

- **Pubkey binding**: Same state at different chain positions produces different tweaks
- **Domain separation**: Tagged hash prevents collision with other protocols
- **Standards alignment**: Compatible with BIP-341 Taproot

All subsequent uses of `scalar(P, state)` refer to this function.

`serialize(s)` MUST be canonical and byte-stable: the same logical state MUST always produce identical bytes. Non-deterministic serialization breaks commitment verification.

The base key `d_base` is immutable for a given trail. Rotation requires starting a new trail.

Fees are paid from the current head output. When balance runs low, top-up (see Scope) adds funds.

Boundary encoding for P2TR output:

```
p2tr_xonly(P) → bytes32:
  if y(P) is even: return x(P)
  else: return x(-P)
```

The witness program is `p2tr_xonly(P)`. Signing uses the corresponding (possibly negated) private key per BIP-340.

Spending this output advances the trail to the next committed state.

## Constraints

Blocktrails use P2TR key-path only and do not require tapscript, script trees, or script-path spends. The only mechanism used is public-key tweaking.

Domain separation is provided by BIP-341 tagged hashes. The "TapTweak" tag ensures commitments cannot collide with other protocols using SHA-256.

## Components

| Component | Description |
|-----------|-------------|
| **Commitment carrier** | Tweaked public key (full point) |
| **State head** | Currently unspent output |
| **History** | The spend chain |

## Design Goals

| Goal | How |
|------|-----|
| Minimal on-chain footprint | One P2TR output per state update |
| Standardness-friendly | Key-path spends only |
| External state | State bytes live off-chain; chain anchors integrity |
| Single-writer | One controlling base key (multi-writer requires extra protocol) |

## Scope

Blocktrails intentionally specify only the minimal primitive required for interoperable verification: commitment-by-output-key and linear progression by spends.

Many lifecycle behaviors — top-up, third-party payments, transfer of control, explicit closure — are compositions of ordinary Bitcoin spends and do not change the core verification rule. This document treats them as optional extensions.

In practice, **top-up** is the only liveness-oriented convention that may be widely needed, since trail progression is fee-bounded.

## Operations

### Genesis

```
t₀ = scalar(P_base, s₀)
d₀ = d_base + t₀
P₀ = P_base + t₀·G
output₀ = p2tr_xonly(P₀)
```

Create P2TR output with witness program `output₀`. Holder knows `d₀`.

### Transition

```
tᵢ = scalar(Pᵢ₋₁, sᵢ)
dᵢ = dᵢ₋₁ + tᵢ
Pᵢ = Pᵢ₋₁ + tᵢ·G
outputᵢ = p2tr_xonly(Pᵢ)
```

Sign with `dᵢ₋₁` to spend previous output. Create new output with witness program `outputᵢ`.

Note: The spending key accumulates all previous tweaks. At state `n`, each tweak `tᵢ` was computed from `Pᵢ₋₁`:
```
dₙ = d_base + t₀ + t₁ + ... + tₙ
```
where `t₀ = scalar(P_base, s₀)`, `t₁ = scalar(P₀, s₁)`, etc.

### Verify

```
verify(P_base, genesis_outpoint, states[]) → bool:
  outpoint = genesis_outpoint
  P = P_base

  for state in states:
    t = scalar(P, state)  // tweak depends on current P; rejects if t = 0
    P = P + t·G           // chain the tweak
    if x(P) ≠ witness_program(outpoint): return false
    outpoint = spending_outpoint(outpoint)

  return is_unspent(outpoint)
```

Verification chains the tweaks: each state adds to the running public key using pubkey-dependent BIP-341 tweaks. Since `x(P) == x(-P)`, comparison uses x-coordinates only. No parity handling needed.

To locate the spending transaction for a given outpoint, implementations MAY use any standard Bitcoin data source: full node with indexing, Electrum-style servers, or externally published transaction feeds.

**SPV compatibility and data dependencies:**

Blocktrails are SPV-compatible. Verification requires:

- **Block headers** — to validate proof-of-work chain
- **Merkle inclusion proofs** — for each transaction in the spend chain
- **Genesis outpoint** — application must know or discover the starting point
- **Spending outpoint lookup** — a method to find which transaction spent a given output (e.g., Electrum query, utreexo proof, or trusted indexer)
- **Current UTXO status** — to determine if the head output is unspent

No full node is necessary, but UTXO status and spend-chain traversal require indexing or proof infrastructure. "SPV-compatible" does not mean "zero external dependencies."

## Interface

Applications implement:

```
serialize(state) → bytes
validate(prev, next) → bool
```

The primitive handles commitment and chaining. Applications define state format and transition rules.

## Guarantees

This primitive provides:

- **Integrity** of state commitments
- **Ordering** relative to the spend chain
- **Finality** — probabilistic via Bitcoin proof-of-work; applications SHOULD wait k confirmations

## Non-Guarantees

This primitive does not provide:

- **On-chain rule enforcement** — `validate(prev, next)` is checked client-side
- **State availability** — state distribution requires a separate channel

## Security Assumptions

| Assumption | Guarantees |
|------------|------------|
| Hash preimage resistance | State binding |
| Discrete log hardness | Output key security |
| Bitcoin consensus | Spend ordering, double-spend prevention |

## Threat Model

### What Blocktrails protect against

| Threat | Mitigation |
|--------|------------|
| State forgery | Hash commitment — cannot find `s'` where `H(s') = H(s)` |
| History rewriting | Bitcoin immutability — confirmed spends are final |
| Double-spending state | UTXO model — each output spendable exactly once |
| Ordering disputes | Blockchain provides canonical order |

### What Blocktrails do NOT protect against

| Threat | Why | Mitigation |
|--------|-----|------------|
| Key compromise | Holder of `d_base` controls all transitions. Compromise means loss of all funds in the head output and full control over all future state transitions. | Secure key storage |
| State withholding | State lives off-chain | Redundant state distribution |
| Invalid state transitions | Validation is client-side | Verifiers must check `validate(prev, next)` |
| Censorship | Depends on Bitcoin transaction inclusion | Standard Bitcoin censorship resistance |
| Front-running | Miner can see pending state in mempool | Commit-reveal if state must be private |

### Trust boundaries

```
┌─────────────────────────────────────────────┐
│            Trusted (cryptography)           │
│  - Hash binds state to output key           │
│  - Signature proves key knowledge           │
│  - Bitcoin prevents double-spend            │
└─────────────────────────────────────────────┘
                      │
┌─────────────────────────────────────────────┐
│         Untrusted (application layer)       │
│  - State availability                       │
│  - Transition validity                      │
│  - Semantic correctness                     │
└─────────────────────────────────────────────┘
```

### Failure modes

| Failure | Consequence | Recovery |
|---------|-------------|----------|
| Lost `d_base` | Cannot advance trail | None — funds/state locked |
| Lost state history | Cannot verify trail | Restore from backup/peers |
| Invalid transition accepted | Verifiers reject; chain forks semantically | Social consensus on valid branch |
| Bitcoin reorg | Recent transitions may revert | Wait for confirmations |

## Reference Implementation

[blocktrails](https://github.com/blocktrails/blocktrails) — JavaScript/Node.js implementation with full test suite.

```bash
npm install blocktrails
```

## Related Specifications

- [Blocktrails Profiles](./profiles.md) — Application-layer profiles defining state formats and validation rules (Monochrome, MRC20)
- [DAG Extension](./dag.md) — Multi-party state graphs (opt-in, backward-compatible)

## References

- [BIP-340: Schnorr Signatures](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)

---

*Blocktrails: Nostr-native state on Bitcoin.*
