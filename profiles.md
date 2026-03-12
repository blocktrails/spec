# Blocktrails Profiles Specification (v0.2)

---

## Abstract

This document defines the Blocktrails profile system and the Monochrome profile framework for building client-validated state contracts on Bitcoin. It includes MRC20, an issuer-controlled fungible token profile.

---

## 1. Introduction

Blocktrails defines a minimal primitive for anchoring and ordering off-chain state using Bitcoin P2TR output-key commitments. It provides ordering and integrity but does not define application semantics.

**Profiles** extend Blocktrails with meaning. A profile specifies:

- State format
- Validation rules
- Operation semantics

Profiles do not modify Blocktrails Core verification.

---

## 2. Profile Model

### 2.1 Identification

A profile is identified by a stable string:

```
profile_id := <namespace>.<name>.<version>
```

Examples:

```
mono.monochrome.v0.1
mono.mrc20.v0.1
```

### 2.2 Requirements

A profile MUST:

- Be compatible with Blocktrails Core verification
- Operate on a single linear state chain
- Use client-side validation only
- Use JCS for canonical serialization

A profile MUST NOT:

- Require tapscript or script-path spends
- Require on-chain rule enforcement
- Alter output-key commitment semantics

---

## 3. Canonical Serialization

Profiles use **JCS (JSON Canonicalization Scheme, RFC 8785)** for deterministic hashing.

### 3.1 Rules

- UTF-8 encoding
- No whitespace between tokens
- Object keys sorted lexicographically
- No duplicate keys
- Numbers as shortest decimal (no scientific notation)

### 3.2 State Hash and Scalar

Profiles define `serialize(state) := jcs(state)`. The Blocktrails Core `scalar(P, state)` function (see parent spec) uses BIP-341 TapTweak derivation:

```
scalar(P, state):
  h = sha256(jcs(state))                          // 32 bytes
  t = tagged_hash("TapTweak", x_only(P) || h)     // BIP-341
  return int(t, big-endian) mod n
```

Serialization MUST be canonical and byte-stable: the same logical state MUST always produce identical bytes. JCS provides this guarantee. Non-deterministic serialization breaks commitment verification.

The resulting tweak MUST be in the range [1, n-1]. If `scalar(P, state) = 0`, the state MUST be rejected (probability ~2^-256).

**Note on collisions:** A hash collision is catastrophic by construction — it breaks commitment binding, not merely on-chain distinguishability. Two different states producing the same scalar value would share an output key, creating semantic ambiguity: verifiers cannot determine which state was intended. The security of client-side validation depends entirely on collision resistance.

### 3.3 Example

Input:

```json
{
  "seq": 1,
  "profile": "mono.mrc20.v0.1",
  "prev": "a1b2..."
}
```

Canonical output:

```
{"prev":"a1b2...","profile":"mono.mrc20.v0.1","seq":1}
```

---

## 4. Common Encodings

### 4.1 Public Keys

```
<pubkey> := 32-byte x-only secp256k1 public key, hex-encoded (64 characters, lowercase)
```

Profiles use x-only pubkeys as account identifiers (e.g., token holders in MRC20). Issuer authority is separate — it derives from the Blocktrails spend signer (`d_base` holder), not from account pubkeys.

Nostr `npub` keys can be converted by bech32-decoding to the 32-byte x-only key; no parity adjustment is needed.

Example:

```
"a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
```

### 4.2 Hashes

```
<hex64> := 32-byte SHA-256 hash, hex-encoded (64 characters, lowercase)
```

### 4.3 Integers

Integers are JSON numbers with no fractional component. Values MUST NOT exceed 2^53-1 (JSON safe integer limit). Values above this limit are invalid.

Note: While fields are documented as `<uint64>` for clarity, the JSON-safe subset applies. Future versions may define a binary encoding for full uint64 range.

---

## 5. URN Vocabulary

Operations MUST use URN identifiers:

| URN | Operation |
|-----|-----------|
| `urn:mono:op:set` | Set key-value |
| `urn:mono:op:mint` | Create units |
| `urn:mono:op:transfer` | Move units |
| `urn:mono:op:burn` | Destroy units |
| `urn:mono:op:append` | Append to log |

Short-form operation names (e.g., `"op": "mint"`) are invalid. Implementations MUST reject states using non-URN operation identifiers.

### 5.1 URN Extensibility

The `urn:mono:` namespace is currently unregistered and defined by convention.

**Normative constraint:** Third parties MUST NOT define new operations under `urn:mono:op:` without governance by the Blocktrails/Monochrome specification maintainers. Namespace squatting creates ambiguity for verifiers.

