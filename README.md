# Family Capsule

**A private, end-to-end encrypted vault for one family's memory:** photos and videos, notes, stories, recipes, people and a family tree, albums and an internal chat. Everything in it outlives the service, because a family can export the whole capsule and open it offline with an open-source reader.

Privacy here is the foundation, not a feature: **the server is blind**. All cryptography runs on the client. The server stores and relays ciphertext only, and it physically lacks the code to decrypt anything.

> **Status:** feature-complete pre-production. Web, Windows, Android and iOS clients run against a local backend; deployment is the next phase. About 1,200 automated tests.
>
> This repository is a **public overview**. The source code is in a private repository.

---

## What it does

| Area | What a family gets |
|---|---|
| **Content** | Notes, stories (with on-device dictation), recipes, people, a family tree, photos and videos. New content types plug in without schema, API or crypto changes |
| **Photos & video** | A paged catalogue with client-generated thumbnails, albums and groups of albums, and progressive playback of video sealed as encrypted chunks |
| **Chat** | Realtime channels (the server relays ciphertext only), channel visibility and admin roles |
| **Members** | Invitation by emailed link, out-of-band fingerprint verification against key substitution, admin roles. One person can belong to several capsules |
| **Export** | One archive (zip: manifest plus the same ciphertext the server held, never re-encrypted) with content, chat and members |
| **Decryptor** | A standalone, **offline, open-source (Apache-2.0)** reader for that archive. It needs no network and no Family Capsule service. *Your archive outlives the company.* |
| **Sign-in** | No password anywhere. A client-generated Key, a one-time recovery code, and local unlock with passkey/PRF (web), Windows Hello, or biometrics/PIN (mobile) |
| **Updates** *(in final verification)* | Installed apps learn when a newer version exists. The server serves metadata and package bytes, and never instructs a client to run anything |

## Security model in brief

- **Blind server, enforced by the dependency graph.** The API project references only the contracts project, so decryption code cannot reach the server. Tests enforce this.
- **Two-tier envelope encryption.** Each item has its own content key (AES-256-GCM). Content keys are wrapped by one *Capsule Key*, which is wrapped to each member's ECDH P-256 public key.
- **Context-bound ciphertext.** AAD binds every ciphertext to its capsule, item, type and key and crypto versions, so a malicious server cannot swap ciphertexts or wrapped keys.
- **Relationships are encrypted too.** Album membership, tags, the people in a story and the family-tree edges live inside the encrypted payloads.
- **Key-only authentication.** Login is a signature challenge; the server stores public keys only, and the unlock secrets never reach it in any form.
- **Honest limits.** Metadata (account email, message timing, file sizes and counts) is still visible to the server.

## Architecture

One shared .NET core and **one Razor UI** serve every platform:

```
            ┌───────────── Shared Razor UI ─────────────┐
            │                                           │
   Blazor WebAssembly (PWA)               .NET MAUI Blazor Hybrid
   + Web Crypto                 (Windows · Android · iOS · macOS) + .NET crypto
            └──── Core: crypto orchestration, item types, archive ────┘
                                   │  ciphertext only
                  ASP.NET Core minimal API + SignalR  (references only Contracts)
                  Postgres (row-level security per capsule) · Blob storage · Redis
```

Local development is fully offline with .NET Aspire and Docker. The target cloud is Azure, with every vendor behind an adapter.

## Documents

| Document | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | Stack, key hierarchy, data model, export, known limits |
| [Authentication model](docs/auth-model.md) | Key-only sign-in, signature-challenge login, recovery, local unlock |
| [Threat model](docs/threat-model.md) | Assets, trust boundaries, adversaries, lifecycle flows, STRIDE per boundary |
| [Export format](docs/export-format.md) | The archive the Decryptor reads |
| [Development process](docs/development-process.md) | How it is built solo with AI coding agents: sections, repo-as-memory, test-first security work |

## Contact

Built by Artem Terekhov. For access to the source code, or to review the cryptography, open an issue in this repository.

## License

The documents in this repository are © 2026 Artem Terekhov, all rights reserved. The Family Capsule application is proprietary; its Decryptor component is licensed under Apache-2.0.
