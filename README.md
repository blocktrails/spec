# Blocktrails

Trust through time — Nostr-native state anchoring on Bitcoin.

## What is a Blocktrail?

A Blocktrail is a sequence of P2TR outputs where each output key commits to application state. Spending an output advances the trail to a new state.

```
state₀ → P₀ → spend → state₁ → P₁ → spend → state₂ → P₂ → ...
```

## Key Properties

- **Nostr-native** — full secp256k1 keys internally, x-only only at P2TR boundary
- **Minimal** — one output per state update, key-path spends only
- **SPV-compatible** — verification needs only output keys and merkle proofs
- **BIP-341 aligned** — uses TapTweak tagged hashes for domain separation
- **No extra tokens** — just Bitcoin

## Quick Example

```
# BIP-341 TapTweak derivation
h = sha256(state)
t = tagged_hash("TapTweak", x_only(P) || h) mod n
d = d_prev + t
P = P_prev + t·G

# On-chain
output = p2tr_xonly(P)
```

Verification: chain tweaks from `P_base`, compare `x(P)` to witness program at each step.

## Specification

See [spec.md](spec.md) for the full specification.

## Use Cases

Blocktrails are a primitive. Applications define state format and validation rules on top:

- Token ledgers
- Version control
- Voting systems
- Audit trails
- Any linear state machine

## Status

Specification: **v0.2**

## License

Public domain.
