# Federated E2EE Version 1

* Version Identifier: `0xE2 0xEE 0x01 0x00` (4 bytes)
* Domain Separation Prefix: `SoatokE2EEProposalV1` (UTF-8, 20 bytes)
* Cryptographic Primitives (Main Operations)
  * Digital Signature Algorithm: Ed25519
  * Key Agreement: X25519 (ECDH over Curve25519), BLAKE2b
  * Key Derivation: HKDF-HMAC-SHA512
  * Symmetric Encryption: XChaCha20-Poly1305 (with an additional HKDF for Key Commitment)
* Supported Cryptographic Primitives (Tertiary Operations; e.g. key backup):
  * Password Protection: OPAQUE-3DH with the ristretto255 group
  * Key Import/Export:
    * PASETO v4.local
    * PASERK k4.seal
