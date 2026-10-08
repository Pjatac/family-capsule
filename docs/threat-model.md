> *Published from the private development repository on 2026-10-08 as `Threat-Model-v1.md`. Unedited except that its two closing sections, the list of open gaps and prioritised follow-ups, are withheld from this public copy. Identifiers such as `S16.38` or `S3c.4` are internal work-section IDs, and file names refer to the private repository.*

# Family Capsule - Threat Model v1

> Status: S11.1 system model, 2026-07-29.
>
> This artifact describes the implemented post-MVP system at commit `5a6ce97` and
> the security work already queued before production. It is a model and review
> input, not a claim that the application has completed the S11.2 code audit or
> production hardening. `Auth-Model-v2.md` is authoritative for current Key-only
> authentication where older architecture text still describes password-based v1.

## Scope And Security Objectives

Family Capsule is a multi-head, end-to-end encrypted family vault. Blazor
WebAssembly and .NET MAUI clients perform cryptography. The ASP.NET Core Api,
PostgreSQL, blob storage, SignalR relay, Redis, and deployment operator are
outside the plaintext trust domain. (A presentation CMS and its own database
engine used to be listed here; **S16.37 withdrew both** — see the cache boundary
below.)

The system is intended to provide:

1. Content confidentiality against storage providers, database readers, the Api,
   operators, and an honest-but-curious server.
2. Content integrity and context binding through AES-GCM and AAD, including
   rejection of tampering and cross-item/cross-capsule substitution.
3. Account authentication through a dedicated ECDSA P-256 signing key and
   fresh server challenges, without sending the Key or recovery code to the
   server.
4. Capsule isolation through Api authorization plus PostgreSQL RLS.
5. Explicit member-key authenticity through human OOB fingerprint verification
   before an invitee receives a Capsule Key wrap.
6. Forward access revocation: a removed member loses server access and cannot
   decrypt new content after Capsule Key rotation.
7. Long-term data ownership through a self-contained encrypted export and an
   offline read-only Decryptor.

The system does not promise:

- recovery after both the Key and recovery code are lost;
- erasure of plaintext or old keys previously copied by an authorized member;
- protection of plaintext on a compromised, unlocked client;
- anonymity or concealment of all operational metadata;
- availability against a malicious server, provider, operator, or network;
- cryptographic isolation between chat-channel participants in the current
  model-A channel implementation;
- detection of every server rollback, omission, or equivocation.

## Assets

| Asset | Required property | Expected location / exposure |
|---|---|---|
| User Key | Confidentiality, integrity, sufficient entropy | Generated and entered on a client; shown once; never sent to the server |
| Recovery code | Confidentiality, integrity, sufficient entropy | Generated and shown on a client; independent backup; never sent to the server |
| ECDH account private key | Confidentiality and correct account binding | Client only in plaintext; server stores only encrypted envelopes |
| ECDSA signing private key | Confidentiality and correct account binding | Client only in plaintext; wrapped with the account key bundle |
| ECDH and ECDSA public keys | Integrity and identity binding | Plaintext on the server; public but security-sensitive |
| Capsule Key and historical versions | Confidentiality, integrity, membership binding | Client plaintext; server only receives member-specific wraps |
| Per-item content keys | Confidentiality, uniqueness, context binding | Client plaintext transiently; stored only wrapped by a Capsule Key |
| Plaintext content and profile PII | Confidentiality and integrity | Client memory/display only |
| Ciphertext envelopes and blobs | Integrity, availability, version/context binding | Api, PostgreSQL, blob store, export archives, local caches |
| AAD identifiers and crypto metadata | Integrity and consistency | Plaintext metadata used to bind ciphertext to capsule, item, type, member, and version |
| Invite token and OOB fingerprint ceremony | Single use, expiry, authenticity | Token crosses the network; fingerprint is compared by humans out of band |
| Auth challenges and signatures | Freshness, single use, account binding | Redis/distributed cache and TLS-protected auth requests |
| Access and refresh tokens | Confidentiality, integrity, rotation, revocation | Client memory/protected storage and server-side token hashes |
| Email OTP and OTP pepper | Confidentiality, expiry, attempt limiting | Auth plane only; currently PostgreSQL plus server secret |
| Membership, role, audit, and tenant state | Integrity, isolation, accountable changes | Api/PostgreSQL plaintext metadata |
| Export archive | Confidentiality, integrity, completeness, availability | User-controlled file containing ciphertext and wrapped keys |
| Server secrets | Confidentiality and rotation | Development secrets now; production Key Vault is S12.2 |

