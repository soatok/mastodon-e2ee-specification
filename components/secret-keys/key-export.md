# Exporting Keys to Multiple Devices

## Overview

This document describes the protocols used to export keys generated on one device onto another device.

At a high level overview, there are three underlying technologies used to perform this task.

1. QR codes for encoding data in a visual format that mobile devices can easily scan
2. An encrypted token format ([PASETO](https://github.com/paseto-standard/paseto-spec))
3. An asymmetric key-wrapping format ([PASERK](https://github.com/paseto-standard/paserk))

Three distinct workflows will be possible with these components:

1. Copying a cryptographic secret from a mobile device to a browser extension
2. Copying a cryptographic secret from a browser extension to a mobile device
3. Copying a cryptographic secret from one mobile device to another

Of these three workflows, the first is the riskiest, because the data must transfer from the device to the browser
extension; which, for usability, calls for passing an encrypted secret to the server and decrypting it in the browser.
This is because calling the web browser on a particular computer from the mobile device may be difficult in many
networking environments.

Workflows 2 and 3 can be facilitated with only a QR code; no server needed.

## Workflows

### Workflow 1

This is exclusively used for devices that do not possess a camera (i.e. browser extensions for desktop computers).

1. **Receiver**: Generate an Ephemeral Ed25519 keypair: `ed25519_sk`, `ed25519_pk`
2. **Receiver**: Call [`CreateQRCodeForImport(ed25519_pk)`](#createqrcodeforimport)
3. **Receiver**: Display QR code, begin polling server for message from Sender
4. **Sender**: Scan QR code (from step 3)
5. **Sender**: Call [`ExportMainKey(main_key, ed25519_pk, device_name = "")`](#exportmainkey)
6. **Sender**: Send the exported key to the server to relay to Receiver
7. **Receiver**: Call [`ImportMainKey(paseto, ed25519_sk, device_name = "")`](#unsealmainkey)

### Workflows 2 and 3

This is the preferred workflow, since the server isn't involved.

1. **Sender**: Prompt user for a password or PIN for the transfer 
2. **Sender**: Call [`GenerateQRCodeForExport(main_key, password, device_name = """)`](#createqrcodeforexport)
3. **Sender**: Display QR code
4. **Receiver**: Scan QR code (from step 3)
5. **Receiver**: Call [`UnwrapMainKeyFromQRCode(paseto, password, device_name = "")](#unwrapmainkeyfromqrcode)

## Operations

### CreateQRCodeForImport

This is used for [Workflow 1](#workflow-1). This will run on the device that receives the key.

**Inputs**:

1. `ed25519_pk`: An ephemeral Ed25519 public key

**Algorithm**:

1. Encode the public key `ed25519_pk` using the
   [PASERK `public`](https://github.com/paseto-standard/paserk/blob/master/types/public.md) type.

### ExportMainKey

This is used for [Workflow 1](#workflow-1). This will run on the device that sends the key.

**Inputs**:

1. `main_key` - The key being exported for a given account
    * This **MUST** be base64url-encoded (as per RFC 4648). It **SHOULD NOT** be padded with `=` characters.
2. `ed25519_pk` - From `CreateQRCodeForImport()`
3. `device_key` - The nickname of the device (optional, defaults to empty string)

**Algorithm**:

1. Generate a one-time 256-bit transfer key `transfer_key`.
2. Use PASERK's [`seal`](https://github.com/paseto-standard/paserk/blob/master/types/seal.md) type to encrypt
   `transfer_key` with `ed25519_pk` (Input 2) to yield `sealed_transfer_key`.
3.  Generate a Local PASETO token using `transfer_key` with the following parameters:
* `payload`:
    * `main_key`: Input 1
* `footer`:
    * `wpk`: `sealed_transfer_key`
* `implicit_assertion`:
    * `device`: Input 3
4. Encode the PASETO token (output of step 3) as a QR code image to be scanned with a mobile device.

The permitted algorithms for this operation (as of the time of this writing) are:

* PASETO `v3.local`
* PASETO `v4.local` (default token format)
* PASERK `k3.seal`
* PASERK `k4.seal` (default public-key encryption for key-wrapping format)

### UnsealMainKey

This is used for [Workflow 1](#workflow-1). This runs on the device that receives the key.

**Inputs**:

1. `paseto` - Local PASETO with a wrapped key in the footer
2. `ed25519_sk` - Ephemeral Ed25519 secret key (64 bytes; the latter half is the public key)
3. `device_name` - The nickname of the device (optional, defaults to empty string)

**Algorithm**:

1. Extract `footer` from `paseto`.
2. Extract the `sealed_transfer_key` from the footer's `wpk` parameter. (Footers are cleartext.)
3. Use PASERK's `seal` type to decrypt `sealed_transfer_key` with `ed25519_sk` to yield `transfer_key`.
4. Decrypt `paseto` yielding `transfer_key`, using `device_name` as the value for `implicit_assertion`.
5. Decode the `main_key` claim (base64url from RFC 4648). Return this.

**Output**: A 256-bit `main_key`.

### CreateQRCodeForExport

This is used for [Workflows 2 and 3](#workflows-2-and-3). This runs on the device that sends the key.

**Inputs**:
1. `main_key` - The key being exported for a given account
    * This **MUST** be base64url-encoded (as per RFC 4648). It **SHOULD NOT** be padded with `=` characters.
2. `password` - Used to encrypt the secret key across devices
3. `device_name` - The nickname of the device (optional, defaults to empty string)

**Algorithm**:

1. Generate a one-time 256-bit transfer key `transfer_key`.
2. Use PASERK's [`local-pw`](https://github.com/paseto-standard/paserk/blob/master/types/local.md) type to encrypt
   `transfer_key` with the `password` (Input 2) to yield `wrapped_transfer_key`.
3. Generate a Local PASETO token using `transfer_key` with the following parameters:
    * `payload`:
        * `main_key`: Input 1
    * `footer`:
        * `wpk`: `wrapped_transfer_key`
    * `implicit_assertion`:
        * `device`: Input 3
4. Encode the PASETO token (output of step 3) as a QR code image to be scanned with a mobile device.

The permitted algorithms for this operation (as of the time of this writing) are:

* PASETO `v3.local`
* PASETO `v4.local` (default token format)
* PASERK `k3.local-pw`
* PASERK `k4.local-pw` (default password-based key-wrapping format)

### UnwrapMainKeyFromQRCode

This is used for [Workflows 2 and 3](#workflows-2-and-3). This will run on the device that receives the key.

**Inputs**:

1. `paseto` - Local PASETO with a wrapped key in the footer
2. `password` - One-time password or PIN provided by the user for this transfer
3. `device_name` - The nickname of the device (optional, defaults to empty string)

**Algorithm**:

1. Extract `footer` from `paseto`.
2. Extract the `wrapped_transfer_key` from the footer's `wpk` parameter. (Footers are cleartext.)
3. Use PASERK's `local-pw` type to decrypt `wrapped_transfer_key` with `password` to yield `transfer_key`.
4. Decrypt `paseto` yielding `transfer_key`, using `device_name` as the value for `implicit_assertion`.
5. Decode the `main_key` claim (base64url from RFC 4648). Return this.

**Output**: A 256-bit `main_key`.

## Questions and Answers

### Why PASETO/PASERK Instead of JWT?

As far as cryptographic token designs go, PASETO was designed to be misuse-resistant. JWT was not.

* PASETO doesn't give you an `{"alg":"none"}` bullet to fire into your foot. JWT does.
* PASETO doesn't let you use an asymmetric public key as a symmetric key. JWT does.
* PASETO doesn't let you use weak keys for symmetric authentication. JWT does.
* Many JWT implementations let you bypass their security hardening against algorithm confusion if you change the key ID
  header. PASETO prohibits this attack and provides [test vectors that **MUST** fail](https://github.com/paseto-standard/test-vectors).
* PASETO is simple for users. You don't have to reason about block cipher modes and their security implications. Your
  choices are simply:
  1. Do I need to use NIST-approved algorithms (i.e. my OS is in FIPS mode)?
    * YES: `v3`
    * NO: `v4` (default)
  2. Do I need a separation of capabilities between token creators and token verifiers?
    * YES: `public`
    * NO: `local`
* PASETO publishes a [clear rationale for its design decisions with new protocol versions](https://github.com/paseto-standard/paseto-spec/blob/master/docs/Rationale-V3-V4.md).
  JWT is an "Everything But the Kitchen Sink" standard whose rationale is unclear.

For balance, the advantages of JWT are:

1. There's an IETF RFC for JWT, and not for PASETO; which matters to some organizations
2. You can find more JWT implementations in more languages than PASETO

PASERK is a PASETO extension that provides a lot of features that were excluded from PASETO. PASERK support is, for most
PASETO-supporting applications, completely optional. This lets PASETO users minimize their attack surface.

To support the features we need, we're including these Types from the PASERK design too:

1. [Password-based key-wrapping](https://github.com/paseto-standard/paserk/blob/master/types/local-pw.md)
2. [Asymmetric (public-key) key-wrapping](https://github.com/paseto-standard/paserk/blob/master/types/seal.md)

This saves us from having to design a new key-encryption scheme.

### Why Use Asymmetric (Public-Key) Encryption for Workflow 1?

Without a QR code, we need to use the Fediverse server as a data mule for the import workflow. We assumed in
[our threat model](../client-side-key-management.md#threat-model) that server administrators are curious.
Therefore, we encrypt this data with a one-time Ed25519 keypair to ensure their curiosity remains unsated.

### Why Use Password-Based Key Wrapping for Workflows 2 and 3?

If someone shoulder-surfs your QR code, this shouldn't immediately give them access to your main key.

This is a defense-in-depth risk mitigation. Do not rely entirely on the security of this password. If you
suspect someone copied your QR code for the device transfer, main key rotation is highly recommended.
