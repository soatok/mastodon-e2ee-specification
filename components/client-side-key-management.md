# Client-Side Key Management

## Executive Summary

One of the most difficult and important components of any cryptosystem is how secrets are managed. The difficulty of
this aspiration is made worse by four notable factors:

1. Cryptographic secrets are highly valuable, and highly sensitive.
2. Cryptographic secrets are almost unavoidably exposed to end users.
3. Limitations with browsers, frameworks, and operating systems.
4. Different users have different appetites for risk, accessibility requirements, and economic factors.

Trying to thread a needle between all four factors and arriving at One True Key Management Solution that makes everyone
happy is a fool's errand.

Instead, we'll start by defining a threat model for different types of users in a federated system. The solutions will
then address different user threat profiles.

With a clear threat model in mind, we describe our solutions at a high level.

Next, we provide technical specifications for each solution; each contained in their own document.

Finally, we offer some general User Experience recommendations for how each key management solution may be offered
to users of a federated system. This final section is non-normative, and only exists to demonstrate the authors'
intention to make cryptography usable by everyone.

## Contents

* [Threat Model](#threat-model)
* [Technical Specifications](#technical-specifications)
* [High-Level Design](#high-level-design)
* [User Experience Recommendations](#user-experience-recommendations)

## Threat Model

### Security Goals

1. **Confidentiality**. The keys managed here are the most sensitive data in this proposal.
2. **Integrity**. False keys shouldn't be injectable by any party.
3. **Availability**. A user shouldn't be locked out forever if they forget a password or
   lose their mobile device.
4. **Zero Trust**. Users shouldn't need to place their trust in a particular Fediverse instance or its
   administrators.

### User Profiles

> Note: I'm not using the standard cryptographic role names in this design, beyond Alice and Bob, because
> cryptographic threat models tended to favor unrealistic threats ("evil maid") instead of real-world problems
> ("abusive spouse"). Please forgive any confusion this causes. All names are unrelated to any humans I know.

**Alice** is a computer security expert that routinely uses a password manager, multiple hardware
security keys, two-factor authentication through TOTP on her mobile device (where FIDO2 isn't supported),
and regularly uses unique, strong passwords everywhere. Alice is a power user; she can clear technical
hurdles with ease, and has strong opinions about her workflow.

**Bob** is a software engineer. He doesn't live and breathe security, but he has the technical background
to find and follow tutorials online when he's stuck. He uses a password manager and two-factor authentication
(TOTP) through his mobile device. He may have heard about hardware security keys, but doesn't own one.

**Chloe** doesn't work in tech, but uses her computer extensively. She knows "just enough to be dangerous".
She tries to follow the security best practices, but her knowledge is out of date, and she's too busy to
relearn these topics right now. She may use a password manager, but that's not guaranteed. If she doesn't,
she relies on password reset mechanisms to get back into accounts when she forgets which password she used.

**Duncan** doesn't work in tech, nor does he use his computer extensively. If you encounter Duncan with his
hands free, chances are he'll be on his mobile device, chatting with friends and family or reading social
media. He doesn't use a password manager, and insists he's lucky to remember his 4-digit pin to unlock his
phone. If he ever owned a security device, he's certainly lost it by now.

**Eli** is trying to escape an abusive relationship, where their partner has started accusing them of
cheating and trying to assert stronger control over their life. This includes demands for physical access
to their phone to inspect message history. They want to communicate with the few friends and family members
that their partner hasn't pushed away to seek help and get out.

> Skipping a few letters in case I'm forgetting anything, because I want to maintain alphabetical order.

### Attacker Profiles

> We'll tackle this one in a different layer, but I'm sketching this one here too:
> 
> **Ivan** is an Internet troll who wants to cause havoc. If he could convince an instance admin that one of
their users is sharing unsolicited nudes with him, he would in a heartbeat, just to see them banned.

**Janet** runs a federated instance and is very curious about what her users are saying to each other.
However, whether through a lack of technical expertise or a dedication to principle, she won't tamper with
the instance to sate her curiosity.

**Lee** is curious like Janet, but they are also willing to deliver a targeted JavaScript payload to pilfer
keys if private messages were encrypted.

**Mason** is a black-hat hacker, armed with a zero-day exploit against Mastodon (or some other fediverse server
software). They want to download the entire database from targeted instances and blackmail users with embarrassing
Direct Messages and related media in exchange for cryptocurrency. They will change the code of the servers they
hack in order to pilfer any data they can get their hands on.

**Niel** is a nation-state attacker with a CA-signed certificate who passively observes all traffic to/from
Fediverse instances within their borders. Their nation's citizens are required to trust their national CA,
and they're constantly looking for evidence of citizens engaging in journalism or thought-crime.

**Ollie** is an unreliable friend. Trusting them with a secret is cheaper than airtime. They will sell you out
for a Klondike bar.

### Feature Requirements

(TODO: Expand the list later.)

#### Feature 1. Support Multiple Devices

### Threats

#### Threat 1. Forgotten Passwords

Any system that deals with human-memorable passwords will have to contend with the fact that
users forget their passwords all the time.

##### Treat 1 Mitigation

Users **MUST NOT** be required to remember a passphrase or key through the normal use of the software.

Users **MAY** be given the option to passphrase-protected their secrets as a recovery mechanism, but it **MUST NOT** be
an obstacle to normal app usage.

Users **MAY** choose an additional password or PIN for locking their encryption keys at rest. 
See [threat 4](#threat-4-domestic-abuse) below.

#### Threat 2. Lost, Damaged, or Stolen Hardware

Phones break. Phones go missing. Phones get stolen, often while unlocked.

##### Threat 2 Mitigation

Users should have a mechanism to recover from a lost, damaged, or stolen device.

#### Threat 3. Malware

Computers and phones get infected with malware all the time.

##### Threat 3 Mitigation

Users should have some mechanism for recovering from an infection and rotating their secret keys.

#### Threat 4. Domestic Abuse

An abusive partner may have direct physical access to a user's mobile device, at least some of the time.

##### Threat 4 Mitigation

Users **MAY** want to password-protect their encryption keys when not in use.

Users **MUST NOT** be required to use threshold recovery mechanisms, since the abuser may be able to bully
the user's friends or family into complying with threshold recovery ceremonies.

#### Threat 5. The Web Server / JavaScript Cryptography

Users that experience the Fediverse through the Internet will regularly re-download all of the JavaScript that the 
server software provides. This is largely opaque to most Internet users.

Consequently, no matter how well-implemented and secure your JavaScript code may be, a malicious instance admin or
black hat hacker may simply substitute it with additional code that pilfers sensitive data for them to read.

##### Threat 5 Mitigation

Users shouldn't expose their keys to the webserver. This means using end-to-end encryption outside the context of
the website. **This implies a Browser Extension.**

#### Threat 6. Compromised Social Network

Users who choose to rely on a quorum of their friends to help with a threshold recovery mechanism must contend with
the risk of their friends all being compromised by a black-hat hacker, social engineered, or bullied by an abusive
partner.

##### Threat 6 Mitigation

The threshold mechanism **MUST NOT** be required.

Additionally, an appropriate threshold **SHOULD** be used to ensure a partial compromise of one's trusted friends 
doesn't cause them to become compromised in turn.

#### Threat 7. Curious Eavesdroppers

Any sensitive information (i.e. keying material) transmitted in plaintext (even over TLS) could be snarfed up by
sufficiently motivated and resourced adversaries.

##### Threat 7 Mitigation

No information should be transmitted in plaintext between devices, even if transport-layer encryption protects 
against some attacks.

#### Threat 8. Adaptive Cryptographic Attacks

Our sophisticated attackers (**Lee**, **Mason**, and **Niel**) may wish to use an adaptive attacks against the
cryptographic algorithms used in whatever protocols we design.

* RSAES-PKCS1v15 will be targeted with Bleichenbacher'98
* RSASSA-PKCS1v15 will be targeted with Bleichenbacher'06
* AES-CBC will be targeted with Vaudenay'02
* Any password-based key derivation functions will be tested with common/weak passwords in a distributed computing 
  cluster.

##### Threat 8 Mitigation

1. Only use modern cryptography, rather than legacy algorithms (RSA) or modes (CBC).
2. For password-hashing algorithms, set the memory/time costs sufficiently high to reduce attacker advantage.
   Additionally, use libraries like zxcvbn to reduce the risk of weak passwords. Encourage (and even integrate with?)
   password management software to make this easier for users to get right.

## High-Level Design

* Client-Side Keys will consist of a 256-bit random key, called the **main key** for an account, which will be used as 
  the IKM input to HKDF for deriving multiple keys, including:
  * An Ed25519 identity key. The public key will be used for the [Federated PKI](federated-pki.md).
  * A symmetric key for wrapping one-time secret keys (for multi-device support).
* Users **SHOULD** generate their main key on a mobile device, but are not strictly required to.
  * For users that do not have a mobile device, the Browser Extension is recommended.
* Main Keys will be exportable. For example:
  * Export to another mobile device, for multi-device support.
  * Export to a Browser Extension for use with Fediverse websites.
  * Export to a desktop app (if applicable).
* Users will be able to encrypt their main key with a passphrase.
  * This passphrase-protected key **MAY** be stored in the Mastodon server.
  * This is beneficial to user profiles: **Alice**, **Bob**, **Chloe**.
* Users will be able to use a threshold scheme to share their key with a group of their most trusted friends.
  * This is beneficial to user profiles: **Bob**, **Chloe**, **Duncan**.
  * Users like **Alice** generally won't want this feature.
  * Threats like **Ollie** make this an unattractive option for many users.
* Users can, at any time, generate a new main key.
  * If they still have their previous main key, they may maintain continuity of key control.
  * If they do not, they can still re-enroll their new key, but will not have access to old messages. All existing 
    one-time keypairs will be invalidated. Additionally, their Direct Message partners will be informed of the
    rotation when they send another message.
  * This is beneficial to user profiles: **Duncan**, **Eli**.
  * Users like **Alice**, **Bob**, and (potentially) **Chloe** generally won't want to rely on this capability.

## Technical Specifications

1. [Key Generation and Derivation](secret-keys/key-generation-derivation.md)
2. [Exporting Keys to Multiple Devices](secret-keys/key-export.md)
3. [Passphrase-Protected Key Wrapping](secret-keys/passphrase-protect.md)
4. [Threshold Key Recovery](secret-keys/threshold-recovery.md)

## User Experience Recommendations

When a user onboards, they should be given multiple options for how they recover from recovery. Forcing them into one 
solution is an antipattern.

It should be acceptable to use multiple recovery mechanisms (i.e. encrypt with passphrase, keep a copy on another 
device, **AND** allow a quorum of your most friends to recover your keys) if the user so desires.

However, it should also be acceptable to opt out of recovery mechanisms entirely, for power users.

## Future Scope

Although it's not captured above, one feature I'd like to support (when browsers allow it) is the ability to encrypt
your key with a FIDO2-compliant hardware security device (e.g. Yubikey or SoloKey) instead of a password.
