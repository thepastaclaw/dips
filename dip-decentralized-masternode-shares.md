<pre>
  DIP: TBD
  Title: Decentralized Masternode Shares
  Author(s): Pasta
  Comments-Summary: No comments yet.
  Status: Draft
  Type: Standard
  Created: 2026-06-20
  Requires: DIP-0002, DIP-0003, DIP-0023, DIP-0026
  License: MIT License
</pre>

## Table of Contents

1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Prior Work](#prior-work)
4. [Relationship to DIP-0026](#relationship-to-dip-0026)
5. [Specification](#specification)
    1. [Terminology](#terminology)
    2. [Parameters](#parameters)
    3. [Provider Transaction Version](#provider-transaction-version)
    4. [Shared Collateral Output](#shared-collateral-output)
    5. [Collateral Share](#collateral-share)
    6. [Shared Registration (ProRegTx v5)](#shared-registration-proregtx-v5)
    7. [Registration Consent Digest](#registration-consent-digest)
    8. [Shared Update Transactions](#shared-update-transactions)
    9. [Authorization Tiers](#authorization-tiers)
    10. [Reward Distribution](#reward-distribution)
    11. [Dissolution (ProDisTx)](#dissolution-prodistx)
    12. [Collateral Spend Enforcement](#collateral-spend-enforcement)
    13. [Deterministic Masternode State](#deterministic-masternode-state)
    14. [Simplified Masternode List and Filters](#simplified-masternode-list-and-filters)
    15. [Governance](#governance)
6. [Deployment and Compatibility](#deployment-and-compatibility)
7. [Rationale](#rationale)
8. [Test Cases](#test-cases)
9. [Implementation Notes](#implementation-notes)
10. [Security Considerations](#security-considerations)
11. [Open Issues](#open-issues)
12. [Copyright](#copyright)

## Abstract

This DIP extends [DIP-0003: Deterministic Masternode Lists](dip-0003.md) by
defining a new provider transaction payload version, `SharedCollateral` (`5`),
that allows between 2 and 8 mutually distrusting participants to jointly fund,
operate, and exit a single Regular or Evolution masternode under
protocol-enforced consent. The shared masternode is funded by an internal
collateral output that is locked by consensus to a dissolution covenant. Owner
rewards are split natively in the coinbase by recorded share amounts. Each
participant may unilaterally dissolve the masternode at any time, subject to a
participant-chosen penalty and the standard transaction fee, and the full set of
participants may dissolve unanimously with no penalty. Updates to fields that
affect all participants require consent from every participant; the operator
role remains unchanged.

Shared collateral is a strict superset of the [DIP-0026: Multi-Party
Payouts](dip-0026.md) reward-splitting mechanism. DIP-0026 leaves the
registrar owner in unilateral control of the payout list. This DIP introduces
a separate provider payload version so that share amounts, refund destinations,
and the collateral itself are bound to per-participant consent and cannot be
redirected by any single owner, operator, miner, or compromised update path.

## Motivation

Dash masternodes require 1000 DASH of collateral for a Regular masternode and
4000 DASH for an Evolution masternode. Many users wish to combine smaller
holdings to operate a single masternode and receive a proportional share of
rewards. Without protocol support, shared ownership requires an off-chain
custodian who controls the collateral, the registration keys, and the
distribution of rewards. This is a custodial relationship: the custodian can
abscond with collateral, freeze a participant out of rewards, refuse to exit,
or be compelled to do any of the above.

DIP-0026 makes the recurring reward split trustless, but a single registrar
owner can still rewrite the payout list, the collateral remains under one
party's control, and there is no consensus-enforced exit path. To make shared
masternodes fully trustless, the protocol must also enforce:

* atomic, multi-party formation of the masternode and its collateral output;
* a collateral covenant that allows funds to leave only through a valid
  dissolution transaction;
* per-participant refund obligations on exit;
* per-share reward shares tied to recorded collateral contributions;
* an authorization model that distinguishes individual, shared-registrar, and
  operator-controlled fields; and
* a unilateral exit path so that an unresponsive or hostile counterparty
  cannot freeze any participant's principal.

This DIP specifies those rules. The initial scope is deliberately narrow:
internal collateral only, immutable share amounts and participant set,
immutable refund scripts, immutable participant owner keys, per-share
reward-script self-update, unanimous shared-registrar updates,
operator-authorized service and operator-payout updates, and unilateral or
unanimous dissolution. Partial exit, participant replacement, external
collateral, owner-key rotation, refund-script mutation, relock transactions,
and on-chain fractional governance are out of scope and may be addressed by
future DIPs.

## Prior Work

* [DIP-0002: Special Transactions](dip-0002.md)
* [DIP-0003: Deterministic Masternode Lists](dip-0003.md)
* [DIP-0023: Enhanced Hard Fork Mechanism](dip-0023.md)
* [DIP-0026: Multi-Party Payouts](dip-0026.md)
* [DIP-0028: Evolution Masternodes](dip-0028.md)

## Relationship to DIP-0026

DIP-0026 introduces provider transaction payload version `4` (`MultiPayout`).
Under DIP-0026 v4, the registrar owner remains the sole signer for ProRegTx
and ProUpRegTx, and may freely rewrite the payout list at any time.

This DIP does not modify DIP-0026 v4 semantics. Shared masternodes use a new
provider transaction payload version, `5` (`SharedCollateral`), with the
following distinctions:

| Property | DIP-0026 v4 | This DIP, v5 |
| --- | --- | --- |
| Ownership model | Single registrar owner | 2 to 8 participants |
| Payout list authority | Registrar owner via ProUpRegTx | Per-share self-update; share amounts immutable |
| Collateral lock | Single owner's UTXO | Shared collateral covenant |
| Exit path | Owner spends collateral | Valid `ProDisTx` only |
| Voting / operator updates | Owner | Unanimous participant consent |
| Refund on exit | None enforced | Per-participant refund script enforced |

A v5 payload MUST NOT be reinterpreted as a v4 payload, and a v4 payload MUST
NOT be reinterpreted as v5. A single-owner masternode using v3 or v4 cannot
be upgraded to v5 in place: shared collateral is established only by a new
v5 registration, and a shared masternode is wound down only by a valid
`ProDisTx`. There is no in-place downgrade from v5 to v3 or v4.

## Specification

### Terminology

| Term | Definition |
| --- | --- |
| Participant | One of the 2 to 8 owners of a shared masternode, identified by a participant owner key. |
| Share | One participant's recorded collateral amount, refund script, reward script, and owner key. |
| Share table | The ordered list of `CollateralShare` entries in a shared registration. |
| Shared collateral | The internal collateral output of a shared masternode, locked to the shared-collateral script template. |
| Shared collateral script template | The fixed serialized output script used by every shared collateral output (see [Shared Collateral Output](#shared-collateral-output)). |
| Registration consent digest | The domain-separated digest each participant signs to authorize a shared registration. |
| Dissolution authorization digest | The domain-separated digest signed to authorize a `ProDisTx`. |
| Unilateral dissolution | A dissolution authorized by a single participant ("the actor") with a penalty redistributed to non-actors. |
| Unanimous dissolution | A dissolution authorized by every participant with no penalty. |
| Actor | The participant chosen by `actorIndex` who pays the transaction fee and, in unilateral mode, the penalty. |
| Early period | The window of `earlyPeriodBlocks` blocks after registration during which `earlyPenalty` applies to unilateral dissolution. |

### Parameters

| Constant | Value | Description |
| --- | ---: | --- |
| `SHARED_MIN_PARTICIPANTS` | 2 | Minimum number of shares. |
| `SHARED_MAX_PARTICIPANTS` | 8 | Maximum number of shares, matching DIP-0026 payout-count limits. |
| `SHARED_MIN_SHARE_DUFFS` | 1000000000 (10 DASH) | Minimum value of any `shares[i].amount`. |
| `SHARED_MAX_EARLY_PERIOD_BLOCKS` | 420480 (~2 years at nominal 2.5 min/block) | Maximum value of `earlyPeriodBlocks`. |

`SHARED_MAX_EARLY_PERIOD_BLOCKS` is given in blocks at Dash's nominal block
interval of 2.5 minutes (150 s), yielding approximately
`365 * 24 * 60 / 2.5 * 2 = 420480` blocks for a two-year ceiling. Realized block
times vary, so the wall-clock equivalent drifts with hashrate; activation may
refine this constant to the actual measured cap selected at deployment time.
Implementations and tooling that prefer a wall-clock interface SHOULD convert
the user-facing value to blocks at the nominal 2.5 min/block rate and reject
any input that exceeds `SHARED_MAX_EARLY_PERIOD_BLOCKS` once converted.

`SHARED_MIN_SHARE_DUFFS` exists to keep recurring per-share coinbase outputs
above policy dust thresholds at present block rewards while permitting modest
participation levels. Wallets SHOULD warn participants that the block reward
declines over time and very small shares may produce dust-class outputs in
the future.

Shared collateral amounts MUST satisfy:

```text
sum(shares[i].amount) == GetMnType(nType).collat_amount
```

where `GetMnType(nType).collat_amount` is 1000 DASH for `Regular` or 4000 DASH
for `Evo`, as defined in DIP-0003 Appendix B and DIP-0028. A registration that
does not exactly sum to the required collateral is invalid.

Penalty parameters MUST satisfy:

```text
0 <= standardPenalty <= earlyPenalty
earlyPenalty   < min(shares[i].amount)
standardPenalty < min(shares[i].amount)
```

The strict `<` ensures that even the smallest share has a positive remainder
for the unilateral actor after the penalty is paid, so a unilateral
dissolution by any participant always returns a non-zero amount to the actor
before fees.

### Provider Transaction Version

This DIP introduces one new ProRegTx payload version and three new special
transaction types, each with its own independent payload version.

| Name | Value | Applies to |
| --- | ---: | --- |
| `ProTxVersion::SharedCollateral` | 5 | `ProRegTx` payload version for shared-collateral registration and the corresponding shared masternode state. |

`ProTxVersion::SharedCollateral = 5` is the provider transaction payload
version of a v5 `ProRegTx` and is also the version tag carried in the
deterministic masternode state for masternodes registered by such a
transaction. It does NOT apply as the payload version of the new special
transaction types introduced below; those payloads carry their own,
independent `nVersion` field that starts at `1` (see each payload
definition).

Provider payload version `5` is reserved for shared-collateral semantics.
Future protocol changes that affect the single-owner `ProRegTx` flow MUST
use the next available provider payload version (`6` or later) rather than
reusing `5` as a generic mode flag.

| Name | Value | Description |
| --- | ---: | --- |
| `TRANSACTION_PROVIDER_DISSOLVE` | 10 | `ProDisTx` payload (this DIP). |
| `TRANSACTION_PROVIDER_UPDATE_SHARE` | 11 | `ProUpShareTx` payload (this DIP). |
| `TRANSACTION_PROVIDER_UPDATE_SHARED_REGISTRAR` | 12 | `ProUpSharedRegTx` payload (this DIP). |

The numeric assignments `10`, `11`, and `12` are tentative. Before this DIP
is merged or activated, the assignments MUST be checked against the
authoritative DIP-0002 registry and the current Dash Core
`TRANSACTION_*` allocation in `primitives/transaction.h`. If any of these
values is already consumed by another DIP or pending allocation at that
time, this DIP MUST be updated to use the next free values. The names
(`PROVIDER_DISSOLVE`, `PROVIDER_UPDATE_SHARE`,
`PROVIDER_UPDATE_SHARED_REGISTRAR`) are normative; this DIP does not
publish an authoritative table of unrelated special transaction type
allocations.

ProRegTx (type `1`) is reused at payload version 5 for shared registration.
The base `ProUpRegTx` (type `3`) is invalid against a v5 masternode and is
unaffected for non-shared masternodes. The base `ProUpServTx` (type `2`)
continues to operate under DIP-0003 / DIP-0028 rules for non-shared
masternodes AND remains valid against a v5 masternode for the
operator-authorized fields enumerated in [Authorization
Tiers](#authorization-tiers); `ProUpServTx` MUST NOT attempt to modify any
owner-controlled or shared-collateral field.

### Shared Collateral Output

A shared registration creates exactly one internal collateral output, here
called the **shared collateral output**. The output value MUST equal
`GetMnType(nType).collat_amount`.

The shared collateral output uses a single, fixed serialized script,
`SHARED_COLLATERAL_SCRIPT`, with the following normative properties:

1. It is a fixed byte sequence specified by this DIP; every shared collateral
   output on every network uses the identical script bytes. Script-template
   equality is the registration-time selector used to identify which output
   of a v5 ProRegTx is the shared collateral output; it is NOT, on its own,
   the consensus identifier of a protected collateral.
2. It contains no participant-specific data. In particular, it does not embed
   any participant owner key, refund script, reward script, share amount, or
   `proTxHash`. The per-masternode binding between an outpoint and a
   `proTxHash` is provided by the deterministic masternode state, not by the
   script itself.
3. It is recognizable by full nodes using a single template comparison, and
   it is recognizable as non-standard by older nodes that do not understand
   shared collateral.
4. After activation, consensus protects **shared collateral outpoints** — the
   set of UTXOs that are either (a) recorded as the collateral outpoint of an
   active v5 masternode in the deterministic masternode list, or (b) created
   by a valid v5 ProRegTx earlier in the same block currently being
   validated. Any spend of an outpoint in this set is rejected unless the
   spending transaction is a valid `ProDisTx` for the corresponding
   masternode. Outpoints in this set are referred to below as
   "active-or-pending shared collateral outpoints" and the rejection is
   enforced at mempool acceptance, at block connection, and during
   deterministic masternode list processing (see [Collateral Spend
   Enforcement](#collateral-spend-enforcement)).
5. Ordinary UTXOs that happen to pay `SHARED_COLLATERAL_SCRIPT` but are NOT
   active-or-pending shared collateral outpoints are NOT bound by the
   dissolution covenant. They behave as ordinary outputs at script-evaluation
   time, subject to whatever spending conditions the recommended template
   imposes (for example, the `OP_TRUE` redeem-script template makes them
   anyone-can-spend). Implementations MUST NOT retroactively lock arbitrary
   pre-existing outputs or unrelated outputs created outside the v5
   registration flow.
6. Before activation, miners and relays MUST treat any output with this
   script as non-standard, so ordinary relay and mining policy discourages
   creating such outputs before activation. After activation, the only way
   for a transaction to register an outpoint into the protected set is a
   valid v5 ProRegTx;
   relay and mining policy MUST continue to treat any other output bearing
   `SHARED_COLLATERAL_SCRIPT` as non-standard to discourage accidental
   creation of stranded anyone-can-spend outputs.

The recommended template is a P2SH of an `OP_TRUE`-equivalent redeem script,
chosen so that the output is recognizable by template equality and so that no
existing wallet would construct an equivalent output by accident. The exact
bytes of `SHARED_COLLATERAL_SCRIPT` are assigned during implementation review,
constrained to the properties above; any implementation choice MUST be
identical across all consensus implementations and MUST be encoded into the
chain parameters used for activation. The DIP is amended with the assigned
bytes before mainnet activation.

This DIP is therefore NOT final for activation: it remains a Draft until
the concrete `SHARED_COLLATERAL_SCRIPT` byte sequence is assigned and
inserted here. The draft is suitable for review of the consensus rules and
covenant model, but no implementation may treat the script as fixed until
this section is updated with the assigned bytes (see [Open
Issues](#open-issues)).

Rationale: a P2SH anyone-can-spend template makes the script-level
satisfaction trivial so that a valid `ProDisTx` can construct a sensible
`scriptSig`, while consensus rules above script evaluation reject all
spending paths except dissolution. Embedding per-masternode commitments
(such as `proTxHash` or participant keys) into the output script was
considered and rejected because it (i) would conflict with future
relock-style updates of participant keys and (ii) would require either large
output scripts or a separate commitment scheme that would still depend on
deterministic masternode state to verify.

### Collateral Share

```text
CollateralShare {
    amount       : CAmount  (8 bytes, little-endian, in duffs)
    refundScript : CScript  (compact-size length-prefixed)
    rewardScript : CScript  (compact-size length-prefixed)
    ownerKey     : CKeyID   (20 bytes)
    joinSig      : vector<unsigned char>  (compact-size length-prefixed, 65 bytes when present)
}
```

| Field | Mutability | Authorization to change |
| --- | --- | --- |
| `amount` | Immutable | Set at registration; cannot be changed. |
| `refundScript` | Immutable | Set at registration; cannot be changed. |
| `rewardScript` | Mutable per-share | Participant owner key of that share, via `ProUpShareTx`. |
| `ownerKey` | Immutable | Set at registration; cannot be rotated in this DIP. |
| `joinSig` | Set only in `ProRegTx` payload | Validated at registration; not stored in deterministic state. |

Field rules at registration:

1. `amount` MUST be at least `SHARED_MIN_SHARE_DUFFS`.
2. `refundScript` MUST be a standard `P2PKH` or `P2SH` script.
3. `rewardScript` MUST be a standard `P2PKH` or `P2SH` script. An empty
   `rewardScript` is interpreted at the consensus layer as `refundScript`.
4. `ownerKey` MUST be distinct from every other `ownerKey` in the share
   table and from `keyIDVoting`.
5. `ownerKey` MUST be distinct from every active owner key recorded for any
   other registered masternode at the registration height. The
   unique-property index defined in DIP-0003 is extended to track every
   `shares[i].ownerKey` for v5 masternodes (see [Deterministic Masternode
   State](#deterministic-masternode-state)).
6. `refundScript` MUST NOT be duplicated within the share table.
7. `rewardScript` MAY be duplicated within the share table.
8. No `refundScript` and no `rewardScript` may be a P2PKH script paying to
   `keyIDOwner` (for v5 the field is absent — see below), `keyIDVoting`, or
   to a key ID that equals any `shares[i].ownerKey`. The intent matches
   the DIP-0026 key-reuse rule: these scripts are spent in lower-trust
   wallet contexts and MUST NOT reuse keys that control masternode state.
9. `joinSig` is a 65-byte compact ECDSA signature by `ownerKey` over the
   registration consent digest defined in [Registration Consent
   Digest](#registration-consent-digest). Any other length or any
   signature that does not verify under `ownerKey` is invalid.

### Shared Registration (ProRegTx v5)

A v5 ProRegTx has the same general layout as the v4 ProRegTx from DIP-0026
with the following differences. Field positions for a v5 payload are
specified normatively below; this list is exhaustive and replaces the v4
ProRegTx layout for `nVersion == 5`.

| Field | Type | Notes |
| --- | --- | --- |
| `nVersion` | `uint16_t` | MUST be `5`. |
| `nType` | `MnType` (`uint16_t`) | `Regular` or `Evo`. |
| `nMode` | `uint16_t` | MUST be `0`. |
| `collateralOutpoint.hash` | `uint256` | MUST be the null hash. |
| `collateralOutpoint.n` | `uint32_t` | Index of the shared collateral output within this transaction. |
| `netInfo` | DIP-0003/0028 net info | As for v4. |
| `pubKeyOperator` | BLS public key (basic scheme) | As for v4. |
| `keyIDVoting` | `CKeyID` | As for v4. |
| `nOperatorReward` | `uint16_t` | Basis points; as for v4. |
| `shares` | `CollateralShare[]` | 2 to 8 entries; per [Collateral Share](#collateral-share). |
| `earlyPeriodBlocks` | `uint32_t` | `0` to `SHARED_MAX_EARLY_PERIOD_BLOCKS`. |
| `earlyPenalty` | `CAmount` (8 bytes) | Duffs. |
| `standardPenalty` | `CAmount` (8 bytes) | Duffs. |
| `platformNodeID`, ports (Evo only) | as DIP-0028 | As for v4. |
| `inputsHash` | `uint256` | `CalcTxInputsHash(tx)`. |
| `vchSig` | `vector<unsigned char>` | MUST be empty for v5 (no external collateral signature). |

There is no `keyIDOwner` field in a v5 ProRegTx. The role of the single owner
key in DIP-0003 is performed in v5 by the collection of `shares[*].ownerKey`.

There is no `scriptPayout` or `payouts` field in a v5 ProRegTx. Owner rewards
are derived directly from the share table as specified in [Reward
Distribution](#reward-distribution).

A v5 ProRegTx is invalid if any of the following conditions hold:

1. `nVersion != 5`.
2. `nType` is not `Regular` or `Evo`.
3. `nMode != 0`.
4. `collateralOutpoint.hash` is not the null hash.
5. `collateralOutpoint.n` does not point to an output of this transaction.
6. The output at `collateralOutpoint.n` does not pay
   `GetMnType(nType).collat_amount` to `SHARED_COLLATERAL_SCRIPT`.
7. The transaction creates more than one output paying
   `SHARED_COLLATERAL_SCRIPT`.
8. `shares.size()` is less than `SHARED_MIN_PARTICIPANTS` or greater than
   `SHARED_MAX_PARTICIPANTS`.
9. Any share fails its per-field validation rules in [Collateral
   Share](#collateral-share).
10. `sum(shares[i].amount) != GetMnType(nType).collat_amount`.
11. Penalty parameter constraints in [Parameters](#parameters) are not
    satisfied.
12. `earlyPeriodBlocks > SHARED_MAX_EARLY_PERIOD_BLOCKS`.
13. `vchSig` is non-empty.
14. `inputsHash != CalcTxInputsHash(tx)`.
15. For any share `i`, `shares[i].joinSig` does not verify against
    `shares[i].ownerKey` over the registration consent digest defined
    below.

Rules 14 and 15 are evaluated in order: `inputsHash` mismatch and the
recomputed `outputsHash` mismatch (see below) are checked before any
ECDSA signature verification.

### Registration Consent Digest

Each participant signs an explicit consent digest that commits to the full
effect of the registration transaction, independent of any
sighash-mode behavior of the funding-input scripts.

```text
SharedRegConsentHash = SHA256d(
    "DashSharedMNReg" ||
    chainGenesisHash ||
    LE16(tx.nVersion)  || LE16(tx.nType)  || LE32(tx.nLockTime) ||
    inputsHash         || outputsHash     ||
    LE16(payload.nVersion) ||
    LE16(payload.nType) || LE16(payload.nMode) ||
    LE32(payload.collateralOutpoint.n) ||
    netInfoSerialized ||
    platformFieldsSerialized ||      // present iff nType == Evo
    pubKeyOperatorSerialized ||
    keyIDVoting ||
    LE16(nOperatorReward) ||
    LE8(shares.size()) ||
    sharesWithoutJoinSigs ||
    LE32(earlyPeriodBlocks) ||
    LE64(earlyPenalty) ||
    LE64(standardPenalty)
)
```

Where:

* `LE8`, `LE16`, `LE32`, `LE64` denote little-endian encodings of the
  indicated width.
* `SHA256d(x)` is the Dash double-SHA-256 used for transaction hashes.
* `chainGenesisHash` is the 32-byte block hash of the genesis block of the
  network on which the registration is being authorized (mainnet, testnet,
  devnet, regtest, or any future fork). Consensus uses the genesis hash of
  the chain that is validating the transaction. Each participant signer
  MUST independently verify the network / chain context of the
  registration before producing `joinSig`; a signature produced against
  one network's `chainGenesisHash` MUST NOT verify on any other network.
* `inputsHash` is `CalcTxInputsHash(tx)`. Consensus MUST recompute this
  from the transaction and compare it to the `payload.inputsHash` field
  before signature verification; mismatch is invalid.
* `outputsHash` is the SHA-256d of the concatenation of every output of
  `tx` in order, each serialized as `(value, scriptPubKey)` using the
  standard Dash transaction-output serialization. Consensus MUST recompute
  this from the transaction; there is no `outputsHash` field on a
  ProRegTx, but the recomputed value is used inside this digest.
* `netInfoSerialized` and `platformFieldsSerialized` use the same encoding
  as in the provider transaction payload itself.
* `pubKeyOperatorSerialized` is the basic-scheme BLS public key
  serialization (48 bytes).
* `sharesWithoutJoinSigs` is the concatenation of every `CollateralShare`
  in share order with the `joinSig` field omitted (treated as a zero-length
  byte string) so that signatures are not self-referential.

The domain separator `"DashSharedMNReg"` is the byte sequence of those
ASCII characters with no trailing NUL.

Funding-input signatures are still required to authorize spending of each
participant's funding UTXOs, but consensus MUST NOT rely on those
signatures to demonstrate consent to shared-masternode parameters: Dash
inherits Bitcoin sighash modes including `SIGHASH_NONE`, `SIGHASH_SINGLE`,
and `SIGHASH_ANYONECANPAY`, any of which permits later modification of
fields a participant would otherwise believe they had committed to.

### Shared Update Transactions

Two new special transaction types specialize updates for shared
masternodes. The base `ProUpRegTx` (type `3`) is invalid for v5 masternodes,
because its single-owner authorization model is incompatible with the
shared-registrar tier defined here. The base `ProUpServTx` (type `2`)
remains valid for v5 masternodes with operator authorization (see
[Authorization Tiers](#authorization-tiers)).

#### ProUpShareTx (type 11)

A `ProUpShareTx` updates exactly one share's mutable fields. In this DIP
the only mutable per-share field is `rewardScript`.

```text
CProUpShareTx {
    nVersion     : uint16_t     // MUST be 1 (payload version of this new special tx type)
    proTxHash    : uint256
    shareIndex   : uint16_t     // index into shares[]
    newRewardScript : CScript
    inputsHash   : uint256
    sig          : vector<unsigned char>  // 65 bytes
}
```

`nVersion` here is the independent payload version of the new
`TRANSACTION_PROVIDER_UPDATE_SHARE` special transaction; it is NOT the
provider payload version `ProTxVersion::SharedCollateral = 5` of the v5
`ProRegTx`. The two version namespaces are separate.

Validation:

1. The masternode identified by `proTxHash` MUST exist and MUST be a v5
   masternode.
2. `shareIndex` MUST be less than `shares.size()` in the current state.
3. `newRewardScript` MUST be a standard `P2PKH` or `P2SH` script. An empty
   `newRewardScript` is interpreted as `shares[shareIndex].refundScript`.
4. `newRewardScript` MUST satisfy the same key-reuse restrictions as
   registration: it MUST NOT be a P2PKH paying to any
   `shares[i].ownerKey` and MUST NOT pay to `keyIDVoting`.
5. `inputsHash` MUST equal `CalcTxInputsHash(tx)`.
6. `sig` MUST be a valid ECDSA compact signature by
   `shares[shareIndex].ownerKey` over:

   ```text
   SHA256d("DashSharedMNUpShare" ||
           proTxHash || LE16(shareIndex) || newRewardScript || inputsHash)
   ```

A valid `ProUpShareTx` replaces `shares[shareIndex].rewardScript` in the
deterministic masternode state. It does not revive a PoSe-banned masternode
and does not affect any other field.

#### ProUpSharedRegTx (type 12)

A `ProUpSharedRegTx` updates fields that affect all participants. In this
DIP the mutable shared-registrar fields are `pubKeyOperator`, `keyIDVoting`,
and `nOperatorReward`.

```text
CProUpSharedRegTx {
    nVersion        : uint16_t     // MUST be 1 (payload version of this new special tx type)
    proTxHash       : uint256
    pubKeyOperator  : CBLSPublicKey  // basic scheme
    keyIDVoting     : CKeyID
    nOperatorReward : uint16_t
    inputsHash      : uint256
    vchSigs         : vector<vector<unsigned char>>  // exactly shares.size() entries, in share order
}
```

As with `ProUpShareTx`, this `nVersion` is the independent payload version
of the new `TRANSACTION_PROVIDER_UPDATE_SHARED_REGISTRAR` special
transaction and is unrelated to `ProTxVersion::SharedCollateral = 5`.

Validation:

1. The masternode identified by `proTxHash` MUST exist and MUST be a v5
   masternode.
2. `pubKeyOperator` MUST be a valid basic-scheme BLS public key and MUST
   NOT collide with the operator key of any other registered masternode.
3. `keyIDVoting` MUST NOT equal any `shares[i].ownerKey`.
4. `inputsHash` MUST equal `CalcTxInputsHash(tx)`.
5. `vchSigs.size()` MUST equal `shares.size()`.
6. For each `i` in `[0, shares.size())`, `vchSigs[i]` MUST be a valid
   ECDSA compact signature by `shares[i].ownerKey` over:

   ```text
   SHA256d("DashSharedMNUpSharedReg" ||
           proTxHash || pubKeyOperatorSerialized ||
           keyIDVoting || LE16(nOperatorReward) || inputsHash)
   ```

7. Signature verification proceeds in share order. Any missing, extra, or
   out-of-order signature is invalid.

A valid `ProUpSharedRegTx` replaces `pubKeyOperator`, `keyIDVoting`, and
`nOperatorReward` in the deterministic masternode state. It does not revive
a PoSe-banned masternode and does not affect share contents, collateral, or
service fields. Operator-controlled fields (`scriptOperatorPayout`,
`netInfo`, Platform identifiers) are unchanged by this transaction; they
continue to be updated by the operator using `ProUpServTx`.

### Authorization Tiers

| Field or action | Authorization | Transaction |
| --- | --- | --- |
| Unilateral dissolution | One signature: `shares[actorIndex].ownerKey` | `ProDisTx` (mode `0`) |
| Unanimous dissolution | One signature per participant in share order | `ProDisTx` (mode `1`) |
| `pubKeyOperator` | One signature per participant in share order | `ProUpSharedRegTx` |
| `keyIDVoting` | One signature per participant in share order | `ProUpSharedRegTx` |
| `nOperatorReward` | One signature per participant in share order | `ProUpSharedRegTx` |
| `rewardScript` for share `i` | `shares[i].ownerKey` | `ProUpShareTx` |
| `refundScript` for share `i` | Immutable | — |
| `ownerKey` for share `i` | Immutable | — |
| `shares[i].amount`, share count, penalties, `earlyPeriodBlocks` | Immutable | — |
| `netInfo` (addresses) | Operator BLS key | `ProUpServTx` |
| `scriptOperatorPayout` | Operator BLS key | `ProUpServTx` |
| Platform service endpoints (Evo) | Operator BLS key | `ProUpServTx` |
| PoSe revocation | Operator BLS key | `ProUpRevTx` |

`ProUpRegTx` (type `3`) is invalid against a v5 masternode. `ProUpServTx` and
`ProUpRevTx` continue to use operator authorization as in DIP-0003 and
DIP-0028. The operator can stop service or revoke the masternode but cannot
redirect owner rewards or spend the shared collateral.

### Reward Distribution

Owner-reward distribution for a v5 masternode reuses the DIP-0026 pipeline
but weights by share amount rather than by basis points.

1. Compute the masternode reward and apply the Platform credit-pool
   reallocation as in current rules.
2. If `nOperatorReward != 0` and the operator has set
   `scriptOperatorPayout`, compute
   `operatorAmount = floor(masternodeReward * nOperatorReward / 10000)`
   and subtract from `masternodeReward` to obtain `ownerReward`. Otherwise
   `operatorAmount = 0` and `ownerReward = masternodeReward`.
3. Distribute `ownerReward` across `shares` using the helper
   `DistributeByWeight(ownerReward, shares[i].amount)` defined below.
4. For each share `i` whose computed reward is non-zero, create one
   coinbase output paying that amount to:
   * `shares[i].rewardScript`, if `shares[i].rewardScript` is non-empty;
   * `shares[i].refundScript`, otherwise.
5. If `operatorAmount != 0`, create the operator payout output to
   `scriptOperatorPayout` using current rules.

```text
DistributeByWeight(total, weights):
    sum_w = sum(weights)
    require sum_w > 0                            // reject all-zero weights
    out[i] = floor(total * weights[i] / sum_w)   for each i
    remainder = total - sum(out[i])
    // remainder is distributed deterministically to indices with positive
    // weights only, in ascending index order, one duff per such index,
    // until exhausted. Indices with weight 0 are skipped so that no duff is
    // ever credited to a zero-weight recipient.
    for i in 0..weights.size():
        if remainder == 0: break
        if weights[i] == 0: continue
        out[i] += 1
        remainder -= 1
    return out
```

`DistributeByWeight` is invalid for an all-zero weight vector and callers
MUST NOT invoke it in that case; for unanimous dissolution, where the
penalty itself is zero, the bonus is taken to be all-zero directly without
calling `DistributeByWeight` (see [Dissolution
(ProDisTx)](#dissolution-prodistx)).

Coinbase validation requires every expected per-share reward output by exact
amount and script. As in DIP-0026, the relative order of outputs within the
coinbase is not consensus-significant.

The `DistributeByWeight` helper is also used in [Dissolution
(ProDisTx)](#dissolution-prodistx) for penalty redistribution.

### Dissolution (ProDisTx)

A `ProDisTx` is the only transaction that can spend a shared collateral
output after activation. It is special transaction type
`TRANSACTION_PROVIDER_DISSOLVE` (`10`).

```text
CProDisTx {
    nVersion    : uint16_t   // MUST be 1 (payload version of this new special tx type)
    proTxHash   : uint256
    mode        : uint8_t    // 0 = unilateral, 1 = unanimous
    actorIndex  : uint16_t
    inputsHash  : uint256
    outputsHash : uint256
    vchSigs     : vector<vector<unsigned char>>
}
```

`nVersion` here is the independent payload version of the new
`TRANSACTION_PROVIDER_DISSOLVE` special transaction and is unrelated to
`ProTxVersion::SharedCollateral = 5`.

#### Transaction shape

A valid dissolution transaction satisfies all of:

1. It has exactly one input, and that input spends the shared collateral
   outpoint of the masternode identified by `proTxHash`. No additional
   funding inputs are permitted in this initial scope.
2. Its outputs are the refund outputs defined below, in share order, with
   no additional outputs of any kind. Change outputs are not permitted.
3. Its `nLockTime` MAY be non-zero. The dissolution authorization digest
   commits to `nLockTime`, so consensus enforces whatever value was signed.

#### Authorization digest

```text
SharedDisHash = SHA256d(
    "DashSharedMNDissolve" ||
    chainGenesisHash ||
    LE16(tx.nVersion) || LE16(tx.nType) || LE32(tx.nLockTime) ||
    inputsHash || outputsHash ||
    LE16(payload.nVersion) ||
    proTxHash ||
    LE8(mode) || LE16(actorIndex)
)
```

Where `inputsHash` is `CalcTxInputsHash(tx)` and `outputsHash` is the
SHA-256d of the concatenated serialized outputs of `tx`, both recomputed
from the actual transaction. Consensus MUST recompute both hashes and
reject mismatches with the payload's `inputsHash` and `outputsHash`
fields before any signature is verified. `chainGenesisHash` is the
32-byte genesis block hash of the network validating the transaction,
identical to its use in [Registration Consent
Digest](#registration-consent-digest); a dissolution signed against one
network's `chainGenesisHash` MUST NOT verify on any other. Each signer
MUST independently verify the network / chain context before producing
its `vchSigs` entry. `payload.nVersion` here is the dissolution payload
version (currently `1`), not the v5 ProRegTx provider payload version.

#### Period and penalty

Let `H` be the height at which the dissolution is connected, and let
`state` be the deterministic masternode state for `proTxHash` at the parent
block of `H`. Define:

```text
early   = (H - state.nRegisteredHeight) < state.earlyPeriodBlocks
penalty = (mode == 1) ? 0 : (early ? state.earlyPenalty : state.standardPenalty)
```

#### Signature cardinality

For `mode == 0` (unilateral):

1. `actorIndex < state.shares.size()`.
2. `vchSigs.size() == 1`.
3. `vchSigs[0]` MUST verify under `state.shares[actorIndex].ownerKey` over
   `SharedDisHash`.

For `mode == 1` (unanimous):

1. `actorIndex < state.shares.size()`.
2. `vchSigs.size() == state.shares.size()`.
3. For each `i`, `vchSigs[i]` MUST verify under `state.shares[i].ownerKey`
   over `SharedDisHash`. Signatures are processed in share order; any
   missing, extra, or out-of-order signature is invalid.

#### Output covenant

Let `a = actorIndex` and `N = shares.size()`. Define the non-actor bonus
distribution:

```text
if mode == 1:
    // unanimous: penalty is zero, bonus is trivially zero everywhere.
    // DistributeByWeight is NOT invoked.
    bonus[i] = 0 for all i in [0, N)
else:
    // unilateral: redistribute the penalty pro rata among non-actors.
    // At least one non-actor weight is positive because N >= 2 and every
    // share amount is positive, so sum_w > 0 and DistributeByWeight is
    // well-defined.
    weights[i] = (i == a) ? 0 : state.shares[i].amount
    bonus      = DistributeByWeight(penalty, weights)
```

Define the per-share dissolution value:

```text
value[i] = (i == a) ? actorValue
                    : state.shares[i].amount + bonus[i]
```

where `actorValue` is the value chosen by the constructor of the
transaction, subject to:

```text
0 <= actorValue <= state.shares[a].amount - penalty
```

The outputs of the dissolution transaction MUST be exactly, in order
matching the share table with the actor slot optionally omitted:

* Walk the share table in ascending `i` from `0` to `N - 1`. For each
  `i`:
  * If `i != a`, the transaction MUST contain the next output at this
    position, paying exactly `state.shares[i].amount + bonus[i]` to
    `state.shares[i].refundScript`.
  * If `i == a` and `actorValue > 0`, the transaction MUST contain the
    next output at this position, paying exactly `actorValue` to
    `state.shares[a].refundScript`.
  * If `i == a` and `actorValue == 0`, the actor's output MUST be
    omitted entirely. No dust, OP_RETURN, or placeholder output stands
    in for the actor.

Non-actor outputs always appear in share order and are exact; the
actor's output appears in its share-order slot when present and is
absent otherwise. No outputs other than those defined above are
permitted. In particular, OP_RETURN outputs, change outputs, and
operator outputs are not permitted.

Consensus matches outputs by walking the share table in share order and
either consuming the next output (for non-actor slots and for the actor
slot when present) or skipping the slot entirely (only for the actor
when `actorValue` would be zero). Any deviation — extra output, missing
non-actor output, wrong refund script, wrong amount, or a stray actor
output when the value would be zero — is invalid.

For `mode == 1` (unanimous), `penalty` is zero, every non-actor output
pays exactly `state.shares[i].amount` to `state.shares[i].refundScript`,
and the actor output pays `actorValue` with `0 <= actorValue <=
state.shares[a].amount` (or is omitted when `actorValue == 0`).

#### Fee accounting

The difference between the spent shared collateral value and the sum of
all outputs is the transaction fee:

```text
fee = state.shared_collateral_value - sum(outputs.value)
```

Because every non-actor output is exact and the actor output is bounded
above by `state.shares[a].amount - penalty`, this fee can be taken only
from the actor's share. The fee MUST be non-negative; consensus enforces
this by the actor-output upper bound and the no-extra-output rule.

A zero-fee dissolution is consensus-valid. Relay and mining policy MAY
require a non-zero fee at the standard minimum relay fee rate; wallets
SHOULD construct dissolutions paying the current standard fee so that
miners include them.

#### Same-block ordering

A `ProDisTx` MAY appear in the same block as the v5 `ProRegTx` that
created the spent shared collateral output, provided every other
shared-collateral consensus rule holds. Block validation MUST track shared
collateral outputs created earlier in the same block (see [Collateral
Spend Enforcement](#collateral-spend-enforcement)).

#### State effect

Connecting a valid `ProDisTx` removes the masternode identified by
`proTxHash` from the deterministic masternode list as a normal
collateral-spend removal would. The masternode entry MUST NOT be removed
until the corresponding `ProDisTx` has been validated.

### Collateral Spend Enforcement

Provider transaction `CheckSpecialTx` validation alone is insufficient to
enforce the shared collateral covenant: an ordinary transaction that
spends the shared collateral outpoint of an active v5 masternode but
bears no special-transaction payload would, under DIP-0003 rules, simply
remove the masternode by collateral spend. Implementations MUST also
enforce the rules below outside `CheckSpecialTx`.

The protected set is the set of **active-or-pending shared collateral
outpoints**, defined as:

* every collateral outpoint recorded against an active v5 masternode in
  the deterministic masternode list at the parent of the block (or
  mempool tip) being validated, plus
* every collateral outpoint created by a valid v5 ProRegTx that has been
  processed earlier in the block currently being validated and that has
  not itself already been dissolved earlier in that block.

A UTXO whose `scriptPubKey` equals `SHARED_COLLATERAL_SCRIPT` but whose
outpoint is not in this set is NOT protected by the covenant and is not
the subject of the rules below. The rules deliberately key off the
recorded outpoint identity, not raw script equality, to avoid
retroactively locking pre-existing or unrelated outputs that happen to
match the template.

1. **Mempool acceptance.** Before accepting any transaction into the
   mempool, scan its inputs. For each input that spends an
   active-or-pending shared collateral outpoint, reject the transaction
   unless it is a valid `ProDisTx` for the masternode whose collateral
   outpoint matches the spent outpoint in the current deterministic
   masternode list.
2. **Block connection, prior blocks.** Before applying the deterministic
   masternode list update for a connected block, scan every non-coinbase
   transaction in that block for inputs that spend collateral outpoints
   of active v5 masternodes in the parent-state deterministic masternode
   list. Reject the block unless every such input is the input of a
   valid `ProDisTx` for the matching masternode.
3. **Block connection, same-block.** Maintain a per-block index of
   shared collateral outputs created by valid v5 ProRegTx earlier in the
   same block, keyed by `(txid, vout, proTxHash)`. For every later
   non-coinbase transaction in the same block, reject inputs that spend
   such outputs unless the spending transaction is a valid `ProDisTx`
   referencing the corresponding `proTxHash`. The matching `ProDisTx`
   MUST appear strictly after the registration in the block ordering.
4. **Deterministic masternode list removal.** Replace the
   "remove on collateral spend" rule from DIP-0003 with the following for
   v5 masternodes: a v5 masternode MUST NOT be removed until a
   corresponding valid `ProDisTx` for that `proTxHash` has been processed.
   Any spend of the masternode's shared collateral outpoint by a
   transaction that is not such a `ProDisTx` is invalid.
5. **Block disconnection and reorg.** Disconnecting a block that
   contained a `ProDisTx` MUST restore the v5 masternode entry to the
   exact pre-dissolution deterministic masternode state. State diffs MUST
   capture the full share vector and shared-registration parameters as a
   single replacement (see [Deterministic Masternode
   State](#deterministic-masternode-state)) so that reorg replay is
   deterministic.

These rules are evaluated before script-level evaluation for inputs that
spend active-or-pending shared collateral outpoints: even if the
underlying redeem script is trivially satisfiable, consensus rejects the
spend of a protected outpoint unless the spend is a valid `ProDisTx` for
the corresponding masternode. Spends of unprotected UTXOs that happen to
bear the same script are not subjected to the covenant and are
script-evaluated normally.

### Deterministic Masternode State

The deterministic masternode state defined in DIP-0003 is extended for
v5 masternodes. The pre-v5 state serialization is unchanged.

For `nVersion >= ProTxVersion::SharedCollateral`, the masternode state
includes the following additional fields:

| Field | Type | Notes |
| --- | --- | --- |
| `shares` | `CollateralShare[]` (without `joinSig`) | 2 to 8 entries; order preserved from registration. |
| `earlyPeriodBlocks` | `uint32_t` | Frozen at registration. |
| `earlyPenalty` | `CAmount` | Frozen at registration. |
| `standardPenalty` | `CAmount` | Frozen at registration. |

The fields `scriptPayout` and `payouts` are unused for v5 masternodes and
MUST be serialized as empty. `keyIDOwner` is unused for v5 masternodes.

State diffs MUST be version-gated:

1. The state-diff bitfield gains a new field bit for the share vector.
   Any change to any `shares[i].rewardScript` produces a state diff in
   which the share vector is fully replaced. Implementations MAY also
   choose to encode per-share rewardScript diffs more compactly; if so,
   the encoding MUST be invertible byte-for-byte from the resulting
   state.
2. State diffs for v5 fields are present only when the masternode is
   v5; for pre-v5 masternodes the corresponding bits MUST NOT appear.
3. Snapshot serialization MUST round-trip across restart, reorg replay,
   and historic block validation without loss.

Unique-property indexes MUST be extended:

* Every `shares[i].ownerKey` participates in the owner-key uniqueness
  index used to reject reuse across masternodes at registration time.
  Pre-v5 masternodes contribute their single `keyIDOwner`; v5
  masternodes contribute every participant owner key.
* The voting-key uniqueness index applies to `keyIDVoting` as in
  DIP-0003.
* The operator-key uniqueness index applies to `pubKeyOperator` as in
  DIP-0003.

A v5 masternode is created only by a valid v5 ProRegTx and removed only
by a valid v5 `ProDisTx` for the same `proTxHash`. Pre-v5 masternodes
continue to be created and removed as in DIP-0003.

### Simplified Masternode List and Filters

`CSimplifiedMNListEntry::CalcHash`, as used by DIP-0004 simplified
masternode list verification, MUST NOT include shares, refund scripts,
reward scripts, penalty parameters, or any other v5-only field. Light
clients receive no SML commitment to shared-collateral metadata.
Diagnostic RPCs and extended JSON output MAY expose v5 fields.

The trust boundary this creates is intentional and mirrors the
direction DIP-0026 took for multi-payout metadata: shared-collateral
fields are not committed to by SML hashes, so light clients cannot
independently verify share amounts, refund scripts, reward scripts,
penalty parameters, or any other v5-only field from an SML proof. A
future extension to DIP-0004 would be required before SPV clients
could verify shared-collateral terms without trusting a serving full
node.

Full nodes are unaffected by this boundary: a full node validates
every v5 field from the deterministic masternode state that it
reconstructs by replaying blocks, exactly as it does for pre-v5
masternode state. The SML/filter limitation applies only to clients
that depend on SML hashes or filter matches as their source of
truth.

Special-transaction and bloom filtering, as defined in DIP-0003 for
`ProRegTx`/`ProUpRegTx` payout scripts, is extended for v5 masternodes:

* For `ProRegTx` v5, the filter MUST match every `shares[i].refundScript`
  and every non-empty `shares[i].rewardScript`, and every
  `shares[i].ownerKey`.
* For `ProUpShareTx`, the filter MUST match `newRewardScript`.
* For `ProUpSharedRegTx`, the filter MUST match `keyIDVoting`.
* For `ProDisTx`, the filter MUST match every output's `scriptPubKey`,
  matching existing transaction-output filter semantics.

These filter extensions exist solely for client-side discovery and
relay; a filter match is NOT a consensus commitment to the matched
data, and any light client that uses a filter match as authoritative
evidence of v5 state still depends on the honesty of the serving full
node.

These filter extensions apply only to v5 special transactions after
activation. Pre-v5 special transactions retain DIP-0003 filtering.

### Governance

Dash governance assigns one vote per masternode, signed by the voting key
recorded in deterministic masternode state. A v5 masternode retains one
`keyIDVoting` and therefore one vote.

`keyIDVoting` MAY be updated only by `ProUpSharedRegTx`, which requires
unanimous participant signatures. There is no protocol-level mechanism
for fractional voting among participants; any coordination of
participant preferences for governance votes is off-chain and outside the
scope of this DIP.

## Deployment and Compatibility

Activation of this DIP is gated by a future deployment defined under
[DIP-0023](dip-0023.md). The exact deployment name and signaling window
are assigned during release engineering. This DIP does not claim a
specific Dash Core release; release engineering MUST verify the live
deployment state of any candidate fork bit before assigning activation.

Before activation:

* Any provider transaction payload with `nVersion == 5` is invalid.
* Special transaction types `10`, `11`, and `12` are invalid.
* Any output with `SHARED_COLLATERAL_SCRIPT` is treated as non-standard
  by relay and mining policy and MUST NOT be created on chain.

After activation:

* v5 ProRegTx, `ProUpShareTx`, `ProUpSharedRegTx`, and `ProDisTx` are
  valid as specified above.
* DIP-0003 single-owner masternodes (v1, v2, v3) and DIP-0026
  multi-payout masternodes (v4) continue to operate unchanged.
* No upgrade path is defined from v1/v2/v3/v4 to v5. A shared masternode
  is established only by a new v5 ProRegTx; a v5 masternode is wound
  down only by a `ProDisTx`.
* No downgrade path is defined from v5 to v1/v2/v3/v4.

A v5 masternode and a non-v5 masternode never appear as the same
`proTxHash`; deterministic masternode state is version-gated as
specified in [Deterministic Masternode
State](#deterministic-masternode-state).

## Rationale

### Strict superset of DIP-0026

DIP-0026 fixes the operational pain of off-chain reward distribution but
does not constrain the registrar owner, the collateral UTXO, or the exit
path. The most common use cases that motivate DIP-0026 (services that
hold the collateral on behalf of multiple beneficiaries) remain
custodial under v4. This DIP addresses that gap by binding share
amounts, refund destinations, and the collateral itself to per-participant
consent.

### Distinct provider payload version

Reusing v4 semantics for shared collateral would either redefine
registrar-controlled fields under a multi-signer rule or carry both a
basis-point payout list and a share table on every v4 payload. Either
choice complicates the v4 deserialization rules and the deterministic
masternode state machine. Allocating a separate v5 keeps each version
self-describing and lets nodes that have implemented v4 reject v5 by
version check until activation.

### Internal collateral only

External shared collateral would require either a multi-signature
collateral UTXO controlled off-chain (defeating the goal of trustless
exit) or a covenant attached to a previously created output (out of
scope for the existing UTXO format). Constraining v5 to internal
collateral makes the covenant model tractable and ensures that every
shared collateral output is created by a v5 ProRegTx whose consent
digest committed every participant.

### Immutable share parameters

Mutable share amounts would let an attacker who compromised one owner
key dilute another participant's stake without unanimous consent.
Mutable refund scripts would create the same risk for the eventual exit
destination. Both are therefore immutable in this DIP. `rewardScript`
is mutable per share because reward destinations are paid in lower-trust
wallet contexts and benefit from key rotation, and because a stale
reward script affects only the participant who owns it.

### Immutable participant owner keys

If the shared collateral output embedded participant owner keys, owner-key
rotation would invalidate the output script or require a relock
transaction that simultaneously spent the old collateral and created a
new one while preserving the masternode entry. Both options were
considered out of scope for the initial DIP. The simpler choice is
specified here: participant owner keys are immutable, and the collateral
output script contains no participant-specific data.

### Operator authority unchanged

Service availability and operator payout are owned by the operator
under DIP-0003. Moving those fields to unanimous participant consent
would prevent legitimate operator-driven operational changes (such as IP
or port reassignment) without coordinating every participant for routine
maintenance, while not actually improving the security model (the
operator can already withhold service unilaterally). Operator authority
is therefore unchanged.

### Explicit registration consent signatures

Funding-input signatures cannot be relied on to authenticate consent to
shared-masternode parameters: Bitcoin/Dash sighash modes
`SIGHASH_NONE`, `SIGHASH_SINGLE`, and `SIGHASH_ANYONECANPAY` each permit
later modification of fields a participant would otherwise believe
they had signed. Each participant therefore signs an explicit
domain-separated consent digest that covers the inputs, outputs, payload
fields, share table, penalty parameters, and version fields.

### Dissolution covenant commits to outputs

A signature over only the payload would leave the actor remainder, fee,
and even the refund destinations open to substitution at relay or mining
time without invalidating the payload signature. The dissolution
authorization digest therefore commits to the full transaction effect:
`nVersion`, `nType`, `nLockTime`, recomputed `inputsHash` and
`outputsHash`, and the payload's `proTxHash`, `mode`, and `actorIndex`.

### Penalty bounds

The strict `<` bounds on `earlyPenalty` and `standardPenalty` ensure
that even the smallest participant retains a positive remainder
*before fees* after a unilateral exit, so that no consent flow can
configure a penalty that exceeds the actor's principal. This bounds the
worst-case griefing cost. The output covenant separately permits the
actor's output to be omitted entirely when the transaction fee consumes
the full pre-fee remainder, so the strict `<` rule on penalties is not
required to keep the post-fee actor remainder positive; that flexibility
exists so that the actor can cover an arbitrary fee at relay time
without violating the covenant.

### Mempool and block enforcement

Spend rejection cannot live only in `CheckSpecialTx`: ordinary
transactions that spend the shared collateral output would not invoke
`CheckSpecialTx` at all, and the DIP-0003 "remove on collateral spend"
rule would otherwise quietly drop the masternode. The required mempool
and block-validation hooks are made explicit.

### Same-block dissolution

Allowing a `ProDisTx` in the same block as the corresponding v5
ProRegTx lets a participant exit immediately after registration, which
is important for use cases where coordination off-chain proved
unsatisfactory. It also matches the existing behavior that a regular
collateral can be spent in the same block as its ProRegTx.

### One vote per masternode

Fractional governance would require either a new vote-aggregation
scheme or a per-share signing tier on every governance message. Both are
out of scope. Preserving the one-masternode-one-vote rule keeps shared
masternodes compatible with the existing governance infrastructure and
defers fractional voting to a future DIP.

## Test Cases

Implementations SHOULD include at minimum the following tests. Test
values for `SHARED_COLLATERAL_SCRIPT` are taken from the activation
chain parameters.

### Helper

1. `DistributeByWeight(10000, [2500, 2500, 2500, 2500])` returns
   `[2500, 2500, 2500, 2500]`.
2. `DistributeByWeight(10001, [2500, 2500, 2500, 2500])` returns
   `[2501, 2500, 2500, 2500]`.
3. `DistributeByWeight(10, [0, 1, 1])` returns `[0, 5, 5]`.
4. `DistributeByWeight(7, [1, 1, 1, 1, 1, 1, 1, 1])` returns one duff
   to each of the first seven indices and zero to the eighth.
5. `DistributeByWeight(3, [0, 1, 0, 1, 0])` returns `[0, 2, 0, 1, 0]`:
   the remainder skips zero-weight indices and is credited only to
   positive-weight indices in ascending order.
6. `DistributeByWeight(5, [0, 0, 0])` is invalid (all-zero weights) and
   MUST be rejected by the helper.

### Registration

1. A v5 ProRegTx with two shares whose amounts sum to 1000 DASH, both
   shares signed correctly, is valid.
2. A v5 ProRegTx with eight shares summing to 1000 DASH, all signed, is
   valid.
3. A v5 ProRegTx with one share is invalid.
4. A v5 ProRegTx with nine shares is invalid.
5. A v5 ProRegTx whose share amounts sum to 999.99 DASH is invalid.
6. A v5 ProRegTx with a duplicate participant owner key is invalid.
7. A v5 ProRegTx with a duplicate refund script is invalid.
8. A v5 ProRegTx with a refund script paying to a participant owner key
   is invalid.
9. A v5 ProRegTx whose collateral output uses a P2PKH script (not
   `SHARED_COLLATERAL_SCRIPT`) is invalid.
10. A v5 ProRegTx whose `vchSig` is non-empty is invalid.
11. A v5 ProRegTx whose `joinSig` for share `i` was produced under a
    different `outputsHash` is invalid.
12. A v5 ProRegTx whose `joinSig` for share `i` was produced under a
    different penalty value is invalid.

### Reward Splitting

1. A v5 masternode with shares `[500, 500]` DASH receives two coinbase
   outputs of equal value to each `rewardScript`.
2. A v5 masternode with shares `[300, 300, 400]` DASH receives three
   coinbase outputs proportional to the shares, with rounding rules per
   `DistributeByWeight`.
3. A v5 masternode with `nOperatorReward = 1000` subtracts the operator
   amount first, then splits the remainder by share.
4. A coinbase missing any expected per-share output is invalid.
5. A coinbase with an extra unexpected output for a v5 masternode is
   invalid.

### Updates

1. A `ProUpShareTx` signed by `shares[0].ownerKey` updating
   `shares[0].rewardScript` is valid.
2. A `ProUpShareTx` for share `0` signed by `shares[1].ownerKey` is
   invalid.
3. A `ProUpShareTx` setting `newRewardScript` to a P2PKH paying
   `shares[0].ownerKey` is invalid.
4. A `ProUpSharedRegTx` with `vchSigs.size() == shares.size()`, signed
   by every share in order, updating `nOperatorReward`, is valid.
5. A `ProUpSharedRegTx` missing one signature is invalid.
6. A `ProUpSharedRegTx` with an extra signature is invalid.
7. A `ProUpSharedRegTx` whose signatures are presented out of share
   order is invalid.
8. A `ProUpRegTx` (type `3`) targeting a v5 masternode is invalid.

### Dissolution

1. A unilateral `ProDisTx` after the early period, with exactly one
   signature by `shares[a].ownerKey`, penalty equal to
   `standardPenalty`, and exact non-actor outputs, is valid.
2. A unilateral `ProDisTx` during the early period uses `earlyPenalty`
   instead of `standardPenalty`.
3. A unanimous `ProDisTx` with `shares.size()` signatures in share
   order and zero penalty is valid.
4. A unilateral `ProDisTx` whose actor output exceeds
   `shares[a].amount - penalty` is invalid.
5. A unilateral `ProDisTx` whose non-actor output `i` pays more or less
   than `shares[i].amount + bonus[i]` is invalid.
6. A unilateral `ProDisTx` whose non-actor output `i` pays to a script
   other than `shares[i].refundScript` is invalid.
7. A unilateral `ProDisTx` with an extra output (e.g. OP_RETURN) is
   invalid.
8. A unilateral `ProDisTx` with an extra input is invalid.
9. A unilateral `ProDisTx` whose actor output is omitted because the
   fee equals `shares[a].amount - penalty` (so the actor remainder
   after the fee is zero) is valid, and its output list is the
   non-actor outputs in share order with no actor slot.
10. A unilateral `ProDisTx` that includes an actor output paying zero
    duffs to `shares[a].refundScript` is invalid; the actor output MUST
    be omitted rather than paid as a zero-value output.
11. A unanimous `ProDisTx` whose actor output is omitted (because the
    fee equals `shares[a].amount`) is valid; every non-actor output
    pays exactly `shares[i].amount`.
12. A unanimous `ProDisTx` missing one participant signature is
    invalid.
13. A `ProDisTx` whose `outputsHash` does not match the recomputed
    transaction `outputsHash` is invalid; the signature check is not
    reached.
14. A normal transaction (`nVersion < 3` or `nType == 0`) that spends
    the collateral outpoint of an active v5 masternode is rejected by
    mempool and by block validation.
15. A non-dissolution special transaction whose input spends the
    collateral outpoint of an active v5 masternode is rejected.
16. A v5 ProRegTx and a unilateral `ProDisTx` for the same `proTxHash`
    in the same block, in that order, are accepted; the masternode is
    created and then removed within the block.
17. An ordinary transaction whose input spends an unrelated UTXO that
    happens to pay `SHARED_COLLATERAL_SCRIPT` but is NOT a recorded
    active or same-block-pending shared collateral outpoint is NOT
    rejected by the covenant; whether it succeeds depends only on
    ordinary script evaluation of the underlying redeem script.

### Reorg

1. Disconnecting a block containing a v5 ProRegTx fully removes the
   masternode entry, including all share state.
2. Disconnecting a block containing a `ProUpShareTx` restores the
   prior `rewardScript`.
3. Disconnecting a block containing a `ProDisTx` restores the v5
   masternode entry with its full share state.
4. A reorg that replays a `ProUpSharedRegTx` followed by a `ProDisTx`
   in a different order produces the same final state.

## Implementation Notes

These notes are non-normative guidance for implementers.

* The deterministic masternode state-diff bitfield must reserve a new bit
  for the share vector. Implementations should follow the bit-allocation
  conventions already used in `CDeterministicMNStateDiff` so that
  pre-v5 snapshots continue to round-trip exactly.
* `CheckSpecialTx` should treat shared collateral outputs as a separate
  spend-rejection rule rather than as part of the per-payload checks; the
  spend rejection applies to *every* transaction in mempool and block
  contexts, not only to special transactions.
* The same-block index of new shared collateral outputs introduced by
  v5 ProRegTx earlier in the block can be implemented as a
  small `std::unordered_map<COutPoint, uint256 /* proTxHash */>` that is
  populated as block transactions are validated and consulted by every
  later transaction's input scan.
* Coinbase construction should reuse the existing payout-pipeline hook
  added by DIP-0026 (PR 184) and append one output per v5 share with the
  computed amount and target script.
* The wallet RPC surface should expose: a coordinator-style flow that
  builds an unsigned v5 ProRegTx given each participant's funding inputs
  and share parameters; per-participant `joinSig` production over the
  consent digest; and a `dissolvemasternode` RPC that builds a
  `ProDisTx`, computes the signed dissolution digest, and either
  collects the actor's signature (unilateral) or the full set of
  signatures (unanimous).
* RPC output for `protx info` for a v5 masternode should expose the
  share table, penalty parameters, and the canonical
  `SHARED_COLLATERAL_SCRIPT` template for diagnostic purposes.

## Security Considerations

### Trust model

A v5 masternode requires only that consensus is honest. No participant
trusts any other participant with custody of funds. Specifically:

* Collateral cannot be spent except through a valid `ProDisTx` for the
  matching masternode, enforced by mempool and block consensus.
* Unilateral dissolution lets any participant exit at any time without
  the cooperation of any other participant; the only cost is the
  configured penalty (paid to non-actors) and the transaction fee
  (paid from the actor's share).
* Per-share reward outputs go to a participant-controlled script set at
  registration; the registrar cannot rewrite them.
* Unanimous registrar updates require every participant's signature.

### Operator compromise

The operator can withhold service, but cannot redirect owner rewards,
spend collateral, or alter share state. Service withholding can cause a
PoSe ban and thereby reduce or stop owner reward accrual. Participants
can mitigate this by replacing the operator via `ProUpSharedRegTx`
(unanimous) or by dissolving the masternode. Participants choosing an
operator SHOULD treat operator selection as a unanimous-consent decision.

### One-participant compromise

If one participant's owner key is compromised, the attacker can:

* update that participant's `rewardScript` to a controlled script,
  redirecting only that participant's future rewards;
* sign a unilateral `ProDisTx`, exiting the masternode with the
  compromised participant's share minus the penalty (paid to the
  non-actor honest participants) and the transaction fee.

The attacker cannot:

* redirect any other participant's reward share;
* redirect any participant's refund on exit (refund scripts are
  immutable);
* spend collateral except through a valid dissolution;
* alter `keyIDVoting`, `pubKeyOperator`, or `nOperatorReward` (those
  require unanimous consent).

This bounds the loss from a single compromised key to that
participant's share plus the loss of future reward share for the
remaining lifetime of the masternode.

### Penalty griefing

If `earlyPenalty` and `standardPenalty` are both zero, a malicious
participant can dissolve the masternode immediately after registration
at no cost to themselves, returning the other participants' principal
but disrupting service. Participants SHOULD agree on non-trivial penalty
values that compensate for the cost of redeploying the masternode.
Wallets SHOULD warn when a participant signs a registration with
zero-valued penalties.

### Future-block reward dust

The block reward declines over time. A future block reward could be
small enough that the per-share coinbase output for the smallest share
falls below policy dust thresholds. `SHARED_MIN_SHARE_DUFFS` is chosen
to keep per-share rewards safely above current dust thresholds at
present reward levels; it does not guarantee that the smallest share's
reward output will remain non-dust forever. Wallets SHOULD warn during
registration when any share is small enough that anticipated future
rewards could approach dust.

### Light client guarantees

Shared-collateral metadata is not committed to by SML hashes. SPV
clients cannot verify v5 share state without a full node or a
DIP-0004-extending future proof. SPV-level features that depend on
v5 metadata (filter matching of refund and reward scripts, for example)
require the full node serving the client to be honest about v5 state.

### Replay across chains

Both the registration consent digest and the dissolution authorization
digest commit explicitly to `chainGenesisHash`, the genesis block hash of
the network on which the transaction is being authorized. The consensus
digest domain therefore includes the network's genesis hash, and any
v5 registration or dissolution signed against one network's
`chainGenesisHash` cannot be replayed onto any other network or fork
because the digest verified by consensus on the target network would
differ. Signers MUST verify the network / chain context (which genesis
hash they are signing against) before producing any `joinSig` or
dissolution signature; relying on `proTxHash` alone is insufficient for
cross-chain replay protection because `proTxHash` collisions across
networks or short-lived forks cannot be ruled out by digest construction
alone and would otherwise expose unsuspecting signers to replay.

## Open Issues

The following implementation details are deferred and MUST be resolved
before activation:

1. **Exact bytes of `SHARED_COLLATERAL_SCRIPT`.** The recommended
   template family is a P2SH of an `OP_TRUE`-equivalent redeem
   script. The chosen bytes are encoded into chain parameters and
   included in this DIP before mainnet activation.
2. **Final numeric values of `TRANSACTION_PROVIDER_DISSOLVE`,
   `TRANSACTION_PROVIDER_UPDATE_SHARE`, and
   `TRANSACTION_PROVIDER_UPDATE_SHARED_REGISTRAR`.** Tentatively
   `10`, `11`, `12`. Before merge and before activation, these
   assignments MUST be re-checked against the authoritative DIP-0002
   registry and the live Dash Core `TRANSACTION_*` enum (as defined in
   `primitives/transaction.h` on the targeted release branch). If any
   value is already taken or pending allocation, this DIP MUST be
   renumbered to the next free values rather than be merged with a
   conflicting allocation. This DIP intentionally does not embed an
   authoritative table of unrelated allocations.
3. **Final value of `SHARED_MIN_SHARE_DUFFS`.** Tentatively
   `1000000000` (10 DASH). Subject to dust-policy review and
   feedback from wallet implementers.
4. **Activation deployment name.** Subject to release engineering
   confirmation that no candidate fork bit has already been consumed.

The following protocol-level extensions are out of scope for this DIP
and may be addressed by future DIPs:

1. External shared collateral.
2. Participant replacement without full dissolution and re-registration.
3. Owner-key rotation, with or without a relock transaction.
4. Refund-script mutation under unanimous consent.
5. Extra fee inputs and explicit change outputs in `ProDisTx`.
6. Protocol-level fractional governance among participants.

## Copyright

Copyright (c) 2026 Dash Core Group, Inc. [Licensed under the MIT License](https://opensource.org/licenses/MIT).