The server necessarily learns account email, account and capsule identifiers,
membership and role, public keys, item type, timestamps, sizes, language, storage
keys, ciphertext hashes, chat routing metadata, and access/audit events. E2E
encryption protects content, not this metadata.

## Trust Boundaries

### Client device boundary

The Web or MAUI client, OS keystore, browser storage, UI, clipboard, screen,
memory, and local files form the plaintext trust domain. A trusted, unlocked
client may derive KEKs, unwrap private keys and Capsule Keys, decrypt content,
and create ciphertext. Browser XSS, malicious extensions, OS compromise,
screen capture, accessibility abuse, local malware, or a malicious app build
can use the client's authority even if raw keys are non-extractable.

### Api blind-server boundary

TLS carries auth and ciphertext traffic to the Api. The Api may authenticate,
authorize, rate-limit, validate metadata, scope requests, and relay/store
ciphertext. It must not receive a Key, recovery code, plaintext private key,
Capsule Key, content key, plaintext payload, or raw dictation audio.
`FamilyCapsule.Api.csproj` references only Contracts and ServiceDefaults.

### Database/RLS boundary

PostgreSQL stores accounts, public keys, encrypted private-key bundles, sessions,
memberships, invitations, wrapped keys, ciphertext records, blob metadata, chat
routing metadata, and security audit metadata. Api scoping is the first tenant
control and RLS is defense in depth. Connection context, pooling, migrations,
background jobs, and BYPASSRLS/service roles are part of this boundary.

### Blob-store boundary

Azurite/Azure Blob or S3-compatible NAS/Garage stores ciphertext bytes. Clients
use the authenticated Api blob proxy; the storage provider must never receive
plaintext. Storage keys, ciphertext sizes, timing, hashes, and access patterns
remain observable.

### Realtime relay boundary

SignalR and the Api route chat ciphertext and expose channel, sender, recipient
set, and timing metadata. Current model-A channel membership is server access
control; all capsule members still possess the Capsule Key, so it is not
cryptographic participant isolation.

### Cache boundary

**S16.37 (2026-10-03): the CMS and its database engine are withdrawn, and this
boundary got smaller rather than safer-by-assertion.** There is no CMS process,
no second database engine, no `:latest` image and no admin credential in the
local stack; presentation content now comes from `FallbackPresentationContent`
compiled into the Api, behind the `IContentSource` seam. The threats below that
concern a CMS are therefore **not mitigated, they are absent** — and they return
the day an implementation is registered behind that seam, which is the reason
they are kept here rather than deleted. Architecture §16 names the withdrawn
product.

Redis is for caching, ephemeral auth challenges, request-rate buckets (S16.59),
and future OTP TTL storage. It must not hold user vault
content, private keys, the Key, recovery code, Capsule Key, or core offline form
labels. Redis compromise can affect availability and challenge freshness but
must not reveal an encryption key.

### Email and external identity boundary

Email delivery sees destination addresses and OTP-bearing messages. External
IdPs operate only on the account-authentication plane and cannot unlock vault
content. Provider compromise may affect account/session access but must not
produce the Key, recovery code, or private-key bundle plaintext.

### Human OOB boundary

The invitee and an existing owner/admin compare a fingerprint through a channel
independent from the Api, such as an in-person or voice conversation. Security
depends on both people comparing the complete presented value and refusing a
mismatch. The server can delay or present inconsistent workflow state but must
not be trusted to assert that the keys match.

