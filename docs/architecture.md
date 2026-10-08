# Family Capsule — Architecture

A condensed, current view of the system. The authentication details are in [auth-model.md](auth-model.md), the archive format in [export-format.md](export-format.md), and the threat analysis in [threat-model.md](threat-model.md).

## Product

A private, end-to-end encrypted "capsule" for one family: photos and videos, notes, stories, recipes, people and a family tree, albums and an internal chat. Built for one family first, and multi-tenant from day zero so it can grow into a service.

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
| Files | Blob storage behind an S3-style `IBlobStore` (Azure Blob; Cloudflare R2 as an escape hatch) |
| Cache | Redis server-side; client-side caches for thumbnails and decrypted content; service worker for the WASM shell |
| Observability | OpenTelemetry → Aspire dashboard (dev) / Application Insights (prod) |
| Local dev | .NET Aspire + Docker: Postgres, Azurite, Redis — fully offline |
| Target cloud | Azure (managed SignalR was the deciding factor) |

## Security model

**Two-tier envelope.** Each item has its own content key (AES-256-GCM). Content keys are wrapped by one symmetric **Capsule Key**, which is wrapped to each member's **ECDH P-256** public key. Adding a member costs one wrap; removing one rotates the Capsule Key.

**Context-bound ciphertext.** Every AES-GCM ciphertext carries AAD binding it to `capsule ‖ item ‖ type ‖ capsule key version ‖ crypto version`. A malicious server cannot move ciphertexts or wrapped keys between items, capsules or versions without the authentication tag failing. Each key purpose has its own HKDF label.

**Key-only authentication.** There is no password. At registration the client generates a high-entropy **Key** (daily unlock) and a separate one-time **recovery code**. Each derives a KEK through HKDF that unwraps the account's private-key bundle, which the server stores only as ciphertext, so multi-device works without the server ever holding a key. Login is a **signature challenge** with a dedicated ECDSA P-256 keypair: the server stores public keys only, and nothing reusable crosses the wire. Trusted devices cache the unlock behind a passkey/PRF (web), Windows Hello, or biometrics/PIN (mobile).

**Members.** An invitation is a single-use token that only authorises joining. The invitee publishes a public key, an existing member wraps the Capsule Key to it, and the two compare a fingerprint **out of band**, which defends against an actively malicious server substituting keys.

**Relationships are encrypted.** Tags, album membership, the people in a story and the family-tree edges live inside the encrypted item payloads. The server sees neither the content nor the graph.

**Crypto agility.** A version tag on every ciphertext and wrapped key allows changing primitives by client-side re-encryption. Post-quantum key encapsulation (ML-KEM) is on the roadmap.

**Verified, not assumed.** Envelope interop between the browser stack and the native stack is verified byte for byte against shared vectors, including negative cases (tamper, wrong key, swapped context).

## Data model

A generic **`items`** store: an open `type` discriminator plus a typed encrypted payload. A new feature is a new type in a `Core` registry, with no schema, API or crypto change. Chat messages are a specialised table for realtime volume and paging. Large media is sealed as chunks, so video can play progressively without decrypting the whole file first.

## Export and the Decryptor

A family can export the whole capsule: a zip with a manifest and the **same ciphertext the server held**, never re-encrypted on the way out. The **Decryptor** is a standalone, offline, open-source reader for that archive. It needs the Key or the recovery code and nothing else — no network, no service. The archive is designed to outlive the company.

## Known limits

- **Metadata** is still visible to the server: account email, message timing and sender, file sizes and counts. The end-to-end claim covers content and relationships, not metadata. Heavier minimisation (sealed sender, size padding) is deferred to the multi-tenant stage.
- **Solo-authored crypto orchestration.** Primitives come from the platform and vetted libraries, and the orchestration is externally anchored and adversarially tested. An independent cryptographic review is still wanted.
- **No in-house PAKE.** If password-style login ever returns, it will be through an audited OPAQUE implementation, never a home-grown protocol.
