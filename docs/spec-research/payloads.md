# Payloads — Research Notes (for spec/payloads.md)

**Ticket:** dsfx3d/bitchat#24 ("Payloads chapter: verify message/payload TLV types (Swift + Kotlin)")
**Map:** dsfx3d/bitchat#16
**Status:** Research only. Facts below are cited file+line; kept internal to this ticket per #11
(someone else folds this into #16's Decisions-so-far).

Source of truth: this repo's Swift payload codec (`bitchat/Models/`, `bitchat/Protocols/`,
`localPackages/BitFoundation/Sources/BitFoundation/`). Cross-checked (read-only) against
`permissionlesstech/bitchat-android` via a `--depth 1` shallow clone on 2026-08-01.

---

## 0. Important framing correction

The ticket's examples ("text, ack, fragment, presence") mix two different protocol layers.
Only some of those actually ride *inside* the Noise-encrypted channel (`MessageType.noiseEncrypted`
/ `NoisePayloadType`); the others are **outer, plaintext `MessageType` packets** that never touch
Noise at all:

| Ticket's example | Actual layer | Type/tag |
|---|---|---|
| "text" (private message) | Inside Noise | `NoisePayloadType.privateMessage = 0x01` |
| "ack" (delivered/read) | Inside Noise | `NoisePayloadType.delivered = 0x03`, `.readReceipt = 0x02` |
| "fragment" | **Outer**, unencrypted framing around any packet (including Noise-encrypted ones) | `MessageType.fragment = 0x20` |
| "presence" (announce) | **Outer**, plaintext, self-signed | `MessageType.announce = 0x01` |

Source: `bitchat/Protocols/BitchatProtocol.swift:76-130` (`NoisePayloadType`),
`localPackages/BitFoundation/Sources/BitFoundation/MessageType.swift:12-63` (`MessageType`).
`spec/payloads.md` should draw this outer/inner distinction explicitly — fragmentation and
presence are transport/discovery concerns, not Noise-payload TLV types.

---

## 1. Outer `MessageType` enumeration (transport envelope, `BitchatPacket.type`)

Swift: `localPackages/BitFoundation/Sources/BitFoundation/MessageType.swift:12-41`

| Value | Case | Description | Swift |
|---|---|---|---|
| `0x01` | `announce` | "I'm here" + nickname (presence) | `MessageType.swift:14` |
| `0x02` | `message` | Public chat message | `MessageType.swift:15` |
| `0x03` | `leave` | "I'm leaving" | `MessageType.swift:16` |
| `0x04` | `courierEnvelope` | Store-and-forward envelope carried by a trusted peer | `MessageType.swift:17` |
| `0x21` | `requestSync` | GCS filter-based sync request (local-only) | `MessageType.swift:18` |
| `0x10` | `noiseHandshake` | Handshake (init/response) | `MessageType.swift:21` |
| `0x11` | `noiseEncrypted` | All encrypted payloads (messages, receipts, etc.) — carries `NoisePayload` | `MessageType.swift:22` |
| `0x20` | `fragment` | Single fragment of a larger packet | `MessageType.swift:25` |
| `0x22` | `fileTransfer` | Binary file/audio/image payloads (public, not Noise) | `MessageType.swift:26` |
| `0x23` | `boardPost` | Signed geohash bulletin-board post or tombstone | `MessageType.swift:27` |
| `0x24` | `prekeyBundle` | Signed batch of one-time prekeys (gossiped) | `MessageType.swift:28` |
| `0x25` | `groupMessage` | Group-encrypted broadcast (cleartext group ID, ChaChaPoly body) | `MessageType.swift:29` |
| `0x26` | `ping` | Directed echo request (mesh diagnostics) | `MessageType.swift:32` |
| `0x27` | `pong` | Directed echo reply | `MessageType.swift:33` |
| `0x28` | `nostrCarrier` | Signed Nostr event ferried mesh↔gateway | `MessageType.swift:37` |
| `0x29` | `voiceFrame` | Public live push-to-talk burst (signed, unencrypted broadcast) | `MessageType.swift:41` |

Kotlin `MessageType`: `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:12-28`.
Android defines **only**: `ANNOUNCE(0x01)`, `MESSAGE(0x02)`, `LEAVE(0x03)`, `NOISE_HANDSHAKE(0x10)`,
`NOISE_ENCRYPTED(0x11)`, `FRAGMENT(0x20)`, `REQUEST_SYNC(0x21)`, `FILE_TRANSFER(0x22)`,
`VOICE_FRAME(0x29)`.

