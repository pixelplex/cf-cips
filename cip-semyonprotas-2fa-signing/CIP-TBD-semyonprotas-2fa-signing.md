# CIP-XXXX: Two-Factor Signing for External Parties

<pre>
  CIP: ?
  Layer: API/RPC
  Title: Two-Factor Signing for External Parties
  Author:
   Semyon Protas (PixelPlex)
   Yury Korzun (PixelPlex)
  Status: Draft
  Type: Standards Track
  Created: 2026-11-17
  License: CC0-1.0
  Requires: 0056, 0103
</pre>

## Abstract

Today an external party on the Canton Network is usually controlled by one
key. Whoever holds that key can move all of the party's assets, and a second
factor checked only in a wallet backend does not help: the network accepts a
transaction signed by that one key.

This CIP standardizes two-factor signing for external parties at the
topology level. After enrollment, every Daml transaction of the party needs
two signatures: one from a key on the user's device and one from a co-signer
key held in a KMS by a co-signer, which is the wallet provider or a separate
service. The co-signer uses its key only after the user passes a second
factor (authenticator app, SMS or email). The party's original root key
becomes a recovery key. It is kept in cold storage and is used only to change
the party's topology, for example to replace a lost device key.

The CIP defines the key roles and the topology layout, the enrollment flow
for existing and new parties, the transaction signing flow, the recovery
flows, storage requirements for each key, the co-signer interface and
registry, and optional additions to the CIP-0103 dApp API so that dApps can
see that an account uses two-factor signing. No changes to the Canton
protocol, to Splice or to Daml models are needed.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they
appear in all capitals, as shown here.

### Definitions

