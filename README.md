# End-to-End Encryption for Mastodon

> **Note:** This document is _very much_ a work in progress.
> Soatok will remove this notice when the specification is ready to review.
> Until then, enjoy seeing my rough drafts evolve into something sane.

## This is NOT ready for your input or feedback yet.

Please watch the repository for updates. It should be ready for feedback in 2023.

Do not ask about sections that have not yet been written, or advocate for specific protocols to be adopted. For all you
know, that's exactly what I was planning to do anyway.

You should assume everything will be rewritten multiple times before I'm happy with it. **Mastodon developers should
especially avoid poisoning their perceptions by early drafts.**

## Background

As a federated system, users may expect that their direct messages between users would be encrypted such that only they
and the people they tag can read the conversation. However, no such end-to-end encryption is currently implemented in 
Mastodon's direct messages.

There have been [prior attempts to deliver E2EE in Mastodon](https://github.com/mastodon/mastodon/pull/13820), but this
work is incomplete (i.e. only implemented in the server-side Ruby code), and was not spearheaded by cryptography experts.
To wit: The linked pull request proposed an HMAC-SHA256 _over the plaintext_ to achieve message franking, when there are 
better ways to achieve these security goals without risking confidentiality.

As cryptographers and security engineers, we aim to deliver end-to-end encryption to the Fediverse through ActivityPub,
so that we might communicate privately when using Mastodon. Thus, while the goal is E2EE DMs in Mastodon, the scope will
be appropriately broad in some areas.

## Executive Summary

To achieve our end goal of end-to-end encryption (E2EE) for Direct Messages in Mastodon (and other ActivityPub
software), we will divide this work into four distinct problem domains and then define how they are unified together.

We propose an architecture that's predicated on key material never being revealed to the server. To this end, we begin
with a proposal for Client-Side Key Management. This will be realized through multiple clients (iOS, Android, Desktop)
as well as a Browser Extension, but not through in-browser JavaScript delivered from a website.

With a reasonable proposal for securing users' secret key material in play, propose a Federated Public Key 
Infrastructure. This allows users to fetch each other's public keys with some assurance that it's the correct public key
for their recipient.

Next, we specify an Asynchronous Forward-Secure Ratcheting Protocol for negotiating the symmetric keys needed to encrypt
messages between users. We focus on group messaging as a first-class feature, with authenticated group actions that do
not depend on a trusted server.

Finally, we propose a Symmetric-Key Message Encryption Format that is fast, secure, and resilient against multi-key
attacks (see: [Invisible Salamanders](https://eprint.iacr.org/2019/016)). We define both message encryption and media
encryption in this level.

The primary output of the Secret Key Management component is a Public Key, which is passed to the Federated Public Key
Infrastructure. Group Messaging uses these Public Keys for group membership verification as it operates a forward-secure
ratchet. This ratcheting protocol produces symmetric keys for encrypting messages and media attachments.

With these pieces in place, we can deliver secure end-to-end encryption to the fediverse. Security experts can focus on
the components independently, or in conjunction with each other.

## Design Tenets

1. **Usability**. The implementation must be both **easy to use** and **difficult to misuse**. 
   * The details of the cryptography should be almost entirely invisible to the end user.
   * Nobody should care about exotic ciphersuites or funky AES modes.
2. **Minimize Agility.** Each version of our protocol will contain a single set of ciphers and modes
   that are permitted.
3. **Minimize Complexity.** Every complication must be justified by a security goal. 
   * Eschew X.509, ASN.1, etc. in favor of simpler binary formats, where possible.
4. **Use State-of-the-Art Cryptography.** We're writing this in the later months of 2022, not the 1990's.
   We don't need to be backwards compatible with legacy formats. Absolutely no RSA, AES-CBC, etc.

### Anti-Tenets

1. **Interoperability with Legacy.** We aren't interested in our proposal interoperating with OpenPGP,
   Matrix, etc.
   * If someone else wants to build something interoperable with our cryptography, that's okay. But we're not aiming
     to support existing designs.
2. **Competing with Secure Messaging apps.** We aren't building a replacement for Signal. We just want direct messages
   to remain confidential.
3. **Deniability and/or Anonymity.** We cannot hide the social graph from ActivityPub, nor escape the use of
   [HTTP Signatures](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-message-signatures-13).
   Given the environment we're operating under, we cannot reasonably tackle these security properties at this time.
4. **Government Compliance.** We aren't selling anything to a government, nor to corporations that sell to governments.
   There's no sense in catering to their lists of approved cryptographic algorithms.

### Security Goals

1. **Confidentiality**. Direct Messages are currently shared between instances such that they are encrypted in transit
   (provided by TLS). However, the contents of the messages are not end-to-end encrypted, so a curious instance admin
   may read their users' DMs. We want to provide confidentiality between all users.
2. **Integrity**. Direct Messages that use end-to-end encryption should be tamper-resistant for all participants in
   a conversation.
3. **Authenticity**. All participants in a group should be able to prove who sent a given encrypted message.
4. **Abuse-Resistant**. It should be possible to report abusive messages to an instance admin.
   * Instead of message franking, we will opt for committing AEAD constructions.
5. **Domain Separation**. Every use of a hash function or KDF will be isolated through domain separation constants to
   prevent cryptographic outputs from being accepted in an inappropriate context.
6. **Protocol Lucidity**. All encrypted messages will be bound to a given context, to avoid Confused Deputy attacks.
   We will use versioned protocols instead of in-band negotiation, to avoid Algorithm Confusion attacks.
7. **Negative Known Answer Tests**. Every security property **SHOULD** be accompanied by a Known Answer Test (also
   sometimes called a Test Vector) that is intended to fail, so that implementations can be secure by design.

## Components

* [Client-Side Key Management](components/client-side-key-management.md)
* [Federated Public Key Infrastructure](components/federated-pki.md)
* [Asynchronous Forward-Secure Ratcheting Protocol](components/async-forward-secure-ratcheting.md)
* [Symmetric-Key Message Encryption Format](components/symmetric-key-encryption-format.md)

## How the Components Will Fit Together

The client-side key management is solely concerned with managing users' secret keys across devices.

E2EE-capable clients will output an Ed25519 public key, which will be used with the Federated Public Key Infrastructure
(FPKI).

When a user wants to send an encrypted message to another user, they will first fetch their public key from the fPKI.
This public key will allow them to verify the protocol messages sent by the asynchronous forward-secure ratcheting
protocol.

Finally, each conversation will involve messages and media encrypted using the keys derived from the ratcheting 
protocol. This is the only part that interacts with user-generated content (rather than device-generated entropy).

## Structure of the Deliverables

### Kotlin Libraries

* Client-Side Key Management:
* Interacting with the Federated Public Key Infrastructure:
* Asynchronous Forward-Secure Ratcheting Protocol:
* Symmetric-Key Message Encryption Format:

### Ruby Libraries

* Interacting with the Federated Public Key Infrastructure:

### Swift Libraries

* Client-Side Key Management:
* Interacting with the Federated Public Key Infrastructure:
* Asynchronous Forward-Secure Ratcheting Protocol:
* Symmetric-Key Message Encryption Format:

### TypeScript Libraries

* Client-Side Key Management:
* Interacting with the Federated Public Key Infrastructure:
* Asynchronous Forward-Secure Ratcheting Protocol:
* Symmetric-Key Message Encryption Format:

### Mastodon-Android

* See: [Kotlin libraries](#kotlin-libraries)
* New classes/functions:
  * ...

### Mastodon-iOS

* See: [Swift libraries](#swift-libraries)
* New classes/functions:
  * ...

### Mastodon (Web Server)

* See: [Ruby libraries](#ruby-libraries)
* New classes/functions:
  * ...

### Browser Extension

* See: [TypeScript libraries](#typescript-libraries)

## Project Plan

This is the plan (as of 2022-12-21) for this project. 

1. Write a design document containing the scope of work, threat models, and a formal specification
   of every component and the overall architecture. **(WIP)**
2. Review these designs with peers from the cryptography community. (2023-01-xx?)
3. Share this design document with the Mastodon community for their consideration. (2023-01-xx?)
4. Investigate formal methods and symbolic model checking before we even implementation. (2023-01-xx?)
5. Implement in JavaScript for inclusion in Mastodon for Web. (2023-02-xx?)
6. Create a Pull Request for Mastodon on GitHub to implement the necessary parts. (2023-04-xx?) 
7. Implement in Kotlin for inclusion in Mastodon for Android. (2023-05-xx?)
8. Implement in Swift for inclusion in Mastodon for iOS. (2023-05-xx?)
9. Beta test the implementations in a limited environment; engage the security community for feedback. (2023-06-xx?)
10. Launch in the next major version of Mastodon. (2023-08-xx?)
11. Investigate improvement plans (i.e. post-quantum cryptography). (2024 and beyond)