**Divergence:** Android's outer `MessageType` is missing `courierEnvelope (0x04)`,
`boardPost (0x23)`, `prekeyBundle (0x24)`, `groupMessage (0x25)`, `ping (0x26)`, `pong (0x27)`,
and `nostrCarrier (0x28)`. These correspond to newer iOS-only features (store-and-forward
courier delivery, geohash bulletin board, async-first-contact prekey bundles, private groups,
mesh diagnostics, and the Nostr gateway carrier) that have not yet been ported to
bitchat-android. The 9 types Android does define match the iOS values exactly.

---

## 2. Inner `NoisePayloadType` enumeration (carried inside `noiseEncrypted`)

Swift: `bitchat/Protocols/BitchatProtocol.swift:76-130`. Wrapper struct
`bitchat/Models/NoisePayload.swift` (`encode()` lines 17-22 prefixes 1 type byte + data;
`decode()` lines 25-40 reads the first byte as type, rest as data).

| Value | Case | Description | Swift |
|---|---|---|---|
| `0x01` | `privateMessage` | Private chat message (TLV, see §3.1) | `BitchatProtocol.swift:78` |
| `0x02` | `readReceipt` | Message was read | `BitchatProtocol.swift:79` |
| `0x03` | `delivered` | Message was delivered | `BitchatProtocol.swift:80` |
| `0x06` | `groupInvite` | Creator-signed group state (invite) | `BitchatProtocol.swift:82` |
| `0x07` | `groupKeyUpdate` | Creator-signed group state (key rotation/roster update) | `BitchatProtocol.swift:83` |
| `0x08` | `voiceFrame` | One live push-to-talk burst packet, private (see §3.2 `VoiceBurstPacket`) | `BitchatProtocol.swift:85` |
| `0x20` | `privateFile` | Finalized private media; encrypted `BitchatFilePacket` (see §3.3) | `BitchatProtocol.swift:89` |
| `0x21` | `authenticatedPeerState` | Versioned peer state (capabilities + signing key) proven inside the Noise session (see §3.4) | `BitchatProtocol.swift:94` |
| `0x10` | `verifyChallenge` | QR-based OOB verification challenge | `BitchatProtocol.swift:96` |
| `0x11` | `verifyResponse` | QR-based OOB verification response | `BitchatProtocol.swift:97` |
| `0x12` | `vouch` | Batch of transitive (web-of-trust) vouch attestations | `BitchatProtocol.swift:99` |

