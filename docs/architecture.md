# Family Capsule — Architecture

A condensed, current view of the system. The authentication details are in [auth-model.md](auth-model.md), the archive format in [export-format.md](export-format.md), and the threat analysis in [threat-model.md](threat-model.md).

## Product

A private, end-to-end encrypted "capsule" for one family: photos, videos and audio, notes, stories, recipes, people and a family tree, albums, an internal chat and (in development) group video calls. Built for one family first, and multi-tenant from day zero so it can grow into a service.

## Core stance

- **Blind server.** All cryptography runs on the client; the server stores and relays ciphertext only.
- **Enforced by the dependency graph.** `Api` references only `Contracts`, so it physically lacks decryption code. Tests guard the boundary.
- **Full .NET, one UI.** One shared `Core` and one Razor UI across the web and every native platform.
- **Multi-tenant from day zero.** `capsule_id` on every row, isolated by Postgres row-level security.
- **Adapters everywhere.** `IBlobStore`, `IContentSource`, OpenTelemetry exporters: vendor choices are configuration, not lock-in.

## Stack

| Layer | Choice |
|---|---|
| Web client | Blazor WebAssembly (PWA), Web Crypto + hash-wasm |
| Native clients | .NET MAUI Blazor Hybrid — Windows, Android, iOS, macOS — sharing the same Razor UI. Native is the higher-trust tier (signed, installed code rather than server-delivered) |
| Backend | ASP.NET Core minimal API; SignalR relays chat ciphertext |
| Data | PostgreSQL (EF Core / Npgsql) with row-level security |
| Files | Blob storage behind an S3-style `IBlobStore` (local file system or Azure Blob today; S3-compatible such as Cloudflare R2 as an escape hatch) |
| Cache | Redis server-side; client-side caches for thumbnails and decrypted content; service worker for the WASM shell |
| Observability | OpenTelemetry → Aspire dashboard (dev) / Application Insights (prod) |
| Local dev | .NET Aspire + Docker: Postgres, Azurite, Redis, the media server and the TURN relay — fully offline |
| Production host | Being chosen: a VPS under the project's own domain, or a managed cloud (the original plan was Azure, for managed SignalR). Requirements: [hosting-requirements.md](hosting-requirements.md) |

## Security model

**Two-tier envelope.** Each item has its own content key (AES-256-GCM). Content keys are wrapped by one symmetric **Capsule Key**, which is wrapped to each member's **ECDH P-256** public key. Adding a member costs one wrap; removing one rotates the Capsule Key.

**Context-bound ciphertext.** Every AES-GCM ciphertext carries AAD binding it to `capsule ‖ item ‖ type ‖ capsule key version ‖ crypto version`. A malicious server cannot move ciphertexts or wrapped keys between items, capsules or versions without the authentication tag failing. Each key purpose has its own HKDF label.

**Key-only authentication.** There is no password. At registration the client generates a high-entropy **Key** (daily unlock) and a separate one-time **recovery code**. Each derives a KEK through HKDF that unwraps the account's private-key bundle, which the server stores only as ciphertext, so multi-device works without the server ever holding a key. Login is a **signature challenge** with a dedicated ECDSA P-256 keypair: the server stores public keys only, and nothing reusable crosses the wire. Trusted devices cache the unlock behind a passkey/PRF (web), Windows Hello, or biometrics/PIN (mobile).

**Members.** An invitation is a single-use token that only authorises joining. The invitee publishes a public key, an existing member wraps the Capsule Key to it, and the two compare a fingerprint **out of band**, which defends against an actively malicious server substituting keys.

**Groups.** A members-only chat channel (and group-scoped content) is sealed under a **group key** instead of the Capsule Key, wrapped only to the group's members. Keys are wrapped to, and accepted from, only members whose key this account has verified (the invitation check, member security codes, or the capsule's encrypted key directory) — a first sight vouches for nothing. A sealed group roster and a key chain make rotation on removal verifiable, and the highest verified version lives in the account's synced pin store, so a rollback is refused on every device.

**Relationships are encrypted.** Tags, album membership, the people in a story and the family-tree edges live inside the encrypted item payloads. The server sees neither the content nor the graph.