### Offline export boundary

An exported archive leaves the service and becomes user-controlled ciphertext.
The offline Decryptor must need no service, must not send archive material over
the network, and must fail closed for a wrong Key, wrong recovery code, tampered
metadata, or tampered ciphertext. File availability, backup handling, and
physical custody become the user's responsibility.

### Build, dependency, and operator boundary

Source control, NuGet/npm packages, SDKs, build machines, signing identities,
release artifacts, and operators can change client or server behavior. A
malicious client release can exfiltrate plaintext before encryption; E2E does
not protect against a compromised client supply chain. S11.4 and S12.4 address
this boundary further.

## Adversaries

| Adversary | Capabilities | Primary protections | Residual capability |
|---|---|---|---|
| Honest-but-curious server/operator | Reads server, DB, logs, storage, and traffic metadata | Client-side encryption, blind-server project boundary, secret-log rules | Metadata, availability, traffic analysis |
| Active malicious server | Substitutes public data, withholds/replays rows, equivocates, modifies ciphertext | AES-GCM/AAD, OOB invite fingerprint, crypto versions | Denial, rollback/omission, workflow equivocation where no transparency log exists |
| Database/blob thief | Copies all rows, blobs, hashes, wraps, and public keys | High-entropy Key/recovery code, encrypted bundles, ciphertext-only blobs | Offline metadata analysis and availability attacks |
| Network attacker | Observes, blocks, replays, or alters traffic | HTTPS, signatures, single-use challenges, JWT validation, AES-GCM | Traffic analysis and denial; TLS trust-store compromise changes the model |
| Malicious outsider | Enumerates accounts, guesses OTPs, steals invite links, floods endpoints | Fake prelogin/challenge responses, rate limits, lockout, OTP attempts/expiry, invite expiry | Denial and metadata inference from imperfect response/timing parity |
| Malicious or removed member | Has legitimate old plaintext, exports, wraps, tokens, or an old Capsule Key | Role checks, session revocation, rotation for new content | Retains anything previously learned or copied |
| Compromised client or browser origin | Executes with user/client authority | CSP, no third-party scripts, non-extractable keys, OS protected storage | Can read displayed plaintext or invoke decryption while the client is unlocked |
| Compromised dependency/build pipeline | Alters shipped code or package behavior | Pinning, review, release controls, future CI gates/SBOM | Full client compromise until detected |
| Lost/stolen device attacker | Obtains local files, tokens, protected blobs, possibly an unlocked session | OS gate, secure storage, Windows Hello, logout/revocation | Unlocked-device compromise; mobile product parity remains S14.10 |
| Malicious CMS/email/IdP provider | Controls presentation copy or auth-plane delivery/assertions | Strict data boundary, output handling, IdP cannot unwrap | Phishing/UI confusion, account-plane denial or takeover without vault unlock |

## Lifecycle Flows

### Registration And Key Bootstrap

1. The client generates a high-entropy Key and independent recovery code and
   shows both once.
2. The client generates separate P-256 ECDH and ECDSA keypairs. ECDH is for
   Capsule Key wrapping; ECDSA is for auth challenge signing.
3. The client derives `KEK = HKDF-SHA256(Key, per-account salt,
   "fc:kek:v2")` and `recovery-KEK = HKDF-SHA256(recoveryCode,
   per-account salt, "fc:recovery-kek:v2")`.
4. The private-key bundle is encrypted into normal and recovery envelopes. The
   server receives the two ciphertext envelopes, salt, public keys, and account
   metadata, never either unlock secret or a private key.
5. The client creates a random Capsule Key and wraps it to the first member's
   ECDH public key. Every ciphertext and wrap carries its applicable version.
6. Email OTP verifies the account on the auth plane only; it never derives or
   unlocks vault keys.

Failure expectations: weak/missing inputs are rejected, wrong Key/recovery code
cannot decrypt the bundle, ECDH and ECDSA private keys are not reused across
algorithms, and registration logs contain no secret.

### Key Unlock And Local Restore

