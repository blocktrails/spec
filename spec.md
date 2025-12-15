# Blocktrails

Nostr-native output-key commitment chaining on Bitcoin.

## Definition

A **Blocktrail** is a sequence of P2TR key-path outputs where each output key is the base key tweaked by `H(state)`, and spending the output advances the trail to a new committed state.

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

For state `s`:

1. Compute `t = H(serialize(s))` interpreted as 256-bit big-endian integer, reduced mod `n`
2. Private key: `d = d_base + t`
3. Public key: `P = d·G` (equivalently, `P = P_base + t·G`)

`H` is SHA-256 unless otherwise specified by the application.

If `t = 0`, the state MUST be rejected as invalid (probability ~2^-256).

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

No domain separation is applied; the commitment binds exactly to the serialized state bytes. This is deliberate — simplicity is a goal. Applications requiring domain separation can include it in their `serialize()` function.

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
t₀ = H(serialize(state₀)) mod n
d₀ = d_base + t₀
P₀ = d₀·G
output₀ = p2tr_xonly(P₀)
```

Create P2TR output with witness program `output₀`. Holder knows `d₀`.

### Transition

```
tᵢ = H(serialize(stateᵢ)) mod n
dᵢ = d_base + tᵢ
Pᵢ = dᵢ·G
outputᵢ = p2tr_xonly(Pᵢ)
```

Sign with `dᵢ₋₁` to spend previous output. Create new output with witness program `outputᵢ`.

### Verify

```
verify(P_base, genesis_outpoint, states[]) → bool:
  outpoint = genesis_outpoint

  for state in states:
    t = H(serialize(state)) mod n
    P = P_base + t·G
    if x(P) ≠ witness_program(outpoint): return false
    outpoint = spending_outpoint(outpoint)

  return is_unspent(outpoint)
```

Since `x(P) == x(-P)`, verification simply compares x-coordinates. No parity handling needed.

To locate the spending transaction for a given outpoint, implementations MAY use any standard Bitcoin data source: full node with indexing, Electrum-style servers, or externally published transaction feeds.

Blocktrails are SPV-compatible. Verification requires only output keys and merkle inclusion proofs — no full node necessary.

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
| Key compromise | Holder of `d_base` controls all transitions | Secure key storage |
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

## References

- [BIP-340: Schnorr Signatures](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)

---

*Blocktrails: Nostr-native state on Bitcoin.*
