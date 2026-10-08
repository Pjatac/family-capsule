> *Published from the private development repository on 2026-10-08 as `Export-Format-v3.md`, unedited. Identifiers such as `S16.38` or `S3c.4` are internal work-section IDs, and file names refer to the private repository.*

# Family Capsule Export Format v3 (S16.20)

v3 is the current archive format. **The container changed and the crypto did not**: the same
envelopes, the same AAD, the same crypto versions. `Export-Format-v1.md` remains the record of the
v1/v2 JSON bundle; nothing in it about how content is sealed has been superseded.

## Why the container changed

v2 put every blob into the manifest as base64. That inflates an archive by roughly a third and — the
part that actually breaks — forces the whole capsule through memory as one JSON string, on both the
writing and the reading side. On the Web head that is how a real photo library fails to export.

v3 is an ordinary zip, so both sides can work one entry at a time.

## Shape

```
family-capsule-<capsule>-<timestamp>.fcexport      (a zip; the extension is not .json any more)
├── manifest.json
├── blobs/<blobId>.bin        raw ciphertext, one per blob
└── blobs/<blobId>.bin
```

- `format_id` MUST be `family-capsule.export`; `format_version` MUST be `3`.
- **Every entry is written stored (`CompressionLevel.NoCompression`).** Ciphertext does not compress,
  so spending CPU on it is waste; the manifest is stored too, so a leak into it would be visible as
  its own bytes rather than hidden behind a deflate stream.
- **What bounds a reader is caps on bytes that actually arrive, not the "stored" flag.** A zip's
  entry lengths live in its central directory and are a claim by whoever wrote the file; .NET does
  not expose an entry's compression method, so requiring the two lengths to agree is a cheap filter
  and nothing more. A reader therefore refuses any entry declaring more than `MaxBlobBytes`
  (512 MiB), reads bodies into a buffer that grows with the bytes delivered, and refuses an entry
  that delivers a different number of bytes than it declared.
- Entry timestamps are fixed at 1980-01-01Z. When the archive was made is in the manifest; the local
  hour and timezone it was made in are nobody's business.
- Entry order carries no meaning — a zip is addressed by name through its central directory. The
  writer emits blobs first and the manifest last, because the manifest is only complete once every
  blob has been written.

### `manifest.json`

Snake-case JSON, as v2. Top-level keys:

| Key | Meaning |
|---|---|
| `format_id`, `format_version` | `family-capsule.export`, `3` |
| `capsule_id`, `created_at` | which capsule, and when the archive was made |
| `key_only_private_key_envelopes` | the member's ECDH private key wrapped under the Key **and**, when the exporter supplied it, the recovery code. **v3 is Key-only** — there is no password-era `private_key_envelopes`, because auth has had no password since Phase 3c and carrying one would describe a way in that does not exist |
| `capsule_key_wrap` | the exporting member's Capsule Key wrap. It is also where the member id is read from — nobody types a membership GUID (S16.3) |
| `items` | item metadata plus each item's own content envelope, including `envelope_item_type` where the AAD item type differs from the item type (a photo's envelope is its metadata, sealed as `photo_metadata`) |
| `blobs` | blob **descriptors**: id, owning item, AAD item type, media type, size, and the ciphertext's `crypto_version`/`algorithm_id`/`iv`. **The ciphertext body is not here** — it is the entry at `blobs/<id>.bin`. v2's `enc_content_hash` is dropped: every producer wrote it as null and no reader checked it, and an integrity field that is neither populated nor verified only invites the belief that something is being checked. The AES-GCM tag is the integrity check |
| `channels`, `messages` | the chat sections. Defined in S16.20, **filled by S16.24**. A reader MUST tolerate their absence |

A blob carries no key of its own: it is sealed under the content key of the item named by its
`item_id`, and the AAD item type is the blob's own (`photo`, `photo_thumb`, `member_avatar`).

### The chat sections

Reserved while the format was being written, because retrofitting a third top-level section into a
shipped container — with a packaged Decryptor already reading it — is not free, and defining it now
is nearly so.

- `channels`: id, type, participants, created-at, and the channel's own `chat_channel` envelope.
- `messages`: id, channel, sender, optional attachment item id, created-at, and the message envelope.

**Both envelopes bind the *channel* id as their AAD item id** — that is what `ChatClientService`
does online, and it is the one thing a reader cannot guess. `attachment_item_id` resolves to an item
already in the archive.

## What a reader must refuse

All of it before any secret is used, because an archive is a file that arrives from anywhere:

- an entry that is not `manifest.json` or `blobs/<guid>.bin` — no traversal, no smuggled payload;
- more than one manifest, or more than one entry for a blob;
- any entry whose declared compressed and uncompressed lengths disagree, or that declares more than
  `MaxBlobBytes`, or that delivers a different number of bytes than it declared;
- a manifest missing a required section (`items`, `blobs`, `capsule_key_wrap`,
  `key_only_private_key_envelopes` are non-nullable in code and absent-able in JSON);
- a duplicate item id or blob id, or a blob whose `item_id` names no item in the archive — the
  builder enforces the last one on write, and a reader may never assume it wrote the file;
- a manifest whose blob descriptors and the archive's blob entries do not match, in either direction;
- an unsupported `format_id`/`format_version`;
- a truncated file (the central directory is at the end, so there is no index to believe).

Beyond that, the crypto refuses what it always refused: a wrong Key or recovery code, a member the
wrap was not issued to, a re-labelled item, and blob bytes moved into another blob's entry.

## Residual risks, stated rather than implied

- **The blob id is not in any AAD.** A blob's AAD binds the capsule, the owning item and the AAD
  item type — not the blob id. Two blobs of the *same* item and the *same* type would therefore be
  interchangeable. Today no such pair exists (`photo`, `photo_thumb`, `member_avatar` are one each
  per item), so the format silently depends on that; anything that adds a second blob of one type to
  an item must add the blob id to the AAD first.
- **The manifest is not authenticated as a whole.** There is no MAC or signature over it, so an
  archive with an item and its blob entry both deleted opens cleanly with a smaller count. This is
  unchanged from v2 and is a property of the format, not an oversight in the reader — every
  *individual* piece of content is still authenticated by its own AES-GCM tag and AAD. An
  authenticated manifest digest is the fix and is not in this section.

## Compatibility

v3 does not read v2 and does not need to: export had no shipped surface before S16.3, and S16.3's
manual smoke had not been run when v3 landed, so **no v2 archive exists**. The v2 reader stays in
`FamilyCapsule.Core` with its S7.1/S3c.5 tests, which still pin the crypto; the Decryptor UI reads
v3 only.