1. A new device obtains the public salt and encrypted bundle through the
   enumeration-mitigated prelogin/bootstrap path.
2. The user enters the Key or recovery code locally. The client derives the
   appropriate KEK and decrypts the private-key bundle.
3. The Api issues a fresh random auth challenge. The client signs a message
   bound to the nonce, challenge id, and normalized email with its dedicated
   ECDSA private key.
4. The server consumes the challenge, verifies against the stored public key,
   and issues a short-lived access token plus rotating refresh token.
5. The client fetches its Capsule Key wrap and unwraps it locally using ECDH.
6. Web persistence stores only a refresh token/account id in session storage
   and keeps key operations behind the non-extractable browser store. Windows
   persistent auth seals account-key restore behind Windows Hello. Android/iOS
   product-facing opt-in and live parity remain S14.10.

Recovery unlock must lead to rotation to a new daily Key in the product flow.
Loss of both independent unlock secrets is intentionally unrecoverable.

### Capsule Key Wrap And Unwrap

1. Each item gets a random content key; AES-GCM encrypts the payload with AAD
   binding capsule id, item id/type, Capsule Key version, and crypto version.
2. The content key is wrapped under the current Capsule Key with its own IV,
   AAD, algorithm id, and version metadata.
3. The Capsule Key is wrapped per member using ECDH P-256 and HKDF-SHA256,
   bound to capsule/member/key-version context.
4. The server stores ciphertext, IVs, public metadata, and wraps. Only an
   authorized client unwraps the Capsule Key and content keys.

Tampering, wrong keys, wrong members, substituted public keys, and cross-context
swaps must fail closed. IV uniqueness relies on cryptographically secure random
generation for every AES-GCM operation.

### Invite And OOB Verification

1. An owner/admin creates a single-use expiring invite token. The token grants
   admission workflow authority, not a decryption key.
2. The invitee accepts and publishes its ECDH public key.
3. Both humans compare a fingerprint bound to invitation id, capsule id, and
   the invitee public key over the human OOB boundary.
4. The inviter's client refuses to create a Capsule Key wrap before a match.
   The Api additionally requires the verified workflow state.
5. The invitee becomes active only after receiving the correctly bound wrap.

A malicious server can withhold the flow or show different statuses, but a
substituted key must produce a different human-verified fingerprint.

### Member Removal And Capsule Key Rotation

1. An authorized owner/admin removes the member and revokes that account's
   capsule sessions/server access.
2. A surviving client generates a new Capsule Key and wraps it only for active
   survivors.
3. New content immediately uses the new key version.
4. Existing content keys are rewrapped progressively and idempotently; mixed
   versions are expected during rotation.
5. Reads select the correct historical Capsule Key version while migration is
   incomplete.

The guarantee is prospective only. A removed member may retain old keys,
plaintext, screenshots, blobs, exports, or caches. Server rollback, incomplete
rotation, and abandoned progress are availability/integrity concerns for S11.2.

### Export And Offline Decryptor

1. An authorized client assembles the archive from ciphertext items, encrypted
   blobs, key wraps, metadata, and Key-only private-key envelopes.
2. No plaintext payload, private key, Capsule Key, content key, Key, or recovery
   code is written into the archive.
3. The user moves the archive outside the service boundary.
4. The offline reader derives a KEK from either the Key or recovery code,
   decrypts the private-key bundle, unwraps the Capsule Key and content keys,
   validates AAD/version metadata, and renders read-only content.
5. A wrong secret or any authenticated-field/ciphertext tamper fails closed.

The core archive logic exists, but discoverable product export and live
cross-head/Decryptor smoke are mandatory S16.3 work before production.

## STRIDE Analysis