Profile authors MAY define additional operations under their own namespace (e.g., `urn:myorg:op:custom`). A formal registry may be established in future versions.

---

## 6. Monochrome Profile Framework (v0.1)

### 6.1 Purpose

Monochrome is a minimal profile framework for linear client-side state contracts. One active state at a time, no branching, no concealment, no concurrency.

### 6.2 Profile ID

```
mono.monochrome.v0.1
```

### 6.3 Contract Identity

```
contract_id := sha256("blocktrails.contract" || genesis_outpoint || P_base_xonly || profile_id)
```

The `"blocktrails.contract"` prefix provides domain separation to prevent cross-protocol identifier collisions.

**Rationale:** The parent spec explicitly omits domain separation for state commitments (simplicity; context is already bound to outpoint + base key). Contract identifiers are different: they are cross-protocol references used outside the commitment context, so domain separation is warranted here.

**Canonical encoding:**

| Component | Encoding |
|-----------|----------|
| `"blocktrails.contract"` | UTF-8 bytes (20 bytes) |
| `genesis_outpoint` | txid (32 bytes, little-endian) ‖ vout (4 bytes, little-endian) |
| `P_base_xonly` | Taproot internal key's x-only encoding (32 bytes) |
| `profile_id` | UTF-8 bytes, no null terminator |

Concatenated directly with no separators.

**Example (pseudocode):**

```
prefix  = utf8("blocktrails.contract") // 20 bytes
txid_le = reverse(txid_hex)            // 32 bytes LE
vout_le = uint32_le(vout)              // 4 bytes LE
pubkey  = x_only_pubkey                // 32 bytes
profile = utf8("mono.mrc20.v0.1")      // 15 bytes (use actual profile_id)

contract_id = sha256(prefix || txid_le || vout_le || pubkey || profile)
```

Note: The `profile_id` in the contract identity is the specific profile layered on Monochrome (e.g., `mono.mrc20.v0.1`), not `mono.monochrome.v0.1`. This binds the contract to its application-layer semantics.

### 6.4 Base State Schema

