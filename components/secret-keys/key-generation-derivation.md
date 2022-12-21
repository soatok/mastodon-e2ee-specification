# Key Generation and Derivation

> Parent Document: [Client-Side Key Management](../client-side-key-management.md)

## Overview

When users enroll in end-to-end encryption with their Fediverse instance, they generate a **Main Key**.
The **Main Key** is used as the Initial Keying Material for a Key Derivation Function.

Any other keys needed for the operation of E2EE will be then be one of the following:

* Derived from the Main Key (long-lived)
* Generated randomly (ephemeral)

## Key Generation

The algorithm for generating keys (including the Main Key) will simply be reading the appropriate number of bytes from
the operating system's random number generator (e.g. `/dev/urandom` on Linux).

Some programming languages expose specific classes rather than reading from a character device. For example, Java:

```java
SecureRandom csprng = new SecureRandom();
byte[] randomBytes = new byte[32];
csprng.nextBytes(randomBytes);
```

Many programming languages wrap OpenSSL's `RAND_bytes()` or `RAND_pseudo_bytes()` APIs. However, OpenSSL's APIs aren't
[fork-safe](https://github.com/ramsey/uuid/issues/80), so we will avoid them when possible.

### Ephemeral Secret Keys

Using libsodium for X25519, the only API we need is [`crypto_box_keypair()`](https://libsodium.gitbook.io/doc/public-key_cryptography/authenticated_encryption#key-pair-generation).

In the future, we may use a hybrid KEM that consists of X25519 and Kyber768. The first 32 bytes of each key will be the
X25519 component. The remaining bytes will be the Kyber768 component.

## Key Derivation

### Ed25519 Keys

For domain separation, we will use the Domain Separation constant defined by the current protocol version
(e.g. `SoatokE2EEProposalV1`), followed by the suffix `-Ed25519SecretKey`.
See [Protocol Versions](../../protocol-versions).

First, derive the secret scalar, using HKDF-HMAC-SHA512 with the IKM set to the user's Main Key, a NULL salt, the info
string defined previously, and an output length of `32` bytes (256 bits):

```typescript
const CONSTANTS_EDDSA_SECRET_V1 = Buffer.from(
    '536f61746f6b4532454550726f706f73616c56312d456432353531395365637265744b6579',
    'hex'
)

const secretSeed = crypto.hkdf(
    'sha512',
    32,
    mainKey,
    null,
    CONSTANTS_EDDSA_SECRET_V1
)
```

You will then use this as an unclamped seed to derive an Ed25519 keypair. If you're using libsodium, you want the
[`crypto_sign_seed_keypair()`](https://libsodium.gitbook.io/doc/public-key_cryptography/public-key_signatures#key-pair-generation)
function.

**Example Output (with all-zero key)**:
```
Input key (hex):
0000000000000000000000000000000000000000000000000000000000000000

HKDF output (hex):
bda508262a31b9b31c09f77c56c85427e6ec49e9d55e4cd804963161b94e68ba

Ed25519 Secret Key (hex):
bda508262a31b9b31c09f77c56c85427e6ec49e9d55e4cd804963161b94e68ba
f858c2a5027cd872e1d2970a457adab59b1b9521afe53ca692c792ac39e0d9c7

Ed25519 Public Key (hex):
f858c2a5027cd872e1d2970a457adab59b1b9521afe53ca692c792ac39e0d9c7
```

### Wrapping Key For Ephemeral Secret Keys

For domain separation, we will use the Domain Separation constant defined by the current protocol version
(e.g. `SoatokE2EEProposalV1`), followed by the suffix `-WrappingKeyForX25519Secrets`.
See [Protocol Versions](../../protocol-versions).

Use HKDF-HMAC-SHA512 with the IKM set to the user's Main Key, a NULL salt, the info string 
defined previously, and an output length of `32` bytes (256 bits):

```typescript
const CONSTANTS_WRAPPING_SECRET_V1 = Buffer.from(
    '536f61746f6b4532454550726f706f73616c56312d5772617070696e674b6579466f7258323535313953656372657473',
    'hex'
)

const wrappingKey = crypto.hkdf(
    'sha512',
    32,
    mainKey,
    null,
    CONSTANTS_WRAPPING_SECRET_V1
)
```

**Example Output**:

```
Input key (hex):
0000000000000000000000000000000000000000000000000000000000000000

Output key (hex):
740a2ba77f32357df7cba231c4f275c77e84804304223da9426c7dfdb88f4df1
```

## Key Rotation

There are two scenarios in which a user might rotate their **Main Key**:

1. They believe one of their devices is compromised. For example, if their mobile device is stolen. 
2. They lost access to their old keying material. For example, if their mobile device is lost or damaged.

In scenario 1, identity continuity can be maintained: Use the old Ed25519 Secret Key to sign the new Ed25519 Public Key.
This will then be published through the [Federated PKI component](../federated-pki.md). Any existing ephemeral secrets
will be wiped and fresh ones will be generated.

In scenario 2, there will be no continuity of identity, which means all participants in the network **MUST** be alerted
to the key rotation. The new Public Key will be published through the [Federated PKI component](../federated-pki.md)
without a signature from the old key.

In both scenarios, users will have to re-import the new Main Key to their other devices (if applicable).

## Future Scope

### Post-Quantum Cryptography

In a future version of this document, we may opt to include the derivation of post-quantum algorithms. Generally, our
strategy will be as follows:

1. Use HKDF-HMAC-SHA512 with a *distinct* `info` parameter to generate a PRNG seed
2. Use the PRNG seed to generate both the secret key and public key for each PQ algorithm

This allows use to continue to leverage a single **Main Key**.

## Questions and Answers

### Why Unsalted HKDF?

We had briefly considered using the federated identity `@username@domain` as an HKDF salt for key derivation, but two
problems arose with this model:

1. If you move servers, key derivation will suddenly produce different keys without an explicit rotation step.
2. If you move servers, you now have multiple salts for a given (IKM, info) tuple, which violates the HKDF security
   definition and downgrades us to mere PRF security. [Read more here](https://soatok.blog/2021/11/17/understanding-hkdf/).

This introduces both operational pain and makes security proofs harder, for no real benefit. 

The input key is a 256-bit secret key. The various KDF outputs are just to allow one compact 256-bit value to be
expanded into multiple related keys when necessary.