### Client device boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Malware or a phishing UI impersonates Family Capsule | Signed/package identity where available, origin/TLS, platform auth prompts | Release signing and user education need production review |
| Tampering | XSS or local malware changes plaintext before encryption | CSP, no third-party scripts, authenticated encryption after creation | An authorized malicious client can create valid malicious ciphertext |
| Repudiation | A user denies a client-side action | Metadata-only server audit events and actor ids | Audit cannot prove the human intended an action on a compromised client |
| Information disclosure | Unlocked UI, memory, clipboard, screenshot, XSS, extension | Non-extractable keys, secure storage, CSP, minimal persistence | Plaintext is necessarily exposed to an authorized UI |
| Denial of service | Corrupt local state or delete protected keys | Recovery code and server-held encrypted bundle | Loss of Key plus recovery code is permanent |
| Elevation of privilege | Token/key material from one account is used as another | Account-bound challenges, AAD, membership checks | Device compromise inherits that user's effective authority |

### Api blind-server boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Forged JWT, signature, refresh token, or external-IdP assertion | JWT validation, ECDSA challenge, token hashes/rotation, provider validation | System-wide re-review is S11.2 |
| Tampering | Api substitutes keys, metadata, ciphertext, or versions | AES-GCM/AAD, OOB fingerprint, explicit versions | No general signed manifest/transparency log for rollback/omission |
| Repudiation | Actor denies membership/auth/management action | Append-only-intended metadata audit log | Audit sink immutability and retention require production validation |
| Information disclosure | Api code/log receives plaintext or secret | Project-reference boundary, ciphertext contracts, redaction tests | Contract/code review across all endpoints remains S11.2 |
| Denial of service | Rate-limit abuse, withheld rows, oversized requests, relay outage | Quotas, rate limits, health checks, policy validation | Malicious server can always deny service |
| Elevation of privilege | Missing role or capsule check exposes another tenant | Endpoint checks plus RLS | Background/service-role and all endpoint paths need holistic audit |

### Database/RLS boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Connection uses forged or stale capsule context | JWT-derived context and transactional `SET LOCAL` model | Verify actual coverage/pooling behavior in S11.2 |
| Tampering | DB writer changes ciphertext, roles, wraps, or audit rows | Authenticated encryption catches content changes; API authorization | Server-side metadata/role tamper is not cryptographically signed |
| Repudiation | Privileged DB operator edits history | Metadata audit design | True external/WORM anchoring is not yet defined |
| Information disclosure | DB dump exposes data | Ciphertext and wrapped keys | Email, membership, sizes, types, timing, and graph/routing metadata remain plaintext |
| Denial of service | Rows deleted, locked, rolled back, or made inconsistent | Backups/migrations and idempotent rotation design | Availability and rollback detection are not E2E guarantees |
| Elevation of privilege | BYPASSRLS role is used for a user request | Separate service/migration role rule and RLS tests | Production identity wiring is pending Phase 12 |

### Blob-store boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Fake storage endpoint returns attacker bytes | Api-controlled provider configuration, AES-GCM/hash checks | Endpoint/configuration trust is operational |
| Tampering | Blob bytes or storage key are replaced | Ciphertext hash plus AES-GCM/AAD | Missing blob or stale valid version is denial/rollback |
| Repudiation | Provider disputes write/delete | Api/storage telemetry | Provider logs are not an end-user authenticity proof |
| Information disclosure | Provider reads blobs | Client-side encryption | Size, timing, access pattern, ciphertext equality hash remain visible |
| Denial of service | Provider/NAS deletes or withholds blobs | Provider abstraction and user exports | Backup/restore objectives need production definition |
| Elevation of privilege | Cross-capsule blob identifier is requested | Authenticated Api proxy and capsule/member scoping | Re-audit all blob lifecycle paths in S11.2 |

### Realtime relay boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Attacker sends as another member | Authenticated hub connection and server sender identity | Compromised token acts as its account |
| Tampering | Relay changes message ciphertext/routing | AES-GCM protects the body | Channel metadata/routing is server-controlled |
| Repudiation | Sender disputes a message | Stored sender/timestamp metadata | No end-user message signature/non-repudiation |
| Information disclosure | Relay observes conversations | Encrypted body | Sender, channel, participants, timing, size are visible; model A is not crypto isolation |
| Denial of service | Drop, reorder, duplicate, or flood messages | Persistence, rate controls, client history | Malicious relay can impair availability/order |
| Elevation of privilege | Non-participant joins a channel | Server membership checks | Any capsule member with the Capsule Key may decrypt obtained ciphertext |

