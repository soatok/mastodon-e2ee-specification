# Passphrase-Protected Key Wrapping

> Parent Document: [Client-Side Key Management](../client-side-key-management.md)

## Overview

Passphrase-protected key backups are required, but may be opted into by users that are confident in the security of 
their passphrases (also known as "passwords" colloquially). A password manager (such as 1Password or KeePassXC) is 
highly recommended for most users that elect this key recovery mechanism.

For simplicity, we are going to implement Draft 09 or later of the 
[OPAQUE RFC](https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/09/). OPAQUE will be used to store and recover
the **export_key**, which will then be used to encrypt the **Main Key**.

## Operations

### BackupMainKey

Refer to OPAQUE RFC sections:

* Section 3.3
* Section 6

**Inputs**:

* `main_key` - The key being backed up
* `passphrase` - Strong, unique passphrase
* `server_public_key` - Server's OPAQUE public key
* `server_identity` - Domain name
* `client_identity` - Fediverse username and domain name, with `@` prefix (e.g. `@soatok@furry.engineer`)

**Outputs**:

* `envelope` - User's Envelope structure as per the OPAQUE RFC
* `encrypted_main_key` - User's **Main Key**, encrypted with the OPAQUE export key
* `client_public_key` - User's AKE public key
* `masking_key` -  an encryption key used by the server with the sole purpose of defending against client enumeration
   attacks

**Pseudocode**:

```python
def BackupMainKey(main_key, passphrase, server_public_key, server_identity, client_identity):
    # From Section 4.1.2 of IETF OPAQUE RFC draft 09:
    (envelope, client_public_key, masking_key, export_key) = Store(
        passphrase,
        server_public_key,
        server_identity,
        client_identity
    )
    
    encrypted_main_key = EncryptMainKey(export_key, main_key)
    return (envelope, encrypted_main_key, client_public_key, masking_key)
```

### RecoverMainKey

Refer to OPAQUE RFC sections:

* Section 4.1
* Section 6

**Inputs**:

* `encrypted_main_key` - User's **Main Key**, encrypted with the OPAQUE export key
* `passphrase` - Strong, unique passphrase
* `server_public_key` - Server's OPAQUE public key
* `envelope` - User's Envelope structure
* `server_identity` - Domain name
* `client_identity` - Fediverse username and domain name, with `@` prefix (e.g. `@soatok@furry.engineer`)

**Outputs**:

* `main_key` - The key being backed up

**Pseudocode**:

```python
def RecoverMainKey(encrypted_main_key, passphrase, server_public_key, envelope, server_identity, client_identity):
    # From Section 4.1.3 of the IETF OPAQUE RFC draft 09:
    (_, export_key) = Recover(
        passphrase,
        server_public_key,
        envelope,
        server_identity,
        client_identity
    )
    return DecryptMainKey(export_key, encrypted_main_key)
```

### EncryptMainKey

**Inputs**:

* `export_key` - OPAQUE export key 
* `main_key` - plaintext Main Key

**Outputs**:

* `encrypted_main_key`

**Algorithm**:

1. Generate a 256-bit random nonce (`N1`).
2. Derive an encryption key and nonce with HKDF-SHA512
   * IKM = export_key
   * info = `SoatokE2EEProposalV1-PBKW-Encrypt` followed by `N1`
   * salt = NULL
   * length = 44
   * The first 32 bytes of output will be the encryption key, `Ek`
   * The remaining 12 bytes of output will be the derived nonce, `N2`
3. Derive an authentication key with HKDF-SHA512
   * IKM = export_key
   * info = `SoatokE2EEProposalV1-PBKW-Auth` followed by `N1`
   * salt = NULL
   * length = 32
   * The total output will be the authentication key, `Ak`
4. Encrypt the payload using ChaCha20
   * Message = `main_key`
   * Nonce = `N2`
   * Key = `Ek`
   * The output will be called `C` (Ciphertext)
5. Authenticate `N1` and `C` using BLAKE2b-MAC
   * Message = `N1` || `C`
   * Key = `Ak`
   * The output will be called `T` (Tag)
6. Return `N1`, `C`, and `T`

**Pseudocode**:

```python
INFO_PREFIX_ENCRYPT = b"SoatokE2EEProposalV1-PBKW-Encrypt"
INFO_PREFIX_AUTH = b"SoatokE2EEProposalV1-PBKW-Auth"

def EncryptMainKey(export_key, main_key):
    n1 = os.urandom(32)
    tmp = hkdf_sha512(export_key, None, concat(INFO_PREFIX_ENCRYPT, n1), 44)
    Ek = tmp[0:31]
    n2 = tmp[32:]
    Ak = hkdf_sha512(export_key, None, concat(INFO_PREFIX_AUTH, n1), 32)
    
    c = crypto_stream_chacha20_xor(key = Ek, nonce = n2, message = main_key)
    t = crypto_generichash(key = Ak, msg = concat(n1, c), length = 32)
    return concat(n1, c, t)
```