| Term | Definition |
| --- | --- |
| Party | An external party: a party whose Daml transactions are authorized by signatures from keys listed in its topology, not by its hosting participant. |
| Recovery key | The party's namespace (root) key. The party ID is derived from it, so it can never be removed from the party. After enrollment it signs only topology changes. |
| User key | A signing key generated and kept on the user's device. |
| Co-signer key | A signing key generated and kept by the co-signer in a KMS or HSM. |
| Co-signer | The service that holds the co-signer key and signs with it after a successful second-factor check. It can be operated by the wallet provider or by a separate provider. |
| Second factor | A check that the user, and not only the user's device, approves an action: a TOTP code, an SMS code or an email code. |
| Wallet | The software the user operates the party with. It implements the dApp API (CIP-0103) where applicable. |
| 2FA party | A party whose topology matches [Topology layout](#topology-layout). |

### Topology layout

A 2FA party MUST have the following topology on every synchronizer it is
hosted on:

1. A root `NamespaceDelegation` for the recovery key, with the restriction
   `CanSignAllMappings`. This root certificate is what lets the recovery key
   control the party after it is removed from `partySigningKeys`.
2. A `PartyToParticipant` mapping whose `party_signing_keys` contains exactly
   two keys, the user key and the co-signer key, with `threshold = 2`.
   - Both keys MUST have the usages `SIGNING_KEY_USAGE_PROTOCOL` and
     `SIGNING_KEY_USAGE_PROOF_OF_OWNERSHIP`.
   - The recovery key MUST NOT be listed in `party_signing_keys`.
   - Hosting participants, their permissions and the confirmation threshold
     are not affected by this CIP.
3. No `PartyToKeyMapping` for the party. This mapping is deprecated, and
   keys MUST be configured in `PartyToParticipant` only.

With this layout:

- a Daml transaction of the party is valid only with valid signatures from
  both the user key and the co-signer key;
- a signature from the recovery key does not count towards the threshold;
- any change of `party_signing_keys` or of the threshold needs the
  authorization of the party namespace, that is, the recovery key. Adding a
  key also needs a signature from the added key.

### Key storage

#### Recovery key

- The wallet MUST NOT use the recovery key to sign Daml transactions of a
  2FA party, and MUST NOT load it into memory for regular transaction
  signing.
- After enrollment the wallet MUST move the recovery key to cold storage:
  encrypted at rest, with a secret separate from the one that unlocks the
  user key for daily use (for example a dedicated passphrase).
- The recovery key MAY stay on the user's device in cold storage. The wallet
  MUST offer to export it to external media (a file on removable storage, a
  printed or written backup) and MUST offer to delete it from the device
  once the user confirms the export.
- Before deleting it from the device, the wallet SHOULD check that the
  export is usable, for example by asking the user to import it back or to
  confirm its fingerprint.
- The wallet provider and the co-signer MUST NOT receive the recovery key in
  any form.

#### User key

- The user key MUST be generated on the user's device. Its private part MUST
  NOT leave the device in plain form, and the wallet provider MUST NOT have
  access to it.
- The user key SHOULD be kept in a hardware-backed key store where the
  platform offers one (for example Secure Enclave or Android Keystore).

#### Co-signer key

- The co-signer key MUST be generated inside a KMS or HSM as a
  non-exportable key. The co-signer can request signatures from it but can
  never read the private key.
- The co-signer MUST use a separate co-signer key for each party.
- The co-signer MUST sign with the co-signer key only as described in
  [Transaction signing](#transaction-signing) and
  [Recovery and maintenance](#recovery-and-maintenance).
- The co-signer MUST keep an audit log of every signature it produces:
  party, transaction hash, second-factor method used, and time.

### Second factors

- The co-signer MUST support at least one of the methods below and SHOULD
  support `totp`.
- The wallet MUST be able to present a challenge of any of these methods.

| Method | Description |
| --- | --- |
| `totp` | A time-based one-time password (RFC 6238) from an authenticator app enrolled with the co-signer. |
| `sms` | A one-time code sent by the co-signer to a phone number enrolled by the user. |
| `email` | A one-time code or confirmation link sent by the co-signer to an email address enrolled by the user. |

Requirements for every method:

- A method MUST be verified at enrollment time with one successful
  challenge before it can be used.
- A challenge MUST be bound to one co-signing request and MUST expire. A
  lifetime of 5 minutes is RECOMMENDED.
- The co-signer MUST NOT send a challenge before the request has passed the
  checks in step 4 of [Transaction signing](#transaction-signing), including
  the user signature. Without the user key nobody can make the co-signer send
  SMS or email.
- The co-signer MUST limit failed attempts per challenge and per party and
  MUST reject the request once the limit is reached.
- For `sms` and `email`, the message MUST include the transaction summary
  (see step 4 of [Transaction signing](#transaction-signing)), so that the
  user checks what is being signed on a channel the wallet does not control.
- Adding, changing or removing a method MUST require a successful challenge
  of a method that is already enrolled.
- If the user has lost access to all enrolled methods, the co-signer MUST NOT
  restore access by any other means, such as a support request or an
  identity check. The only way is a `resetSecondFactor` request signed by the
  recovery key (see [Co-signer interface](#co-signer-interface)).

### Enrollment of an existing party

An existing external party has one key, the recovery key, which is also its
only signing key. Depending on how the party was onboarded, the party either
has a root `NamespaceDelegation` already, or is self-signed: its
`PartyToParticipant` authorizes itself because the namespace key is listed in
its own `party_signing_keys`.

Enrollment consists of these steps, in this order:

1. **Second factor.** The user enrolls at least one second-factor method
   with the co-signer, and the co-signer verifies it with one challenge.
2. **Keys.** The wallet generates the user key on the device. The co-signer
   generates the co-signer key in its KMS and returns the public key to the
   wallet.
3. **Root certificate.** If the party has no root `NamespaceDelegation`, the
   wallet publishes one for the recovery key, signed by the recovery key.
   This MUST happen before step 4. Otherwise nothing would control the
   namespace once the recovery key leaves `party_signing_keys`.
4. **Signing keys.** The wallet publishes a new `PartyToParticipant` with
   `serial` equal to the current serial plus one. It keeps the hosting
   participants and the confirmation threshold unchanged and sets
   `party_signing_keys` to the user key and the co-signer key with
   `threshold = 2`. The transaction MUST be signed by:
   - the recovery key, as the namespace key;
   - the user key, as proof of ownership;
   - the co-signer key, as proof of ownership. The co-signer signs only this
     exact topology transaction, after it has checked the mapping and the
     user has passed a second-factor challenge.
5. **Cold storage.** Once the new mapping is effective, the wallet moves the
   recovery key to cold storage as described in
   [Recovery key](#recovery-key).
6. **Registry.** The co-signer publishes its registry entry (see
   [Co-signer registry](#co-signer-registry)).

If step 4 fails, the party stays a single-key party. The co-signer SHOULD
delete a co-signer key that has not appeared in the party's topology within
24 hours.

### Enrollment of a new party

A wallet MAY create a party with two-factor signing from the start. In that
case the recovery key never signs a Daml transaction.

1. The user enrolls a second-factor method with the co-signer, as in step 1
   above.
2. The wallet generates the recovery key and the user key on the device. The
   co-signer generates the co-signer key.
3. The wallet onboards the party (Ledger API `AllocateExternalParty`) with
   the following topology transactions:
   - a root `NamespaceDelegation` for the recovery key, signed by the
     recovery key;
   - a `PartyToParticipant` with the hosting participant or participants,
     and with `party_signing_keys` set to the user key and the co-signer key
     with `threshold = 2`. It is signed by the recovery key, the user key,
     the co-signer key, and the hosting participant (the participant adds
     its signature during allocation).
4. The wallet moves the recovery key to cold storage right away.
5. The co-signer publishes its registry entry.

### Transaction signing

Every Daml transaction of a 2FA party follows this flow. The wallet submits
the transaction; the co-signer never submits.

1. **Prepare.** The wallet calls `PrepareSubmission` on a participant and
   receives the prepared transaction, its hash and the hashing scheme
   version.
2. **User signature.** The wallet shows the user what the transaction does
   and, after the user approves, signs the hash with the user key.
3. **Co-signing request.** The wallet sends the co-signer a co-signing
   request (see [Co-signer interface](#co-signer-interface)) with the
   prepared transaction, the hashing scheme version and the user signature.
4. **Checks by the co-signer.** The co-signer MUST, in this order:
   - recompute the transaction hash from the prepared transaction itself,
     following the Canton interactive submission hashing specification for
     the given hashing scheme version. It MUST NOT trust a hash supplied by
     the wallet;
   - verify the user signature over the recomputed hash with the user key
     registered with the co-signer at enrollment or at the last user key
     replacement;
   - check that `actAs` is exactly the enrolled party;
   - check that `max_record_time` is set and is not further ahead than the
     co-signer's limit (see [Time limits](#time-limits));
   - build a summary of the transaction for the user. For token transfers
     under CIP-0056 the summary MUST state the receiver, the amount and the
     instrument. For other transactions it MUST at least list the templates
     and choices exercised and the parties that act.

   These checks use only the request. The co-signer needs no access to a
   participant. If the registered user key is outdated, for example after a
   recovery the co-signer did not learn about, the network rejects the
   transaction because the user signature does not count towards the
   threshold.
5. **Second factor.** The co-signer sends a challenge through one of the
   user's enrolled methods, and the wallet presents it to the user.
6. **Co-signature.** After the challenge is passed, the co-signer signs the
   recomputed hash with the co-signer key and returns the signature. The
   co-signer SHOULD NOT sign the same hash more than once.
7. **Execute.** The wallet calls `ExecuteSubmission` (or
   `ExecuteSubmissionAndWait`) with both signatures.

### Time limits

The time between steps 1 and 7 includes waiting for the user and for an SMS
or email. The synchronizer limits this time in two ways:

- `preparation_time_record_time_tolerance` (24 hours by default) limits how
  long after prepare a transaction can still be executed;
- `ledger_time_record_time_tolerance` (60 seconds by default) applies when
  the ledger time of the transaction is fixed at prepare. This is the case
  when a choice reads the time with `getTime`. One minute is too short for a
  human second factor. Choices that only check a deadline
  (`assertWithinDeadline`, used by Splice for external parties) set only an
  upper bound on ledger time and are not affected.

Requirements:

- The wallet MUST set `max_record_time` in `PrepareSubmission`. It is the
  expiry of the signatures: after it, the transaction is rejected even with
  both signatures. A value of 5 to 10 minutes after prepare is RECOMMENDED.
- The co-signer MUST refuse a transaction without `max_record_time`, and
  SHOULD refuse one whose `max_record_time` is more than 10 minutes ahead.
- The co-signer SHOULD refuse a transaction whose ledger time is fixed at
  prepare (`min_ledger_effective_time` equal to
  `max_ledger_effective_time`).

### Recovery and maintenance

All flows below change the party's `PartyToParticipant` with `serial` equal
to the current serial plus one. The hosting participants and the
confirmation threshold stay unchanged.

| Situation | New `party_signing_keys` | Signed by |
| --- | --- | --- |
| User key lost or device replaced | new user key and co-signer key, `threshold = 2` | recovery key and new user key |
| Co-signer key replaced or co-signer provider changed | user key and new co-signer key, `threshold = 2` | recovery key and new co-signer key |
| Two-factor signing disabled | user key, `threshold = 1` | recovery key |

- For a user key replacement, the co-signer MUST NOT register the new user
  key for its checks unless the request carries the topology transaction
  that makes the change, signed by the recovery key. It SHOULD also require
  a second-factor challenge.
- The co-signer checks a recovery key signature without participant access:
  the request carries the recovery public key, and its fingerprint MUST equal
  the namespace part of the party ID.
- If the recovery key is lost, the party's topology can no longer be
  changed. Transactions still work with the user key and the co-signer key.
  The wallet SHOULD then advise the user to move the assets to a new party.
- If both the user key and the recovery key are lost, the party can no
  longer act. The co-signer key alone cannot sign for the party, and the
  co-signer cannot recover it.

#### Moving to another co-signer

No keys or second-factor data move between co-signers. The old co-signer
key cannot be exported, and second factors are enrolled again with the new
co-signer.

1. The user enrolls a second-factor method with the new co-signer, which
   creates a new co-signer key.
2. The wallet publishes a `PartyToParticipant` with the user key and the new
   co-signer key, `threshold = 2`, signed by the recovery key and the new
   co-signer key.
3. The new co-signer publishes its registry entry.
4. The wallet sends the old co-signer a `handover` request with the signed
   topology transaction from step 2. The old co-signer MUST check that it is
   signed by the recovery key (as above) and that its own key is not in the
   new `party_signing_keys`. It then MUST delete its co-signer key, its
   registry entry and the party's second-factor data.

The old co-signer cannot see whether the transaction became effective,
because it has no participant access. This is safe: only the holder of the
recovery key can produce the request, and once the transaction is
effective the old key no longer counts towards the threshold anyway.

### Co-signer interface

The wallet and the co-signer talk over JSON-RPC 2.0 over HTTPS, the same
transport style as CIP-0103. Each message below is a JSON-RPC method. Binary
values (prepared transactions, topology transactions, signatures, public
keys) are base64-encoded.

| Message | Direction | Content |
| --- | --- | --- |
| `enrollFactor` | wallet → co-signer | party ID, method, method details (phone number, email address, or TOTP setup request) |
| `verifyFactor` | wallet → co-signer | party ID, method, code |
| `createCoSignerKey` | wallet → co-signer | party ID. Returns the co-signer public key and its fingerprint. |
| `requestCoSignature` | wallet → co-signer | party ID, prepared transaction, hashing scheme version, user signature. Returns a request ID, the transaction summary, the challenge method and the expiry time. |
| `submitSecondFactor` | wallet → co-signer | request ID, code. Returns the co-signer signature or an error. |
| `requestTopologyCoSignature` | wallet → co-signer | party ID, serialized topology transaction. Used in enrollment step 4 and when the co-signer key is replaced. Followed by `submitSecondFactor`. |
| `replaceUserKey` | wallet → co-signer | party ID, the signed topology transaction that replaces the user key. The co-signer registers the new user key for its checks. |
| `resetSecondFactor` | wallet → co-signer | party ID, new method and its details, recovery public key, signature by the recovery key over the request. Replaces all enrolled methods after the user lost them. The new method is verified with `verifyFactor`. |
| `handover` | wallet → old co-signer | party ID, recovery public key, the signed topology transaction that replaces this co-signer's key. The co-signer deletes its key, registry entry and second-factor data. |
| `revoke` | wallet → co-signer | party ID. The co-signer stops signing for the party. Requires a second-factor challenge. |

Errors MUST at least distinguish: unknown party, unsupported hashing scheme,
invalid user signature, wrong `actAs`, missing or too distant
`max_record_time`, invalid recovery key signature, co-signer key still in
the new topology, expired challenge, wrong code, and attempt limit reached.

### Co-signer registry

The registry lets any wallet find the co-signer of a 2FA party from the
party ID alone, for example after the user moves to another wallet.

A registry entry has these fields:

| Field | Content |
| --- | --- |
| `fingerprint` | Fingerprint of the co-signer key. |
| `endpoint` | HTTPS URL of the co-signer's JSON-RPC interface. |
| `provider` | Display name of the co-signer provider. |
| `signature` | Signature by the co-signer key over the other fields. |

- The co-signer MUST publish an entry for each co-signer key once the key
  is in the party's `party_signing_keys`, and MUST remove it after `revoke`
  or after the key is replaced.
- The signed message MUST start with the fixed prefix
  `canton-2fa-cosigner-registry-v1`, so that it can never be read as a
  transaction hash.
- To find the co-signer, the wallet reads the party's `PartyToParticipant`,
  looks up the fingerprints of its signing keys in the registry, and MUST
  verify the entry signature with the public key from the topology.

Because each entry is signed by a key from the party's topology, whoever
runs the registry cannot redirect wallets to another co-signer. It can only
withhold entries. Where entries are stored is an open question.

### dApp API additions

The following additions to CIP-0103 are optional and backward compatible.
Under the CIP-0103 section "API Evolution" they are non-breaking changes
that need a new minor version of the API artifacts and no amendment to
CIP-0103.

The `Account` type gets these optional fields:

```typescript
{
  // ...existing CIP-0103 Account fields
  signingKeys?: {
    publicKey: string,
    fingerprint: string,
    role: Enum[user, coSigner]
  }[],
  signingThreshold?: number,
  twoFactor?: {
    enabled: boolean,
    methods: Enum[totp, sms, email][]
  }
}
```

For a 2FA party, `publicKey` MUST be the user key, and `namespace` MUST be
the fingerprint of the recovery key.

`txChanged` gets one more event type, emitted after the user signature (step
2) and before the co-signature (step 6):

```typescript
// TxChangedAwaitingSecondFactorEvent
{
  status: Enum[awaitingSecondFactor],
  commandId: string,
  payload: {
    method: Enum[totp, sms, email],
    expiresAt: string // ISO 8601
  }
}
```

`signMessage` for a 2FA party uses the user key.

dApps do not take part in enrollment or recovery. As set out in the CIP-0103
section "Topology-related capabilities", topology changes stay with the
wallet.

### Security considerations

What this scheme protects against:

- **Theft or compromise of the user key** (malware on the device, a stolen
  backup). The attacker cannot move assets without the co-signer
  signature, and cannot change the topology without the recovery key.
- **Compromise of the co-signer or of its KMS.** The co-signer key alone
  cannot sign for the party.
- **Substitution of the transaction after approval.** The co-signature is
  over the hash of one prepared transaction. A co-signature for one
  transaction is not valid for another.
- **A wallet frontend that shows false transaction details.** With `sms`
  and `email`, the user sees the summary built by the co-signer from the
  prepared transaction.

What it does not protect against:

- **Theft of the recovery key.** The recovery key alone can replace both
  signing keys. This is why it MUST stay in cold storage.
- **Simultaneous compromise of the user key and the co-signer** (or of the
  user key and the user's second factor, for example a SIM swap for `sms`).
- **A compromised device combined with `totp`.** A TOTP code is not bound to
  a transaction, so the user sees only what the wallet shows. Co-signers
  SHOULD prefer methods that carry the summary for high-value transactions.
- **An unfamiliar transaction approved by the user.** The co-signer signs
  any transaction once the second factor is passed. Outside CIP-0056
  transfers, the summary lists only templates and choices, and the user may
  not understand what they do.
- **Recovery keys of existing parties.** Before enrollment, an existing
  party's recovery key signed every transaction as its only key. Copies may
  already exist outside cold storage. Enrollment cannot rotate the namespace
  key without changing the party ID. New parties SHOULD be created with
  two-factor signing from the start.

Privacy:

- The co-signer sees every transaction of the party in full (amounts,
  counterparties, assets) and the user's phone number or email address.
  When the co-signer is a separate provider, this is data the wallet
  provider would otherwise keep to itself.
- The co-signer MUST use transaction data only for the checks and the
  summary, MUST NOT pass it to third parties, and MUST NOT keep prepared
  transactions after the request is finished. The audit log keeps only the
  fields listed in [Co-signer key](#co-signer-key).
- When the co-signer is not the wallet provider, the wallet MUST tell the
  user who operates it before enrollment.

Regulation: a co-signer holds one of the two keys a transaction needs and can
refuse to sign. In some jurisdictions this may count as partial custody.
Providers should check this before they offer the service.

## Motivation

External parties put users in control of their keys, but a single key is a
single point of failure. If the key leaks, all assets of the party can be
moved at once, and nothing on the network can stop it.

Wallets can add a second factor in their backend, but the network accepts
any transaction with a valid signature from the party's key. Anyone who
holds the key can skip the wallet backend and submit directly. A second
factor only protects the party if the network itself refuses transactions
without it.

Canton 3.x already has what is needed: `PartyToParticipant` can list several
party signing keys with a threshold, and the namespace key controls changes
to that list. What is missing is a common way to use these features:

- which key plays which role;
- how a wallet moves an existing party to two-factor signing;
- what the co-signer must check before it signs;
- how a user recovers from a lost device;
- how dApps learn that an account uses two-factor signing.

Without a standard, each wallet would build its own variant. Users could not
switch wallets or co-signers without losing the protection or moving their
assets.

## Rationale

### Topology threshold instead of Daml contracts

A second factor can also be built in Daml, for example with a guardian party
that must co-sign a custody contract holding the assets. That approach needs
custody wrappers around holdings and changes in every application that
should honor them. The topology threshold works below Daml: every
transaction of the party needs both signatures, whatever the application,
and existing token-standard contracts work unchanged.

### The root key as recovery key

The namespace key cannot be rotated without changing the party ID, so
recovery is built around it. Keeping it out of `party_signing_keys` and in
cold storage separates the keys by use: two hot keys for daily transactions
and one cold key for rare topology changes. Removing the namespace key from
`party_signing_keys` requires a root `NamespaceDelegation` first, which is
why enrollment issues one for self-signed parties.

A 2-of-3 setup with the recovery key in `party_signing_keys` was considered
and rejected. There, the recovery key and the user key together could move
assets without the second factor. The recovery key would also become a
transaction key that wallets have reason to keep at hand, which works
against cold storage.

### The co-signer recomputes the hash

If the co-signer signed whatever hash the wallet sends, a compromised wallet
could get a co-signature for any transaction, and the second factor would
only confirm "sign what was sent". Recomputing the hash from the prepared
transaction lets the co-signer show the user what is actually signed.

### The co-signer as a separate role

The co-signer's checks use only the request: the hash is recomputed from
the prepared transaction, and the user signature is checked with the key
registered at enrollment. If that key is outdated, the network rejects the
transaction anyway. So the co-signer needs no participant access, and
nothing ties it to the wallet. A separate co-signer also means that a
compromised wallet backend does not hold either of the two signing keys.
The cost is privacy (see [Security considerations](#security-considerations)).

### The wallet executes

The wallet already holds the prepared transaction and the user signature,
and it reports the result to the dApp through `txChanged`. Letting the
wallet submit keeps the co-signer a signing service only, with no
submission rights and no need to track completions.

### Rejected: codes derived from public data

Another option was a code computed from public data (for example the
contract ID and the party fingerprint) and checked inside a Daml contract.
Everything a contract can check is visible to whoever submits the
transaction. Such a code adds no secret and is not a second factor. Storing
a hash of a short code in a contract fails too, because a six-digit code can
be brute-forced offline in milliseconds.

## Backwards compatibility

- No changes to the Canton protocol, to Splice or to Daml models. The CIP
  uses topology features available in Canton 3.x. The reference
  implementation runs on Canton 3.6.3 (Splice 0.9.3).
- Parties that do not enroll are not affected.
- The dApp API additions are optional fields and one new `txChanged`
  status. dApps that do not know them keep working, but a dApp that treats
  unknown `txChanged` statuses as errors would need an update.
- A wallet that does not implement this CIP cannot sign for a 2FA party,
  because it has no access to a co-signer. Moving a 2FA party to such a
  wallet requires disabling two-factor signing first, with the recovery
  key.

## Reference implementation

A proof of concept runs on Canton 3.6.3 over the gRPC Ledger API and Admin
API only, with all keys held outside Canton. Its co-signer has no access to
any participant: it receives the serialized prepared transaction, the
hashing scheme version and the user signature. All 26 cases pass:

- an existing self-signed party is enrolled: root certificate, then a
  `PartyToParticipant` with the user key and the co-signer key, threshold 2;
- transactions signed by the user key only, the co-signer key only or the
  recovery key only are rejected, and so is a co-signature taken from
  another transaction. The user key and the co-signer key together are
  accepted;
- a topology change signed by the user key without the recovery key is
  rejected with `TOPOLOGY_UNAUTHORIZED_TRANSACTION`;
- the recovery key replaces a lost user key. The old user key stops working
  even while the co-signer still has it registered;
- a transaction prepared on one participant and executed on another that
  does not host the party is checked the same way;
- the co-signer's recomputed hash equals the participant's hash (scheme
  V2), and its summary states the receiver and amount;
- the co-signer refuses before sending any challenge in these cases: a
  transaction sent with the user signature of another transaction, a
  receiver changed in the transaction body, no `max_record_time`,
  `max_record_time` one hour ahead. It also refuses a wrong code and a
  second attempt on the same request;
- after 90 seconds between prepare and execute, a plain transfer and a
  transfer with `assertWithinDeadline` are accepted. A transfer that reads
  `getTime` is rejected (`NOT_SEQUENCED_TIMEOUT`), and so is a transfer whose
  `max_record_time` has passed (`LOCAL_VERDICT_MAX_RECORD_TIME_EXCEEDED`).

Not covered: enrollment of a new party with two-factor signing from the
start, the SMS and email methods, the co-signer interface over a network
transport, and token-standard decoding of summaries. The proof of concept
will be published with the next revision of this draft.

## Future updates

This CIP sets a minimal base. Later revisions, or CIPs that list it in
`Requires`, can extend it, for example with more second-factor methods,
without changing what is defined here.

## Open questions

- **Registry storage.** Where registry entries live: a Daml contract on the
  network, a public HTTPS location, or another store.
