# CIP-XXXX: Signing Policies for Two-Factor External Parties

<pre>
  CIP: ?
  Layer: API/RPC
  Title: Signing Policies for Two-Factor External Parties
  Author:
   Semyon Protas (PixelPlex)
   Yury Korzun (PixelPlex)
  Status: Draft
  Type: Standards Track
  Created: 2026-11-26
  License: CC0-1.0
  Requires: TBD (Two-Factor Signing for External Parties), 0056, 0103
</pre>

## Abstract

The Two-Factor Signing CIP makes every transaction of an external party
need two signatures: one from the user's device and one from a co-signer
that signs only after a second-factor check. The co-signer already reads
each transaction before it signs. This CIP lets it also refuse a
transaction that breaks rules the user has set.

The CIP defines four rules: a list of allowed or blocked receiver
addresses, a list of allowed or blocked participants that host the
receiver, per-transaction and daily limits for each token, and a freeze
that stops all signing. The co-signer reads the token movements from the
token standard choices in the transaction and trusts only the code of
token implementations its provider has approved. The CIP also adds methods
to the co-signer interface and optional fields to the CIP-0103 dApp API.
No changes to the Canton protocol, to Splice or to Daml models are needed.

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

Terms defined in the Two-Factor Signing CIP (party, 2FA party, user key,
recovery key, co-signer, wallet, second factor) keep their meaning.

