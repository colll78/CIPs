---
CIP: 160
Title: Receiving Script Purpose and Addresses
Category: Ledger
Status: Proposed
Authors:
    - Philip DiSarro <philipdisarro@gmail.com>
Implementors: []
Discussions:
    - Original PR: https://github.com/cardano-foundation/CIPs/pull/1063
Created: 2025-07-24
License: CC-BY-4.0
---

## Abstract

This CIP proposes a protected form of payment address and a corresponding `Receiving` script purpose. This enhancement allows a smart contract to validate not only UTxO spends from its address, but also UTxO creations to its address. The lack of such a mechanism today forces developers to implement complex workarounds involving authentication tokens, threaded NFTs, and registry UTxOs to guard against unauthorized or malformed deposits. This CIP aims to provide a native mechanism to guard script addresses against incoming UTxOs, thereby improving protocol safety, reducing engineering overhead, and eliminating a wide class of vulnerabilities in the Cardano smart contract ecosystem.

## Motivation: Why is this CIP necessary?

In Cardano’s current eUTxO model, smart contracts can enforce logic only when their locked UTxOs are being spent. They have no ability to reject or validate UTxOs being sent to them. This leads to a fundamental weakness: anyone can send arbitrary tokens and datum to a script address, potentially polluting its state or spoofing valid contract UTxOs. To mitigate this, developers today must:

- Mint authentication tokens
- Use threading tokens to track contract state
- Build registry systems with always-fails scripts
- Validate datums defensively with token-datum context coupling

These workarounds add significant complexity, on-chain cost, and surface area for bugs. A native mechanism to guard UTxO creations at a script address would eliminate the need for most of these patterns.

## Specification

The following is the proposed protocol contract. Initial Dijkstra with a
receiving-aware Plutus V4 is the preferred target, subject to Ledger, Plutus and
release coordination. Publication of this amendment does not establish feature
admission, an activation protocol version, or that the V4 format remains open.
If either format is frozen, use an agreed successor era/language rather than
change an already deployed language schema.

### Protected payment addresses

Protection is an opt-in property of a Shelley payment address. Payment and
staking credential types do not change. The protected and unprotected forms
with identical credentials are different addresses.