### CMS/cache boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Fake CMS/cache supplies presentation content or challenge state | Server-side adapters/configuration and fallback | Presentation content can still phish if rendered unsafely |
| Tampering | Cached copy or auth challenge is modified | Random nonce plus ECDSA verification; typed content boundary | Atomic single-use consume requires focused audit |
| Repudiation | Cache mutation has little durable evidence | Api security audit events | Ephemeral state intentionally disappears |
| Information disclosure | Redis dump | No vault data or encryption keys allowed there | Auth timing, cache keys, and presentation content remain visible. **S16.37: no CMS or Mongo store exists to dump** |
| Denial of service | Eviction/outage breaks challenges or content | Memory fallback in dev; presentation content is compiled-in fallback since **S16.37**, so no CMS can be down | Production Redis availability is Phase 12 work |
| Elevation of privilege | CMS data is interpreted as executable/core policy | **S16.37: absent, not mitigated** — no CMS supplies data. Returns with any implementation registered behind `IContentSource`, which must then restrict itself to presentation copy | Output encoding and schema enforcement require review **before** a CMS is re-added |

### Human OOB boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Attacker impersonates a relative on the OOB channel | Human recognition and independent contact | Social engineering remains possible |
| Tampering | One side reads/enters a different fingerprint | Fingerprint binds invitation, capsule, and public key | Users may compare incompletely or ignore warnings |
| Repudiation | Participant denies confirming | Server workflow state | Server state cannot prove a careful human comparison |
| Information disclosure | Fingerprint is overheard | Fingerprint/public key is not secret | Relationship and invitation timing may leak |
| Denial of service | Server or attacker prevents comparison/wrap | Explicit pending state | No safe automated bypass should exist |
| Elevation of privilege | Admin confirms an attacker's key | Role check plus OOB ceremony | A malicious authorized admin can grant access |

### Offline export boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Fake Decryptor or archive impersonates a real artifact | Open format and future signed releases | Archive does not currently carry an end-user signature |
| Tampering | IDs, versions, ciphertext, or wraps are edited | AES-GCM/AAD and strict format/version checks | Deletion/omission of entries may be hard to distinguish from an incomplete export |
| Repudiation | Export creator denies producing archive | Creation metadata only | No author signature/non-repudiation |
| Information disclosure | Archive file is stolen | Key/recovery-wrapped private bundle and ciphertext | Metadata and offline availability enable analysis; weak user custody loses confidentiality if secret is also stolen |
| Denial of service | Archive is truncated, deleted, or future reader unavailable | Self-contained open format and standalone reader | User backup discipline and compatibility testing remain necessary |
| Elevation of privilege | Decryptor imports active content or contacts service | Read-only/offline design | Product smoke and network-negative proof remain S16.3 |

### Build, dependency, and operator boundary

| Threat | Scenario | Existing control | Residual / follow-up |
|---|---|---|---|
| Spoofing | Unofficial package/build is distributed | Repository history and platform packaging | Formal release provenance is not yet implemented |
| Tampering | Dependency or pipeline injects exfiltration | Pinned packages, reviews, no new crypto dependency without approval | S11.4 audit and S12.4 CI gates remain open |
| Repudiation | Operator denies release/config change | Git history and future pipeline audit | Local/manual release steps have weaker provenance |
| Information disclosure | Build logs/config contain secrets | Secret-handling rules and user-secrets | Production secret scan/Key Vault are pending |
| Denial of service | Dependency removal or compromised update breaks builds | Lock/pin discipline and source history | External registries remain dependencies |
| Elevation of privilege | Pipeline identity deploys malicious code or reads secrets | Least-privilege target design | Managed identity and CI isolation are Phase 12 work |

