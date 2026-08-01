# Wire-Format Research Notes (for `spec/wire-format.md`)

Research ticket: dsfx3d/bitchat#18, off the wayfinder map dsfx3d/bitchat#16
("Map: Implementation of the bitchat protocol spec").

Purpose: capture the byte-level facts about the BitChat packet header and TLV
encoding, with file+line citations, so whoever drafts `spec/wire-format.md`
can lift facts directly from here. **The citation trail in this document is
internal research scaffolding — do not carry file+line citations into the
published spec prose** (per dsfx3d/bitchat#11).

Source of truth: `localPackages/BitFoundation` in this repo (Swift), the
canonical codec per jackjackbits. Cross-checked read-only against
`permissionlesstech/bitchat-android` (Kotlin), shallow-cloned to
`/tmp/bitchat-android-research-wireformat` at commit reachable via
`git clone --depth 1` on 2026-08-01. No obligation to fix bitchat-android
divergence (per dsfx3d/bitchat#14) — divergence is flagged, not patched.

---

## 1. Packet header layout (`BitchatPacket` / `BinaryProtocol`)

Swift source of truth:
- `localPackages/BitFoundation/Sources/BitFoundation/BitchatPacket.swift:15-38`
  (struct fields: `version, type, senderID, recipientID, timestamp, payload,
  signature, ttl, route, isRSR`)
- `localPackages/BitFoundation/Sources/BitFoundation/BinaryProtocol.swift`
  (encode: lines 132-253; decode: lines 256-408)

Kotlin cross-check:
- `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:37-176`
  (`BitchatPacket` data class + doc comment)
- `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:181-531`
  (`BinaryProtocol` object: encode 201-334, decode 336-530)

### 1.1 Fixed header (big-endian / network byte order throughout)

Two header sizes exist, selected by the `version` byte:

| Field | Offset (v1 & v2) | Size | Notes |
|---|---|---|---|
| `version` | 0 | 1 byte | `1` or `2`; any other value is rejected |
| `type` | 1 | 1 byte | `MessageType` raw value |
| `ttl` | 2 | 1 byte | hop limit, decremented on relay |
| `timestamp` | 3 | 8 bytes | `UInt64`, milliseconds since epoch, big-endian |
| `flags` | 11 | 1 byte | bitfield, see §1.2 |
| `payloadLength` | 12 | 2 bytes (v1) / 4 bytes (v2) | big-endian; payload-section length only (excludes route bytes) |

- Header size: **14 bytes for v1, 16 bytes for v2** (the extra 2 bytes come
  from the wider `payloadLength` field, not from a new fixed field).
  - Swift: `v1HeaderSize = 14`, `v2HeaderSize = 16` —
    `localPackages/BitFoundation/Sources/BitFoundation/BinaryProtocol.swift:100-101`,
    `headerSize(for:)` at lines 111-117.
  - Kotlin: `HEADER_SIZE_V1 = 14`, `HEADER_SIZE_V2 = 16` —
    `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:182-183`,
    `getHeaderSize()` at lines 194-199.
  - Both docstrings restate the same layout in prose:
    Swift `BinaryProtocol.swift:24-37`; Kotlin `BinaryProtocol.kt:40-53`.
- The `flags` byte's fixed offset of **11** is asserted directly in Swift as
  `Offsets.flags = 11  // After version(1)+type(1)+ttl(1)+timestamp(8)` —
  `localPackages/BitFoundation/Sources/BitFoundation/BinaryProtocol.swift:107-109`,
  and exercised by tests reading `encoded[BinaryProtocol.Offsets.flags]` —
  `localPackages/BitFoundation/Tests/BitFoundationTests/BinaryProtocolTests.swift:87-88,175,207,238`.
  Kotlin has no named offset constant but the encode order (version, type,
  ttl, 8-byte timestamp, flags) is identical —
  `BinaryProtocol.kt:238-260` — so the flags byte lands at the same offset 11.
- `payloadLength` encoding: Swift writes 2 bytes (`UInt16`) for v1 or 4 bytes
  (`UInt32`) for v2, big-endian, at `BinaryProtocol.swift:198-207`; Kotlin
  mirrors this with `buffer.putInt`/`buffer.putShort` in BIG_ENDIAN byte order
  at `BinaryProtocol.kt:262-272` (`ByteBuffer` is set to
  `ByteOrder.BIG_ENDIAN` at `BinaryProtocol.kt:235`).
- Max packet size implied by the 16-bit v1 length field: 65,535 bytes,
  documented at `BinaryProtocol.swift:59`.

### 1.2 Flags byte (bit 0 = LSB)

| Bit | Name | Swift | Kotlin |
|---|---|---|---|
| 0x01 | hasRecipient | `Flags.hasRecipient` — `BinaryProtocol.swift:124` | `Flags.HAS_RECIPIENT` — `BinaryProtocol.kt:188` |
| 0x02 | hasSignature | `Flags.hasSignature` — `BinaryProtocol.swift:125` | `Flags.HAS_SIGNATURE` — `BinaryProtocol.kt:189` |
| 0x04 | isCompressed | `Flags.isCompressed` — `BinaryProtocol.swift:126` | `Flags.IS_COMPRESSED` — `BinaryProtocol.kt:190` |
| 0x08 | hasRoute (v2+ only) | `Flags.hasRoute` — `BinaryProtocol.swift:127` | `Flags.HAS_ROUTE` — `BinaryProtocol.kt:191` |
| 0x10 | **isRSR** | `Flags.isRSR` — `BinaryProtocol.swift:128` | **absent** — see Divergence §4.1 |
| 0x20-0x80 | reserved | — | — |

### 1.3 Variable sections (in wire order, after the fixed header)

| Section | Size | Present when | Swift | Kotlin |
|---|---|---|---|---|
| `senderID` | 8 bytes, fixed (truncated/zero-padded) | always | `BinaryProtocol.swift:102,209-213` (`senderIDSize = 8`) | `BinaryProtocol.kt:184,274-279` (`SENDER_ID_SIZE = 8`) |
| `recipientID` | 8 bytes, fixed | `hasRecipient` flag set | `BinaryProtocol.swift:103,215-221` | `BinaryProtocol.kt:185,282-288` |
| `route` | 1 byte hop-count + N×8 bytes | `hasRoute` flag set (v2+ only); route bytes are **not** counted in `payloadLength` | `BinaryProtocol.swift:154-166,223-228` (encode), `343-355` (decode) | `BinaryProtocol.kt:231-234,290-298` (encode), `405-420,437-450` (decode) |
| `originalSize` preamble | 2 bytes (v1) / 4 bytes (v2) | `isCompressed` flag set; precedes the compressed payload bytes, counted inside `payloadLength` | `BinaryProtocol.swift:230-241` (encode), `358-371` (decode) | `BinaryProtocol.kt:301-310` (encode), `452-500` (decode) |
| `payload` | variable, `payloadLength` bytes total (incl. `originalSize` preamble if compressed) | always | `BinaryProtocol.swift:242` | `BinaryProtocol.kt:311` |
| `signature` | 64 bytes, fixed | `hasSignature` flag set | `BinaryProtocol.swift:104,244-246` (`signatureSize = 64`) | `BinaryProtocol.kt:186,314-316` |

- Padding: after building the unpadded frame, both implementations round up to
  an "optimal block size" and PKCS#7-style pad for traffic-analysis
  resistance — Swift `BinaryProtocol.swift:248-251` calls into
  `MessagePadding`; Kotlin `BinaryProtocol.kt:322-326` calls
  `MessagePadding.pad`. Decode tries the raw frame first, then strips padding
  and retries — Swift `BinaryProtocol.swift:256-263`; Kotlin
  `BinaryProtocol.kt:336-357`.
- Minimum valid frame size for decode: `v1HeaderSize + senderIDSize` = 14 + 8
  = 22 bytes — Swift `BinaryProtocol.swift:267` (note: the doc comment at
  line 61 says 21, which is stale relative to the actual guard on line 267);
  Kotlin's equivalent guard is `HEADER_SIZE_V1 + SENDER_ID_SIZE` at
  `BinaryProtocol.kt:367`.

### 1.4 Version / route notes

- `route` (source routing) is v2+ only. Swift explicitly zeroes it for v1 —
  `BinaryProtocol.swift:154` (`(version >= 2) ? (packet.route ?? []) : []`)
  and ignores the `hasRoute` flag bit when decoding a v1 packet —
  `BinaryProtocol.swift:320`. Kotlin matches — encode guard at
  `BinaryProtocol.kt:231,257`, decode guard at `BinaryProtocol.kt:389`.
- Route hop count is capped at 255 (fits the 1-byte count prefix) — Swift
  `BinaryProtocol.swift:163`; Kotlin coerces to 255 —
  `BinaryProtocol.kt:232,294`.

---

## 2. TLV encoding schemes

There are **two distinct TLV byte layouts** in the Swift codebase, both
called "TLV" in comments/names but with different length-field widths. Any
spec text should call this out explicitly rather than describing "the" TLV
format as one thing.

### 2.1 TLV-8: `[type: 1 byte][length: 1 byte][value: length bytes]`

Used for the announce/identity packet and its embedded gossip TLV. This one
lives in `bitchat/Protocols/Packets.swift` (app layer, not BitFoundation) and
in `localPackages/BitFoundation/Sources/BitFoundation/PeerCapabilities.swift`
(the capabilities bit-field payload carried inside one such TLV value).

- Swift `AnnouncementPacket`:
  `bitchat/Protocols/Packets.swift:6-157`. TLV types (lines 33-39):
  `nickname=0x01, noisePublicKey=0x02, signingPublicKey=0x03,
  directNeighbors=0x04, capabilities=0x05, bridgeGeohash=0x06`.
  Encode appends `[type][UInt8(count)][bytes]` per field, e.g. nickname at
  `Packets.swift:49-51`, capabilities at `Packets.swift:79-81`. Decode reads
  type (1 byte) then length (1 byte) then that many value bytes, skipping
  unrecognized types for forward compatibility —
  `Packets.swift:105-113,141-144`.
- Swift `PeerCapabilities` wire value (the bytes carried *inside* TLV
  `0x05`): a minimal little-endian bitfield, at least 1 byte, trailing zero
  bytes dropped —
  `localPackages/BitFoundation/Sources/BitFoundation/PeerCapabilities.swift:44-64`.
- Kotlin cross-check — `IdentityAnnouncement`:
  `app/src/main/java/com/bitchat/android/model/IdentityAnnouncement.kt:14-150`.
  TLV type enum only lists
  `NICKNAME=0x01, NOISE_PUBLIC_KEY=0x02, SIGNING_PUBLIC_KEY=0x03,
  CAPABILITIES=0x05` (lines 22-27) — `directNeighbors=0x04` and
  `bridgeGeohash=0x06` are **not** named constants on Android but are
  preserved verbatim as opaque `UnknownAnnouncementTLV(type, value)` entries
  (lines 52-67, 108-133) so round-tripping stays byte-compatible even though
  Android doesn't natively interpret those two TLV types. Encode/decode use
  the same `[type: 1][len: 1][value]` framing —
  `IdentityAnnouncement.kt:47-77` (encode), `86-149` (decode).
- Kotlin `PeerCapabilities.encoded()`: same minimal little-endian bitfield
  scheme as Swift — `app/src/main/java/com/bitchat/android/model/IdentityAnnouncement.kt:15-49`.
- Kotlin also has a standalone `GossipTLV` helper that independently encodes
  TLV type `0x04` (`DIRECT_NEIGHBORS_TYPE`) as `[0x04][len: 1][N*8 bytes of
  peer IDs]`, matching Swift's `directNeighbors` TLV byte-for-byte —
  `app/src/main/java/com/bitchat/android/services/meshgraph/GossipTLV.kt:1-77`
  (constant at line 11, encode at 16-24, decode at 30-54).

### 2.2 TLV-16: `[type: 1 byte][length: 2 bytes big-endian][value: length bytes]`

Used for `PrekeyBundle` and `CourierEnvelope`, both in
`localPackages/BitFoundation`. **No Kotlin equivalent exists yet** in
bitchat-android as of this research (see Divergence §4.2) — these are newer
BitFoundation-only payloads.

- `PrekeyBundle` (MessageType `0x24`):
  `localPackages/BitFoundation/Sources/BitFoundation/PrekeyBundle.swift`.
  TLV types (lines 52-57):
  `noiseStaticPublicKey=0x01, prekeys=0x02, generatedAt=0x03, signature=0x04`.
  Encode appends `[type][UInt16 big-endian length][value]` per field —
  lines 103-117 (`appendBE` helper at 192-195 writes big-endian). Decode
  reads type (1 byte), then a 2-byte big-endian length, then that many value
  bytes, tolerating unknown types — lines 131-141,165-167.
- `CourierEnvelope`:
  `localPackages/BitFoundation/Sources/BitFoundation/CourierEnvelope.swift`.
  TLV types (lines 48-54):
  `recipientTag=0x01, expiry=0x02, ciphertext=0x03, copies=0x04,
  prekeyID=0x05`. Same `[type][UInt16 BE length][value]` framing — encode at
  lines 89-115, decode at lines 130-160. `copies` and `prekeyID` TLVs are
  omitted entirely when they hold their default value, specifically so the
  encoded bytes stay identical to the older pre-spray / pre-prekey wire
  format — comments at lines 101-102, 109-110.

---

## 3. Message-type byte (top-level `type` field, header offset 1)

- Swift `MessageType`:
  `localPackages/BitFoundation/Sources/BitFoundation/MessageType.swift:12-42`.
  Full set: `announce=0x01, message=0x02, leave=0x03, courierEnvelope=0x04,
  requestSync=0x21, noiseHandshake=0x10, noiseEncrypted=0x11, fragment=0x20,
  fileTransfer=0x22, boardPost=0x23, prekeyBundle=0x24, groupMessage=0x25,
  ping=0x26, pong=0x27, nostrCarrier=0x28, voiceFrame=0x29`.
- Kotlin `MessageType`:
  `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:12-28`.
  Only defines `ANNOUNCE=0x01, MESSAGE=0x02, LEAVE=0x03, NOISE_HANDSHAKE=0x10,
  NOISE_ENCRYPTED=0x11, FRAGMENT=0x20, REQUEST_SYNC=0x21, FILE_TRANSFER=0x22,
  VOICE_FRAME=0x29`. See Divergence §4.3 — this is a feature/type-coverage
  gap, not a conflicting byte value (every value Kotlin does define agrees
  with Swift's assignment for that same byte).

---

## 4. Swift/Kotlin divergence found

Divergence **was found**. None of it appears to break basic decode
compatibility (unknown flag bits and unknown TLV types are both designed to
be ignored), but each item below is a real gap between the two codebases as
of this research (bitchat-android shallow-cloned 2026-08-01,
`/tmp/bitchat-android-research-wireformat`).

### 4.1 Header flags: `isRSR` bit (0x10) is Swift-only

Swift defines and sets/reads a fifth flag bit, `Flags.isRSR = 0x10`
(`BinaryProtocol.swift:128`, set at line 195, read at line 321), carried on
`BitchatPacket.isRSR` (`BitchatPacket.swift:25`). Kotlin's `Flags` object
(`BinaryProtocol.kt:187-192`) defines only `HAS_RECIPIENT, HAS_SIGNATURE,
IS_COMPRESSED, HAS_ROUTE` — there is no `isRSR`-equivalent bit or field on
Kotlin's `BitchatPacket` data class (`BinaryProtocol.kt:55-65`). A packet
with bit 0x10 set, decoded by bitchat-android, will silently lose that bit
(it isn't masked into anything Android reads), so this degrades rather than
breaks decoding.

### 4.2 TLV-16 payloads (`PrekeyBundle`, `CourierEnvelope`) have no Kotlin counterpart

`grep -rli "courierenvelope\|prekeybundle"` over bitchat-android's
`app/src/main/java/` returned no matches. `MessageType.prekeyBundle = 0x24`
and `MessageType.courierEnvelope = 0x04` (Swift) have no corresponding
Kotlin `MessageType` entries either (see §4.3). These are BitFoundation-only
wire payloads at the time of this research.

### 4.3 `MessageType` enum coverage gap

Swift's `MessageType` (16 cases) is a superset of Kotlin's `MessageType` (9
cases). Missing on Kotlin: `courierEnvelope=0x04, boardPost=0x23,
prekeyBundle=0x24, groupMessage=0x25, ping=0x26, pong=0x27,
nostrCarrier=0x28`. Every byte value Kotlin *does* define matches Swift's
assignment for that value, so there's no conflicting reuse of a type byte —
Android just hasn't implemented these message kinds yet.

### 4.4 `AnnouncementPacket` TLV types 0x04/0x06 not named on Kotlin's `IdentityAnnouncement`

See §2.1 — Android's `IdentityAnnouncement` TLV enum omits `directNeighbors
(0x04)` and `bridgeGeohash (0x06)` as named cases, but its decoder is
forward-compatible (unknown TLVs preserved verbatim), and a separate
`GossipTLV` helper independently reimplements the `0x04` framing
byte-for-byte. Net effect: wire-compatible, but the type is split across two
files on the Kotlin side instead of one, and `bridgeGeohash (0x06)` has no
Kotlin encoder/decoder at all (opaque passthrough only).

---

## 5. Suggested facts to lift into `spec/wire-format.md`

(No citations in the published spec — see the note at the top of this file.)

- Two packet versions, v1 (14-byte header) and v2 (16-byte header); the only
  structural difference is the width of the payload-length field (2 vs 4
  bytes) and the availability of the optional `route` section.
- Fixed header field order: version, type, ttl, timestamp (8 bytes BE),
  flags, payload length (BE).
- All multi-byte integers are big-endian (network byte order).
- Flags bitfield: hasRecipient, hasSignature, isCompressed, hasRoute (v2+),
  and — Swift/BitFoundation only, flagged as a cross-client gap — isRSR.
- Variable section order after the header: senderID (8B), recipientID (8B,
  optional), route (optional, v2+, 1B count + 8B/hop), payload (optionally
  prefixed by a 2B/4B original-size field when compressed), signature (64B,
  optional).
- Two TLV framings exist in the protocol family: a 1-byte-length TLV (used
  by the announce/identity packet and gossip neighbor lists) and a
  2-byte-big-endian-length TLV (used by prekey bundles and courier
  envelopes). Spec should name these distinctly rather than describing one
  universal TLV format.
- Both TLV framings use an unknown-type-skip decoding rule for forward
  compatibility.