Set header bit 3 (`0x08`) for a protected address. Only base address types
0–3 and enterprise types 6–7 support protection. The high-nibble type, credential
payloads and lengths are unchanged. In these payment addresses, bit 0 still
identifies testnet (`0`) or mainnet (`1`); bits 1–2 remain reserved and zero.
Thus protected testnet/mainnet low nibbles are `1000`/`1001`, respectively.
They do not identify new networks 8 or 9. See the proposed
[CIP-19 amendment](../CIP-0019/README.md#proposed-protected-payment-address-extension).

Protected pointer addresses (types 4–5), Byron addresses, reward/account
addresses, other families, reserved bits, malformed payloads and incorrect
lengths are invalid. Protection retains the `addr`/`addr_test` Bech32 prefixes.
Unprotected address bytes and historical decoding rules remain unchanged.
Before activation, transaction decoding/validation rejects protected outputs.
State restoration preserves previously validated protected addresses without
repeating creation authorization; production initial-fund injection must not
create protected outputs without that authorization and is rejected.

The proposed receiving-aware V4 address shape uses that language's existing
account-based staking representation:

```haskell
data Address
  = Address Credential (Maybe AccountId)
  | AddressProtected Credential (Maybe AccountId)
```

This replaces the candidate V4 Address's list Data schema with an indexed sum,
proposing ordinary constructor index `0` and protected index `1`. It is a
language-format change requiring explicit Plutus agreement before activation.
It must not be applied to a frozen or deployed V4 schema. Existing V1–V3
address schemas are unchanged.

### Receiving domain and redeemer indexes

The per-output model below is this amendment's proposed implementation
reconciliation of the [existing per-output validation rule](https://github.com/cardano-foundation/CIPs/blob/b4a593c960f2751fef2ddc8df28bec7b22c68eb5/CIP-0160/README.md#receiving-validation-rule)
and [the Ledger WG suggestion to resolve a TxOut into the purpose](https://github.com/cardano-foundation/CIPs/pull/1063#issuecomment-3222306948).
Those sources support individual output execution and visibility. The precise
raw-index mapping and Data fields proposed here still require upstream review;
the cited discussion does not establish approval of those formats or activation.

For each transaction body, including each nested body, enumerate all ordinary
outputs in authored order before selecting protected script outputs:

```text
receivingScriptTargets(body) =
  [(originalIndex, paymentScriptHash(output.address))
   | (originalIndex, output) <- enumerate body.ordinaryOutputs,
     output.address.isProtected,
     output.address.paymentCredential is a script]
receivingKeyHashes(body) = unique
  [paymentKeyHash(output.address)
   | output <- body.ordinaryOutputs, output.address.isProtected]
```

The index is the raw original zero-based position in that body's ordinary output
sequence. It is not a filtered-target rank or a sorted-script-hash rank. Ordinary,
key and native outputs retain their sequence positions; filtering does not
compress indexes. Script witness contents or language cannot renumber outputs.
An unprotected output creates no Receiving obligation, even when its payment
hash matches a protected output. No target list is added to transaction bytes.

Every protected Plutus output requires a separate Receiving execution, redeemer
and execution budget, including outputs sharing the same script hash, different
staking credentials, or byte-identical contents. Success for one output does not
authorize another output. Native scripts use existing phase-1 authorization and
require no Plutus Receiving redeemer. Script/key witness availability may
normally deduplicate hashes; execution identity is the individual output.

Ledger purpose item and redeemer-pointer views carry the original `Word32`
output index. Forward/inverse lookup verifies that index resolves to a protected
script output in the same body; the resolved item/pointer pair retains that
index in both fields. Never infer an index from the first equal TxOut: identical
duplicates remain distinct occurrences. Out-of-range, ordinary/key-output and
native-only redeemers are invalid under existing witness/exact-redeemer checks.

The same script hash and index in a parent and child body have independent
purposes, redeemers and budgets. Each Receiving context exposes the specific
output resolved at its index. `txInfoOutputs` retains authored order.

### Creation witnesses

Every ordinary protected output requires its payment credential's authorization
for its own transaction body:

- A payment key requires its existing signature over that body's hash. An
  existing matching signature can satisfy it. A signature for a different
  staking credential does not authorize the payment key; the same underlying
  key may satisfy existing payment and staking witness requirements.
- A native script requires its script and successful existing phase-1 checks,
  using that body's native-script environment. It has no Receiving redeemer.
- A Plutus script requires its script, a Receiving redeemer and successful
  phase-2 evaluation in the receiving-aware language.

Script witnesses and existing selected-input/reference-input reference scripts
are resolved under the era's ordinary sharing rules, including batch sharing
where provided. A script carried only by a newly created output cannot witness
that output's creation. A shared script may serve several purposes; each Plutus
purpose retains its own redeemer and execution budget.

Receiving adds no implicit datum argument and no blanket phase-1 requirement
for a new output datum-hash preimage. Validators can inspect inline datums or
preimages supplied through existing permitted datum witnesses. Existing datum,
missing/extraneous script and exact-redeemer checks continue to apply.

Protection governs output creation. Spending a protected UTxO still requires
the usual key/native/Plutus spending authorization; reference inputs do not
trigger Receiving. Network and stake accounting use the original credentials.
Deposits/withdrawals affecting reward accounts are outside this payment-output
domain. Receiving key witnesses do not implicitly add explicit guards.

### Receiving-aware script contexts

Extend the receiving-aware language's ScriptPurpose with
`Receiving ScriptHash Integer`, retaining the executing hash first and the raw
original body-local output index second. For proposed V4, extend ScriptInfo with
`ReceivingScript Integer TxOut`, exposing the original index and the exact
resolved output. Retain the executing hash in `scriptContextScriptHash`.
Existing V4 purposes, including
Guarding, remain available. `txInfoRedeemers` includes Receiving entries.

Preserve protection in all visible translated outputs, consumed inputs,
reference inputs and nested views, including those exposed to Guarding. The
Receiving validator checks its resolved output's datum, assets and staking
credentials as required by its contract. It may inspect other visible outputs
for wider invariants, but that does not replace their separate executions.

### Ledger redeemer tags and Plutus Data

Propose the following ledger CBOR redeemer tags, retaining Guarding at `6`:

```cddl
redeemer_tag =
    0 ; spend
  / 1 ; mint
  / 2 ; cert
  / 3 ; reward
  / 4 ; voting
  / 5 ; proposing
  / 6 ; guarding
  / 7 ; receiving
```

Ledger CBOR tags and Plutus Data constructor indexes are separate formats.
For the proposed receiving-aware V4, retain the existing ScriptPurpose Data
indexes (Minting 0, Spending 1, Withdrawing 2, Certifying 3, Voting 4,
Proposing 5, Guarding 6) and append Receiving at `7`. Retain the corresponding
ScriptInfo indexes and append ReceivingScript at `7`. Receiving encodes as
`Constr 7 [scriptHash, originalOutputIndex]`; ReceivingScript encodes as
`Constr 7 [originalOutputIndex, resolvedOutput]`. The matching numeric
Receiving indexes do not make the two complete numbering schemes identical.
Publish independent vectors for each format.

### Batch budgets, collateral and failure

Targets, signatures, redeemers, language views and integrity hashes are
body-local. Existing batch script availability, phase-2 collection/evaluation,
execution-unit limits, fees and block accounting include every Receiving
invocation across the batch.

A child-only Plutus invocation requires the top-level transaction's collateral
under the existing batch rule. Subtransactions gain no collateral-return field.
Key/native-only Receiving adds no Plutus collateral requirement.

A protected top-level collateral-return output is always invalid in phase 1,
even if scripts would succeed, and never enters the Receiving target domain.
A protected key UTxO used as collateral input follows existing eligibility and
key-witness rules.

If any Receiving evaluation fails or exceeds its budget, no ordinary output in
the batch is created. A matching phase-2-invalid transaction may follow the
existing collateral-only ledger transition, consuming collateral and creating
only an eligible unprotected collateral return. A claimed-valid transaction
whose scripts fail is rejected for the validity mismatch. Receiving adds no
separate rollback or evaluation pipeline.

### Vectors

[Protected address vectors](test-vectors/protected-addresses.json) reuse the
unchanged credential payloads from CIP-19's published address examples and set
only `0x08` for the six supported families on testnet and mainnet. They are
proposed format vectors, not evidence of network activation.

## Rationale: How does this CIP achieve its goals?

Protected payment addresses and the `Receiving` script purpose let contracts prevent unauthorized UTxO creation at their protected destinations. By giving smart contracts the ability to validate outputs being sent to them during phase-2 validation, developers can:

- Ensure only valid state transitions or authenticated deposits are accepted
- Enforce access control and structural correctness of datums before a UTxO is created
- Remove the need for workaround patterns such as:
  - Authentication tokens
  - Threading/state tokens
  - The issues from cyclic depenencies that are inherent in both of the above. 

Contracts opt in through address protection, without a new credential type.
Anyone may still send to an unprotected address with the same payment hash.
Contracts that rely on authenticated creation must verify protection during
spending as well as validate each specific protected output during Receiving.
Protection
does not automatically validate assets, datums or staking credentials, nor
replace every protocol's state-token invariants.

### Alternatives considered

- **Status quo**: Relying on auth tokens, thread tokens, and registry patterns introduces complexity, performance bottlenecks, and cyclic dependencies that are fragile and hard to audit. It has also led to serious exploits in real-world protocols due to misused or mishandled tokens.

- **Off-chain filtering**: While indexers and DApp backends can attempt to filter out junk UTxOs, they provide no on-chain security and cannot be relied on in adversarial environments or in composable settings.

- **Multivalidator pattern**: While technically feasible, this couples minting and spending logic into a single script, constrained by the 16KB script size limit. Furthermore, this introduces a huge layer of complexity and an associated attack surface. Managing the lifecycle of minted state tokens to prevent smuggling is extremely difficult, and becomes more impractical as the complexity of the dApp increases (ie. The attack surface for state token smuggling in a protocol that has 12 different validator scripts is nearly impossible to secure). In practice, this constraints the realm of what types of dApps are feasible on Cardano, you cannot build a cutting-edge financial instrument like AAVE, Balancer, or MakerDAO on Cardano because managing the lifecycle of dozens of state tokens across dozens of scripts while preventing smuggling is infeasible. 

### Backward Compatibility

Historical unprotected addresses and supported ordinary transactions retain
their encodings and behavior. A preactivation transaction cannot introduce a
protected address or Receiving redeemer.

When an older language's actual context would contain an unrepresentable
protected address or Receiving purpose, collection fails in phase 1 with a
specific context-translation error. This includes protected key/native outputs
visible to an older Plutus script, even when no Receiving Plutus script runs.
Protection must not silently be projected away. Apply this policy to the
context required by the language/purpose; it is not a blanket ban on every
mixed-language batch. Existing subtransaction restrictions on V1–V3 remain.

Wallets, nodes, and off-chain tooling must be updated to:
- Recognize and encode/decode protected payment addresses
- Include distinct output-indexed `Receiving` redeemers and budgets for every
  protected Plutus output, including identical duplicates
- Extend phase-2 validation to evaluate `Receiving` scripts

Node software, CLI, Plutus libraries, and serialization tooling (e.g., `cardano-api`, `cardano-ledger`, `plutus-ledger-api`) would require coordinated upgrades.

## Path to Active

### Acceptance Criteria

- Agreement from Cardano Ledger and Plutus teams
- Implementation of:
  - Protected payment addresses in address serialization
  - `Receiving` in ledger script validation rules
  - `Receiving` in Plutus
  - Phase-2 validation for transactions with outputs to protected addresses
- An agreed activation era/protocol version and receiving-aware language;
  Dijkstra/V4 is proposed, with successor-era/language fallback if frozen
- CIP-19, ledger CBOR/Huddle/CDDL, Plutus Data and formal-spec agreement
- Non-skipped executable-spec conformance, historical replay, performance and
  complete node/consensus construction/validation integration
- Named release-critical tooling/serialization/wallet readiness and an actual
  upgrade rehearsal; integration testing alone does not establish activation

### Implementation Plan

1. Extend ledger address types to represent protected payment addresses.
2. Modify transaction validation logic to detect outputs to protected addresses and invoke appropriate scripts.
3. Introduce CDDL changes for redeemer tags and the new address variant.
4. Update transaction witnesses, CLI tooling, and Plutus interpreter to support `Receiving`.
5. Provide test cases for:
   - Correct execution of `Receiving` scripts
   - Rejection of transactions that include outputs to protected addresses and do not have the required witnesses or where the associated `Receiving` script execution fails. 
6. Provide examples and documentation for contract authors.

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