Compatibility alias: raw value `0x09` decodes to `.privateFile` (`prereleasePrivateFileRawValue`,
`BitchatProtocol.swift:101-108`) — a short-lived prerelease encoding (#1434) that decoders must
still accept but must never emit; `isPrivateFile(rawValue:)` canonicalizes both `0x09` and `0x20`.

Kotlin `NoisePayloadType`: `app/src/main/java/com/bitchat/android/model/NoiseEncrypted.kt:20-46`.
Defines: `PRIVATE_MESSAGE(0x01)`, `READ_RECEIPT(0x02)`, `DELIVERED(0x03)`, `VOICE_FRAME(0x08)`,
`VERIFY_CHALLENGE(0x10)`, `VERIFY_RESPONSE(0x11)`, `FILE_TRANSFER(0x20)` (Android's name for
`privateFile`), `PEER_STATE(0x21)` (Android's name for `authenticatedPeerState`). Android's
`fromValue()` (`NoiseEncrypted.kt:38-44`) implements the same `0x09`→`FILE_TRANSFER` prerelease
alias as Swift, decode-only, matching the comment at `NoiseEncrypted.kt:33-35`.

**Divergence:** Android's `NoisePayloadType` is missing `groupInvite (0x06)`,
`groupKeyUpdate (0x07)`, and `vouch (0x12)` — the same private-groups and web-of-trust features
flagged missing at the outer-`MessageType` layer in §1. Every value Android *does* define is
byte-identical in tag and semantics to Swift (naming differs only cosmetically: `FILE_TRANSFER`
↔ `privateFile`, `PEER_STATE` ↔ `authenticatedPeerState`). The `NoisePayload` 1-byte-type-prefix
wrapper format itself matches exactly:
`app/src/main/java/com/bitchat/android/model/NoiseEncrypted.kt:52-89` (`encode()`/`decode()`).

Construction/dispatch of these types on iOS: `bitchat/Services/BLE/BLENoisePayloadFactory.swift`
— `privateMessage()` (lines 4-10), `readReceipt()` (12-14, raw UTF-8 `originalMessageID` bytes,
**no TLV**), `delivered()` (16-18, raw UTF-8 `messageID` bytes, **no TLV**), `privateFile()`
(20-23), `authenticatedPeerState()` (25-28), and the shared `typedPayload()` helper (30-34) that
prepends the 1-byte type tag. Kotlin mirrors the raw-UTF8-bytes encoding for ack payloads exactly:
`app/src/main/java/com/bitchat/android/mesh/MessageHandler.kt:233-244` (DELIVERED: `messageID.toByteArray(Charsets.UTF_8)`)
and `app/src/main/java/com/bitchat/android/mesh/BluetoothMeshService.kt:1131-1136` (READ_RECEIPT,
same encoding). **Verdict: match** for `readReceipt`/`delivered` wire bytes.

---

## 3. TLV encodings of individual Noise-payload types

### 3.1 `privateMessage` (0x01) — `PrivateMessagePacket`

Swift: `bitchat/Protocols/Packets.swift:242-297`. TLV, 1-byte type + 1-byte length (max 255
per field):
- `0x00`: `messageID` (UTF-8) — `Packets.swift:247, 256-259`
- `0x01`: `content` (UTF-8) — `Packets.swift:248, 262-265`

Kotlin: `app/src/main/java/com/bitchat/android/model/NoiseEncrypted.kt:114-209`. Same two TLV
tags (`MESSAGE_ID = 0x00`, `CONTENT = 0x01`, `NoiseEncrypted.kt:123-125`), same 1-byte
type + 1-byte length framing (`encode()`: `NoiseEncrypted.kt:138-160`; `decode()`:
`NoiseEncrypted.kt:166-203`). **Verdict: match**, byte-for-byte.

### 3.2 `voiceFrame` (0x08, private) — `VoiceBurstPacket`

Swift: `bitchat/Protocols/VoiceBurstPacket.swift:20-31` (doc comment) and `:33-46` (struct).
Fixed-field wire format, not TLV: `[burstID: 8 bytes][seq: UInt16 BE][flags: UInt8][payload…]`.
Flags: `0x01` START → payload `[codec: UInt8]`; `0x02` END → payload
`[totalDataPackets: UInt16 BE][durationMs: UInt32 BE]`; `0x04` CANCELED → empty payload;
`0x00` data → repeated `[length: UInt16 BE][AAC frame]`. Codec `0x01 = aacLC16kMono` (AAC-LC,
16 kHz mono, ~16 kbps) — `VoiceBurstPacket.swift:13-18`.

Kotlin: `app/src/main/java/com/bitchat/android/features/voice/VoiceBurstPacket.kt:1-70+`. Doc
comment explicitly states "Wire format (shared with iOS)" (`VoiceBurstPacket.kt:15-19`); same
codec enum (`AAC_LC_16K_MONO(0x01)`, `VoiceBurstPacket.kt:7-8`); `encode()`
(`VoiceBurstPacket.kt:39-6x`) writes burstID, then big-endian `seq`, then the same flag bytes
and per-kind payload layout. **Verdict: match**, byte-for-byte (this packet type is shared
between the private Noise-wrapped burst and the public unencrypted `MessageType.voiceFrame`
broadcast; both platforms use one `VoiceBurstPacket` codec for both cases per
`VoiceBurstPacket.swift:20-22`).

### 3.3 `privateFile` (0x20) — `BitchatFilePacket`

Swift: `bitchat/Protocols/BitchatFilePacket.swift:15-156`. TLV, 1-byte type; length field is
2 bytes big-endian **except** `content` which uses a 4-byte big-endian length
(`encode()`: lines 31-66; `TLVType`: lines 22-27):
- `0x01` `fileName` (UTF-8, optional) — 2-byte length
- `0x02` `fileSize` (4-byte big-endian `UInt32`, canonical v2) — 2-byte length field = `4`
- `0x03` `mimeType` (UTF-8, optional) — 2-byte length
- `0x04` `content` (opaque bytes) — 4-byte big-endian length prefix

Decoder tolerance (backward compatibility): accepts legacy `fileSize` TLVs of length `8`
(old 8-byte size field) as well as the canonical `4` (`BitchatFilePacket.swift:124-133`), and
falls back from the canonical 4-byte `content` length to a legacy 2-byte length if the 4-byte
read doesn't fit the remaining bytes (`BitchatFilePacket.swift:98-111`).

Kotlin: `app/src/main/java/com/bitchat/android/model/BitchatFilePacket.kt:1-178`. Same 4 TLV
tags (`FILE_NAME/FILE_SIZE/MIME_TYPE/CONTENT = 0x01..0x04`, lines 34-45); `encode()` (lines
47-97) writes `fileSize` as a 4-byte value behind a 2-byte length field (line 62 comment:
"UInt32 for FILE_SIZE (changed from 8 bytes)") and `content` behind a 4-byte length prefix
(lines 85-88) — matches Swift's canonical v2 encoding exactly.

**Divergence (decode-only, one-directional):** Kotlin's `decode()` **rejects** the legacy
8-byte `fileSize` encoding (`BitchatFilePacket.kt:146-150`: `if (len != 4) return null`) and
has **no fallback** to a legacy 2-byte `content` length (`BitchatFilePacket.kt:118-128`
always reads a 4-byte content length). Swift's decoder is strictly more permissive than
Android's here: a file packet built by a very old iOS client using the legacy 8-byte
fileSize/2-byte content-length encoding would decode fine on current iOS but be rejected by
current Android. Current-iOS-canonical-encoding output (4-byte fileSize, 4-byte content
length) is unaffected and round-trips identically on both platforms. Unknown-TLV skip
behavior matches on both sides (Swift: `case nil: continue`, `BitchatFilePacket.swift:142-143`;
Kotlin: explicit comment + skip, `BitchatFilePacket.kt:130-141`).

Size ceilings referenced by the encoder: `FileTransferLimits.maxPayloadBytes = 1 MiB`,
`maxVoiceNoteBytes = 512 KiB`, `maxImageBytes = 512 KiB` —
`localPackages/BitFoundation/Sources/BitFoundation/FileTransferLimits.swift:2-8`. Not
independently re-verified against a matching Android constants file in this pass.

### 3.4 `authenticatedPeerState` (0x21) — `AuthenticatedPeerStatePacket`

Swift: `bitchat/Protocols/Packets.swift:159-240`. Wire format: `[version=0x01][TLV]...`, 1-byte
type + 1-byte length:
- `0x01`: canonical minimal little-endian `PeerCapabilities` (1-8 bytes) — `Packets.swift:180, 190-192`
- `0x02`: 32-byte Ed25519 signing public key — `Packets.swift:181, 193-195`

Decoder rejects duplicate TLVs, non-canonical capability re-encodings, and unknown versions
(`Packets.swift:199-238`); unknown TLV tags are skipped (`Packets.swift:215-217`).

Kotlin: `app/src/main/java/com/bitchat/android/model/AuthenticatedPeerState.kt:1-87`. Identical
wire format documented verbatim in its doc comment (lines 4-11): `[version=0x01][type=0x01]
[len=1..8][minimal LE capabilities][type=0x02][len=32][Ed25519 public key]`. Same TLV tag
values (`CAPABILITIES_TLV = 0x01`, `SIGNING_PUBLIC_KEY_TLV = 0x02`, `SIGNING_PUBLIC_KEY_SIZE = 32`,
lines 37-39), same duplicate/canonical-re-encode rejection (lines 57-61). **Verdict: match**,
byte-for-byte, including validation strictness.

### 3.5 `PeerCapabilities` bitfield (carried inside `authenticatedPeerState` TLV `0x01`, and also
inside the outer plaintext `AnnouncementPacket` TLV `0x05`)

Swift: `localPackages/BitFoundation/Sources/BitFoundation/PeerCapabilities.swift:9-65`. Minimal
little-endian bitfield, at least 1 byte, trailing zero bytes dropped (`encoded()`: lines 46-54);
decodes any length, keeping only the low 64 bits (`init(encoded:)`: lines 58-64). Bits defined:

| Bit | Name | Swift line |
|---|---|---|
| 0 | `prekeys` | `PeerCapabilities.swift:16` |
| 1 | `wifiBulk` | `PeerCapabilities.swift:17` |
| 2 | `gateway` | `PeerCapabilities.swift:18` |
| 3 | `groups` | `PeerCapabilities.swift:19` |
| 4 | `board` | `PeerCapabilities.swift:20` |
| 5 | `vouch` | `PeerCapabilities.swift:21` |
| 6 | `meshDiagnostics` | `PeerCapabilities.swift:22` |
| 7 | `bridge` | `PeerCapabilities.swift:26` |
| 8 | `privateMedia` | `PeerCapabilities.swift:30` |
| 9 | `privateMediaReceipts` | `PeerCapabilities.swift:37` |
| 10 | `nonDestructiveNoiseReplacement` (reserved, never advertised) | `PeerCapabilities.swift:41-42` |

Kotlin: `app/src/main/java/com/bitchat/android/model/PeerCapabilities.kt:1-50`. Same minimal
LE bitfield encode/decode (`encoded()` lines 19-27, `decode()` lines 42-48, matching Swift's
algorithm exactly). **But** Android only *defines* bit 8: `PRIVATE_MEDIA = PeerCapabilities(1L
shl 8)` (line 33), and `LOCAL_SUPPORTED = PRIVATE_MEDIA` (line 36) — no named constants for
bits 0-7, 9, or 10.

**Divergence:** the wire encoding (bitfield format) matches exactly, but Android only advertises
and recognizes the `privateMedia` (bit 8) capability by name; it has no constants for
`prekeys`, `wifiBulk`, `gateway`, `groups`, `board`, `vouch`, `meshDiagnostics`, or `bridge`.
This is consistent with §1/§2's findings — those capabilities gate the same newer features
(prekey bundles, private groups, board posts, vouch, mesh diagnostics, geohash bridging) that
are absent from Android's `MessageType`/`NoisePayloadType` enums. Because the bitfield decoder
preserves unknown high bits verbatim (both platforms), an Android peer that round-trips a
capabilities TLV it partially understands will not corrupt bits it doesn't recognize.

---

## 4. Types verified as outer-layer only (not inside Noise) — included for completeness

### 4.1 `announce` (0x01, "presence") — `AnnouncementPacket`

Swift: `bitchat/Protocols/Packets.swift:6-157`. Plaintext, self-signed TLV (1-byte type +
1-byte length): `0x01` nickname, `0x02` Noise static public key, `0x03` Ed25519 signing public
key, `0x04` direct neighbors (8-byte peer IDs, optional), `0x05` `PeerCapabilities` (optional),
`0x06` bridge rendezvous geohash cell (optional, ≤12 bytes) — tags at `Packets.swift:33-40`.
Not cross-checked against Kotlin in this pass (out of the Noise-payload scope this ticket
targets); flagged here only to correct the ticket's "presence is a Noise payload" framing (§0).

### 4.2 `fragment` (0x20) — outer fragment header

Swift: `bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:103-131`. Fixed-field (not TLV):
`[fragmentID: 8 bytes][index: UInt16 BE][total: UInt16 BE][original packet type: 1 byte]
[fragment data]`, wrapped as a new `BitchatPacket` with `type = MessageType.fragment`
(`BLEOutboundFragmentPlanner.swift:119-130`). This wraps the *entire* original packet
(including already-Noise-encrypted ones) — the fragment header itself is always outer/plaintext
framing, never inside the Noise payload. Byte-for-byte parity with
`app/src/main/java/com/bitchat/android/model/FragmentPayload.kt` (13-byte header: 8+2+2+1) was
already verified in the prior BLE-transport research ticket
(`docs/spec-research/ble-transport.md` §2.1, dsfx3d/bitchat#20); re-confirmed here via
`app/src/main/java/com/bitchat/android/mesh/FragmentManager.kt:90-91,188-189`, whose own
comments cite the matching iOS lines directly.

### 4.3 `courierEnvelope` (0x04) — `CourierEnvelope`

Swift: `localPackages/BitFoundation/Sources/BitFoundation/CourierEnvelope.swift:20-194`. TLV
(1-byte type + 2-byte big-endian length): `0x01` `recipientTag` (16 bytes, rotating
HMAC-SHA256-derived hint), `0x02` `expiry` (8-byte ms-since-epoch), `0x03` `ciphertext`
(opaque one-way Noise X ciphertext — the *payload* is Noise-sealed, but the envelope TLV
wrapping it is not itself carried inside a `noiseEncrypted` transport packet; it's its own
outer `MessageType`), `0x04` `copies` (1 byte, spray-and-wait budget, omitted when 1), `0x05`
`prekeyID` (4 bytes, optional, selects v2 forward-secret seal vs. v1 static-key seal) — tags at
`CourierEnvelope.swift:48-54`. **No Kotlin equivalent file found** in bitchat-android (searched
for `CourierEnvelope`/`courier` — no matches); this is an iOS-only, unported feature.

### 4.4 `prekeyBundle` (0x24) — `PrekeyBundle`

Swift: `localPackages/BitFoundation/Sources/BitFoundation/PrekeyBundle.swift:20-190`. TLV
(1-byte type + 2-byte big-endian length): `0x01` `noiseStaticPublicKey` (32 bytes), `0x02`
`prekeys` (repeated `[id: UInt32 BE][Curve25519 public key: 32 bytes]` entries, up to 8), `0x03`
`generatedAt` (8-byte ms-since-epoch), `0x04` `signature` (64-byte Ed25519 signature over
domain-separated canonical bytes) — tags at `PrekeyBundle.swift:52-57`. **No Kotlin equivalent
found** in bitchat-android (searched for `Prekey`/`prekey` — no matches); iOS-only, unported.

### 4.5 `vouch` / `groupInvite` / `groupKeyUpdate` payloads

No dedicated Swift source file matched these by name in the files this ticket scoped, beyond
their `NoisePayloadType` case declarations (§2); their TLV bodies were not read in this pass.
**No Kotlin equivalent found at all** (searched for `vouch`/`Vouch`, `groupInvite`, `GroupInvite`,
`groupKeyUpdate`, `GroupKeyUpdate` in bitchat-android — zero matches). Flagged for a follow-up
ticket if `spec/payloads.md` needs their exact TLV layout; the outer-enum-level divergence is
already captured in §1/§2.

### 4.6 Not wire formats (included in the ticket's file list, but no TLV to verify)

- `bitchat/Models/RequestSyncPacket.swift:1-138`: local-only sync-request TLV
  (`MessageType.requestSync = 0x21`), never carried inside Noise. Tags documented in the file's
  own header comment (lines 4-12). Android has a corresponding `GossipSyncManager.kt` /
  `GCSFilter` (`app/src/main/java/com/bitchat/android/sync/GossipSyncManager.kt`), but exact
  byte-for-byte TLV parity was **not verified** in this pass — out of scope for the
  Noise-payload chapter; flag for the sync-chapter ticket instead.
- `bitchat/Models/ReadReceipt.swift:12-96`: a separate `Codable`/binary-encodable model used for
  the **Nostr/geohash** read-receipt path (`toBinaryData()`/`fromBinaryData()`, lines 46-95) and
  local UI/persistence. This is *not* what rides inside the BLE mesh Noise payload — that path
  uses the raw-UTF8-messageID encoding described in §2 (`BLENoisePayloadFactory.readReceipt()`).
  Two different wire representations exist for "read receipt" depending on transport; worth
  calling out explicitly in `spec/payloads.md` so it isn't assumed to be one format.
- `localPackages/BitFoundation/Sources/BitFoundation/FileTransferLimits.swift:1-23`: constants
  only (no wire format of its own); referenced by §3.3.
- `bitchat/Models/BitchatMessage+Media.swift`: local filesystem path resolution for voice/image
  attachments (keyed off a `content` string prefix + `MimeType.Category`), not a network TLV.

---

## 5. Summary for spec/payloads.md drafting

1. Split the payloads chapter into **outer `MessageType`** (plaintext transport envelope) and
   **inner `NoisePayloadType`** (carried only inside `noiseEncrypted` packets) — the ticket's own
   framing conflated these (§0).
2. The inner `NoisePayloadType` set that should appear in the "message/payload TLV types" chapter:
   `privateMessage (0x01)`, `readReceipt (0x02)`, `delivered (0x03)`, `groupInvite (0x06)`,
   `groupKeyUpdate (0x07)`, `voiceFrame (0x08)`, `privateFile (0x20)`,
   `authenticatedPeerState (0x21)`, `verifyChallenge (0x10)`, `verifyResponse (0x11)`,
   `vouch (0x12)` — plus the `0x09`→`privateFile` decode-only compatibility alias.
3. **Swift/Kotlin divergence found** (not a byte-format bug — a feature-parity gap): Android's
   `bitchat-android` implements 8 of Swift's 11 `NoisePayloadType` cases and 9 of Swift's 15
   outer `MessageType` cases; every type Android *does* implement matches Swift's tag values and
   TLV layout byte-for-byte, **except** `BitchatFilePacket` decode leniency (§3.3: Android
   rejects two legacy encodings Swift still accepts for backward compatibility). Missing on
   Android: courier envelopes, prekey bundles, private groups (invite/key-update), vouch/web-of-
   trust, geohash board posts, mesh diagnostics ping/pong, and the Nostr gateway carrier — plus
   the corresponding `PeerCapabilities` bits that advertise them. These are read-only findings
   per #14; no fix is proposed or expected here.
