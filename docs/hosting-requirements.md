# Family Capsule — Hosting requirements

What a production deployment has to run, what each part needs from the machine and the network, and what stays outside it. Written for choosing the first host (a VPS under the project's own domain is the option being weighed against a managed cloud). Sizes marked *estimate* have not been measured under real load yet.

## What runs where

Everything the server side runs is **blind**: it stores and forwards ciphertext and never holds a key that opens content.

| Component | What it is | Needs |
|---|---|---|
| **Api** | ASP.NET Core (.NET 10) minimal API + SignalR (chat, invitations; in-process, one instance) | HTTPS behind a reverse proxy; outbound HTTPS to push providers and the mail server |
| **Database** | PostgreSQL with row-level security on every capsule-scoped table; EF Core migrations | Persistent disk, backups; reachable only from the Api |
| **Cache / rate limits** | Redis (token buckets for the two rate-limit planes; caches) | Small memory; reachable only from the Api |
| **Blob storage** | Encrypted chunks (one blob = one chunk of at most 8 MiB plaintext; at most 128 MiB per blob). Behind an `IBlobStore` adapter: **local file system** or **Azure Blob** today; S3-compatible (e.g. Cloudflare R2) is the planned escape hatch | Disk or object storage sized for the capsules (default quota **25 GiB per capsule**); backups |
| **Media server** *(calls, in development)* | Node.js 24 + mediasoup 3 behind the project's own signalling. Forwards encrypted media frames; admitted per call by a one-use token the Api signs | A **public IP**; one WebSocket signalling port (behind the proxy, `wss://`); a **UDP port range** for media (plus TCP fallback); CPU for packet forwarding |
| **TURN relay** *(calls)* | coturn, credentials issued by the Api per call (TURN REST), relaying **only** to the media server | Public IP; `3478/udp` (and `5349/tcp` once TURN over TLS is configured); a UDP relay port range |
| **Data Protection keys** | ASP.NET Core key ring (protects push registrations at rest) | A persistent, private directory (or a key vault) |
| **Update channel** | The Api serves "is there a newer version" metadata and package bytes for the installed apps (Windows App Installer, Decryptor) | Static files on disk next to the Api |

## Network

- **One domain with TLS** (`https://` for the Api and the Web app, `wss://` for SignalR and call signalling). The Web app is a static Blazor WebAssembly PWA and can be served by the same proxy or any static host.
- **Inbound ports:** 443/tcp (proxy); for calls, the media server's UDP range and 3478/udp (+ 5349/tcp with TURN over TLS) and the TURN relay range. Postgres, Redis and the Api's internal roster-check port are **never** exposed.
- **Reverse proxy:** the Api honours `X-Forwarded-For` only from an explicitly configured proxy list (an empty list means the header is ignored), so the proxy's address must be configured for rate limiting by client network to work.
- **Outbound:** HTTPS to Google FCM (Android), Microsoft WNS (Windows) and the browsers' Web Push services; SMTP to the mail provider.

## Sizing *(estimate)*

A first deployment serves a handful of families.

| Resource | Starting point | Grows with |
|---|---|---|
| CPU | 2 vCPU | Calls (media forwarding), uploads |
| Memory | 4 GB (Api ~0.5–1 GB, Postgres ~1 GB, Redis and coturn small, media server ~0.3 GB + per call) | Concurrent calls and connections |
| Disk | System + database: 20–40 GB; blobs: 25 GiB × capsules if stored locally | Photos, video and audio — the dominant cost |
| Traffic | Uploads and downloads of encrypted media | Media libraries; **calls dominate when used** |

**Calls, back of the envelope:** each sender produces roughly 1 Mbit/s (VP8 at 640×360 plus Opus at a constant bitrate). The media server receives one stream per participant and sends each one to everybody else, so a full call of 8 people is about **8 Mbit/s in and ~56 Mbit/s out**; a family call of 4 is about 4 in and 12 out. Traffic through the TURN relay counts twice. A VPS for calls needs an unmetered or generous bandwidth allowance and a stable public IPv4 address.

## Outside the host

| Service | Used for |
|---|---|
| Google Firebase Cloud Messaging | Android push (sealed fixed-length records) |
| Microsoft WNS (an Entra app registration) | Windows push |
| Web Push (VAPID key pair) | Browser push |
| SMTP provider | Sign-up codes and invitation links |
| App stores / signing | Android and Apple builds; Windows packages are signed and offered through App Installer |

None of these receives content: a push carries an event kind and a per-device alias, sealed for FCM/WNS and RFC 8291-encrypted for Web Push.

## Secrets the host must keep

The Api's signing and rate-limit settings, the database and Redis credentials, three distinct call secrets (join tokens, roster check, TURN), the push provider credentials, the SMTP credential and the Data Protection key ring. None of them can decrypt user content — that needs keys only the clients hold — but each would let an attacker impersonate the service. They belong in environment variables or a secrets store, never in the repository.

## Not decided yet

- Managed cloud (the original Azure plan: managed Postgres, Blob, SignalR) versus a single VPS with local disk or S3-compatible storage. The code keeps both open: storage, SignalR and telemetry are behind adapters, and SignalR runs in-process on one instance.
- Backups and their encryption at rest (the blobs are already ciphertext; the database holds metadata and wrapped keys).
- A CDN for the static Web app.