| Term | Definition |
| --- | --- |
| Policy | The set of rules the co-signer applies to the transactions of one 2FA party. |
| Provider | The operator of the co-signer. |
| Instrument | A token, identified by `admin` and `id` as in CIP-0056 `InstrumentId`. |
| Token standard choice | One of the choices listed in [Token standard choices](#token-standard-choices), exercised through its CIP-0056 interface. |
| Implementation profile | A token implementation the provider trusts: the ids of its Daml packages and the actions outside the token standard a user may take with it. |
| Trusted contract | A contract whose template comes from a package listed in an implementation profile. |
| Wrapper | A contract whose template comes from a package that no implementation profile lists, for example a wallet's own contract that bundles several transfers. |
| Receiver | A party other than the 2FA party to which a transaction sends assets: the receiver of a transfer, or a receiver or the executor of an allocation. |
| Spend | For one instrument, the amount a transaction sends, allocates or burns from the party's holdings. |

### Scope

This CIP covers tokens whose holdings implement the CIP-0056 interfaces and
whose transfers, allocations and their follow-up steps go through the
CIP-0056 choices. Actions outside the token standard are allowed only where
an implementation profile lists them. Other token implementations are left
to later revisions (see [Future updates](#future-updates)).

### Where the policy check runs

The policy check is a new step in the transaction signing flow of the
Two-Factor Signing CIP. It runs after the checks of step 4 (hash
recomputation, user signature, `actAs`, `max_record_time`) and before the
second factor of step 5.

- The co-signer MUST NOT send a second-factor challenge for a transaction
  that fails the policy check.
- A transaction passes the policy check only if every rule passes. Any
  failure is a refusal.
- The policy check uses only the co-signing request, the implementation
  profiles, the stored policy and usage of the party, and the participant
  lookup described in [Participant list](#participant-list). The co-signing
  request does not change: the wallet sends the prepared transaction as
  under the Two-Factor Signing CIP.

### Policy

A policy has the following settings:

| Setting | Values |
| --- | --- |
| `addressList` | mode `off`, `allow` or `deny`, and a list of party IDs |
| `participantList` | mode `off`, `allow` or `deny`, and a list of participant IDs |
| `limits` | for each instrument: optional `perTransaction` and optional `daily` amount |
| `frozen` | `true` or `false` |

- Each list has one mode. A list is either an allow list or a deny list,
  never both.
- The lists and limits apply to all instruments. There are no separate lists
  per instrument.
- An instrument without an entry in `limits` has no limits.

Who sets the policy:

- The provider sets the default policy for new parties.
- The user MAY change any setting of the policy. A change that is not a
  freeze MUST pass a second-factor challenge, the same as a transaction.
- While the party is frozen, the co-signer MUST refuse every change to the
  policy except an unfreeze.
- Only the co-signer stores the policy. It is not published on the ledger.
- When the provider takes a fee as a transfer to its own party, its default
  policy SHOULD let that party and its participant pass the lists.
- Implementation profiles are part of the co-signer's configuration and are
  set by the provider. The user MUST NOT be able to add or change them.

### Token standard choices

| Interface | Choice | Counted as |
| --- | --- | --- |
| `TransferFactory` | `TransferFactory_Transfer` | spend of `transfer.amount` to `transfer.receiver` |
| `AllocationFactory` | `AllocationFactory_Allocate` | spend of `allocation.transferLeg.amount` to the leg `receiver` and the settlement `executor` |
| `TransferInstruction` | `TransferInstruction_Accept`, `_Reject`, `_Withdraw` | no spend |
| `Allocation` | `Allocation_ExecuteTransfer`, `_Cancel`, `_Withdraw` | no spend |
| `AllocationInstruction` | `AllocationInstruction_Withdraw` | no spend |

- A transfer or allocation counts only when the party is its sender. A
  transfer to the party itself (for example to merge holdings) has no spend.
- The co-signer MUST check that the interface of the exercise node comes
  from a CIP-0056 API package it knows, by package id. An interface with the
  same name in another package is not a token standard choice.

### Implementation profiles

An implementation profile describes one token implementation:

```jsonc
{
  "name": "splice-amulet",
  "packages": ["<package id>", "..."],
  "actions": {
    "AmuletRules.AmuletRules_CreateTransferPreapproval": {
      "payer":  { "argument": "provider" },
      "amount": { "result": "amuletPaid" },
      "admin":  { "contract": "dso" },
      "id":     { "const": "Amulet" }
    },
    "TransferPreapproval.TransferPreapproval_Renew": {
      "payer":  { "contract": "provider" },
      "amount": { "result": "amuletPaid" },
      "admin":  { "contract": "dso" },
      "id":     { "const": "Amulet" }
    },
    "TransferPreapproval.TransferPreapproval_Cancel": {},
    "ExternalPartySetupProposal.ExternalPartySetupProposal_Accept": {},
    "AmuletRules.AmuletRules_BuyMemberTraffic": {
      "payer":  { "argument": "provider" },
      "amount": { "result": "amuletPaid" },
      "admin":  { "contract": "dso" },
      "id":     { "const": "Amulet" }
    }
  }
}
```

```jsonc
{
  "name": "utility-registry",
  "packages": ["<package id>", "..."],
  "actions": {
    "TransferPreapproval.create": {},
    "TransferPreapproval.TransferPreapproval_Withdraw": {},
    "TransferPreapproval.TransferPreapproval_Modify": {}
  }
}
```

- `packages` lists package ids, not package names. A package id is the hash
  of the package, so it names the exact code. Every package version the
  provider trusts is listed.
- An action is a choice of a template from the profile's packages, written
  `Template.Choice`, or the creation of such a template, written
  `Template.create`.
- An action with an empty value has no spend. An action that takes assets
  from the party gives the amount and the instrument. Each value is read
  from the choice `argument`, the choice `result` (`exercise_result` in the
  prepared transaction), the exercised `contract`, or is a `const`. A path
  is a dot-separated list of record field labels; a field missing from a
  record reads as none, because serialized records leave out trailing
  `Optional` fields that are `None`.
- `payer`, when present, names the party that pays. The spend counts only
  when the payer is the 2FA party.
- An implementation does not have to be run by one party. One profile
  covers every token built on the same packages, whatever their admin,
  registrar or provider parties.

### Reading the transaction

The co-signer walks the transaction tree from its root nodes and applies
these rules to each node, in this order.

A node uses the party's authority if it is:

- an exercise node in which the party is an acting party, or in which the
  party is a signatory of the exercised contract;
- a create node in which the party is a signatory.

1. **Token standard choice on a trusted contract.** The co-signer counts it
   by its arguments as given in
   [Token standard choices](#token-standard-choices) and does not walk its
   subtree, which is the implementation's code.
2. **Action from a profile on a trusted contract.** An exercise or create
   that uses the party's authority, on a template from a profile's packages,
   is allowed if the profile lists it. The co-signer counts its spend as the
   profile says and does not walk its subtree. If the profile does not list
   it, the co-signer MUST refuse the transaction.
3. **Other node on a trusted contract.** An exercise on a trusted contract
   that does not use the party's authority is not walked.
4. **Exercise on a wrapper.** The co-signer walks the children with the
   same rules. Every effect of the wrapper's code is a node of the
   transaction, so the code itself is not reviewed.
5. **Create of a wrapper signed by the party.** The co-signer MUST refuse
   the transaction. Such a contract can give another party the right to act
   for the party later, without the co-signer.

Fetch nodes have no effect and are skipped. Rollback nodes are walked like
any other subtree. Canton rejects transactions that roll back creates or
archives (`DAML_EFFECTFUL_ROLLBACK_ERROR`), so they never reach the
co-signer.

The spend of the transaction, per instrument, is the sum of everything
counted under rules 1 and 2. Its receivers are the parties counted with it.
A burn has no receiver, so the lists do not apply to it. The limits do.

Rule 1 relies on the trusted code doing what its arguments say. An
attacker can create a contract of a trusted template with their own party
as admin or registrar. It still runs the trusted code, and that code checks
that the instrument and the input holdings belong to the parties named in
the contract. Holdings of other admins cannot be moved through it.

### Rules

#### Address list

- Mode `allow`: every receiver MUST be in the list.
- Mode `deny`: no receiver may be in the list.
- Mode `off`: no check.

#### Participant list

The co-signer looks up the participants that host each receiver on the
synchronizer of the transaction.

- The co-signer MUST NOT take this data from the wallet. It SHOULD use the
  Scan API (`getPartyToParticipant`) of a Super Validator. It MAY ask
  several Super Validators and compare the answers, or use another source
  it trusts.
- Mode `allow`: every hosting participant of every receiver MUST be in the
  list.
- Mode `deny`: no hosting participant of any receiver may be in the list.
- Mode `off`: no lookup and no check.
- If the co-signer cannot determine the hosting participants of a receiver,
  it MUST refuse while the mode is `allow` or `deny`.

The address list and the participant list are independent. A receiver
passes only if it passes both. An allowed address on a blocked participant
is refused, and so is a blocked address on an allowed participant.

#### Limits

For each instrument with an entry in `limits`:

- `perTransaction`: the spend of the transaction MUST NOT exceed it.
- `daily`: the spend of the transaction plus the usage of the party in the
  last 24 hours MUST NOT exceed it.

Usage:

- The window is a sliding window of 24 hours, not a calendar day.
- The co-signer adds the spend of a transaction to the usage when it issues
  the co-signature.
- The co-signer MUST NOT reduce the usage when it is told that a
  transaction was not executed. It cannot check this without participant
  access, and a wallet controlled by an attacker could use it to reset the
  limit.
- Incoming transfers do not change the usage.

#### Freeze

While the party is frozen, the co-signer MUST refuse every
`requestCoSignature` and every policy change except an unfreeze.

- **Freeze** MAY be requested by the user with a message signed by the user
  key, without a second factor. The provider MAY also freeze a party, for
  example on a user's call to support. A freeze MUST NOT be possible with a
  second factor alone, so that an attacker who controls only the user's
  phone number or email cannot block the party.
- **Unfreeze** MUST require either a message signed by the recovery key, or
  a decision of the provider under its own identity checks. A second factor
  together with the user key is not enough, because a freeze is meant for
  the case where both may be compromised.
- A freeze does not block requests signed by the recovery key: user key
  replacement, `resetSecondFactor` and `handover` keep working. A user who
  froze the party after losing the device needs the recovery key both to
  replace the user key and to unfreeze, so it is taken out of cold storage
  once for both.
- Signed freeze and unfreeze messages MUST start with the fixed prefixes
  `canton-2fa-policy-freeze-v1` and `canton-2fa-policy-unfreeze-v1`, MUST
  name the party and MUST carry a timestamp. The co-signer MUST refuse a
  message older than 10 minutes and a message it has already seen.

### Co-signer interface additions

These additions follow "Interface evolution" in the Two-Factor Signing CIP.

| Message | Direction | Content |
| --- | --- | --- |
| `getPolicy` | wallet → co-signer | party ID. Returns the policy and, for each instrument with a daily limit, the usage in the last 24 hours. |
| `setPolicy` | wallet → co-signer | party ID, the new policy without `frozen`. Returns a request ID and the challenge method, as `requestCoSignature` does, and is followed by `submitSecondFactor`. |
| `freeze` | wallet → co-signer | party ID, timestamp, signature by the user key. |
| `unfreeze` | wallet → co-signer | party ID, timestamp, recovery public key, signature by the recovery key. The fingerprint of the key MUST equal the namespace part of the party ID. |

New errors:

- `policyRejected`: the transaction or the change breaks the policy;
- `accountFrozen`: the party is frozen.

The message that comes with an error, and how much it says about the rule
that failed, is up to the co-signer.

### dApp API additions

The following additions to CIP-0103 are optional and backward compatible.
Under the CIP-0103 section "API Evolution" they are non-breaking changes
that need a new minor version of the API artifacts and no amendment to
CIP-0103.

`TxChangedFailedEvent` gets an optional payload:

```typescript
// TxChangedFailedEvent
{
  status: Enum[failed],
  commandId: string,
  payload?: {
    reason: Enum[policyRejected, accountFrozen],
    message?: string
  }
}
```

The `twoFactor` field of `Account`, added by the Two-Factor Signing CIP,
gets one more optional field:

```typescript
twoFactor?: {
  // ...fields from the Two-Factor Signing CIP
  frozen?: boolean
}
```

Policy settings and usage are not exposed to dApps.

### Security considerations

What the policy adds:

- **A stolen device with the second factor.** An attacker who controls the
  user key and the second factor, for example through a SIM swap, can still
  only send to receivers the policy allows and only up to the limits.
- **A user tricked into a transfer.** A user who approves a transfer to a
  scam address is stopped if the address or its participant is not allowed.
- **A factory made by an attacker.** A contract that implements
  `TransferFactory` but is not from a trusted package is a wrapper. Its
  arguments are not counted, and the co-signer checks what its code does
  node by node.
- **Standing authority.** A transaction that would create a contract signed
  by the party from untrusted code is refused, so an attacker cannot get a
  contract that lets them act for the party later.

What it does not cover:

- **Policy changes by an attacker with the user key and the second factor.**
  Such an attacker can loosen the policy and then send. The policy helps
  only against attackers who do not hold both. The freeze covers this case: only
  the recovery key or the provider can lift it.
- **The recovery key.** The holder of the recovery key can move the party to
  another co-signer that applies no policy. This is intended: the recovery
  key controls the party's topology, including which co-signer key it
  uses.
- **Network-level enforcement.** The policy is enforced by the co-signer,
  not by the network. A transaction that the co-signer refuses cannot be
  executed, because the threshold needs its signature, but nothing on the
  ledger shows the policy.
- **Faults in trusted code.** The co-signer counts token standard choices by
  their arguments. If the code of a trusted package moves more than its
  arguments say, the policy does not see it. The provider is responsible
  for the packages it trusts.
- **Receivers that move after signing.** The participant check uses the
  hosting at the time of signing. A receiver can move to another
  participant before the transaction is executed. With the `max_record_time`
  limit of the Two-Factor Signing CIP this window is minutes.
- **Implementations outside the profiles.** Transactions with tokens or
  actions that no profile covers are refused. This protects the limits but
  also blocks such tokens until the provider adds a profile.

Privacy: the policy, the lists and the usage tell the co-signer who the user
trades with and how much. The co-signer MUST treat them like the transaction
data in the Two-Factor Signing CIP: use them only for the checks and not pass
them to third parties.

## Motivation

Two-factor signing stops an attacker who holds only the user's device. It
does not stop one who also gets the second factor, and it does not stop a
user who approves a transfer to the wrong address. In both cases the
co-signer sees a valid request and signs.

Rules such as allowed addresses and daily limits are common in exchanges and
banks, and users expect them. The co-signer is a good place to check them.
It already reads every transaction before it signs, and the network cannot
execute a transaction without its signature. Without a standard, each
co-signer would invent its own way to read token movements, and wallets
would have to integrate with each of them separately.

## Rationale

### Token standard arguments

The token standard choices carry the sender, the receiver, the amount and
the instrument as arguments, and the arguments are part of the signed
transaction. Counting them works the same for every token that implements
the standard, whatever its holding contracts look like. Holding contracts
do not carry their interface views in the transaction, so reading them
would need per-template knowledge of each token.

### Trust is tied to code

Token implementations differ in which party signs the factory. Canton Coin
uses a factory signed by the DSO. The DA Utility Registry uses the
receiver's own preapproval as the factory for direct transfers, signed only
by the receiver, and each token there has its own registrar and provider.
Within one implementation, all tokens run the same code. A profile keyed by
package ids covers all of them, and trusted code with an attacker's parties
can only move that attacker's tokens.

### Wrappers are checked by their effects

Wallets bundle a fee and a transfer, or many transfers, in their own
contracts. Canton does not prepare several commands in one transaction for
an external party, so these bundles need a wrapper. A wrapper's code does
not need review: everything it does appears as nodes of the transaction,
and the co-signer applies the same rules to them. The only effect that a
later transaction could use without the co-signer is a contract signed by
the party, so creating one from untrusted code is refused.

### Profiles are kept by the provider

A profile decides whose code is counted by its arguments. A user who could
add a profile could add a factory that says one thing in its arguments and
does another. Profiles therefore belong to the provider, like the choice of
the co-signer itself.

### The participant lookup does not trust the wallet

A wallet that reports where a receiver is hosted could report an allowed
participant for any receiver. Scan is a public service run by every Super
Validator, so the co-signer can look up the hosting itself without a
participant of its own.

### Freeze and unfreeze need different keys

A freeze only stops signing and cannot move assets, so the user key alone
is enough for it. An unfreeze restores the ability to sign, so it needs
something an attacker with the device and the second factor does not have:
the recovery key, or the provider's own checks.

### Usage is not reduced

The co-signer has no participant access and cannot see whether a signed
transaction was executed. Trusting the wallet on this would let an attacker
reset the daily limit. A user whose signed transaction expired waits for the
window to move on.

## Backwards compatibility

- The new co-signer messages and the new errors are non-breaking changes
  under "Interface evolution" in the Two-Factor Signing CIP.
- The co-signing request does not change. A wallet without support for this
  CIP keeps working with a co-signer that applies a policy; it only cannot
  read or change the policy, and it shows a refusal as an unknown error.
- Wallets whose wrappers call only token standard choices and profile
  actions keep working. Wallets that move tokens outside the token standard,
  or create contracts signed by the user from their own packages, are
  refused until a later revision covers them.
- The dApp API additions are optional fields. dApps that do not know them
  keep working.

## Reference implementation

A proof of concept extends the one of the Two-Factor Signing CIP and runs on
Canton 3.6.3. Its co-signer walks the prepared transaction with one
implementation profile and applies the policy before the second factor. Its
test token implements the CIP-0056 `Holding`, `TransferFactory`,
`TransferInstruction`, `AllocationFactory` and `Allocation` interfaces, with
an admin factory for two-step transfers and allocations and a receiver
preapproval as the factory for direct transfers. All 34 cases pass:

- a direct transfer through the receiver's preapproval, disclosed to the
  sender, and a two-step transfer through the admin's factory;
- a wallet wrapper with a fee and two transfers: each receiver is checked
  against the lists. Preparing the same transfers as two commands without a
  wrapper is refused by the participant;
- an allocation: the receiver and the executor are both checked;
- accepting an incoming transfer has no spend and passes an empty allow
  list;
- actions outside the token standard: a direct holding transfer is
  refused; creating and withdrawing the party's own preapproval and a burn
  listed in the profile are allowed, and the burn counts towards the limits;
- a factory made by an attacker, whose arguments say one token to an
  allowed receiver while its code sends the holding to the attacker, is
  refused. Signed with both keys without the co-signer, the same transaction
  moves the holding to the attacker;
- a trusted factory template with the attacker as admin cannot take a
  holding of another admin: preparation fails in the trusted code;
- a contract signed by the party that lets the attacker move the party's
  tokens later is refused. Signed without the co-signer, it lets the
  attacker take a holding later with no signature of the party;
- creates and archives inside a caught `try` block are refused by the
  participant at prepare time;
- allow and deny lists of participants, per-transaction and daily limits
  (a signed transaction that is never executed still counts), freeze,
  refused unfreeze with the user key, unfreeze with the recovery key, and a
  replayed unfreeze message;
- in every executed transaction, the spend the co-signer counted equals
  what left the party's holdings according to the participant.

Not covered: the Splice and DA Utility Registry packages themselves, and the
Scan lookup (the proof of concept reads the topology of its own participant
instead). The proof of concept will be published with the next revision of
this draft.

## Future updates

- **Wallet profiles.** Wallets with their own contracts that the user signs
  (settings, delegations to the wallet's own service) are refused under
  rule 5. A wallet profile would let such wallets work. In outline:
  - the wallet provider publishes a profile in the format of an
    implementation profile: package ids, the templates the user may create
    and the choices the user may exercise, and for each the spend, if any;
  - the profile is signed by the wallet provider's key, never by the user's
    device, because a captured device must not be able to declare code as
    safe;
  - the co-signer provider reviews the profile and its packages and accepts
    it into its configuration; package ids pin the reviewed code, and a new
    package version needs a new review;
  - the user selects which accepted wallet profiles apply to the party, with
    a second-factor challenge; the co-signer only offers profiles its
    provider has accepted;
  - a transport for publishing profiles and their updates, for example a
    signed document at a well-known HTTPS location of the wallet provider,
    and a way to revoke a profile.
- **Implementations outside the token standard.** Tokens that do not
  implement the CIP-0056 interfaces, or move holdings with their own
  choices, could be supported by reading holdings instead of arguments: the
  co-signer gets owner, amount and instrument from contract arguments with
  per-template knowledge and counts what the transaction archives and
  creates.
- **Token Standard V2.** CIP-0112 adds new interfaces, accounts and a
  standard batching package deployed on all Splice validators. A later
  revision can add the V2 choices to the token standard table and list the
  batching package in a standard profile.
- **Limits in a common currency.** This CIP sets limits in units of each
  instrument. Limits in USD or another currency need a price source the
  co-signer can trust.

## Open questions

- **Profile publication.** Who publishes the package ids and actions for
  common implementations (Splice releases, the DA Utility Registry), and in
  which format, so that every provider does not build them alone.
- **Multi-hosted receivers.** The Scan `getPartyToParticipant` endpoint
  fails for parties with more than one hosting participant. The DSO party,
  which is hosted on the participants of all Super Validators, is one of
  them. While the participant list is on, co-signers need another source
  for such receivers or refuse them.
