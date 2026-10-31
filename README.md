# Family Capsule

**A private, end-to-end encrypted vault for one family's memory:** photos, videos and audio, notes, stories, recipes, people and a family tree, albums, an internal chat and — in development — group video calls. Everything in it outlives the service, because a family can export the whole capsule and open it offline with an open-source reader.

Privacy here is the foundation, not a feature: **the server is blind**. All cryptography runs on the client. The server stores and relays ciphertext only, and it physically lacks the code to decrypt anything.

> **Status (31 October 2026):** feature-complete pre-production. Web, Windows, Android and iOS clients run against a local backend; **choosing the production host is the next step** — see [Hosting requirements](docs/hosting-requirements.md). End-to-end encrypted group calls are being built (the Web client is done and verified; the installed apps are next). About 1,840 automated tests.
>
> This repository is a **public overview**. The source code is in a private repository.

---

## What it does

| Area | What a family gets |
|---|---|
| **Content** | Notes, stories (with on-device dictation, the people in them, a timeline, comments and earlier versions), recipes, people, a family tree, photos, videos and audio. Content carries its language, so a family writing in several can filter by it. New content types plug in without schema, API or crypto changes |
| **Photos, video & audio** | A paged catalogue with client-generated thumbnails, albums by category, the date a photo was taken, and progressive playback of media sealed as encrypted chunks. Metadata embedded in a file is stripped on the device **before** encryption; the few details the app shows (such as the date taken and the place) travel inside the encrypted item instead |
| **Chat** | Realtime channels (the server relays ciphertext only), emoji, channel visibility and admin roles. A **members-only channel is sealed under its own group key**, so other capsule members cannot read it |
| **Calls** *(in development)* | Group video calls of up to 8 people in a chat channel, **end-to-end encrypted from the first frame**: the media server forwards frames it cannot read |
| **Notifications** | Push on Web, Android and Windows, per category, with nothing readable in the push — a fixed-length record with an event kind and a per-device alias |
| **Members** | Invitation by emailed link, out-of-band fingerprint verification against key substitution, admin roles. One person can belong to several capsules |
| **Export** | One archive (zip: manifest plus the same ciphertext the server held, never re-encrypted) with content, chat and members |
| **Decryptor** | A standalone, **offline, open-source (Apache-2.0)** reader for that archive. It needs no network and no Family Capsule service. *Your archive outlives the company.* |
| **Sign-in** | No password anywhere. A client-generated Key, a one-time recovery code, and local unlock with passkey/PRF (web), Windows Hello, or biometrics/PIN (mobile). A revoked session loses access within seconds |
| **Updates** | Installed apps learn when a newer version exists; Windows updates in-app through App Installer. The server serves metadata and package bytes, and never instructs a client to run anything |

## Security model in brief

- **Blind server, enforced by the dependency graph.** The API project references only the contracts project, so decryption code cannot reach the server. Tests enforce this.
- **Two-tier envelope encryption.** Each item has its own content key (AES-256-GCM). Content keys are wrapped by one *Capsule Key*, which is wrapped to each member's ECDH P-256 public key. **Group keys** seal members-only channels the same way, wrapped only to the group.
- **Context-bound ciphertext.** AAD binds every ciphertext to its capsule, item, type and key and crypto versions, so a malicious server cannot swap ciphertexts or wrapped keys.
- **Relationships are encrypted too.** Album membership, tags, the people in a story and the family-tree edges live inside the encrypted payloads.
- **Key-only authentication.** Login is a signature challenge; the server stores public keys only, and the unlock secrets never reach it in any form.
- **Calls keyed on the clients.** A fresh random call key per roster change, wrapped only to members whose keys this device has verified, and a sealed list of who holds it — the call screen shows that list, not the server's.
- **Honest limits.** Metadata (account email, message timing, file sizes and counts; for calls, who is in a call and when) is still visible to the server.

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
                  Media server (mediasoup) + TURN relay — encrypted frames only
```

Local development is fully offline with .NET Aspire and Docker. Every vendor sits behind an adapter; the production host is being chosen (a VPS under the project's own domain, or a managed cloud).

## Documents

| Document | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | Stack, key hierarchy, data model, calls, export, known limits |
| [Hosting requirements](docs/hosting-requirements.md) | What production runs, ports, sizing, external services — for choosing the host |
| [Authentication model](docs/auth-model.md) | Key-only sign-in, signature-challenge login, recovery, local unlock |
| [Threat model](docs/threat-model.md) | Assets, trust boundaries, adversaries, lifecycle flows, STRIDE per boundary |
| [Export format](docs/export-format.md) | The archive the Decryptor reads |
| [Development process](docs/development-process.md) | How it is built solo with AI coding agents: sections, repo-as-memory, test-first security work |

## Contact

Built by Artem Terekhov. For access to the source code, or to review the cryptography, open an issue in this repository.

## License

The documents in this repository are © 2026 Artem Terekhov, all rights reserved. The Family Capsule application is proprietary; its Decryptor component is licensed under Apache-2.0.
