# Noise chapter research: handshake payload mappings, pattern, framing

Research for issue #22 ("Noise chapter: verify handshake payload mappings
(Swift + Kotlin)") on the `spec/noise.md` map (issue #16). Source of truth is
this repo's Swift Noise implementation; cross-checked (read-only) against
`permissionlesstech/bitchat-android`'s Kotlin implementation. Every claim below
cites file+line in both trees as of the commit/clone used for this research:

- Swift: `dsfx3d/bitchat` at commit `0152196` (branch `main`, this worktree).
- Kotlin: `permissionlesstech/bitchat-android`, shallow clone
  (`git clone --depth 1`) taken 2026-08-01 into a scratch directory (not
  committed anywhere; read-only reference, per #14).

## 1. Pattern

**Noise_XX_25519_ChaChaPoly_SHA256** — DH: Curve25519 (X25519), cipher:
ChaCha20-Poly1305 AEAD, hash: SHA-256.

- Swift: pattern enum and message-pattern table —
  `bitchat/Noise/NoiseProtocol.swift:92-97` (`enum NoisePattern { case XX, IK, NK, X }`),
  protocol name construction `bitchat/Noise/NoiseProtocol.swift:115-124`
  (`Noise_\(pattern)_25519_ChaChaPoly_SHA256`), XX message pattern table
  `bitchat/Noise/NoiseProtocol.swift:912-919`.
- Swift session always requests `.XX` for interactive peer sessions:
  `bitchat/Noise/NoiseSession.swift:57-63` (initiator) and
  `bitchat/Noise/NoiseSession.swift:88-94` (responder).
- Kotlin: protocol name constant —
  `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:24`
  (`PROTOCOL_NAME = "Noise_XX_25519_ChaChaPoly_SHA256"`), passed to the
  southernstorm `HandshakeState` at
  `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:244`.

**Match: identical pattern and primitive suite on both platforms.**

### One-way X pattern (courier / prekey envelopes) — Swift-only extension, not yet in Kotlin

The Swift client also uses Noise **X** (one-way, single message, responder
static known in advance) for two store-and-forward use cases layered on top of
the interactive XX session:
- Courier envelopes sealed to a peer's long-term static key:
  `bitchat/Services/NoiseEncryptionService.swift:434-475`
  (`sealCourierPayload` / `openCourierPayload`, pattern `.X` at line 450/463).
- Forward-secret one-time-prekey envelopes (X where the responder static is a
  gossiped ephemeral prekey, not the identity key):
  `bitchat/Services/NoiseEncryptionService.swift:492-537`
  (`sealPrekeyPayload` / `openPrekeyPayload`, pattern `.X` at line 502/524).
- The `X` pattern's single message pattern (`[.e, .es, .s, .ss]`) is defined at
  `bitchat/Noise/NoiseProtocol.swift:930-933`.

A grep of the bitchat-android clone (`grep -rn Noise_X app/`, and the full
`noise/` package) found **no equivalent one-way X-pattern envelope path** —
Kotlin's `NoiseSession.kt` only constructs `HandshakeState` with the XX
protocol name (`app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:24,244`).
This is a real Swift/Kotlin divergence: courier/prekey store-and-forward mail
sealed by an iOS/macOS sender cannot yet be produced or opened by the Android
client's Noise layer (whether it has an equivalent courier feature built a
different way was not investigated — out of scope for this read-only
cross-check). **Flagging for #14, not fixing.**

## 2. Handshake message sequence (XX)

```
Initiator                              Responder
---------                              ---------
-> e                                   (32-byte ephemeral public key)
<- e, ee, s, es                        (32-byte ephemeral + DH + 48-byte encrypted static [32 + 16-byte Poly1305 tag] + DH)
-> s, se                               (48-byte encrypted static [32 + 16-byte tag] + DH)
```

- Swift message-pattern table (source of truth for the sequence):
  `bitchat/Noise/NoiseProtocol.swift:912-919`:
  ```
  case .XX:
      return [
          [.e],               // -> e
          [.e, .ee, .s, .es], // <- e, ee, s, es
          [.s, .se]           // -> s, se
      ]
  ```
