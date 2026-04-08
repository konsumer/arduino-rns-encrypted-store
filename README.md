# arduino-rns-encrypted-store

Authenticated encrypted file storage for [microReticulum](https://github.com/attermann/microReticulum) messages on Arduino/ESP32 with an SD card.

Encrypts a message blob using an identity's private key and writes it to a file. Reading back requires the same identity. Designed to protect sent/received Reticulum messages at rest — announcements and peer records are public and do not need this treatment.

Use [arduino-rns-password](https://github.com/konsumer/arduino-rns-password) to load the identity's private key from SD before calling these functions. See [arduino-rns-identity-vault](https://github.com/konsumer/arduino-rns-identity-vault) for a complete working example.

## Installation (PlatformIO)

Add to `lib_deps` in your `platformio.ini`:

```ini
lib_deps =
    https://github.com/konsumer/arduino-rns-encrypted-store.git
    https://github.com/attermann/Crypto.git
    https://github.com/attermann/microReticulum.git
```

## API

```cpp
#include "encrypted_store.h"

// Encrypt `data` (len bytes) and write to `path` using `identity`'s key.
// Returns true on success.
bool encstore_write(const char* path, const RNS::Identity& identity,
                    const uint8_t* data, size_t len);

// Decrypt the file at `path` into `data` (must be encstore_size(path) bytes).
// Returns true on success.
// Returns false if the file is missing, wrong size, or authentication fails
// (wrong identity or corrupted file). No plaintext is written on failure.
bool encstore_read(const char* path, const RNS::Identity& identity,
                   uint8_t* data, size_t len);

// Returns the plaintext byte count stored at `path` (file_size - 49),
// or 0 if the file does not exist or is too small to be valid.
size_t encstore_size(const char* path);
```

The identity must have its **private key loaded** before calling these functions.

## Usage

```cpp
#include "encrypted_store.h"

// After loading identity via arduino-rns-password ...

// Store an encrypted message
const char* msg = "hello reticulum";
encstore_write("/msgs/001.enc", identity,
               (const uint8_t*)msg, strlen(msg));

// Read it back
size_t sz = encstore_size("/msgs/001.enc");
uint8_t* buf = (uint8_t*)malloc(sz + 1);
if (encstore_read("/msgs/001.enc", identity, buf, sz)) {
    buf[sz] = '\0';
    Serial.println((char*)buf);
}
free(buf);
```

## Typical flow

```
password_open("/identity.bin", pw, prv, 64)   ← unlock identity from SD
  └─ identity.load_private_key(prv)            ← restore into RNS::Identity
       └─ encstore_read/write(...)             ← use identity to protect messages
```

## Security design

### Key derivation — HKDF-SHA256

Two independent 32-byte keys are derived from the identity's X25519 encryption private key using HKDF-SHA256 (the same key derivation primitive Reticulum uses for packet encryption):

- **keys[0:32]** → AES-256-CTR encryption key
- **keys[32:64]** → HMAC-SHA256 authentication key

The identity's **hash** is used as the HKDF salt, which binds the derived keys to this specific identity.

### Authenticated encryption

The HMAC-SHA256 tag covers the version byte, IV, and ciphertext. It is verified with a **constant-time comparison** before any decryption happens. A wrong identity, corrupted file, or tampered ciphertext all fail here. No plaintext is ever produced from an unauthenticated file.

### File format

```
┌──────────┬──────────┬──────────────┬─────────────────┐
│ version  │ IV       │ ciphertext   │ HMAC-SHA256     │
│ 1 byte   │ 16 bytes │ N bytes      │ 32 bytes        │
└──────────┴──────────┴──────────────┴─────────────────┘
Total overhead: 49 bytes beyond the plaintext.
```

## Dependencies

- [attermann/microReticulum](https://github.com/attermann/microReticulum) — `Identity`, `Cryptography::hkdf`, `Cryptography::HMAC`, `Cryptography::random`
- [attermann/Crypto](https://github.com/attermann/Crypto) — AES256, CTR
- `SD` — file I/O (Arduino built-in)