### DecryptMainKey

**Inputs**:

* `export_key` - OPAQUE export key
* `encrypted_main_key`

**Outputs**:

* `main_key` - plaintext Main Key

**Algorithm**:

1. Split `encrypted_main_key` into `N1`, `C`, and `T`
   * The first 32 bytes are `N1`
   * The next 32 bytes are `C`
   * The final 32 bytes are `T`
2. Derive an authentication key with HKDF-SHA512
   * IKM = export_key
   * info = `SoatokE2EEProposalV1-PBKW-Auth` followed by `N1`
   * salt = NULL
   * length = 32
   * The total output will be the authentication key, `Ak`
3. Recalculate the authentication tag (`T2`) over `N1` and `C` using BLAKE2b-MAC
   * Message = `N1` || `C`
   * Key = `Ak`
   * The output will be called `T` (Tag)
4. Compare `T2` with `T` in constant-time. Abort if it doesn't match.
5. Derive an encryption key and nonce with HKDF-SHA512
   * IKM = export_key
   * info = `SoatokE2EEProposalV1-PBKW-Encrypt` followed by `N1`
   * salt = NULL
   * length = 44
   * The first 32 bytes of output will be the encryption key, `Ek`
   * The remaining 12 bytes of output will be the derived nonce, `N2`
6. Decrypt the payload using ChaCha20
   * Message = `C`
   * Nonce = `N2`
   * Key = `Ek`
7. Return the output of step 6 as `main_key`.

**Pseudocode**:

```python
INFO_PREFIX_ENCRYPT = b"SoatokE2EEProposalV1-PBKW-Encrypt"
INFO_PREFIX_AUTH = b"SoatokE2EEProposalV1-PBKW-Auth"

def DecryptMainKey(export_key, encrypted_main_key):
    n1 = encrypted_main_key[0:31] # First 32 bytes
    c = encrypted_main_key[32:63] # Next 32 bytes
    t = encrypted_main_key[64:] # Final 32 bytes
    
    Ak = hkdf_sha512(export_key, None, concat(INFO_PREFIX_AUTH, n1), 48)
    t2 = crypto_generichash(key = Ak, msg = concat(n1, c), length = 32)
    if not hmac.compare_digest(t2, t):
        raise Exception("Incorrect payload")
    
    tmp = hkdf_sha512(export_key, None, concat(INFO_PREFIX_ENCRYPT, n1), 48)
    Ek = tmp[0:31]
    n2 = tmp[32:]
    
    return crypto_stream_chacha20_xor(key = Ek, nonce = n2, message = c)
```

## Design Decisions

### OPAQUE Usage

In an earlier (unpublished) draft of this document, we sketched a protocol that looked something like this:

**Backup**:

1. Generate two random salts (S1, S2).
2. Use Argon2id with the user's passphrase (P) and S1 to derive a wrapping_key, with parameters targeting 1.0
   seconds per guess (or greater).
3. Encrypt the **Main Key** with wrapping_key to obtain the encrypted_main_key.
4. Use OPAQUE with Argon2id (with the user's passphrase (P) and S2) to encrypt S1 (envelope).
5. Store the outputs of steps 3 and 4 in the server.

**Recovery**:

1. Download encrypted_main_key and envelope from the server.
2. Use OPAQUE with Argon2id (with the user's passphrase (P) and S2) to decrypt the envelope to obtain the salt, S1.
3. Use Argon2id with the user's passphrase (P) and S1 to derive a wrapping_key.
4. Decrypt wrapping_key to obtain the **Main Key**.
5. Verify that you derive the correct public key from **Main Key** (step 4).

The intent of this design was to only trust the server with an *encrypted* salt, which would then need to be used with
the correct passphrase to successfully recover the wrapping_key that protects the **Main Key**.

However, the latest OPAQUE RFC draft exposes a consistent export_key (distinct from the randomized session_key).
This export_key is ideal for our use case, and saves us from having to use two invocations of Argon2id.

### Why Argon2 and not PBKDF2?

While it's true that [PBKDF2 is easier in JavaScript than Argon2](https://soatok.blog/2022/12/29/what-we-do-in-the-etc-shadow-cryptography-with-passwords/),
we aren't building a product to be sold nor a service to subscribe to.

If users cannot access their backed up private keys in lockdown mode, they're free to choose other backup methods.
