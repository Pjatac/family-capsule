> *Published from the private development repository on 2026-10-08 as `Auth-Model-v2.md`, unedited. Identifiers such as `S16.38` or `S3c.4` are internal work-section IDs, and file names refer to the private repository.*

# Family Capsule — Authentication & Key-Unlock Model v2 (Key-only)

> Status: **DECISION (2026-07-23) + design agreed (S3c.1, 2026-07-26).** Supersedes **`Auth-Model-v1.md`** (password + Key). v2 removes the password entirely: the client-generated **Key** is the only unlock factor, a separate one-time **recovery code** is the backup, and **server login is a signature-challenge** (Artem's S3c.1 choice: mechanism **B**, not the Key-derived verifier). Pre-launch → **no data migration**. Fixed crypto primitives only (AES-GCM, ECDH P-256, ECDSA P-256, HKDF-SHA256) — **no new crypto dependency**. SEC throughout: tests/vectors first + mandatory security-review.

This document is the spec that **S3c.2 (core crypto)**, **S3c.3 (passwordless server login)**, **S3c.4 (registration + Web UI)**, and **S3c.5 (cascade re-verify)** implement. It also underpins **S14.8** (Windows persistent auth via Windows Hello), which caches the signing key behind an OS gate.

---

## 1. What changed from v1

| | v1 (password + Key) | **v2 (Key-only)** |
|---|---|---|
| Unlock factors | password **+** Key | **Key only** |
| Server login | password **auth-verifier** (hash stored) | **signature-challenge** (server stores a **signing public key**; client signs a fresh nonce) |
| KEK | `HKDF(password-KEK-material ‖ Key)` (one Argon2id run) | `KEK = HKDF(Key)` (no Argon2id — Key is high-entropy) |
| Recovery | Key alone (`recovery-KEK=HKDF(Key)`) | **separate one-time recovery code** (`recovery-KEK=HKDF(recoveryCode)`) |
| Argon2id | yes (password) | **not used** (no low-entropy secret in the unlock/auth path) |
| Secrets on the wire to the server | reusable password-verifier | **only per-login signatures over server nonces** (no reusable auth secret) |

The **two-plane rule is unchanged**: nothing the server knows ever enters the encryption plane. In v2 the auth plane holds only **public** signing keys and issues sessions; the encryption plane holds the Key / recovery code, which never reach the server in any form (not even one-way-derived).

## 2. Secrets and keys

| Secret / key | Origin | Plane | Server sees? | Role |
|---|---|---|---|---|
| **Key** | client-generated, ≥128-bit, shown **once** at registration | encryption | **never** | daily unlock: `KEK=HKDF(Key)` unwraps the private-key bundle |
| **Recovery code** | client-generated, ≥128-bit, shown **once** at registration | encryption | **never** | backup unlock: `recovery-KEK=HKDF(recoveryCode)` unwraps the same bundle |
| **ECDH account keypair** (P-256) | client | encryption | **public key only** | capsule-key wrapping (unchanged from before) |
| **ECDSA signing keypair** (P-256) | client, **new in v2** | auth | **public key only** | proves account possession by signing server challenges |
| **Session tokens** (JWT access + rotating refresh) | server | auth | yes | issued after a verified signature; unchanged rotation/reuse-detection |
| **Biometric / Windows Hello / passkey** | device | local unlock | no | releases the cached signing + ECDH keys on a trusted device (S3.5/S3.6/S14.8) |
| **Email OTP** | server | auth | yes | one-time email verification at registration (unchanged, S3.7) |

Note the deliberate **separate ECDSA signing keypair**: reusing the ECDH P-256 key for ECDSA signatures is a cross-algorithm key-reuse footgun, so v2 uses a dedicated signing keypair. Both private keys are wrapped together as one **bundle** so a single unlock yields both.

## 3. Key derivation (fixed primitives only)

```
KEK           = HKDF-SHA256(IKM=Key,          salt=perAccountSalt, info="fc:kek:v2",          len=32)
recovery-KEK  = HKDF-SHA256(IKM=recoveryCode, salt=perAccountSalt, info="fc:recovery-kek:v2", len=32)
```

- **No Argon2id**: both the Key and the recovery code are high-entropy (≥128-bit), so a memory-hard KDF adds nothing — HKDF is correct (same reasoning as v1's `recovery-KEK`).
- `perAccountSalt` is random per account, stored server-side, returned by prelogin (domain-separation / defence-in-depth; it is not secret).
- The private-key **bundle** `{ecdhPrivateKey, signPrivateKey}` is stored server-side as **two ciphertext envelopes**:
  - **envelope-normal** — AES-GCM under `KEK` → needs the **Key**.
  - **envelope-recovery** — AES-GCM under `recovery-KEK` → needs the **recovery code**.
- Every envelope/wrapped key carries the crypto-version tag; **bump the crypto-version** for the v2 derivation (S9.2 / S3c.5).

## 4. Server login — signature-challenge (mechanism B)

The server stores, per account: `signing_public_key` (P-256), `ecdh_public_key`, `perAccountSalt`, both envelopes, email. **No auth-verifier, no password hash.**

**Login handshake:**
1. `POST /auth/challenge { email }` → server creates a single-use, short-TTL **challenge** (random ≥256-bit nonce + id, stored in Redis with TTL — reuse the S14.R store) and returns `{ challengeId, nonce, perAccountSalt, envelopeNormal }`. Envelope is ciphertext → server stays blind. **Enumeration-safe:** for an unknown/unverified email, return a deterministic *fake* salt + fake envelope + a real-looking challenge (mirror S9.4 prelogin fake-salt), so existence is not revealed.
2. Client: `KEK = HKDF(Key, salt)` → AES-GCM-decrypt `envelopeNormal` → `{ecdhPriv, signPriv}` → `signature = ECDSA-P256-SHA256(signPriv, nonce ‖ challengeId ‖ email)`.
3. `POST /auth/login { email, challengeId, signature, deviceName }` → server loads the challenge (must exist, unexpired, unconsumed), **consumes it** (single-use), verifies `signature` against `signing_public_key`. On success → issue JWT access + rotating refresh token (existing S3.2/S3.3 machinery). On failure → count against per-email rate-limit/lockout (reuse S9.4), constant-time where relevant.
4. Then `/devices/bootstrap` returns the wrapped capsule key as today; the client already holds `ecdhPriv` from step 2 → unwraps the capsule key.

**Routine access (trusted device):** the signing key is cached in the OS keystore (Android Keystore / iOS Secure Enclave / **Windows Hello**, S3.5/S14.8), gated by biometric/PIN → the client can sign the challenge **without re-entering the Key**. New/untrusted device → full Key entry (step 2).

**Recovery (lost Key):** the client uses the **recovery code** → prelogin/`/auth/challenge` returns `envelopeRecovery` (add it to the challenge response, or a dedicated `recoveryChallenge`) → `recovery-KEK` → decrypt the bundle → sign the challenge (log in) → the app then **re-wraps** the bundle under a **new Key** (new `envelope-normal`) and shows the new Key once. Recovery is thus "log in with the backup, then rotate the daily Key".

**Lost both Key and recovery code:** the vault is unrecoverable (both envelopes need one of them). Mandates strong "save your Key AND recovery code" UX; roadmap: **Shamir social recovery** (Architecture §168) to reconstruct from family shares.

## 5. Why B is safe (and what it costs)

- **Server holds only public keys** + ciphertext envelopes. A full DB leak reveals no auth secret and no decryption key; unwrapping still requires the Key or recovery code (≥128-bit → offline brute force infeasible).
- **No reusable bearer secret on the wire.** Each login is a signature over a fresh single-use nonce → **replay-resistant**: a captured signature cannot be reused, and there is nothing like a password-verifier to steal in transit.
- **Auth ⇒ possession of the signing key ⇒ possession of the Key (or a trusted-device cache).** There is no "auth-only" secret separate from the Key, so v2 does not add a weaker credential; stealing the Key is full compromise, which is inherent to any Key-only design.
- **Cost (honest):** a **second keypair** (ECDSA signing) per account + its two envelope slots + public-key column; a **challenge endpoint** with a single-use TTL nonce store; prelogin now returns an envelope **before** a session exists (still ciphertext → blind-server preserved). This is more SEC surface than mechanism A, accepted for the stronger zero-knowledge/replay properties (Artem 2026-07-26).

## 6. Honest limits (documented, not hidden)

- The **Key is a full standalone credential**; the **recovery code is a second standalone credential**. Two independent unlock paths; losing one is fine, losing **both** = unrecoverable.
- ECDSA P-256 is a **fixed primitive** (built into .NET `System.Security.Cryptography.ECDsa` and WebCrypto `ECDSA`) — **no new dependency**. Do **not** reuse the ECDH key for signing (separate keypair, §2).
- The challenge nonce store is ephemeral (Redis TTL, memory-fallback for dev) — single-use, short TTL; pairs with the S14.R OTP→Redis move.
- **No PAKE / no self-rolled crypto.** Passkey PRF, platform keystore, and Windows Hello are OS APIs, not crypto libraries.
- Email OTP (S3.7) still gates **account** verification at registration; it is auth-plane only and never unlocks the vault.
- 2FA/TOTP remains out of scope (separate post-MVP analysis, unchanged).

## 7. Cascade — sections to re-verify when v2 lands (S3c.5)

Mark `needs-reverify` and re-run: **S3.1** (registration: generate Key + recovery code + signing keypair; drop password), **S3.2** (login → signature-challenge; drop password-verifier), **S3.3** (multi-device new-device login = Key entry then sign), **S3.4** (unlock derivation → `KEK=HKDF(Key)`, no Argon2id), **S1.5** (recovery-envelope), **S3.5/S3.6** (biometric/passkey now cache the signing key too), **S5.2** (registration UX: show Key **and** recovery code once), **S7.1 + `Export-Format-v1.md`** and **S7.2 decryptor** (unlock inputs become **Key OR recovery code**; export bundle wrapped under the v2 KEKs). **Bump the crypto-version tag** (S9.2). No data migration (pre-launch).

## 8. Section order (Phase 3c)

`S3c.1` (this doc, agreed) → **`S3c.2`** core crypto (`KEK=HKDF(Key)`, recovery-code envelope, signing keypair generation + wrap; interop vectors + negatives) → **`S3c.3`** passwordless server login (challenge endpoint + signature verify; drop password-verifier; keep rate-limit/lockout + enumeration-safety) → **`S3c.4`** registration + Web UI (generate Key + recovery code, show once; Login = Key → sign; remove password fields; recovery-code entry flow) → **`S3c.5`** cascade re-verify + crypto-version bump. Each is SEC: tests/vectors first + security-review before merge.
