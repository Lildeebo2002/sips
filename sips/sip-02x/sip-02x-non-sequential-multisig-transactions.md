# Preamble

SIP Number: 02x

Title: Non-sequential Multisig Transactions

Authors: Jeff Bencin <jbencin@hiro.so>, Vlad Bespalov <vlad@asigna.io>

Consideration: Technical

Type: Consensus

Status: Draft

Created: 2023-08-30

License: BSD 2-Clause

Sign-off:

Discussions-To: https://github.com/stacksgov/sips

# Abstract

This SIP proposes a new multisig transaction format which is intended to be easier to use than the current format described in SIP-005.
It does not remove support for the current format, rather it is intended to co-exist with the old format and give users a choice of which format to use.

The issue with the current format is that it establishes a signer order when funds are sent to multisig account address, and requires signers to sign in the same order to spend the funds.
In practice, the current format has proven difficult to understand and implement, as evidenced by the lack of Stacks multisig implementations today.

This new format intends to simplify the signing algorithm and remove the requirement for in-order signing, without comprimising on security or increasing transaction size.
It is expected that this will lead to better wallet support for Stacks multisig transactions.

# Introduction

Currently, a multisig transaction requires the first signer to sign the transaction itself, and subsequent signers to sign the signature of the previous signer.
For a transaction with *n* signers, the final signature is generated in the following way:

```
signature_n(...(signature_2(signature_1(tx))))
```

There are a few drawbacks to doing it this way:

- The order in which the signers must sign is fixed as soon as funds are sent to a multisig account, which limits flexibility when creating a spending transaction from a multisig account
- The process of signing a transaction requires each signer to validate the entire signature chain before signing, in order to make sure it matches the transaction, leading to `O(n^2)` signing times
- This does not reduce the size of a transaction, as each intermediate signature must still be included
- The algorithm for doing this is complex, and several developers have a hard time understanding and implementing it correctly

This document proposes having each signer sign the transaction directly:

```
signature_1(tx), signature_2(tx), ..., signature_n(tx)
```

This would address all of the concerns listed above, and would not increase transaction size or make it easier to forge a signature

## Examples

Imagine a DAO that has a management team comprised of five members.
They create a 3 out of 5 multisig account on Stacks.
The existing multisig standard mandates that all transactions from this account be signed in an order which is established upon account creation.
The ordering requirement creates a hierarchy where keys near the start of the sequence have more flexibility than those near the end. 
To illustrate some of the limitations this creates:

- In a scenario where the 1st member initiates a transaction and the 4th signs it, it prohibits the 2nd and 3rd members from signing. The responsibility then falls solely on the 5th member to finalize the transaction.
- Once the 5th member has signed a transaction, no further signatures are possible.
- If the 3rd member initiates a transaction, only the 4th and 5th members are eligible to provide subsequent signatures.
- Initiating a transaction by the 4th or 5th member is impossible, as there are insufficient subsequent members to complete the signing process.

While such a multisig setup might suffice for smaller teams, as the number of required signers increases, it becomes increasingly difficult to create a transaction. This SIP aims to remove these limitations.

# Specification

This section should be interpreted as a patch to the existing "[Specification](https://github.com/stacksgov/sips/blob/main/sips/sip-005/sip-005-blocks-and-transactions.md#specification)" section of SIP-005.
For anything not mentioned here, the rules from SIP-005 still apply.

## Transactions

### Transaction Encoding

#### Transaction Authorization

Add new hash modes `0x05` and `0x07`. These numbers were chosen in order to maintain the following relationships:
 - `is_multisig = hash_mode & 0x1`
 - `is_p2wsh_p2sh = hash_mode & 0x2`
 - `is_non_sequential_multisig = hash_mode & 0x4`

| Hash mode | Spending Condition | Mainnet version | Hash algorithm |
| --------- | ------------------ | --------------- | -------------- |
| `0x00` | Single-signature | 22 | Bitcoin P2PKH |
| `0x01` | Multi-signature | 20 | Bitcoin redeem script P2SH |
| `0x02` | Single-signature | 20 | Bitcoin P2WPK-P2SH |
| `0x03` | Multi-signature | 20 | Bitcoin P2WSH-P2SH |
| `0x05` | Non-sequential multi-signature | 20 | Bitcoin redeem script P2SH |
| `0x07` | Non-sequential multi-signature | 20 | Bitcoin P2WSH-P2SH |

#### Transaction Signing and Verifying

The steps for signing a non-sequential multisig transaction (hash modes `0x05` and `0x07`) shall be as follows:

0. Set the spending condition address, and optionally, its signature count.
1. Clear the other spending condition fields, using the appropriate algorithm below.
   If this is a sponsored transaction, and the signer is the origin, then set the sponsor spending condition
   to the "signing sentinel" value (see below).
2. Serialize the transaction into a byte sequence, and hash it to form an
   initial `sighash`.
3. Calculate the `presign-sighash` over the `sighash` by hashing the
   `sighash` with the authorization type byte (0x04 or 0x05), the fee (as an 8-byte big-endian value),
   and the nonce (as an 8-byte big-endian value).
4. Calculate the ECDSA signature over the `presign-sighash` by treating this
   hash as the message digest.  Note that the signature must be a `libsecp256k1`
   recoverable signature. Store the message signature and public key encoding byte as a signature auth field.
5. Repeat step 4 until the signer threshold is reached.

The steps for verifying a non-sequential multisig transaction (hash modes `0x05` and `0x07`) shall be as follows:

0. Extract the public key(s) and signature(s) from the spending condition.
1. Clear the spending condition.
2. Serialize the transaction into a byte sequence, and hash it to form an
   initial `sighash`.
3. Calculate the `presign-sighash` from the `sighash`, authorization type byte,
   fee, and nonce.
4. Use the `presign-sighash` and the next (public key encoding byte,
   ECDSA recoverable signature) pair to recover the public key that generated it.
6. Repeat step 4 for each signature, so that all of the public keys are
   recovered.
7. Verify that the sequence of public keys hash to the address, using
   the address's indicated public key hashing algorithm.

#### Additional Recommendations

While this SIP allows signers to sign in any order, the ordering of public keys in the transaction auth fields still affects multisig account address generation.
When funding a multisig account or creating a transaction, it is strongly recommended, but not required, to order public keys from least to greatest (equivalant to lexographically sorting the hex-encoded strings).
This will remove the requirement to remember key order and result in consistent address generation.

# Related Work

[PR #139](https://github.com/stacksgov/sips/pull/139): This draft SIP was created earlier but lacked the technical specifications for implementation. The author has since closed this PR in favor of this draft

# Layer

Consensus (hard fork)

# Requires

[SIP-005](https://github.com/stacksgov/sips/blob/main/sips/sip-005/sip-005-blocks-and-transactions.md)

# Backwards Compatibility

The Stacks Blockchain will continue to treat multisig transactions using the current format as valid.
Existing multisig accounts will be able to use the new transaction types to spend previously recieved funds.

# Activation

This SIP will activate with Stacks epoch 3.0

# Reference Implementations

To be implemented in Rust. See https://github.com/stacks-network/stacks-blockchain/pull/3710.