```json
{
  "profile": <string>,
  "seq": <uint64>,
  "prev": <hex64>,
  "ops": [ <operation> ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `profile` | string | Profile identifier |
| `seq` | integer | Sequence number, starts at 0 |
| `prev` | hex64 | `sha256(serialize(previous_state))`, 64 zeros for genesis |
| `ops` | array | Operations applied in this transition |

The `ops` array contains only the operations applied in this state transition, not a cumulative log. Historical operations are recoverable by replaying the state chain.

### 6.5 Base Validation

A transition is valid if:

1. Blocktrails Core verification succeeds
2. `seq` increments by exactly 1
3. `prev` equals `sha256(serialize(previous_state))`
4. All operations use valid URN identifiers
5. All operations are well-formed per profile schema

### 6.6 Authorization

The signer of the Blocktrails spend is the sole authority for state transitions.

This is a **single-writer model**. Multi-writer or per-account authorization requires external protocol and is out of scope for Monochrome v0.1.

### 6.7 Fees

Fees are paid from the current head output. Insufficient balance halts trail progression until the output is topped up (see [Blocktrails Core: Scope](https://blocktrails.org/#scope)).

---

## 7. MRC20: Fungible Token Profile (v0.1)

### 7.1 Purpose

MRC20 is a minimal fungible token profile built on Monochrome.

### 7.2 Authority Model

**MRC20 is an issuer-controlled ledger.**

The holder of the Blocktrails base key (`d_base`) has full authority over all operations, including:

- Minting to any address
- Transferring from any address
- Burning from any address

This is analogous to a centralized ledger with cryptographic auditability. The issuer cannot forge history, but can authorize any valid state transition.

**Implications:**

- Users trust the issuer not to act maliciously
- All operations are publicly auditable
- The issuer cannot rewrite confirmed history

**For self-sovereign tokens** where account holders authorize their own transfers, see Section 7.11.

### 7.2.1 Trust Through Observed Auditability

MRC20 operates on an **earned trust model**. The issuer cannot:

- Forge historical states (hash-bound)
- Reorder past transitions (Bitcoin-anchored)
- Hide operations from verifiers (explicit state)

Trust is established through:

1. **Complete history visibility** — Any party can replay and verify all transitions
2. **Immutable audit trail** — Misbehavior is permanently recorded
3. **Reputation at stake** — Observed honesty over time builds confidence

This differs from trustless systems (e.g., Bitcoin L1) where rules are enforced by consensus. In MRC20, rules are enforced by **social accountability** — the issuer *can* misbehave, but *cannot hide* misbehavior.

**Practical implication:** Users should verify issuer history before accepting tokens. Long-running contracts with clean histories are more trustworthy than new ones.

### 7.3 Profile ID

```
mono.mrc20.v0.1
```

### 7.4 State Schema

```json
{
  "profile": "mono.mrc20.v0.1",
  "seq": <uint64>,
  "prev": <hex64>,
  "ops": [ <operation> ],
  "balances": { "<pubkey>": <uint64> },
  "supply": <uint64>,
  "name": <string>,
  "ticker": <string>,
  "decimals": <uint8>
}
```

| Field | Type | Required | Mutable |
|-------|------|----------|---------|
| `profile` | string | Yes | No |
| `seq` | integer | Yes | Yes |
| `prev` | hex64 | Yes | Yes |
| `ops` | array | Yes | Yes |
| `balances` | object | Yes | Yes |
| `supply` | integer | Yes | Yes |
| `name` | string | Yes | No |
| `ticker` | string | Yes | No |
| `decimals` | integer | Yes | No |

**Integer limits:** Per Section 4.3, all integer values are constrained to 2^53-1 (JSON safe integer). For MRC20, this means maximum representable balance or supply per contract is 9,007,199,254,740,991 base units. With 8 decimals, this is ~90 million tokens — sufficient for most use cases.

### 7.5 State Computation Model

**`ops` is authoritative. `balances` and `supply` are computed state.**

Verifiers MUST:

1. Start from genesis state (`balances: {}`, `supply: 0`)
2. Apply each operation in sequence
3. Verify computed `balances` and `supply` match the state exactly

**Rationale:** Storing computed state enables efficient verification of the current state without replaying full history. However, the operation log is the source of truth.

**Verification pseudocode:**

```
verify_transition(S, S'):
  computed_balances = copy(S.balances)
  computed_supply = S.supply

  for op in S'.ops:
    apply_op(op, computed_balances, computed_supply)

  assert S'.balances == computed_balances
  assert S'.supply == computed_supply
```

Transition verification is local: it only requires the previous state `S` and the new state `S'`. Full-chain replay from genesis is also valid and produces equivalent results.

### 7.6 Operations

**MINT**

```json
{ "op": "urn:mono:op:mint", "to": "<pubkey>", "amt": <uint64> }
```

**TRANSFER**

```json
{ "op": "urn:mono:op:transfer", "from": "<pubkey>", "to": "<pubkey>", "amt": <uint64> }
```

**BURN**

```json
{ "op": "urn:mono:op:burn", "from": "<pubkey>", "amt": <uint64> }
```

### 7.7 Validation Rules

A transition from `S` to `S'` is valid if:

1. `S'.profile == "mono.mrc20.v0.1"`
2. `S'.seq == S.seq + 1`
3. `S'.prev == sha256(serialize(S))`
4. Immutable fields unchanged: `name`, `ticker`, `decimals`
5. All operations use URN identifiers
6. For each operation, computed deltas are valid:

| Op | Precondition | Effect |
|----|--------------|--------|
| MINT | — | `balances[to] += amt`; `supply += amt` |
| TRANSFER | `balances[from] >= amt` | `balances[from] -= amt`; `balances[to] += amt` |
| BURN | `balances[from] >= amt` | `balances[from] -= amt`; `supply -= amt` |

7. `S'.balances` equals computed balances after applying all ops
8. `S'.supply` equals computed supply after applying all ops
9. Blocktrails spend signed by issuer (`d_base` holder)

**Note:** Zero balances MAY be omitted from `balances` object. Implementations MUST treat missing keys as zero.

### 7.8 Genesis State

```json
{
  "profile": "mono.mrc20.v0.1",
  "seq": 0,
  "prev": "0000000000000000000000000000000000000000000000000000000000000000",
  "ops": [],
  "balances": {},
  "supply": 0,
  "name": "Example Token",
  "ticker": "EXT",
  "decimals": 8
}
```

### 7.9 Example Lifecycle

**Genesis (State 0)**

```json
{
  "profile": "mono.mrc20.v0.1",
  "seq": 0,
  "prev": "0000000000000000000000000000000000000000000000000000000000000000",
  "ops": [],
  "balances": {},
  "supply": 0,
  "name": "Satoshi Token",
  "ticker": "SATO",
  "decimals": 8
}
```

**Mint 1000 to Alice (State 1)**

```json
{
  "profile": "mono.mrc20.v0.1",
  "seq": 1,
  "prev": "7a3b9c2d1e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b",
  "ops": [
    { "op": "urn:mono:op:mint", "to": "a1b2c3d4e5f6...", "amt": 1000 }
  ],
  "balances": {
    "a1b2c3d4e5f6...": 1000
  },
  "supply": 1000,
  "name": "Satoshi Token",
  "ticker": "SATO",
  "decimals": 8
}
```

**Alice sends 400 to Bob (State 2)**

```json
{
  "profile": "mono.mrc20.v0.1",
  "seq": 2,
  "prev": "1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c",
  "ops": [
    { "op": "urn:mono:op:transfer", "from": "a1b2c3d4e5f6...", "to": "b2c3d4e5f6a7...", "amt": 400 }
  ],
  "balances": {
    "a1b2c3d4e5f6...": 600,
    "b2c3d4e5f6a7...": 400
  },
  "supply": 1000,
  "name": "Satoshi Token",
  "ticker": "SATO",
  "decimals": 8
}
```

### 7.10 Indexing and Discovery (Non-Normative)

Clients require a method to:

1. Discover the current head UTXO for a contract
2. Retrieve state data for each transition
3. Locate spending transactions for verification

Verification requires:

- Block headers (to validate proof-of-work chain)
- Merkle inclusion proofs for each relevant transaction
- A method to determine current UTXO status (e.g., Electrum query, utreexo proof, or trusted indexer)

No full node is necessary, but UTXO status requires some form of indexing or proof infrastructure. The choice is application-specific and does not affect verification semantics.

### 7.11 Future: Self-Sovereign Variant

MRC20 v0.1 is issuer-controlled by design. A self-sovereign variant would require:

- **Option A: Embedded signatures** — Each TRANSFER/BURN includes a signature from `from` pubkey over the operation
- **Option B: Per-account sub-trails** — Each account has its own Blocktrail; transfers require cross-trail coordination
- **Option C: Delegation registry** — Issuer pre-authorizes account holders via signed delegation

These introduce significant complexity and are deferred to a future profile (e.g., `mono.mrc20-ss.v0.1`).

---

## 8. Security Considerations

### 8.1 Inherited from Blocktrails Core

- Hash preimage resistance → state binding
- Discrete log hardness → output key security
- Bitcoin consensus → spend ordering

### 8.2 Profile-Specific

| Threat | Mitigation |
|--------|------------|
| Invalid state accepted | Client MUST validate all transitions |
| Balance manipulation | Recompute from ops, verify match |
| Issuer misbehavior | Publicly auditable; cannot rewrite history |
| State withholding | Redundant state distribution |

### 8.3 Trust Boundaries

```
┌─────────────────────────────────────┐
│     Trusted (Blocktrails Core)      │
│  - Commitment integrity             │
│  - Spend ordering                   │
│  - Double-spend prevention          │
└─────────────────────────────────────┘
                  │
┌─────────────────────────────────────┐
│     Trusted (MRC20 Issuer)          │
│  - Honest operation authorization   │
│  - State availability               │
└─────────────────────────────────────┘
                  │
┌─────────────────────────────────────┐
│     Untrusted (Verifiable)          │
│  - Transition validity              │
│  - Balance correctness              │
│  - History integrity                │
└─────────────────────────────────────┘
```

---

## 9. Transport (Non-Normative)

Profiles do not mandate transport. Common patterns:

- Nostr events (kind TBD)
- HTTPS endpoints
- IPFS/Arweave blobs

---

## 10. Non-Goals

This specification does not provide:

- Confidentiality
- Zero-knowledge proofs
- DAG state graphs (see [DAG Extension](./dag.md) for opt-in multi-party DAG support)
- Script-enforced logic
- Multi-writer concurrency (see [DAG Extension](./dag.md) for multi-party trails)
- Self-sovereign account authorization (see 7.11)
- Allowances/approvals

---

## 11. Relationship to RGB

Monochrome occupies similar design space to RGB with different trade-offs:

| Aspect | Monochrome/MRC20 | RGB |
|--------|------------------|-----|
| State model | Linear | DAG |
| Visibility | Explicit | Concealed |
| Complexity | Minimal | Expressive |
| Authority | Issuer-controlled | Self-sovereign |
| Validation | JSON + JCS | Strict types |

---

## 12. References

- [RFC 8785: JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [Blocktrails Core Specification](https://blocktrails.org)
- [BIP-340: Schnorr Signatures](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)

---

## 13. Summary

| Layer | Purpose |
|-------|---------|
| Blocktrails Core | Ordering and anchoring |
| Monochrome | Profile framework |
| MRC20 | Issuer-controlled fungible tokens |

---

*Blocktrails Profiles v0.2 — Minimal client-validated state on Bitcoin.*