**Crypto agility.** A version tag on every ciphertext and wrapped key allows changing primitives by client-side re-encryption. Post-quantum key encapsulation (ML-KEM) is on the roadmap.

**Verified, not assumed.** Envelope interop between the browser stack and the native stack is verified byte for byte against shared vectors, including negative cases (tamper, wrong key, swapped context).

## Data model

A generic **`items`** store: an open `type` discriminator plus a typed encrypted payload. A new feature is a new type in a `Core` registry, with no schema, API or crypto change. Chat messages are a specialised table for realtime volume and paging. Large media is sealed as chunks, so video can play progressively without decrypting the whole file first.

## Calls *(in development)*

A call lives in a chat channel: up to 8 people, through **our own media server** (mediasoup behind our own signalling) and, where a network needs it, our own **TURN relay** that reaches only the media server. Both see opaque call and participant ids, never a member, account, capsule or channel, and both forward ciphertext.

- **The call key is made on a client.** One random key per *epoch*. The keyholder (the earliest connected participant, a rule the Api also enforces) starts a new epoch with a fresh key on every join and leave, so a newcomer cannot read what came before and a leaver cannot read what comes after. The key is wrapped to each participant with the same primitive as the Capsule Key, **only if that member's key is verified on the keyholder's device**, and for a members-only channel only if the member is in the verified group.
- **A sealed call roster** (under an HKDF-derived key) names who holds the key, bound to capsule, channel, call, epoch, keyholder and each participant's own join nonce. A participant accepts an epoch only if every check passes and it is newer than anything it has accepted. **The call screen lists that sealed roster**, so a server cannot hide or relabel someone who can see and hear the call.
- **Frames are `fc-frame/v1`:** per-sender, per-track, per-stream AES-GCM keys derived by HKDF, counter nonces, a small trailer (SFrame's structure, our wire format). The transform is attached when a sender is created and drops every frame until a key exists, so **nothing ever leaves a device unencrypted**; a browser that cannot transform encoded frames is refused before it joins. A test with real browsers (Chromium, Chromium with `RTCRtpScriptTransform`, Firefox) taps everything the media server forwards and requires every frame, from the first, to be `fc-frame/v1` under the call key.
- **Stated limits:** the media server and the relay see IP addresses, who is in a call and when, packet sizes and timing, and — by decision — who is speaking (the audio-level header stays outside the encryption); audio runs at a constant bitrate so packet sizes do not trace speech. Anyone holding an epoch key could forge another participant's frames (as in SFrame without signatures). Recording (with every participant's consent) is a later increment.

## Notifications

A push is a new channel from a blind server to third parties, so it carries almost nothing: a **fixed-length record** with an event kind and a 16-character per-device alias of the capsule (unlinkable across devices). No name, text, sender or channel; the words are made on the device. FCM and WNS payloads are sealed under a per-registration key (this hides them from Google and Microsoft, not from our server); Web Push is encrypted to the browser by RFC 8291. Registrations are per session and are deleted when the session ends.

## Export and the Decryptor

A family can export the whole capsule: a zip with a manifest and the **same ciphertext the server held**, never re-encrypted on the way out. The **Decryptor** is a standalone, offline, open-source reader for that archive. It needs the Key or the recovery code and nothing else — no network, no service. The archive is designed to outlive the company.

## Known limits

- **Metadata** is still visible to the server: account email, message timing and sender, file sizes and counts. The end-to-end claim covers content and relationships, not metadata. Heavier minimisation (sealed sender, size padding) is deferred to the multi-tenant stage.
- **Push and calls add metadata**: push providers learn that an event happened for a device and when; the media server and the relay learn who calls whom, when and for how long.
- **Served code on the web.** The Web app's code, including the call client, is delivered by our server; the installed apps are the higher-trust tier.
- **Solo-authored crypto orchestration.** Primitives come from the platform and vetted libraries, and the orchestration is externally anchored and adversarially tested. An independent cryptographic review is still wanted.
- **No in-house PAKE.** If password-style login ever returns, it will be through an audited OPAQUE implementation, never a home-grown protocol.