- `writeMessage`/`readMessage` token interpreters that turn each pattern token
  into wire bytes and DH/MixHash/MixKey operations:
  `bitchat/Noise/NoiseProtocol.swift:614-780`.
- Message-size constants confirming the byte counts above (32-byte XX message
  1, 2048-byte handshake ceiling to accommodate message 2/3 plus any embedded
  payload): `bitchat/Noise/NoiseSecurityConstants.swift:35-38`
  (`maxHandshakeMessageSize = 2048`, `xxInitialMessageSize = 32`).
- Kotlin mirrors the same sequence and exact byte sizes explicitly:
  `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:30-33`:
  ```
  private const val XX_MESSAGE_1_SIZE = 32      // -> e
  private const val XX_MESSAGE_2_SIZE = 96      // <- e, ee, s, es (32 + 48) + 16 (MAC)
  private const val XX_MESSAGE_3_SIZE = 48      // -> s, se
  ```
  (Kotlin's message 3 comment "+ 16 (MAC)" is on message 2's line only; message
  3's 48 bytes = 32-byte encrypted static + 16-byte tag, same shape as
  message 2's static-key field.)
- Kotlin handshake driver: `startHandshake()` writes message 1
  (`app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:274-311`);
  `processHandshakeMessage()` reads an inbound message and, depending on the
  southernstorm library's reported `action`, either writes the next message or
  completes (`app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:317-376`).

**Match: identical 3-message XX flow, identical per-message byte sizes.**

## 3. Handshake payload → packet/TLV field mapping

### Wire-level packet types carrying handshake bytes

Both platforms use the same two message-type byte values for the outer
transport packet:

| Wire byte | Swift | Kotlin |
|---|---|---|
| `0x10` | `MessageType.noiseHandshake` — `localPackages/BitFoundation/Sources/BitFoundation/MessageType.swift:21` | `MessageType.NOISE_HANDSHAKE` — `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:16` |
| `0x11` | `MessageType.noiseEncrypted` — `localPackages/BitFoundation/Sources/BitFoundation/MessageType.swift:22` | `MessageType.NOISE_ENCRYPTED` — `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:17` |

The raw Noise handshake message bytes (whatever `writeMessage`/`readMessage`
produced — see §2) are carried **verbatim, unwrapped**, as the
`BitchatPacket.payload` of a `noiseHandshake` (`0x10`) packet — there is no
extra handshake-specific TLV framing on top of the Noise wire format itself:

- Swift: handshake initiation returns raw handshake bytes directly, comment
  states "The Noise protocol handles its own message format" —
  `bitchat/Services/NoiseEncryptionService.swift:713-719` (`initiateHandshake`)
  and `:838-847` (`processHandshakeMessageWithResult`, "we process the raw
  data directly without NoiseMessage wrapper").
  The response packet is built directly from that raw payload:
  `bitchat/Services/BLE/BLENoisePacketHandler.swift:112-130`
  (`env.processHandshakeMessage(...)` → `BitchatPacket(type:
  MessageType.noiseHandshake.rawValue, ..., payload: response, ...)`).
- Kotlin: `startHandshake()` / `processHandshakeMessage()` return the raw
  `HandshakeState.writeMessage`/`readMessage` byte buffers with no additional
  wrapper — `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:275-376`.

There *is* a `NoiseMessage` Codable/binary wrapper struct defined in Swift
(`type`/`sessionID`/`payload`, with both JSON and binary `toBinaryData()`
encodings) — `bitchat/Services/NoiseEncryptionService.swift:1103-1173`
(`NoiseMessageType` enum) and `:1116-1173` (`NoiseMessage` struct). **This
wrapper is dead code for the current wire protocol**: nothing in
`bitchat/Services/BLE/` constructs or parses a `NoiseMessage` for actual
handshake/transport traffic (only the raw payload path above is used); it
appears to be a leftover from an earlier wire format. Do not cite
`NoiseMessage`/`NoiseMessageType` as the live payload mapping in `spec/noise.md`.

### Application payload TLV inside the post-handshake transport (`0x11 noiseEncrypted`)

Once the session is established, the *decrypted* Noise transport plaintext
itself carries a 1-byte type tag (`NoisePayloadType`) followed by
type-specific bytes — this is the "TLV" that actually matters for the spec:

- Swift type enum: `bitchat/Protocols/BitchatProtocol.swift:76-113`
  (`enum NoisePayloadType: UInt8`) — values: `privateMessage=0x01`,
  `readReceipt=0x02`, `delivered=0x03`, `groupInvite=0x06`,
  `groupKeyUpdate=0x07`, `voiceFrame=0x08`, `privateFile=0x20`,
  `authenticatedPeerState=0x21`, `verifyChallenge=0x10`,
  `verifyResponse=0x11`, `vouch=0x12`. (`0x09` is accepted as a decode-only
  legacy alias for `privateFile`: `bitchat/Protocols/BitchatProtocol.swift:101-108`.)
- Swift encode/decode helper: `bitchat/Models/NoisePayload.swift:12-41`
  (`struct NoisePayload { type; data }`, `encode()` prepends `type.rawValue`,
  `decode(_:)` reads first byte as type, rest as payload).
- Swift construction call sites for each typed payload:
  `bitchat/Services/BLE/BLENoisePayloadFactory.swift:4-35`
  (`privateMessage`, `readReceipt`, `delivered`, `privateFile`,
  `authenticatedPeerState`, generic `typedPayload(_:payload:)` at line 30-34).
- Swift dispatch on decrypt: `bitchat/Services/BLE/BLENoisePacketHandler.swift:216-242`
  (`handleEncrypted`) — decrypts, reads `decrypted[0]` as payload type
  (line 222-228), special-cases `.authenticatedPeerState` (line 232-239),
  otherwise delivers to UI via `deliverNoisePayload` (line 242).
- Kotlin mirrors this exactly: `NoisePayloadType` enum —
  `app/src/main/java/com/bitchat/android/model/NoiseEncrypted.kt:20-46` —
  values present: `PRIVATE_MESSAGE=0x01`, `READ_RECEIPT=0x02`,
  `DELIVERED=0x03`, `VOICE_FRAME=0x08`, `VERIFY_CHALLENGE=0x10`,
  `VERIFY_RESPONSE=0x11`, `FILE_TRANSFER=0x20`, `PEER_STATE=0x21` (with the
  same `0x09` legacy-decode alias for `FILE_TRANSFER`, lines 33-44).
  `NoisePayload` encode/decode: same file, lines 52-89 (`[type_byte][data]`,
  comment "exactly like iOS").

**Divergence found:** the Kotlin `NoisePayloadType` enum is missing
`groupInvite (0x06)`, `groupKeyUpdate (0x07)`, and `vouch (0x12)` that exist in
the current Swift enum
(`bitchat/Protocols/BitchatProtocol.swift:82-83,99`). These correspond to
newer Swift-side features (private groups, transitive web-of-trust vouching)
that were not found anywhere in the bitchat-android clone's `noise` or
`model` packages. This is a feature-parity gap, not a wire-format
incompatibility (the byte values `0x06/0x07/0x12` are simply unused/unknown on
Android today) — flag per #14, no fix expected here.

Inner sub-payload structures referenced by `NoisePayloadType` (`PrivateMessagePacket`
TLV encoding, `AuthenticatedPeerStatePacket`, `BitchatFilePacket`) were not
independently re-verified byte-for-byte in this pass; only the outer
type-tag + framing was in scope for #22. A follow-up ticket could verify those
inner encodings if `spec/noise.md` needs that level of detail.

## 4. Message framing after handshake completes (transport phase)

### Key derivation / split

- Swift: `NoiseHandshakeState.getTransportCiphers(useExtractedNonce:)` —
  `bitchat/Noise/NoiseProtocol.swift:861-875` — calls `symmetricState.split(...)`
  (`bitchat/Noise/NoiseProtocol.swift:481-494`, standard Noise HKDF split into
  two `NoiseCipherState`s) and returns `(send, receive, handshakeHash)`, with
  initiator using `c1` to send / `c2` to receive and vice versa for the
  responder (`:871-874`). Called with `useExtractedNonce: true` from
  `bitchat/Noise/NoiseSession.swift:110,136`.
- Kotlin: `activeHandshake.split()` (southernstorm library) at
  `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:405-407`,
  producing `sendCipher`/`receiveCipher` from the `CipherStatePair`.

### Transport ciphertext framing: `<4-byte big-endian nonce> || <ciphertext> || <16-byte Poly1305 tag>`

This is the key framing fact for the spec — post-handshake messages are
**not** using Noise's default implicit-nonce transport framing; both clients
prepend an explicit 4-byte big-endian nonce and validate it with a 1024-message
sliding-window replay window, rather than relying on strict monotonic nonce
ordering:

- Swift cipher: `bitchat/Noise/NoiseProtocol.swift:132-406` (`NoiseCipherState`).
  - Nonce size / replay window constants: `:134-137`
    (`NONCE_SIZE_BYTES = 4`, `REPLAY_WINDOW_SIZE = 1024`).
  - `encrypt(...)`: seals with ChaCha20-Poly1305 using an internal little-endian
    12-byte nonce built from the 8-byte counter (`:274-279`), then builds the
    wire payload as `<4-byte big-endian nonce> + ciphertext + tag` when
    `useExtractedNonce == true` (`:283-290`).
  - `decrypt(...)`: extracts the leading 4 bytes as nonce
    (`extractNonceFromCiphertextPayload`, `:225-247`), validates via sliding
    window (`isValidNonce`, `:171-188`), then opens the ChaChaPoly box
    (`:300-380`).
  - `useExtractedNonce` is always `true` for interactive-session transport
    ciphers (`bitchat/Noise/NoiseSession.swift:110,136` pass
    `useExtractedNonce: true` to `getTransportCiphers`).
  - Overhead constant confirming the framing shape: `transportCiphertextOverhead
    = 20` (4-byte nonce + 16-byte tag) — `bitchat/Noise/NoiseSecurityConstants.swift:16-18`.
  - Max plaintext size: `maxMessageSize = 65535` (64 KiB, per Noise spec) —
    `bitchat/Noise/NoiseSecurityConstants.swift:14`.
- Kotlin: identical scheme.
  - Constants: `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:39-41`
    (`NONCE_SIZE_BYTES = 4`, `REPLAY_WINDOW_SIZE = 1024`).
  - `encrypt(...)`: builds `<4-byte nonce> + ciphertext(+tag)` at
    `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:442-501`,
    explicit comment "Returns: <nonce><ciphertext> where nonce is 4 bytes
    (matching iOS implementation)" (line 440).
  - `decrypt(...)`: extracts nonce, validates sliding window, decrypts —
    `app/src/main/java/com/bitchat/android/noise/NoiseSession.kt:507-568`,
    comment "matching iOS implementation" (line 505).

**Match: identical transport framing, identical replay-window size (1024
messages), identical nonce size (4 bytes) and placement (prefix).**

### BLE-level padding of Noise frames (privacy, not Noise-protocol framing per se)

Both clients additionally pad the *outer* BLE frame (not the Noise ciphertext
itself) to one of the block sizes `{256, 512, 1024, 2048}` bytes with PKCS#7-style
padding, but only for `noiseHandshake`/`noiseEncrypted` packet types:

- Swift: block sizes `bitchat/Protocols/BitchatProtocol.swift:41-43` (doc
  comment) and `localPackages/BitFoundation/Sources/BitFoundation/MessagePadding.swift:24`
  (`blockSizes = [256, 512, 1024, 2048]`); which packet types get padded:
  `bitchat/Services/BLE/BLEOutboundPacketPolicy.swift:11-21`
  (`padsBLEFrame(for:)` — `true` only for `.noiseEncrypted`/`.noiseHandshake`).
- Kotlin: `app/src/main/java/com/bitchat/android/mesh/BLEPacketPaddingPolicy.kt:1-18`
  (`shouldPadForBLE` — same two types), block sizes in
  `app/src/main/java/com/bitchat/android/protocol/MessagePadding.kt`.

**Match.**

## 5. Outer packet header (shared by handshake and transport packets)

14-byte v1 header, 8-byte sender ID, 8-byte optional recipient ID — identical
on both platforms:

- Swift: `localPackages/BitFoundation/Sources/BitFoundation/BinaryProtocol.swift:100-103`
  (`v1HeaderSize = 14`, `senderIDSize = 8`, `recipientIDSize = 8`);
  `BitchatPacket` struct: `localPackages/BitFoundation/Sources/BitFoundation/BitchatPacket.swift:15-38`.
- Kotlin: header doc comment and `BitchatPacket` data class at
  `app/src/main/java/com/bitchat/android/protocol/BinaryProtocol.kt:33-46`+
  (v1 = 14 bytes, v2 = 16 bytes; senderID/recipientID 8 bytes each, matching
  the Swift v2 note in `bitchat/Noise/NoiseSecurityConstants.swift:24-27`
  `"v2 adds two length bytes"`).

## 6. Note on `BRING_THE_NOISE.md` (repo-root design doc)

Per the ticket instructions, `BRING_THE_NOISE.md` was read but treated as
prose-only, not a source of truth. Its high-level claims (XX pattern, 3-message
flow `-> e` / `<- e, ee, s, es` / `-> s, se`) match the current source exactly
(`BRING_THE_NOISE.md:18-26` vs. `bitchat/Noise/NoiseProtocol.swift:912-919`).
However, several of its code snippets are **stale** and do not match current
source — do not lift these into `spec/noise.md`:

- It shows `NoiseEncryptionService` holding a `channelEncryption =
  NoiseChannelEncryption()` property (`BRING_THE_NOISE.md:37-45`). No
  `NoiseChannelEncryption` type exists anywhere in `bitchat/` today (confirmed
  by repo-wide grep); the real class has no such property
  (`bitchat/Services/NoiseEncryptionService.swift:153-360`).
- It shows a `NoiseIdentityAnnouncement` struct with a `signature` field
  (`BRING_THE_NOISE.md:109-117`). No such type exists in `bitchat/` today;
  the actual mechanism is the announce-signature helper functions
  `buildAnnounceSignature`/`verifyAnnounceSignature` in
  `bitchat/Services/NoiseEncryptionService.swift:610-627`, plus the separate
  `authenticatedPeerState` Noise payload type (§3 above) for post-handshake
  capability/identity binding.
- Its `NoiseSession` sketch shows `remoteStaticKey` as a `let` constant
  (`BRING_THE_NOISE.md:47-56`); the real property is a mutable `var`
  populated only after the handshake completes
  (`bitchat/Noise/NoiseSession.swift:25,114-115,141`).

Interpretation: `BRING_THE_NOISE.md` is an early/illustrative design writeup
that predates several refactors (prekeys, group messaging, vouching,
authenticated-peer-state payload). Its pattern-level claims are still
accurate; its class-shape code samples are not and should not be cited in
`spec/noise.md`.

## Summary for `spec/noise.md` drafting

1. Pattern: `Noise_XX_25519_ChaChaPoly_SHA256` for interactive sessions; Noise
   `X` (one-way) for Swift-only courier/prekey store-and-forward envelopes
   (§1) — the X usage has no Kotlin equivalent found (divergence, #14).
2. XX sequence: `-> e` (32 B) / `<- e, ee, s, es` (96 B) / `-> s, se` (48 B) —
   identical Swift/Kotlin (§2).
3. Handshake bytes ride as the raw, unwrapped `payload` of a `noiseHandshake
   (0x10)` `BitchatPacket`; no extra TLV. Post-handshake application payloads
   are `[1-byte NoisePayloadType][type-specific bytes]` inside the decrypted
   plaintext of a `noiseEncrypted (0x11)` packet (§3). Swift has three
   payload types (`groupInvite 0x06`, `groupKeyUpdate 0x07`, `vouch 0x12`)
   that Kotlin's enum does not yet define (divergence, #14).
4. Transport framing: `<4-byte big-endian nonce> || ciphertext || 16-byte
   Poly1305 tag>`, 1024-message sliding-window replay protection — identical
   both platforms (§4). BLE-level PKCS#7 padding to `{256,512,1024,2048}`
   applies only to `noiseHandshake`/`noiseEncrypted` frames on both platforms.
5. No fixes were made to bitchat-android; divergences are flagged above for
   whoever triages #14.
