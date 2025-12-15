# Blocktrails

Nostr-native output-key commitment chaining on Bitcoin.

## What is a Blocktrail?

A Blocktrail is a sequence of P2TR outputs where each output key commits to application state. Spending an output advances the trail to a new state.

```
state₀ → P₀ → spend → state₁ → P₁ → spend → state₂ → P₂ → ...
```

## Key Properties

- **Nostr-native** — full secp256k1 keys internally, x-only only at P2TR boundary
- **Minimal** — one output per state update, key-path spends only
- **SPV-compatible** — verification needs only output keys and merkle proofs
- **No extra tokens** — just Bitcoin

## Quick Example

```
# State commitment
t = SHA256(state) mod n
d = d_base + t
P = d·G

# On-chain
output = p2tr_xonly(P)
```

Verification: compare `x(P)` to witness program for each state in sequence.

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

Specification: **draft**

## License

Public domain.
