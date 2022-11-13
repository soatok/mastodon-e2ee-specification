# End-to-End Encryption for Mastodon

> **Note:** This document is _very much_ a work in progress.
> Soatok will remove this notice when the specification is ready to review.
> Until then, enjoy seeing my rough drafts evolve into something sane.

## Background

As a federated system, users may expect that their direct messages between users would be encrypted such that only they
and the people they tag can read the conversation. However, no such end-to-end encryption is currently implemented in 
Mastodon's direct messages.

There have been [prior attempts to deliver E2EE in Mastodon](https://github.com/mastodon/mastodon/pull/13820), but they
were not spearheaded by cryptography experts. The linked pull request proposed an HMAC-SHA256 _over the plaintext_ to 
achieve message franking. This was also implemented entirely in Ruby, which means that the Mastodon server administrator
could read everything... thereby failing to achieve the _end-to-end_ part of end-to-end encryption.

## Executive Summary

We propose an architecture that's predicated on key material never being revealed to the server. To this end, we begin
with a proposal for Client-Side Key Management for Mastodon Web, Mastodon Android, and Mastodon iOS.

With a reasonable proposal for securing users' secret key material in play, propose a Federated Public Key 
Infrastructure.

Next, we specify an Asynchronous Forward-Secure Ratcheting Protocol for negotiating the symmetric keys needed to encrypt
messages between Mastodon users.

Finally, we propose a Symmetric-Key Message Encryption Format that is fast, secure, and resilient against multi-key
attacks (see: [Invisible Salamanders](https://eprint.iacr.org/2019/016)).

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

## Components

* [Client-Side Key Management](components/client-side-key-management.md)
* [Federated Public Key Infrastructure](components/federated-pki.md)
* [Asynchronous Forward-Secure Ratcheting Protocol](components/async-forward-secure-ratcheting.md)
* [Symmetric-Key Message Encryption Format](components/symmetric-key-encryption-format.md)

## How the Components Will Fit Together

The client-side key management is solely concerned with managing users' secret keys across devices.

## Structure of the Deliverables

### Mastodon-Android

* External Libraries
  * Client-Side Key Management:
  * Interacting with the Federated Public Key Infrastructure:
  * Asynchronous Forward-Secure Ratcheting Protocol:
  * Symmetric-Key Message Encryption Format:
* New classes/functions:
  * ...

### Mastodon-iOS


* External Libraries
  * Client-Side Key Management:
  * Interacting with the Federated Public Key Infrastructure:
  * Asynchronous Forward-Secure Ratcheting Protocol:
  * Symmetric-Key Message Encryption Format:
* New classes/functions:
  * ...

### Mastodon (Web)

* External Libraries
  * Client-Side Key Management:
  * Interacting with the Federated Public Key Infrastructure:
  * Asynchronous Forward-Secure Ratcheting Protocol:
  * Symmetric-Key Message Encryption Format:
* New classes/functions:
  * ...

## Project Plan

This is the plan (as of 2022-11-13) for this project. 

1. Write a design document containing the scope of work, threat models, and a formal specification
   of every component and the overall architecture. **(WIP)**
2. Review these designs with peers from the cryptography community. (2022-12-xx?)
3. Share this design document with the Mastodon community for their consideration. (2022-12-xx?)
4. Investigate formal methods and symbolic model checking before we even implementation. (2023-01-xx?)
5. Implement in JavaScript for inclusion in Mastodon for Web. (2023-02-xx?)
6. Create a Pull Request for Mastodon on GitHub to implement the necessary parts. (2023-04-xx?) 
7. Implement in Kotlin for inclusion in Mastodon for Android. (2023-05-xx?)
8. Implement in Swift for inclusion in Mastodon for iOS. (2023-05-xx?)
9. Beta test the implementations in a limited environment; engage the security community for feedback. (2023-06-xx?)
10. Launch in the next major version of Mastodon. (2023-08-xx?)
11. Investigate improvement plans (i.e. post-quantum cryptography). (2024 and beyond)
