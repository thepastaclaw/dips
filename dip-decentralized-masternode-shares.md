<pre>
  DIP: TBD
  Title: Decentralized Masternode Shares
  Author(s): Pasta
  Comments-Summary: No comments yet.
  Status: Draft
  Type: Standard
  Created: 2026-06-20
  Requires: DIP-0002, DIP-0003, DIP-0023, DIP-0026, DIP-0028
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
    3. [Shared-Collateral Mode Discriminator](#shared-collateral-mode-discriminator)
    4. [Shared Collateral Output](#shared-collateral-output)
    5. [Collateral Share](#collateral-share)
    6. [Shared Registration (ProRegTx v4, shared mode)](#shared-registration-proregtx-v4-shared-mode)
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
defining a **shared-collateral mode** within the existing DIP-0026 v4 provider
transaction payload, selected by an explicit `isSharedCollateral` discriminator
flag. In shared-collateral mode, between 2 and 8 mutually distrusting
participants jointly fund, operate, and exit a single Regular or Evolution
masternode under protocol-enforced consent. The shared masternode is funded by
an internal collateral output that is locked by consensus to a dissolution
covenant. Owner rewards are split natively in the coinbase by recorded share
amounts. Each participant may unilaterally dissolve the masternode at any time,
subject to a participant-chosen penalty and the standard transaction fee, and
the full set of participants may dissolve unanimously with no penalty. Updates
to fields that affect all participants require consent from every participant;
the operator role remains unchanged.

Shared collateral is a strict superset of the [DIP-0026: Multi-Party
Payouts](dip-0026.md) reward-splitting mechanism. DIP-0026 v4 with
`isSharedCollateral == false` leaves the registrar owner in unilateral control
of the payout list and is unchanged by this DIP. This DIP introduces the
shared-collateral discriminator into the same v4 payload version so that, when
the discriminator is true, share amounts, refund destinations, and the
collateral itself are bound to per-participant consent and cannot be redirected
by any single owner, operator, miner, or compromised update path. No new
ProRegTx provider payload version is reserved.

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
Under DIP-0026 v4 with `isSharedCollateral == false`, the registrar owner
remains the sole signer for ProRegTx and ProUpRegTx, and may freely rewrite
the payout list at any time.

This DIP does not modify DIP-0026 v4 non-shared semantics. Shared masternodes
reuse the same v4 provider transaction payload version with
`isSharedCollateral == true`, with the following distinctions:

| Property | v4, `isSharedCollateral == false` (DIP-0026) | v4, `isSharedCollateral == true` (this DIP) |
| --- | --- | --- |
| Ownership model | Single registrar owner | 2 to 8 participants |
| Payout list authority | Registrar owner via ProUpRegTx | Per-share self-update; share amounts immutable |
| Collateral lock | Single owner's UTXO | Shared collateral covenant |
| Exit path | Owner spends collateral | Valid `ProDisTx` only |
| Voting / operator updates | Owner | Unanimous participant consent |
| Refund on exit | None enforced | Per-participant refund script enforced |

The two modes share `nVersion == 4` but are not interchangeable on the wire:
the discriminator selects mutually exclusive variant fields (see
[Shared-Collateral Mode Discriminator](#shared-collateral-mode-discriminator)),
so a payload serialized with `isSharedCollateral == true` MUST NOT be
reinterpreted as a non-shared v4 payload, and a non-shared v4 payload MUST NOT
be reinterpreted as shared. A single-owner masternode using v3 or non-shared
v4 cannot be converted to shared in place: shared collateral is established
only by a new v4 registration with `isSharedCollateral == true`, and a shared
masternode is wound down only by a valid `ProDisTx`. There is no in-place
conversion from a shared v4 masternode back to a non-shared v3 or v4
masternode.

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
| Early period | The first `earlyPeriodBlocks` eligible dissolution blocks after registration during which `earlyPenalty` applies to unilateral dissolution. Same-block dissolution is not eligible; if `earlyPeriodBlocks == 0`, no block is early. |

### Parameters

| Constant | Value | Description |
| --- | ---: | --- |
| `SHARED_MIN_PARTICIPANTS` | 2 | Minimum number of shares. |
| `SHARED_MAX_PARTICIPANTS` | 8 | Maximum number of shares. The limit is deliberately stricter than current legacy payout-list limits and is aligned with the pending DIP-0026 v4 design goal of bounding coinbase fan-out. |
| `SHARED_MIN_SHARE_DUFFS` | 1000000000 (10 DASH) | Minimum value of any `shares[i].amount`. |
| `SHARED_MAX_EARLY_PERIOD_BLOCKS` | 400305 (~2 years at 2.626 min/block) | Maximum value of `earlyPeriodBlocks`. |

`SHARED_MAX_EARLY_PERIOD_BLOCKS` is given in blocks using Dash's measured
long-run block interval estimate of 2.626 minutes, yielding approximately
`365 * 24 * 60 / 2.626 * 2 = 400305` blocks for a two-year ceiling. Realized
block times vary, so the wall-clock equivalent drifts with hashrate.
Implementations and tooling that prefer a wall-clock interface SHOULD convert
the user-facing value to blocks at the 2.626 min/block rate and reject any input
that exceeds `SHARED_MAX_EARLY_PERIOD_BLOCKS` once converted.

`SHARED_MIN_SHARE_DUFFS` exists to keep recurring per-share coinbase outputs
above policy dust thresholds at present block rewards while permitting modest
participation levels. Wallets SHOULD warn participants that the block reward
declines over time and very small shares may produce dust-class outputs in
the future.

Shared collateral amounts MUST satisfy:

```text
for every i:
    MoneyRange(shares[i].amount)
    SHARED_MIN_SHARE_DUFFS <= shares[i].amount <= GetMnType(nType).collat_amount
sum_overflow_safe(shares[i].amount) == GetMnType(nType).collat_amount
```

where `GetMnType(nType).collat_amount` is 1000 DASH for `Regular` or 4000 DASH
for `Evo`, as defined in DIP-0003 Appendix B and DIP-0028. The sum MUST be
computed with exact overflow-safe arithmetic; any overflow, non-`MoneyRange`
share amount, share amount above the required collateral amount, or non-exact
sum is invalid.

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

### Shared-Collateral Mode Discriminator

This DIP does NOT introduce a new ProRegTx provider payload version.
Shared-collateral registration reuses the existing DIP-0026 v4 ProRegTx
(`nVersion == 4`) and selects shared-collateral semantics by an explicit
discriminator field, `isSharedCollateral`. This DIP introduces three new
special transaction types, each with its own independent payload version,
to update and dissolve shared masternodes.

The discriminator is a single `uint8_t` field in the v4 ProRegTx payload,
serialized immediately after `nMode`. This wire position is a cross-DIP
activation dependency: DIP-0026 v4 MUST reserve the same discriminator byte for
all v4 ProRegTx payloads before this DIP can activate (see
[Open Issues](#open-issues)).

| Name | Type | Encoding |
| --- | --- | --- |
| `isSharedCollateral` | `uint8_t` | `0x00` for non-shared (DIP-0026 multi-payout) mode; `0x01` for shared-collateral mode. Any other value is invalid. |

The discriminator selects mutually exclusive variant fields in the same v4
payload:

* `isSharedCollateral == false`: after the shared discriminator byte reserved
  by DIP-0026 v4, the remaining v4 payload follows the unchanged DIP-0026
  multi-payout layout (single `keyIDOwner`, `scriptPayout` / `payouts`, no
  share table, no shared collateral output, no penalty parameters, no
  `earlyPeriodBlocks`). All DIP-0026 v4 semantics, validation, state, and
  authorization apply verbatim and are unaffected by this DIP.
* `isSharedCollateral == true`: the v4 payload follows the shared-collateral
  layout defined in [Shared Registration (ProRegTx v4, shared
  mode)](#shared-registration-proregtx-v4-shared-mode) (no `keyIDOwner`, no
  `scriptPayout`, no `payouts`, a `CollateralShare[]` table, an internal
  collateral output that pays `SHARED_COLLATERAL_SCRIPT`, and per-masternode
  penalty parameters). The shared-collateral variant is selected solely by
  the discriminator; consensus MUST NOT infer the variant from any other
  field length, script shape, or output count.

The discriminator is the mode tag carried in the deterministic masternode
state: a masternode created by a v4 ProRegTx with `isSharedCollateral == true`
carries `state.isSharedCollateral == true` for the rest of its lifetime, and
all shared-collateral state fields and authorization rules in this DIP gate on
that flag. A masternode created by a v4 ProRegTx with
`isSharedCollateral == false` carries `state.isSharedCollateral == false` and
is governed by DIP-0026.

The discriminator does NOT apply as the payload version of the new special
transaction types introduced below; those payloads carry their own,
independent `nVersion` field that starts at `1` (see each payload
definition). The independent payload versions of `ProDisTx`,
`ProUpShareTx`, and `ProUpSharedRegTx` evolve separately from the v4
ProRegTx payload version.

| Name | Value | Description |
| --- | ---: | --- |
| `TRANSACTION_PROVIDER_DISSOLVE` | 10 | `ProDisTx` payload (this DIP). |
| `TRANSACTION_PROVIDER_UPDATE_SHARE` | 11 | `ProUpShareTx` payload (this DIP). |
| `TRANSACTION_PROVIDER_UPDATE_SHARED_REGISTRAR` | 12 | `ProUpSharedRegTx` payload (this DIP). |

Values `10`, `11`, and `12` are assigned by this DIP and are also registered in
[DIP-0002's special transaction type table](dip-0002/special-transactions.md).
If another accepted DIP consumes any of these values before this DIP is merged,
this DIP MUST be updated to use the next free values before merge; it MUST NOT
ship with conflicting type assignments.

ProRegTx (type `1`) is reused at payload version 4 with
`isSharedCollateral == true` for shared registration. The base `ProUpRegTx`
(type `3`) is invalid against a shared-collateral masternode
(`state.isSharedCollateral == true`) and is unaffected for non-shared
masternodes. The base `ProUpServTx` (type `2`) continues to operate under
DIP-0003 / DIP-0028 rules for non-shared masternodes AND remains valid against a shared-collateral masternode for the
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
   of a v4 shared-collateral ProRegTx is the shared collateral output; it is
   NOT, on its own, the consensus identifier of a protected collateral.
2. It contains no participant-specific data. In particular, it does not embed
   any participant owner key, refund script, reward script, share amount, or
   `proTxHash`. The per-masternode binding between an outpoint and a
   `proTxHash` is provided by the deterministic masternode state, not by the
   script itself.
3. It is recognizable by full nodes using a single template comparison, and
   it is recognizable as non-standard by older nodes that do not understand
   shared collateral.
4. After activation, consensus and mempool policy protect the following shared collateral outpoint sets:
   * (a) **Registered shared collateral outpoints** — outpoints recorded
     as the collateral of a registered shared-collateral masternode
     (`state.isSharedCollateral == true`) in the deterministic
     masternode list at the parent state of the block (or mempool tip)
     being validated, including PoSe-banned entries that remain registered.
     A spend of an outpoint in this class is rejected unless the spending
     transaction is a valid `ProDisTx` for the corresponding masternode.
   * (b) **Same-block pending shared collateral outputs** — shared
     collateral outputs created by a valid v4 ProRegTx with
     `isSharedCollateral == true` earlier in the block currently being
     validated. A spend of an output in this class later in the same
     block is rejected unconditionally: ordinary spends are rejected
     because the outpoint represents a freshly created masternode
     commitment, and `ProDisTx` is rejected because the registration is
     not yet part of the parent deterministic state a dissolution would
     consult. Dissolution of such a masternode is permitted only in a
     strictly later block, after the registration has been recorded in
     the deterministic masternode list.
   * (c) **Mempool pending shared collateral outputs** — shared
     collateral outputs created by an unconfirmed valid v4 ProRegTx with
     `isSharedCollateral == true` in the mempool. A mempool spend of an
     output in this class is rejected unconditionally until the
     registration confirms.
   Rejection is enforced at mempool acceptance, at block connection, and
   during deterministic masternode list processing (see [Collateral
   Spend Enforcement](#collateral-spend-enforcement)).
5. Ordinary UTXOs that happen to pay `SHARED_COLLATERAL_SCRIPT` but are
   NOT in any protected class above are NOT bound by the dissolution
   covenant. They behave as ordinary outputs at script-evaluation time,
   subject to whatever spending conditions the recommended template
   imposes (for example, the `OP_TRUE` redeem-script template makes them
   anyone-can-spend). Implementations MUST NOT retroactively lock arbitrary
   pre-existing outputs or unrelated outputs created outside the
   shared-collateral registration flow.
6. Before activation, upgraded relays and miners MUST treat any output with
   this script as non-standard and SHOULD NOT mine it, so ordinary relay and
   mining policy discourages creating such outputs before activation. After
   activation, the only way for a transaction to register an outpoint into the
   protected set is a valid v4 ProRegTx with `isSharedCollateral == true`;
   relay and mining policy MUST continue to treat any other output bearing
   `SHARED_COLLATERAL_SCRIPT` as non-standard to discourage accidental
   creation of stranded anyone-can-spend outputs.

The shared collateral script is the single-byte script `OP_TRUE`:

```text
SHARED_COLLATERAL_SCRIPT = 0x51
```

This bare anyone-can-spend script is deliberately non-standard except when it
appears as the collateral output of a valid v4 ProRegTx with
`isSharedCollateral == true` after activation.
Script-level satisfaction is trivial, but consensus rules above script
evaluation reject every protected spend except a valid `ProDisTx`. Embedding
per-masternode commitments (such as `proTxHash` or participant keys) into the
output script was considered and rejected because it would either conflict with
future relock-style updates of participant keys or require a separate commitment
scheme that would still depend on deterministic masternode state to verify.

### Collateral Share

```text
CollateralShare {
    amount       : CAmount  (8 bytes, little-endian, in duffs)
    refundScript : CScript  (compact-size length-prefixed)
    rewardScript : CScript  (compact-size length-prefixed)
    ownerKey     : CKeyID   (20 bytes)
    joinSig      : vector<unsigned char>  (compact-size length-prefixed, 65 bytes when present; CompactSize(0) in join-signature preimage)
}
```

| Field | Mutability | Authorization to change |
| --- | --- | --- |
| `amount` | Immutable | Set at registration; cannot be changed. |
| `refundScript` | Immutable | Set at registration; cannot be changed. |
| `rewardScript` | Mutable per-share | Participant owner key of that share, via `ProUpShareTx`. |
| `ownerKey` | Immutable | Set at registration; cannot be rotated in this DIP. |
| `joinSig` | Set only in `ProRegTx` payload | Validated at registration; not stored in deterministic state. In the registration join-signature digest, this field is serialized byte-exactly as an empty vector (`CompactSize(0)`), not omitted. |

Field rules at registration:

1. `amount` MUST satisfy `MoneyRange(amount)` and MUST be between
   `SHARED_MIN_SHARE_DUFFS` and `GetMnType(nType).collat_amount`, inclusive.
2. `refundScript` MUST be a standard `P2PKH` or `P2SH` script.
3. `rewardScript` MUST be empty or a standard `P2PKH` or `P2SH` script. An
   empty `rewardScript` is interpreted at the consensus layer as
   `refundScript`.
4. `ownerKey` MUST be distinct from every other `ownerKey` in the share
   table and from `keyIDVoting`.
5. `ownerKey` MUST be distinct from every active owner key recorded for any
   other registered masternode at the registration height. The
   unique-property index defined in DIP-0003 is extended to track every
   `shares[i].ownerKey` for shared-collateral masternodes (see
   [Deterministic Masternode State](#deterministic-masternode-state)).
6. `refundScript` MUST NOT be duplicated within the share table.
7. Effective reward scripts MUST NOT be duplicated within the share table. For
   this rule, an empty `rewardScript` is interpreted as `refundScript` before
   duplicate detection.
8. No `refundScript` and no `rewardScript` may be a P2PKH script paying to
   `keyIDVoting` or to a key ID that equals any `shares[i].ownerKey`. There
   is no `keyIDOwner` field in shared-collateral mode — see [Shared
   Registration (ProRegTx v4, shared
   mode)](#shared-registration-proregtx-v4-shared-mode). The intent matches
   the DIP-0026 key-reuse rule: these scripts are spent in lower-trust
   wallet contexts and MUST NOT reuse keys that control masternode state.
9. `joinSig` is a 65-byte compact ECDSA signature by `ownerKey` over the
   registration consent digest defined in [Registration Consent
   Digest](#registration-consent-digest). Any other length or any
   signature that does not verify under `ownerKey` is invalid.

### Shared Registration (ProRegTx v4, shared mode)

A v4 ProRegTx with `isSharedCollateral == true` uses the field layout below.
Field positions for the shared-collateral variant are specified normatively
here; this list is exhaustive and defines the v4 ProRegTx layout for
`isSharedCollateral == true`. The v4 layout for `isSharedCollateral == false`
is unchanged from DIP-0026 and is NOT defined by this DIP.

| Field | Type | Notes |
| --- | --- | --- |
| `nVersion` | `uint16_t` | MUST be `4`. |
| `nType` | `MnType` (`uint16_t`) | `Regular` or `Evo`. |
| `nMode` | `uint16_t` | MUST be `0`. |
| `isSharedCollateral` | `uint8_t` | MUST be `0x01` to select the shared-collateral variant defined in this section. `0x00` selects the unchanged DIP-0026 multi-payout variant and is not governed by this DIP. Any other value is invalid. |
| `collateralOutpoint.hash` | `uint256` | MUST be the null hash. |
| `collateralOutpoint.n` | `uint32_t` | Index of the shared collateral output within this transaction. |
| `netInfo` | DIP-0003/0028 net info | As for non-shared v4. |
| `pubKeyOperator` | BLS public key (basic scheme) | As for non-shared v4. |
| `keyIDVoting` | `CKeyID` | As for non-shared v4. |
| `nOperatorReward` | `uint16_t` | Basis points; MUST be from 0 to 10000. |
| `shares` | `CollateralShare[]` | CompactSize-prefixed vector of 2 to 8 entries; per [Collateral Share](#collateral-share). |
| `earlyPeriodBlocks` | `uint32_t` | `0` to `SHARED_MAX_EARLY_PERIOD_BLOCKS`. |
| `earlyPenalty` | `CAmount` (8 bytes) | Duffs. |
| `standardPenalty` | `CAmount` (8 bytes) | Duffs. |
| `inputsHash` | `uint256` | `CalcTxInputsHash(tx)`. |
| `platformNodeID`, `platformNetInfo` (Evo only) | as DIP-0028 | Serialized after `inputsHash` and before `vchSig`, preserving the DIP-0028 insertion point. |
| `vchSig` | `vector<unsigned char>` | MUST be empty in shared-collateral mode (no external collateral signature). |

There is no `keyIDOwner` field in a shared-collateral v4 ProRegTx. The role
of the single owner key in DIP-0003 is performed in shared-collateral mode by
the collection of `shares[*].ownerKey`. The non-shared v4 ProRegTx retains
its `keyIDOwner` field unchanged.

There is no `scriptPayout` or `payouts` field in a shared-collateral v4
ProRegTx. Owner rewards are derived directly from the share table as
specified in [Reward Distribution](#reward-distribution). The non-shared v4
ProRegTx retains its DIP-0026 payout list unchanged.

Unless explicitly replaced by this section, the base ProRegTx validation rules
from DIP-0003, DIP-0026, and DIP-0028 continue to apply to shared-collateral
v4 registrations. This includes network-info validation, Evo Platform-field
validation, operator-key validity, and operator-key uniqueness.

A v4 ProRegTx with `isSharedCollateral == true` is invalid if any of the
following conditions hold:

1. `nVersion != 4`.
2. `isSharedCollateral` is neither `0x00` nor `0x01`. (When
   `isSharedCollateral == 0x00`, the payload is governed by DIP-0026 and the
   remaining rules in this list do not apply.)
3. `nType` is not `Regular` or `Evo`.
4. `nMode != 0`.
5. `netInfo` fails the applicable DIP-0003 network-info validation rules.
6. For Evo masternodes, any Platform field fails the applicable DIP-0028
   validation rules.
7. `pubKeyOperator` is not a valid basic-scheme BLS public key or collides
   with the operator key of any other registered masternode.
8. `collateralOutpoint.hash` is not the null hash.
9. `collateralOutpoint.n` does not point to an output of this transaction.
10. The output at `collateralOutpoint.n` does not pay
    `GetMnType(nType).collat_amount` to `SHARED_COLLATERAL_SCRIPT`.
11. The transaction creates more than one output paying
    `SHARED_COLLATERAL_SCRIPT`.
12. `shares.size()` is less than `SHARED_MIN_PARTICIPANTS` or greater than
    `SHARED_MAX_PARTICIPANTS`.
13. Any share fails its per-field validation rules in [Collateral
    Share](#collateral-share).
14. Any `shares[i].amount` fails `MoneyRange`, is less than
    `SHARED_MIN_SHARE_DUFFS`, or is greater than
    `GetMnType(nType).collat_amount`.
15. The exact overflow-safe sum of all `shares[i].amount` values does not
    equal `GetMnType(nType).collat_amount`, or the summation overflows.
16. `nOperatorReward > 10000`.
17. Penalty parameter constraints in [Parameters](#parameters) are not
    satisfied.
18. `earlyPeriodBlocks > SHARED_MAX_EARLY_PERIOD_BLOCKS`.
19. `vchSig` is non-empty.
20. `inputsHash != CalcTxInputsHash(tx)`.
21. For any share `i`, `shares[i].joinSig` does not verify against
    `shares[i].ownerKey` over the registration consent digest defined
    below.

Rules 19 and 20 are evaluated before any ECDSA signature verification.
Consensus computes `outputsHash` from the transaction outputs when building
the registration consent digest; there is no payload `outputsHash` field in
a shared-collateral v4 ProRegTx.

### Registration Consent Digest

Each participant signs an explicit consent digest that commits to the full
effect of the registration transaction, independent of any
sighash-mode behavior of the funding-input scripts.

```text
SharedRegConsentHash = SHA256d(
    "DashSharedMNReg" ||
    chainGenesisHash ||
    LE16(tx.nVersion)  || LE16(tx.nType)  || LE32(tx.nLockTime) ||
    inputsHash         || sequencesHash   || outputsHash ||
    LE16(payload.nVersion) ||
    LE16(payload.nType) || LE16(payload.nMode) ||
    LE8(payload.isSharedCollateral) ||
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

The digest commits explicitly to `payload.isSharedCollateral` so that a
signature produced under one variant cannot be reinterpreted under the
other.

Where:

* `LE8`, `LE16`, `LE32`, `LE64` denote little-endian encodings of the
  indicated width.
* `SHA256d(x)` is the Dash double-SHA-256 used for transaction hashes.
* `chainGenesisHash` is the 32-byte block hash of the genesis block of the
  network on which the registration is being authorized (mainnet, testnet,
  devnet, regtest, or any future network with a distinct genesis block).
  Consensus uses the genesis hash of the chain that is validating the
  transaction. Each participant signer MUST independently verify the
  network / chain context of the registration before producing `joinSig`;
  a signature produced against one `chainGenesisHash` MUST NOT verify on a
  network with a different genesis hash.
* `inputsHash` is `CalcTxInputsHash(tx)`. Consensus MUST recompute this
  from the transaction and compare it to the `payload.inputsHash` field
  before signature verification; mismatch is invalid.
* `sequencesHash` is the SHA-256d of the concatenation of every input's
  `nSequence` in transaction-input order, each encoded as `LE32(nSequence)`.
  Consensus MUST recompute this from the transaction before signature
  verification. This prevents a coordinator from changing finality, RBF, or
  `nLockTime` behavior after participants produce `joinSig`.
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
  in share order, with each `joinSig` field serialized byte-exactly as an
  empty vector (`CompactSize(0)`) so that signatures are not
  self-referential. The field MUST NOT be omitted from the preimage.

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
masternodes. The base `ProUpRegTx` (type `3`) is invalid for shared-collateral
masternodes (`state.isSharedCollateral == true`), because its single-owner
authorization model is incompatible with the shared-registrar tier defined
here. The base `ProUpServTx` (type `2`) remains valid for shared-collateral
masternodes with operator authorization (see [Authorization
Tiers](#authorization-tiers)).

#### ProUpShareTx (type 11)

A `ProUpShareTx` updates exactly one share's mutable fields. In this DIP
the only mutable per-share field is `rewardScript`.

```text
CProUpShareTx {
    nVersion     : uint16_t     // MUST be 1 (payload version of this new special tx type)
    proTxHash    : uint256
    shareIndex   : uint16_t     // index into shares[]
    newRewardScript : CScript  // compact-size length-prefixed
    inputsHash   : uint256
    sig          : vector<unsigned char>  // 65 bytes
}
```

`nVersion` here is the independent payload version of the new
`TRANSACTION_PROVIDER_UPDATE_SHARE` special transaction; it is NOT the v4
ProRegTx provider payload version. The two version namespaces are separate,
and the `ProUpShareTx` payload version evolves independently of the v4
ProRegTx payload version.

Validation:

1. `nVersion` MUST be `1`.
2. The masternode identified by `proTxHash` MUST exist and MUST satisfy
   `state.isSharedCollateral == true`.
3. `shareIndex` MUST be less than `shares.size()` in the current state.
4. `newRewardScript` MUST be empty or a standard `P2PKH` or `P2SH` script.
   An empty `newRewardScript` is interpreted as
   `shares[shareIndex].refundScript`.
5. `newRewardScript` MUST satisfy the same key-reuse restrictions as
   registration: it MUST NOT be a P2PKH paying to any
   `shares[i].ownerKey` and MUST NOT pay to `keyIDVoting`.
6. After interpreting an empty `newRewardScript` as
   `shares[shareIndex].refundScript`, the resulting effective reward script
   MUST NOT equal any other share's current effective reward script.
7. `inputsHash` MUST equal `CalcTxInputsHash(tx)`.
8. `sig` MUST be a valid ECDSA compact signature by
   `shares[shareIndex].ownerKey` over:

   ```text
   SHA256d("DashSharedMNUpShare" ||
           chainGenesisHash ||
           LE16(tx.nVersion) || LE16(tx.nType) || LE32(tx.nLockTime) ||
           inputsHash || sequencesHash || outputsHash ||
           LE16(payload.nVersion) ||
           proTxHash || LE16(shareIndex) || newRewardScriptSerialized)
   ```

   `chainGenesisHash` is the 32-byte genesis block hash of the network
   validating the transaction. `payload.nVersion` is the ProUpShareTx payload
   version (currently `1`). `sequencesHash` and `outputsHash` use the same
   definitions as in [Registration Consent Digest](#registration-consent-digest)
   and are recomputed from the actual transaction before signature
   verification. `newRewardScriptSerialized` is the standard compact-size
   length-prefixed `CScript` serialization used in the payload.

A valid `ProUpShareTx` replaces `shares[shareIndex].rewardScript` in the
deterministic masternode state. It does not revive a PoSe-banned masternode
and does not affect any other field. Duplicate effective reward scripts are
therefore invalid both at registration and after every per-share update.

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
transaction and is unrelated to the v4 ProRegTx provider payload version.

Validation:

1. `nVersion` MUST be `1`.
2. The masternode identified by `proTxHash` MUST exist and MUST satisfy
   `state.isSharedCollateral == true`.
3. `pubKeyOperator` MUST be a valid basic-scheme BLS public key and MUST
   NOT collide with the operator key of any other registered masternode.
4. `keyIDVoting` MUST be non-null and MUST NOT equal any
   `shares[i].ownerKey`.
5. No current `refundScript` and no current effective `rewardScript` may be a
   P2PKH script paying to `keyIDVoting`. For this check, an empty
   `rewardScript` is interpreted as the corresponding `refundScript`.
6. `nOperatorReward <= 10000`.
7. `inputsHash` MUST equal `CalcTxInputsHash(tx)`.
8. `vchSigs.size()` MUST equal `shares.size()`.
9. For each `i` in `[0, shares.size())`, `vchSigs[i]` MUST be a valid
   ECDSA compact signature by `shares[i].ownerKey` over:

   ```text
   SHA256d("DashSharedMNUpSharedReg" ||
           chainGenesisHash ||
           LE16(tx.nVersion) || LE16(tx.nType) || LE32(tx.nLockTime) ||
           inputsHash || sequencesHash || outputsHash ||
           LE16(payload.nVersion) ||
           proTxHash || pubKeyOperatorSerialized ||
           keyIDVoting || LE16(nOperatorReward))
   ```

   `chainGenesisHash` is the 32-byte genesis block hash of the network
   validating the transaction. `payload.nVersion` is the ProUpSharedRegTx
   payload version (currently `1`). `sequencesHash` and `outputsHash` use the
   same definitions as in [Registration Consent Digest](#registration-consent-digest)
   and are recomputed from the actual transaction before signature
   verification.

10. Signature verification proceeds in share order. Any missing, extra, or
    out-of-order signature is invalid.

A valid `ProUpSharedRegTx` replaces `pubKeyOperator`, `keyIDVoting`, and
`nOperatorReward` in the deterministic masternode state. It does not revive
a PoSe-banned masternode and does not affect share contents or collateral.

If `pubKeyOperator` is unchanged, existing operator-controlled service fields
(`scriptOperatorPayout`, `netInfo`, and Platform identifiers) are unchanged by
this transaction and continue to be updated by the operator using
`ProUpServTx`.

If `pubKeyOperator` changes, all operator-controlled service fields MUST be
reset: `scriptOperatorPayout` becomes empty, `netInfo` becomes the empty
network-info value for the masternode's state version, Platform identifiers
become null/empty, and the masternode is marked PoSe-banned until the new
operator submits a valid `ProUpServTx`. This mirrors the existing
operator-key-reset behavior for non-shared masternodes and prevents stale
operator payout or endpoint state from carrying across an operator change.

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

`ProUpRegTx` (type `3`) is invalid against a shared-collateral masternode
(`state.isSharedCollateral == true`). `ProUpServTx` and `ProUpRevTx` continue
to use operator authorization as in DIP-0003 and DIP-0028. The operator can
stop service or revoke the masternode but cannot redirect owner rewards or
spend the shared collateral.

### Reward Distribution

Owner-reward distribution for a shared-collateral masternode
(`state.isSharedCollateral == true`) reuses the DIP-0026 pipeline but weights
by share amount rather than by basis points.

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
    remainder_index = greatest i where weights[i] > 0
    out[i] = floor(total * weights[i] / sum_w)   for each i != remainder_index
    out[remainder_index] = total - sum(out[i] for i != remainder_index)
    return out
```

The final positive-weight entry receives the rounding remainder, matching the
shape of DIP-0026's final-entry remainder rule while allowing weights to be
collateral amounts rather than basis points. `DistributeByWeight` is invalid
for an all-zero weight vector and callers
MUST NOT invoke it in that case. All intermediate arithmetic in
`DistributeByWeight`, including `total * weights[i]`, `sum_w`, and
`sum(out[i])`, MUST be evaluated exactly without overflow. Implementations
MUST use at least 128-bit integer intermediates or a mathematically equivalent
overflow-safe quotient/remainder algorithm that produces the same floor and
remainder results for every consensus-valid input. For unanimous dissolution,
where the penalty itself is zero, the bonus is taken to be all-zero directly
without calling `DistributeByWeight` (see [Dissolution
(ProDisTx)](#dissolution-prodistx)).

Coinbase validation requires every expected per-share reward output by exact
amount and script for every shared-collateral masternode scheduled in the
block. As in DIP-0026, the relative order of outputs within the coinbase is
not consensus-significant.

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
`TRANSACTION_PROVIDER_DISSOLVE` special transaction and is unrelated to the
v4 ProRegTx provider payload version.

#### Transaction shape

A valid dissolution transaction satisfies all of:

1. `nVersion` MUST be `1`.
2. It has exactly one input, and that input spends the shared collateral
   outpoint of the masternode identified by `proTxHash`. No additional
   funding inputs are permitted in this initial scope.
3. Its outputs are the refund outputs defined below, in share order, with
   no additional outputs of any kind. Change outputs are not permitted.
4. Its `nLockTime` MAY be non-zero. The dissolution authorization digest
   commits to `nLockTime`, so consensus enforces whatever value was signed.
5. `mode` MUST be either `0` (unilateral) or `1` (unanimous). Any other
   value is invalid.

#### Authorization digest

```text
SharedDisHash = SHA256d(
    "DashSharedMNDissolve" ||
    chainGenesisHash ||
    LE16(tx.nVersion) || LE16(tx.nType) || LE32(tx.nLockTime) ||
    LE32(tx.vin[0].nSequence) ||
    inputsHash || outputsHash ||
    LE16(payload.nVersion) ||
    proTxHash ||
    LE8(mode) || LE16(actorIndex)
)
```

Where `inputsHash` is `CalcTxInputsHash(tx)` and `outputsHash` is the
SHA-256d of the concatenated serialized outputs of `tx`, both recomputed
from the actual transaction. The digest also commits to the sequence of
the single dissolution input (`tx.vin[0].nSequence`) so that changing the
sequence cannot disable or alter the signed `nLockTime` semantics.
Consensus MUST recompute both hashes and reject mismatches with the
payload's `inputsHash` and `outputsHash` fields before any signature is
verified. `chainGenesisHash` is the 32-byte genesis block hash of the
network validating the transaction, identical to its use in [Registration
Consent Digest](#registration-consent-digest); a dissolution signed against
one `chainGenesisHash` MUST NOT verify on a network with a different
genesis hash. Each signer MUST independently verify the network / chain
context before producing its `vchSigs` entry. `payload.nVersion` here is
the dissolution payload version (currently `1`), not the v4 ProRegTx
provider payload version.

#### Period and penalty

Let `H` be the height at which the dissolution is connected, and let
`state` be the deterministic masternode state for `proTxHash` at the parent
block of `H`. Define:

```text
early   = (state.earlyPeriodBlocks > 0) &&
          ((H - state.nRegisteredHeight) <= state.earlyPeriodBlocks)
penalty = (mode == 1) ? 0 : (early ? state.earlyPenalty : state.standardPenalty)
```

Because same-block dissolution is invalid, the first eligible dissolution
height is `state.nRegisteredHeight + 1`. Therefore `earlyPeriodBlocks == 1`
means only that first eligible block uses `earlyPenalty`, and
`earlyPeriodBlocks == 0` disables the early penalty window.

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
else if mode == 0:
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
fee = GetMnType(state.nType).collat_amount - sum(outputs.value)
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

A `ProDisTx` MUST NOT appear in the same block as the v4 ProRegTx with
`isSharedCollateral == true` that creates the shared collateral output it
would spend. Dissolution operates against the deterministic masternode state
at the parent block, so the registering masternode is not yet part of that
state until the registering block has been connected. A `ProDisTx` whose
input refers to a shared collateral output created earlier in the same block
is therefore invalid even if every other covenant rule would otherwise hold;
dissolution is permitted only in a strictly later block. Block validation
MUST track shared collateral outputs created earlier in the same block solely
so that any spend of those outputs later in the same block — ordinary or
`ProDisTx` — can be rejected (see [Collateral Spend
Enforcement](#collateral-spend-enforcement)).

#### State effect

Connecting a valid `ProDisTx` removes the masternode identified by
`proTxHash` from the deterministic masternode list as a normal
collateral-spend removal would. The masternode entry MUST NOT be removed
until the corresponding `ProDisTx` has been validated.

### Collateral Spend Enforcement

Provider transaction `CheckSpecialTx` validation alone is insufficient to
enforce the shared collateral covenant: an ordinary transaction that spends
the shared collateral outpoint of a registered shared-collateral masternode
(`state.isSharedCollateral == true`) but bears no special-transaction payload
would, under DIP-0003 rules, simply remove the masternode by collateral
spend. Implementations MUST also enforce the rules below outside
`CheckSpecialTx`.

The protected set has three parts with different rules:

* **Registered set.** Every collateral outpoint recorded against a
  registered shared-collateral masternode
  (`state.isSharedCollateral == true`) in the deterministic masternode list at
  the parent of the block
  (or mempool tip) being validated, including PoSe-banned entries that
  remain registered until removed by a valid `ProDisTx`. A spend of an
  outpoint in the registered set is rejected unless it is a valid
  `ProDisTx` for the matching masternode.
* **Same-block pending set.** Every shared collateral outpoint created
  by a valid v4 ProRegTx with `isSharedCollateral == true` earlier in
  the block currently being validated. A spend of an outpoint in the
  same-block pending set is rejected unconditionally: ordinary spends
  are rejected because the outpoint represents an active masternode
  commitment, and `ProDisTx` spends are rejected because the
  registration is not yet part of the parent deterministic state.
  Dissolution of such a masternode is permitted only in a strictly
  later block.
* **Mempool pending set.** Every shared collateral outpoint created by
  an unconfirmed v4 ProRegTx with `isSharedCollateral == true` accepted
  into the mempool. A mempool transaction that spends an outpoint in
  this set is rejected unconditionally until the registration confirms
  and the outpoint moves into the registered set.

A UTXO whose `scriptPubKey` equals `SHARED_COLLATERAL_SCRIPT` but whose
outpoint is in neither set is NOT protected by the covenant and is not
the subject of the rules below. The rules deliberately key off the
recorded outpoint identity, not raw script equality, to avoid
retroactively locking pre-existing or unrelated outputs that happen to
match the template.

1. **Mempool acceptance.** Before accepting any transaction into the
   mempool, scan its inputs. For each input that spends an outpoint in
   the registered set, reject the transaction unless it is a valid
   `ProDisTx` for the masternode whose collateral outpoint matches the
   spent outpoint in the current deterministic masternode list. For
   each input that spends an outpoint in the mempool pending set, reject
   the transaction unconditionally, including if the transaction is a
   `ProDisTx`. The same-block pending set is computed per block by
   block-connection logic; the mempool pending set prevents miners from
   assembling an invalid parent-plus-child package around an
   unconfirmed shared-collateral registration.
2. **Block connection, prior blocks.** Before applying the deterministic
   masternode list update for a connected block, scan every non-coinbase
   transaction in that block for inputs that spend outpoints in the
   registered set (collateral outpoints of registered shared-collateral
   masternodes, including PoSe-banned entries, in the parent-state
   deterministic masternode list). Reject the block unless every such
   input is the input of a valid `ProDisTx` for the matching masternode.
3. **Block connection, same-block.** Maintain a per-block index of
   shared collateral outputs created by valid v4 ProRegTx with
   `isSharedCollateral == true` earlier in the same block, keyed by
   `(txid, vout, proTxHash)`. For every later non-coinbase transaction
   in the same block, reject any input that spends such an output. Both
   ordinary spends and `ProDisTx` spends are rejected: a `ProDisTx`
   whose input refers to a shared collateral output created in the same
   block is invalid even if every other covenant rule would otherwise
   hold.
4. **Deterministic masternode list removal.** Replace the
   "remove on collateral spend" rule from DIP-0003 with the following
   for shared-collateral masternodes: a shared-collateral masternode
   (`state.isSharedCollateral == true`) MUST NOT be removed until a
   corresponding valid `ProDisTx` for that `proTxHash` has been
   processed in a strictly later block than the block that contained
   the registering v4 ProRegTx. Any spend of the masternode's shared
   collateral outpoint by a transaction that is not such a `ProDisTx`
   is invalid.
5. **Block disconnection and reorg.** Disconnecting a block that
   contained a `ProDisTx` MUST restore the shared-collateral masternode
   entry to the exact pre-dissolution deterministic masternode state.
   State diffs MUST capture the full share vector and
   shared-registration parameters as a single replacement (see
   [Deterministic Masternode
   State](#deterministic-masternode-state)) so that reorg replay is
   deterministic.

These rules are evaluated before script-level evaluation for inputs
that spend outpoints in the registered set or same-block pending set: even
if the underlying redeem script is trivially satisfiable, consensus
rejects the spend of a registered-set outpoint unless the spend is a valid
`ProDisTx` for the corresponding masternode, and rejects every spend of
a same-block pending outpoint unconditionally. Mempool policy likewise
rejects every spend of a mempool-pending outpoint before script-level
standardness or validity can make the child acceptable. Spends of
unprotected UTXOs that happen to bear the same script are not subjected
to the covenant and are script-evaluated normally.

### Deterministic Masternode State

The deterministic masternode state defined in DIP-0003 is extended for
shared-collateral masternodes. The pre-shared and non-shared serialization
is unchanged.

The deterministic masternode state for every masternode includes the
`isSharedCollateral` flag carried from its registering v4 ProRegTx (or
implicitly `false` for masternodes registered under earlier provider payload
versions). When `state.isSharedCollateral == true`, the masternode state
includes the following additional fields:

| Field | Type | Notes |
| --- | --- | --- |
| `shares` | `CollateralShare[]` (without `joinSig`) | 2 to 8 entries; order preserved from registration. |
| `earlyPeriodBlocks` | `uint32_t` | Frozen at registration. |
| `earlyPenalty` | `CAmount` | Frozen at registration. |
| `standardPenalty` | `CAmount` | Frozen at registration. |

The fields `scriptPayout` and `payouts` are unused when
`state.isSharedCollateral == true` and MUST be serialized as empty.
`keyIDOwner` is absent from the shared-collateral v4 ProRegTx and MUST NOT
participate in shared-collateral validation or uniqueness indexes. If an
implementation's deterministic state object contains a legacy `keyIDOwner`
field, that field MUST be serialized as null/zero whenever
`state.isSharedCollateral == true` and MUST be ignored whenever
`state.isSharedCollateral` is true.

State diffs MUST be gated on `state.isSharedCollateral`:

1. The state-diff bitfield gains a new field bit for the share vector
   (exact bit value deferred; see [Open Issues](#open-issues)).
   Any change to any field of any `shares[i]` (including a
   per-share `rewardScript` change from a `ProUpShareTx`) MUST produce
   a state diff in which the entire share vector is fully replaced.
   This full-replacement encoding is the single canonical
   representation of a shared-collateral share-state diff: compact
   diffs, per-share field diffs, and any other alternative encoding of
   share-vector changes are NOT permitted. Implementations MUST emit
   and accept share-vector changes only as a full replacement of the
   share vector.
2. State diffs for shared-collateral fields are present only when
   `state.isSharedCollateral == true`; for non-shared masternodes (both
   pre-v4 and v4 with `isSharedCollateral == false`) the corresponding
   bits MUST NOT appear.
3. Snapshot serialization MUST round-trip across restart, reorg replay,
   and historic block validation without loss.

Unique-property indexes MUST be extended:

* Every `shares[i].ownerKey` participates in the owner-key uniqueness
  index used to reject reuse across masternodes at registration time.
  Non-shared masternodes contribute their single `keyIDOwner`;
  shared-collateral masternodes contribute every participant owner key.
* No network-wide uniqueness index is added for `keyIDVoting`. As in
  DIP-0003, the voting key may be delegated and reused; shared-collateral
  validation only forbids `keyIDVoting` from equaling any participant owner
  key for the same masternode.
* The operator-key uniqueness index applies to `pubKeyOperator` as in
  DIP-0003.

A shared-collateral masternode is created only by a valid v4 ProRegTx with
`isSharedCollateral == true` and removed only by a valid `ProDisTx` for the
same `proTxHash`. Non-shared masternodes continue to be created and removed
as in DIP-0003 and DIP-0026.

### Simplified Masternode List and Filters

`CSimplifiedMNListEntry::CalcHash`, as used by DIP-0004 simplified
masternode list verification, MUST NOT include shares, refund scripts,
reward scripts, penalty parameters, or any other shared-collateral-only
field. Light clients receive no SML commitment to shared-collateral
metadata. Diagnostic RPCs and extended JSON output MAY expose
shared-collateral fields.

The trust boundary this creates is intentional and mirrors the
direction DIP-0026 took for multi-payout metadata: shared-collateral
fields are not committed to by SML hashes, so light clients cannot
independently verify share amounts, refund scripts, reward scripts,
penalty parameters, or any other shared-collateral-only field from an
SML proof. A future extension to DIP-0004 would be required before SPV
clients could verify shared-collateral terms without trusting a serving
full node.

Full nodes are unaffected by this boundary: a full node validates
every shared-collateral field from the deterministic masternode state
that it reconstructs by replaying blocks, exactly as it does for
non-shared masternode state. The SML/filter limitation applies only to
clients that depend on SML hashes or filter matches as their source of
truth.

Special-transaction and bloom filtering, as defined in DIP-0003 for
`ProRegTx`/`ProUpRegTx` payout scripts, is extended for shared-collateral
transactions:

| Transaction | Filter elements |
| --- | --- |
| `ProRegTx` v4 with `isSharedCollateral == true` | Every `shares[i].refundScript`; every non-empty `shares[i].rewardScript`; every `shares[i].ownerKey`; `keyIDVoting`; `pubKeyOperator`; the shared collateral outpoint. |
| `ProUpShareTx` | `proTxHash`; the current `shares[shareIndex].ownerKey`; `newRewardScript` when non-empty. |
| `ProUpSharedRegTx` | `proTxHash`; every current `shares[i].ownerKey`; `keyIDVoting`; `pubKeyOperator`. |
| `ProDisTx` | `proTxHash`; every signing participant owner key required by `mode`; every refund output `scriptPubKey`, matching existing transaction-output filter semantics. |

These filter extensions exist solely for client-side discovery and
relay; a filter match is NOT a consensus commitment to the matched
data, and any light client that uses a filter match as authoritative
evidence of shared-collateral state still depends on the honesty of the
serving full node.

These filter extensions apply only to shared-collateral special transactions
after activation. Non-shared special transactions (including DIP-0026 v4
with `isSharedCollateral == false`) retain their existing DIP-0003 and
DIP-0026 filtering.

### Governance

A shared-collateral masternode (`state.isSharedCollateral == true`) retains
one `keyIDVoting`, signed by the voting key recorded in deterministic
masternode state. This DIP does not introduce fractional participant voting
or per-share governance keys.

Vote weight remains defined by existing masternode-type rules. A shared
`Regular` masternode has the regular masternode vote weight, while a shared
`Evo` masternode retains the DIP-0028 evonode vote weight of four relative to
a regular masternode. The single shared-collateral `keyIDVoting` authorizes
that whole masternode-type vote weight.

`keyIDVoting` MAY be updated only by `ProUpSharedRegTx`, which requires
unanimous participant signatures. There is no protocol-level mechanism for
fractional voting among participants; any coordination of participant
preferences for governance votes is off-chain and outside the scope of this
DIP.

## Deployment and Compatibility

**Draft status.** This DIP is a Draft. The DIP-0023 deployment name and
signaling window are NOT finalized in this revision. The consensus model and
covenant rules below are written to be reviewable as a draft and MUST NOT be
treated as activation-ready until deployment parameters are assigned and this
DIP is updated accordingly (see [Open Issues](#open-issues)).

Activation of this DIP is gated by an intended future deployment
defined under [DIP-0023](dip-0023.md). The exact deployment name and
signaling window are assigned during release engineering. This DIP
does not claim any specific Dash Core release for activation; release
engineering MUST verify the live deployment state of any candidate
fork bit before assigning activation, and the activation target is
subject to DIP editor assignment.

Before activation:

* Any v4 ProRegTx payload with `isSharedCollateral == true` is invalid.
  DIP-0026 v4 payloads with `isSharedCollateral == false` are unaffected by
  this DIP and remain governed by DIP-0026.
* Special transaction types `10`, `11`, and `12` are invalid.
* Upgraded relay and mining policy MUST treat any output with
  `SHARED_COLLATERAL_SCRIPT` as non-standard and SHOULD NOT mine it.
  Pre-activation consensus validity for such outputs is otherwise unchanged
  unless a separate deployment says otherwise.

After activation:

* v4 ProRegTx with `isSharedCollateral == true`, `ProUpShareTx`,
  `ProUpSharedRegTx`, and `ProDisTx` are valid as specified above.
* DIP-0003 single-owner masternodes (v1, v2, v3) and DIP-0026 v4
  masternodes with `isSharedCollateral == false` continue to operate
  unchanged.
* No conversion path is defined from a non-shared masternode (v1/v2/v3 or
  v4 with `isSharedCollateral == false`) to a shared-collateral masternode.
  A shared-collateral masternode is established only by a new v4 ProRegTx
  with `isSharedCollateral == true`; it is wound down only by a `ProDisTx`.
* No conversion path is defined from a shared-collateral masternode back to
  a non-shared masternode.

A shared-collateral masternode and a non-shared masternode never appear as
the same `proTxHash`; deterministic masternode state is gated on
`state.isSharedCollateral` as specified in [Deterministic Masternode
State](#deterministic-masternode-state).

## Rationale

### Strict superset of DIP-0026

DIP-0026 fixes the operational pain of off-chain reward distribution but
does not constrain the registrar owner, the collateral UTXO, or the exit
path. The most common use cases that motivate DIP-0026 (services that
hold the collateral on behalf of multiple beneficiaries) remain
custodial under v4 with `isSharedCollateral == false`. This DIP
addresses that gap by binding share amounts, refund destinations, and
the collateral itself to per-participant consent.

### Explicit discriminator instead of a distinct provider payload version

A provider transaction payload version is a wire-format identifier, not
a semantic mode flag. Allocating a new ProRegTx provider payload version
solely for shared collateral would conflate two orthogonal axes: the
on-the-wire layout of the v4 payload, and whether the masternode that v4
payload describes is shared. The version number would then drift apart from
any future layout changes that are unrelated to shared collateral, and every
downstream consumer would have to learn to treat one specific version value
as a stand-in for "shared".

Selecting shared-collateral semantics by an explicit discriminator
(`isSharedCollateral`) within the existing v4 payload keeps the wire
version describing only the wire format and keeps the semantic mode
selection a first-class field that callers, validators, indexers, and
state diffs all read directly. The discriminator is committed to by the
registration consent digest and carried into deterministic masternode
state, so a participant signature, a state entry, and an SML query all
agree on the mode without inferring it from the version number, output
shape, or script length. Future layout-level changes to the v4 payload
can advance the wire version without altering what "shared" means; new
shared-mode variants can advance the discriminator without consuming
provider payload version numbers.

The discriminator also keeps the non-shared v4 variant unchanged after the
shared discriminator byte that DIP-0026 v4 must reserve for all v4 ProRegTx
payloads: a v4 ProRegTx with `isSharedCollateral == false` deserializes and
validates exactly as DIP-0026 specifies after that byte, and the variant fields
specific to shared mode appear only when the discriminator selects them.

### Internal collateral only

External shared collateral would require either a multi-signature
collateral UTXO controlled off-chain (defeating the goal of trustless
exit) or a covenant attached to a previously created output (out of
scope for the existing UTXO format). Constraining shared-collateral mode
to internal collateral makes the covenant model tractable and ensures
that every shared collateral output is created by a v4 ProRegTx with
`isSharedCollateral == true` whose consent digest committed every
participant.

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

### Same-block dissolution forbidden

A `ProDisTx` is invalid in the same block as the v4 ProRegTx with
`isSharedCollateral == true` that creates the shared collateral output it
would spend. Dissolution
operates against the deterministic masternode state at the parent
block, so allowing same-block dissolution would require either a
mid-block speculative state machine or a duplicated validation path
that reads the in-progress block under construction. Forbidding the
same-block case keeps state transitions atomic at block boundaries,
matches how DIP-0003 ProUp*Tx are evaluated against the parent state,
and aligns the protected-set rule with a single canonical
share-vector diff (see [Deterministic Masternode
State](#deterministic-masternode-state)). A participant who wishes to
exit immediately after registration can do so in the next block at no
additional cost beyond standard fees.

### One voting key per masternode

Fractional governance would require either a new vote-aggregation
scheme or a per-share signing tier on every governance message. Both are
out of scope. Preserving one voting key per masternode while retaining
existing masternode-type vote weights keeps shared masternodes compatible
with the current governance infrastructure and defers fractional voting
to a future DIP.

## Test Cases

Implementations SHOULD include at minimum the following tests.

### Helper

1. `DistributeByWeight(10000, [2500, 2500, 2500, 2500])` returns
   `[2500, 2500, 2500, 2500]`.
2. `DistributeByWeight(10001, [2500, 2500, 2500, 2500])` returns
   `[2500, 2500, 2500, 2501]`.
3. `DistributeByWeight(10, [0, 1, 1])` returns `[0, 5, 5]`.
4. `DistributeByWeight(7, [1, 1, 1, 1, 1, 1, 1, 1])` returns seven duffs
   to the final index and zero to the other indices.
5. `DistributeByWeight(3, [0, 1, 0, 1, 0])` returns `[0, 1, 0, 2, 0]`:
   the final positive-weight index receives the remainder.
6. `DistributeByWeight(5, [0, 0, 0])` is invalid (all-zero weights) and
   MUST be rejected by the helper.

### Registration

1. A v4 ProRegTx with `isSharedCollateral == true` and two shares whose
   amounts sum to 1000 DASH, both shares signed correctly, is valid.
2. A v4 ProRegTx with `isSharedCollateral == true` and eight shares summing
   to 1000 DASH, all signed, is valid.
3. A v4 ProRegTx with `isSharedCollateral == true` and `nOperatorReward >
   10000` is invalid.
4. A v4 ProRegTx with `isSharedCollateral == true` and one share is invalid.
5. A v4 ProRegTx with `isSharedCollateral == true` and nine shares is
   invalid.
6. A v4 ProRegTx with `isSharedCollateral == true` whose share amounts sum
   to 999.99 DASH is invalid.
7. A v4 ProRegTx with `isSharedCollateral == true` whose share summation
   overflows is invalid.
8. A v4 ProRegTx with `isSharedCollateral == true` and any individual share
   amount outside `MoneyRange` or greater than `GetMnType(nType).collat_amount`
   is invalid.
9. A v4 ProRegTx with `isSharedCollateral == true` and a duplicate
   participant owner key is invalid.
10. A v4 ProRegTx with `isSharedCollateral == true` and a duplicate refund
    script is invalid.
11. A v4 ProRegTx with `isSharedCollateral == true` and duplicate effective
    reward scripts is invalid.
12. A v4 ProRegTx with `isSharedCollateral == true` and a refund script
    paying to a participant owner key is invalid.
13. A v4 ProRegTx with `isSharedCollateral == true` whose collateral output
    uses a P2PKH script (not `SHARED_COLLATERAL_SCRIPT`) is invalid.
14. A v4 ProRegTx with `isSharedCollateral == true` whose `vchSig` is
    non-empty is invalid.
15. A v4 ProRegTx with `isSharedCollateral == true` whose `joinSig` for
    share `i` was produced under a different `outputsHash` is invalid.
16. A v4 ProRegTx with `isSharedCollateral == true` whose `joinSig` for
    share `i` was produced under different input sequences is invalid.
17. A v4 ProRegTx with `isSharedCollateral == true` whose `joinSig` for
    share `i` was produced under a different penalty value is invalid.
18. A v4 ProRegTx with `isSharedCollateral == true` whose `joinSig` for
    share `i` was produced under `isSharedCollateral == false` is invalid;
    the consent digest commits to the discriminator.
19. A v4 ProRegTx with `isSharedCollateral` set to a value other than `0x00`
    or `0x01` is invalid.
20. A v4 ProRegTx with `isSharedCollateral == false` continues to be
    validated by DIP-0026 unchanged and is unaffected by the rules in this
    section.

### Reward Splitting

1. A shared-collateral masternode with shares `[500, 500]` DASH receives
   two coinbase outputs of equal value to each `rewardScript`.
2. A shared-collateral masternode with shares `[300, 300, 400]` DASH
   receives three coinbase outputs proportional to the shares, with
   rounding rules per `DistributeByWeight`.
3. A shared-collateral masternode with `nOperatorReward = 1000` subtracts
   the operator amount first, then splits the remainder by share.
4. A coinbase missing any expected per-share output is invalid.
5. A coinbase with an extra unexpected output for a shared-collateral
   masternode is invalid.

### Updates

1. A `ProUpShareTx` signed by `shares[0].ownerKey` updating
   `shares[0].rewardScript` is valid.
2. A `ProUpShareTx` for share `0` signed by `shares[1].ownerKey` is
   invalid.
3. A `ProUpShareTx` setting `newRewardScript` to a P2PKH paying
   `shares[0].ownerKey` is invalid.
4. A `ProUpShareTx` setting `newRewardScript` to another share's current
   effective reward script is invalid.
5. A `ProUpShareTx` whose signature was produced under different input
   sequences or transaction outputs is invalid.
6. A `ProUpShareTx` with `nVersion != 1` is invalid.
7. A `ProUpSharedRegTx` with `vchSigs.size() == shares.size()`, signed
   by every share in order, updating `nOperatorReward` to a value no
   greater than 10000, is valid.
8. A `ProUpSharedRegTx` with `nVersion != 1` is invalid.
9. A `ProUpSharedRegTx` setting `keyIDVoting` to null is invalid.
10. A `ProUpSharedRegTx` setting `nOperatorReward > 10000` is invalid.
11. A `ProUpSharedRegTx` setting `keyIDVoting` to a key hash already used by
    any current refund script or effective reward script is invalid.
12. A `ProUpSharedRegTx` whose signature was produced under different input
    sequences or transaction outputs is invalid.
13. A `ProUpSharedRegTx` missing one signature is invalid.
14. A `ProUpSharedRegTx` with an extra signature is invalid.
15. A `ProUpSharedRegTx` whose signatures are presented out of share
    order is invalid.
16. A `ProUpSharedRegTx` signature produced for another network's
    `chainGenesisHash` is invalid.
17. A `ProUpSharedRegTx` that changes `pubKeyOperator` resets
    `scriptOperatorPayout`, `netInfo`, and Platform identifiers and leaves the
    masternode PoSe-banned until the new operator submits a valid
    `ProUpServTx`.
18. A `ProUpRegTx` (type `3`) targeting a shared-collateral masternode
    (`state.isSharedCollateral == true`) is invalid.

### Dissolution

1. A unilateral `ProDisTx` after the early period, with exactly one
   signature by `shares[a].ownerKey`, penalty equal to
   `standardPenalty`, and exact non-actor outputs, is valid.
2. A unilateral `ProDisTx` during the early period uses `earlyPenalty`
   instead of `standardPenalty`.
3. A unanimous `ProDisTx` with `shares.size()` signatures in share
   order and zero penalty is valid.
4. A `ProDisTx` with `nVersion != 1` is invalid.
5. A unilateral `ProDisTx` whose actor output exceeds
   `shares[a].amount - penalty` is invalid.
6. A unilateral `ProDisTx` whose non-actor output `i` pays more or less
   than `shares[i].amount + bonus[i]` is invalid.
7. A unilateral `ProDisTx` whose non-actor output `i` pays to a script
   other than `shares[i].refundScript` is invalid.
8. A unilateral `ProDisTx` with an extra output (e.g. OP_RETURN) is
   invalid.
9. A unilateral `ProDisTx` with an extra input is invalid.
10. A unilateral `ProDisTx` whose actor output is omitted because the
   fee equals `shares[a].amount - penalty` (so the actor remainder
   after the fee is zero) is valid, and its output list is the
   non-actor outputs in share order with no actor slot.
11. A unilateral `ProDisTx` that includes an actor output paying zero
    duffs to `shares[a].refundScript` is invalid; the actor output MUST
    be omitted rather than paid as a zero-value output.
12. A unanimous `ProDisTx` whose actor output is omitted (because the
    fee equals `shares[a].amount`) is valid; every non-actor output
    pays exactly `shares[i].amount`.
13. A unanimous `ProDisTx` missing one participant signature is
    invalid.
14. A `ProDisTx` with `mode` other than `0` or `1` is invalid.
15. A `ProDisTx` whose input sequence is changed after signing is invalid,
    including when the sequence change would alter `nLockTime` behavior.
16. A `ProDisTx` whose `outputsHash` does not match the recomputed
    transaction `outputsHash` is invalid; the signature check is not
    reached.
17. A normal transaction (`nVersion < 3` or `nType == 0`) that spends
    the collateral outpoint of a registered shared-collateral masternode,
    including a PoSe-banned masternode that remains registered, is rejected
    by mempool and by block validation.
18. A non-dissolution special transaction whose input spends the
    collateral outpoint of a registered shared-collateral masternode is
    rejected.
19. A normal (non-`ProDisTx`) transaction later in the same block as a
    v4 ProRegTx with `isSharedCollateral == true` that spends the
    just-created shared collateral output is rejected at block connection.
20. A unilateral `ProDisTx` for the same `proTxHash` in the same block
    as the v4 ProRegTx with `isSharedCollateral == true` that creates
    its shared collateral output is invalid, regardless of ordering
    within the block and regardless of whether every other covenant
    rule holds.
21. A unanimous `ProDisTx` for the same `proTxHash` in the same block
    as the v4 ProRegTx with `isSharedCollateral == true` that creates
    its shared collateral output is invalid on the same basis as the
    unilateral case.
22. A `ProDisTx` (unilateral or unanimous) that spends the shared
    collateral outpoint of a shared-collateral masternode whose
    registering v4 ProRegTx with `isSharedCollateral == true` was
    confirmed in a strictly earlier block, and that otherwise
    satisfies every covenant rule, is valid; the masternode is
    removed when this block is connected.
23. An ordinary transaction whose input spends an unrelated UTXO that
    happens to pay `SHARED_COLLATERAL_SCRIPT` but is NOT a recorded
    registered shared collateral outpoint and is NOT a same-block pending
    shared collateral outpoint is NOT rejected by the covenant;
    whether it succeeds depends only on ordinary script evaluation of
    the underlying redeem script.

### Reorg

1. Disconnecting a block containing a v4 ProRegTx with
   `isSharedCollateral == true` fully removes the masternode entry, including
   all share state.
2. Disconnecting a block containing a `ProUpShareTx` restores the
   prior `rewardScript`.
3. Disconnecting a block containing a `ProDisTx` restores the
   shared-collateral masternode entry with its full share state.
4. Disconnecting and reconnecting a chain segment that contains a
   `ProUpSharedRegTx` followed later by a `ProDisTx` in the original
   chain order produces the same final state.

## Implementation Notes

These notes are non-normative guidance for implementers.

* The deterministic masternode state-diff bitfield must reserve a new bit
  for the share vector. Implementations should follow the bit-allocation
  conventions already used in `CDeterministicMNStateDiff` so that
  non-shared snapshots continue to round-trip exactly.
* `CheckSpecialTx` should treat shared collateral outputs as a separate
  spend-rejection rule rather than as part of the per-payload checks; the
  spend rejection applies to *every* transaction in mempool and block
  contexts, not only to special transactions.
* The same-block index of new shared collateral outputs introduced by
  v4 ProRegTx with `isSharedCollateral == true` earlier in the block can
  be implemented as a small `std::unordered_set<COutPoint>` (or
  `std::unordered_map<COutPoint, uint256 /* proTxHash */>` if the
  diagnostic value is wanted) that is populated as block transactions
  are validated and consulted by every later transaction's input scan.
  Any spend of an outpoint in this set within the same block is
  rejected, including by `ProDisTx`; same-block dissolution is not a
  legal path.
* Coinbase construction should reuse the existing payout-pipeline hook
  added by DIP-0026 (PR 184) and append one output per share of every
  scheduled shared-collateral masternode with the computed amount and
  target script.
* The wallet RPC surface should expose: a coordinator-style flow that
  builds an unsigned v4 ProRegTx with `isSharedCollateral == true` given
  each participant's funding inputs and share parameters; per-participant
  `joinSig` production over the consent digest; and a `dissolvemasternode`
  RPC that builds a `ProDisTx`, computes the signed dissolution digest,
  and either collects the actor's signature (unilateral) or the full set
  of signatures (unanimous).
* RPC output for `protx info` for a shared-collateral masternode should
  expose `isSharedCollateral`, the share table, penalty parameters, and
  the canonical `SHARED_COLLATERAL_SCRIPT` template for diagnostic
  purposes.

## Security Considerations

### Trust model

A shared-collateral masternode (`state.isSharedCollateral == true`)
requires only that consensus is honest. No participant trusts any other
participant with custody of funds. Specifically:

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
clients cannot verify shared-collateral share state without a full node
or a DIP-0004-extending future proof. SPV-level features that depend on
shared-collateral metadata (filter matching of refund and reward
scripts, for example) require the full node serving the client to be
honest about shared-collateral state.

### Replay across chains

Both the registration consent digest and the dissolution authorization
digest commit explicitly to `chainGenesisHash`, the genesis block hash of
the network on which the transaction is being authorized. The consensus
digest domain therefore separates networks with different genesis hashes:
a shared-collateral registration or dissolution signed against one
`chainGenesisHash` cannot be replayed onto a network with a different
genesis hash because the digest verified by consensus on the target
network would differ.

This does not provide replay separation between forks or deployments that
share the same genesis block. Signers MUST verify the intended network /
chain context before producing any `joinSig` or dissolution signature, and
implementations deploying shared-collateral mode on same-genesis forks
MUST add any additional fork-specific replay protection they require.
Relying on `proTxHash` alone is insufficient for cross-chain replay
protection because matching transaction and state context across networks
cannot be ruled out by digest construction alone.

### Mode-confusion across the discriminator

Because the v4 ProRegTx payload version is shared between non-shared
(DIP-0026) and shared-collateral modes, an incorrectly implemented validator
that ignores `isSharedCollateral` could mis-parse one variant as the other.
The registration consent digest commits to `payload.isSharedCollateral`,
so a `joinSig` produced under one variant does not verify under the
other; the deterministic masternode state carries
`isSharedCollateral` as a first-class field so that all downstream
authorization, reward, and covenant rules gate on the same flag a
participant signed against. Implementations MUST treat
`isSharedCollateral` as a payload-shape selector that strictly
determines which fields are deserialized and which validation path
runs; falling back to a "best effort" decode that tries both variants
is invalid.

## Open Issues

The following implementation details are deferred and MUST be resolved
before activation:

1. **Final value of `SHARED_MIN_SHARE_DUFFS`.** Currently
   `1000000000` (10 DASH). Subject to dust-policy review and feedback from
   wallet implementers.
2. **Activation deployment name.** Subject to release engineering confirmation
   that no candidate fork bit has already been consumed.
3. **DIP-0026 v4 discriminator reservation.** DIP-0026 v4 MUST reserve the
   `isSharedCollateral` discriminator byte immediately after `nMode` for every
   v4 ProRegTx payload, including non-shared multi-payout registrations, before
   this DIP can activate. Without that reservation, shared and non-shared v4
   parsers would frame the same bytes differently.
4. **State-diff bit value for the share vector.** The exact bit position for
   the shared-collateral share-vector full-replacement diff MUST be assigned
   before activation.

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
