---
title: "Member-only Secrets in the Messaging Layer Security Protocol"
abbrev: "MLS Member-only Secrets"
category: info

docname: draft-mahy-mls-member-secrets-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Messaging Layer Security"
keyword:
 - member secrets
 - DS privacy
 - Distribution Service privacy
 - metadata privacy
 - MIMI pseudonyms
venue:
  group: "Messaging Layer Security"
  type: ""
  mail: "mls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mls/"
  github: "rohanmahy/mls-member-secrets"
  latest: "https://rohanmahy.github.io/mls-member-secrets/draft-mahy-mls-member-secrets.html"

author:
 -
    fullname: Rohan Mahy
    organization: Rohan Mahy Consulting Services
    email: rohan.ietf@gmail.com

normative:

informative:
  MMR:
    target: https://github.com/ietf-wg-mimi/mimi-protocol/pull/101
    title: "Add section on minimal metadata rooms #101"
    author:
        name: Konrad Kohbrok

--- abstract

This document describes a way to keep secrets in MLS groups that can be read by members, but not by in intermediary.


--- middle

# Introduction

This is a Work in Progress mechanism to keep some secrets with respect to non-members, including from the MLS Distribution Service (DS).

Realted work on a similar topic includes Minimal Metadata Rooms (MMR) {{MMR}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

In MLS, the term Directory Server (DS) refers to...

# Mechanisms

One key per group
(MMR) in PR #102

Use a separate MLS group using PrivateMessage to generate secrets


~~~ tls
struct {

} EphemeralSecret

EphemeralSecret EphemeralSecretsData<V>;

struct {

} EphemeralSecretsUpdate;

~~~

# Member-only disclosures

This document defines two new MLS application components to facilitate sharing secrets with members, that are hidden from the DS.
It defines a new MLS key schedule Exporter Label, `member_identity_disclosure_secret`.

These two application components provide a way to efficiently update encryption secrets inside LeafNode and GroupContext objects, when needed to achieve privacy from the DS, including former members colluding with the DS.

~~~ aasvg
Leaf Node or GroupContext
  app_data_dictionary or specific credentials type
    some specific component
     (individually AEAD encrypted) <------------------------------+
                          ^                                       |
                          |                                       |
    leaf_index, blinded_claim_hash,                               |
        encrypted_during_epoch                                    |
        (individual disclosure key encrypted w/ per-epoch key)    |
                          ^                                       |
                          |        OldEpochMemberEncryptionKeys:  |
                          |         (epoch, old_member_secret encrypted
                          |          with current
                          |          member_identity_disclosure_secret)
                          |                    ^
                          |                    |
MLS Key schedule, member_identity_disclosure_secret
~~~

During any epoch when a disclosure is added or updated, the new (added or updated) ephemeral keys are encrypted using the `member_identity_disclosure_secret` for the epoch about to be committed.
Also the `member_identity_disclosure_secret`s for any previous epochs that are still being used to encrypt individual credentials are encrypted using the target epoch `member_identity_disclosure_secret` in the OldEpochMemberEncryptionKeys struct.

~~~ tls
struct {
    opaque bytes<V>;
} BlindedClaimHash;

struct {
    uint32 leaf_index;
    BlindedClaimHash blinded_claim_hash;
    uint64 encrypted_during_epoch;
} PerDisclosureEpochEncryptor;

struct {
    PerDisclosureEpochEncryptor per_disclosure_epoch_map<V>;
} PerDisclosureEncryptionMap;

PerDisclosureEncryptionMap per_disclosure_encryption_map;

struct {
    BlindedClaimHash removed_disclosures<V>;
    PerDisclosureEpochEncryptor updated_disclosures<V>;
} PerDisclosureEncryptionMapUpdate;

PerDisclosureEncryptionMapUpdate per_disclosure_encryption_map_update;

struct {
    uint64 epoch;
    opaque encrypted_member_secret<V>;
} OldEpochEncryptionKey;

struct {
    OldEpochEncryptionKey old_epoch_encyption_keys<V>;
} OldEpochMemberEncryptionKeys;

OldEpochMemberEncryptionKeys old_epoch_member_encryption;
OldEpochMemberEncryptionKeys old_epoch_member_encryption_update;
~~~

## Post-Leaver Privacy (PLP)

Using the scheme above, any member-only disclosures are encrypted with a member-only key from the epoch in which these disclosures originally appear.
New joiners are provided a way to view the old per-epoch keys, so the new joiners can decrypt the member-only disclosures of pre-existing members, but that not every disclosure needs to be updated during every commit.

In many implementations, the AS and DS are tightly-coupled, so hiding information from the DS which is known to the SD-CWT Issuer is not a priority.
However, if PLP is desired, PLP updates are needed only when there is a Remove proposal (or SelfRemove proposal) and an Add proposal in the same or later commit.
In other words if Alice removes Bob in epoch 4, but the next add isn't until Alice adds Cathy in epoch 6, then a PLP update needs to be included in epoch 6.
This prevents Cathy from colluding with the DS to decrypt a disclosure about Bob.

To perform a PLP update, the (PerDisclosureEpochEncryptor) ephemeral keys encrypted from any epoch or earlier than the epoch containing a Remove or SelfRemove, are re-encrypted using the about-to-be committed `member_identity_disclosure_secret`. As with any other commit, unused previous epochs are pruned from OldEpochMemberEncryptionKeys, and all the remaining `member_identity_disclosure_secret`s for old epochs are encrypted using the target epoch `member_identity_disclosure_secret`.

# Generic form of EncryptWithLabel/DecryptWithLabel

To possibly modify later with the SafeEncryptWithLabel. ???

~~~
encrypted_disclosure = EncryptWithLabel(public_key, label, context, plaintext)

disclosure = DecryptWithLabel(private_key, label, context, kem_output, ciphertext)
~~~


# Security Considerations

This mechanism could

If the same signature keys are used across groups, a former member colluding


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
