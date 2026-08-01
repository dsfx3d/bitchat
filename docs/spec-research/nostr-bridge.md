# Nostr Bridge Research Notes

Research for issue #28 (map: #16). Source of truth is this repo's Swift
implementation; cross-checked (read-only) against
`permissionlesstech/bitchat-android` at the commit fetched via
`git clone --depth 1` on 2026-08-01. Every claim below cites Swift file+line
and, where one exists, the corresponding Kotlin file+line. This file is scoped
for someone drafting `spec/nostr-bridge.md` to lift facts from directly.

## 1. Event kinds

Swift `NostrProtocol.EventKind` (`bitchat/Nostr/NostrProtocol.swift:21-37`):

| Kind | Name | Purpose |
|---|---|---|
| 0 | `metadata` | Declared, unused in the bridge flows below |
| 1 | `textNote` | Persistent geohash-tagged location note (NIP-01 text note) — `createGeohashTextNote`, `NostrProtocol.swift:428-455` |
| 5 | `deletion` | NIP-09 deletion request for a self-authored event — `createDeleteEvent`, `NostrProtocol.swift:460-473` |
| 13 | `seal` | Sender-signed inner layer of a private envelope (BitChat-proprietary payload, reuses NIP-59 kind number) — `NostrProtocol.swift:24-28`, `477-500` |
| 14 | `dm` | Unsigned "rumor" (innermost layer) of a private message — `NostrProtocol.swift:27` |
| 20000 | `ephemeralEvent` | Geohash public chat message, **and** mesh-bridge rendezvous message (same kind, distinguished by tags — see §3) — `NostrProtocol.swift:30`, `244-261`, `348-377` |
| 20001 | `geohashPresence` | Geohash presence heartbeat / mesh-bridge presence heartbeat (same kind, distinguished by tags) — `NostrProtocol.swift:31`, `319-335`, `382-395` |
| 1059 | `giftWrap` | Outer public envelope of a private message (one-time ephemeral key, reuses NIP-17 kind number) — `NostrProtocol.swift:29`, `502-530` |
| 1401 | `courierDrop` | Store-and-forward sealed Noise-X envelope parked on relays under a rotating recipient tag, used when the recipient is offline/unreachable — `NostrProtocol.swift:33-36`, `397-423` |

**Explicit non-compatibility note in the source** (`NostrProtocol.swift:10-17`): the
private-envelope construction (kinds 14/13/1059) is "deliberately BitChat-specific"
and **not** NIP-17/NIP-44/NIP-59 compatible, despite reusing those NIPs' kind
numbers and a `v2:` content-prefix naming convention. Encryption is
XChaCha20-Poly1305 with BitChat-specific HKDF parameters (`NostrProtocol.swift:583-587, 901-916`), not the NIP-44 ChaCha20 schedule.

**Kotlin cross-check** (`bitchat-android`, `app/src/main/java/com/bitchat/android/nostr/NostrEvent.kt:210-218`):
```
const val DIRECT_MESSAGE = 14
const val FILE_MESSAGE = 15
const val SEAL = 13
const val GIFT_WRAP = 1059
const val EPHEMERAL_EVENT = 20000
const val GEOHASH_PRESENCE = 20001
```
Android's `NostrProtocol.kt` header comment (line 10) even calls the private-message
construction "NIP-17 ... Compatible with iOS implementation" — this is stale/incorrect
per the Swift source's own disclaimer above.

**Divergence found:**
- Kind 5 (NIP-09 deletion) and kind **1401 (courier drop)** have no equivalent
  anywhere in `bitchat-android` (`grep -rn "1401|courierDrop" --include=*.kt` = no
  hits). The courier/store-and-forward delivery path does not exist on Android.
- Android declares an additional kind, `FILE_MESSAGE = 15`
  (`NostrEvent.kt:214`), with no Swift counterpart found in `NostrProtocol.swift`.
- Kind 20000/20001 are **not** used for mesh-bridge rendezvous on Android at all
  — see §3/§5 below; Android's uses of 20000/20001 are geohash-chat only
  (`NostrProtocol.kt:157-214`, `135-151`).

## 2. Tags used

From `NostrProtocol.swift`:
- `p` — recipient pubkey tag on the outer gift-wrap (`524`) and, for
  compatibility with current Android's inner-tag shape, optionally on the
  inner rumor (`create PrivateMessage` `messageTags`, `59-72`, `172-181`).
- `g` — geohash tag on ephemeral chat events, presence heartbeats, and text
  notes (`309`, `325`, `436`).
- `n` — nickname tag, optional, on ephemeral/text-note/bridge-mesh events
  (`311`, `357-359`, `437-439`).
- `t` — `["t","teleport"]` marks a teleported (non-local) geohash post
  (`313-315`); `["t","urgent"]` marks an urgent geohash text note (`443-445`).
- `r` — **mesh-bridge rendezvous cell** tag, distinct from `g` specifically so
  bridge traffic does not leak into geohash-channel `#g` subscriptions
  (`339-347`, `356`, `390`).
- `m` — `[stableID, meshSenderIDHex, meshTimestampMs]` radio-copy hint on
  bridge-mesh events, used only as an unauthenticated dedup hint (never trusted
  to own the timeline ID) — `348-367`, and consumed in
  `BridgeService.classify` (`bitchat/Services/Gateway/BridgeService.swift:849-866`).
- `x` — hex recipient tag (day-rotating) on courier drops (`410-412`).
- `expiration` — NIP-40 expiry, on courier drops (`412`) and optionally on
  geohash text notes (`441`).
- `e` — referenced event ID on NIP-09 deletion requests (`468`).

**Kotlin cross-check**: `p`, `g`, `n`, `t` (teleport) all present and matching
(`NostrProtocol.kt:34, 114, 117, 173`). **`r`, `m`, `x`, `expiration` tags do
not appear anywhere in bitchat-android** (confirmed via
`grep -rn "\"r\"|\"m\"|\"x\"|expiration" app/src/main/java/com/bitchat/android/nostr/*.kt`
finding no bridge/courier usage) — consistent with §1's finding that the
mesh-bridge and courier-drop features are Swift/iOS-only.

## 3. Relay selection

Two independent relay-selection mechanisms:

### 3a. Default relay set (private messages / gift wraps / courier drops)

Hardcoded built-in list, `NostrRelayManager.builtInRelays`
(`bitchat/Nostr/NostrRelayManager.swift:156-162`):
```
wss://relay.damus.io
wss://nos.lol
wss://relay.primal.net
wss://offchain.pub
```
merged with up to 8 user-added custom relays (`NostrRelaySettings.maxCustomRelays`,
`bitchat/Nostr/NostrRelaySettings.swift:25`), deduped/normalized
(`NostrRelayManager.swift:177-183`). This set is gated behind a policy —
default relays are only connected when the user has mutual favorites, location
permission, or is actively in a location channel
(`applyDefaultRelayPolicy`, `NostrRelayManager.swift:809-825`).

**Kotlin cross-check** (`NostrRelayManager.kt:43-48, 375-383`):
```
wss://relay.damus.io
wss://relay.primal.net
wss://offchain.pub
wss://nostr21.com
```
The Kotlin comment literally says "Default relay list (same as iOS)"
(`NostrRelayManager.kt:43`) — **this is false**. Android is missing
`wss://nos.lol` and has `wss://nostr21.com` in its place, which Swift does not
have at all. **Divergence found** — flagging per #14 rather than fixing.

### 3b. Geohash-proximity relay set (geohash chat, presence, mesh-bridge rendezvous)

`GeoRelayDirectory.closestRelays(toGeohash:count:)`
(`bitchat/Nostr/GeoRelayDirectory.swift:225-241`) decodes the geohash to a
lat/lon center (`Geohash.decodeCenter`) and returns the `count` (default 5,
`TransportConfig.nostrGeoRelayCount = 5`, `bitchat/Services/TransportConfig.swift:215`)
nearest relays by haversine distance, ties broken by hostname for
publisher/subscriber agreement (`GeoRelayDirectory.swift:230-232`).

Relay coordinates come from a CSV directory (`relays/online_relays_gps.csv` at
repo root, host/lat/lon columns, 441 entries as of this research —
`relays/online_relays_gps.csv:1-5`), loaded in priority order: on-disk cache →
bundled app resource → filesystem dev path
(`GeoRelayDirectory.swift:398-437`). The runtime refresh fetches from
**this repo's own reviewed copy** on GitHub
(`https://raw.githubusercontent.com/permissionlesstech/bitchat/refs/heads/main/relays/online_relays_gps.csv`,
`GeoRelayDirectory.swift:73`) — the code comment explains this deliberately:
"Upstream georelays/main is imported by a validator-backed pull request, so an
upstream mutation cannot immediately retarget clients" (`GeoRelayDirectory.swift:70-72`).
Fetched data is validated all-or-nothing (bounded bytes/rows/entries, valid
lat/lon ranges, minimum overlap with the previous baseline) —
`GeoRelayDirectory.swift:442-513`.

**Kotlin cross-check** (`RelayDirectory.kt:26-115`): same haversine-nearest
approach (`closestRelaysForGeohash`, `RelayDirectory.kt:88-106`), but fetches
directly from **`permissionlesstech/georelays`'s own repo**
(`ASSET_FILE_URL = "https://raw.githubusercontent.com/permissionlesstech/georelays/refs/heads/main/nostr_relays.csv"`,
`RelayDirectory.kt:29`) rather than from a reviewed copy vendored into
`bitchat-android` itself, and with much lighter validation (parses rows,
skips malformed ones, no byte/row caps, no minimum-retained-fraction check
against the previous baseline) — `RelayDirectory.kt:258-278`. **Divergence
found**: Android trusts upstream `georelays/main` directly at fetch time,
which is exactly the trust gap the Swift comment (`GeoRelayDirectory.swift:70-72`)
says it avoids by fetching from bitchat's own reviewed copy.

## 4. Private-message (DM) bridging semantics

Trigger: `NostrTransport.sendPrivateMessage` / `sendPrivateMessageGeohash`
(`bitchat/Services/NostrTransport.swift:254-266, 313-324`) — called by the
message router when a peer is not reachable over the BLE mesh transport but
has a known Nostr public key (favorite relationship). The BLE payload is
embedded into the Nostr event content via `NostrEmbeddedBitChat.encodePMForNostr`
(`NostrTransport.swift:260`), then wrapped as a gift-wrap
(`NostrProtocol.createPrivateMessage`, `NostrProtocol.swift:46-94`) and sent to
the default relay set. Delivery/read acks (`sendDeliveryAck`,
`sendReadReceipt`) follow the identical path, paced by `AckPacer`
(`NostrTransport.swift:73-129`) to avoid relay rate limits.

`canDeliverPromptly`/`canDeliverSecurely` (`NostrTransport.swift:226-239`)
report reachability based on a known npub plus a live relay connection to the
DM-capable relay set (`isDMRelayConnected`,
`NostrRelayManager.swift:191-195`) — a connected geohash/custom relay alone
does not count, since DMs only target the default relay set.

When relays are unreachable, `BridgeCourierService.depositDrop`
(`bitchat/Services/Gateway/BridgeCourierService.swift:258-299`) parks a sealed
Noise-X envelope as a kind-1401 courier drop under the recipient's day-rotating
tag (`x` tag), as a parallel delivery path to the direct gift-wrap send; the
recipient (or a gateway peer acting for them) subscribes for candidate tags and
opens matching drops (`handleDropEvent`, `BridgeCourierService.swift:554-588`).

**Kotlin cross-check**: `NostrTransport.kt` implements the equivalent
gift-wrap send/receive path (kinds 14/13/1059) matching Swift's construction
(confirmed in §1/§2). **The courier-drop store-and-forward path has no Android
equivalent** — confirmed no `1401`/`x`-tag/`expiration`-tag usage anywhere in
`bitchat-android`.

## 5. Mesh <-> Nostr bridging semantics (Gateway + Bridge services)

This is the "what triggers a bridge, what gets translated" core of the ticket.
**Two distinct, Swift-only features**, both opt-in toggles, both absent from
`bitchat-android` (confirmed: no `gateway`/`bridge`/`toGateway`/`fromGateway`/
`toBridge`/`fromBridge`/`rendezvous` hits anywhere in the Kotlin tree, and
Android's own `TransportBridgeService.kt` is an unrelated BLE<->WiFi-Aware
mesh-to-mesh relay, not a Nostr bridge — `TransportBridgeService.kt:13-19`).

### 5a. GatewayService — geohash channel gateway

Purpose (`bitchat/Services/Gateway/GatewayService.swift:14-21`): while enabled,
a device with internet advertises a `.gateway` capability bit, and:
- **Uplink** (mesh → Nostr): publishes signed geohash events deposited by
  mesh-only peers to Nostr relays. Trigger: a directed `toGateway`
  `NostrCarrierPacket` arrives (`handleMeshCarrier`, `GatewayService.swift:167-186`,
  `handleUplinkDeposit`, `190-237`). Structural validation (kind must be
  `ephemeralEvent`/20000, must carry a matching `#g` tag, must be fresh) runs
  before any Schnorr-verify cost, then signature verification, then either
  immediate publish or bounded per-depositor queueing if relays are down
  (`Limits.maxQueuedUplinks=20`, `maxQueuedUplinksPerDepositor=5`, `uplinkEventsPerMinutePerDepositor=10` — `GatewayService.swift:62-66`).
- **Downlink** (Nostr → mesh): every event the gateway's own geohash
  subscription receives is rebroadcast onto the mesh as a broadcast
  `fromGateway` carrier packet (`rebroadcastRelayEvent`, `293-333`), rate-limited
  to `downlinkEventsPerMinute=30` (`72`).
- **Loop prevention** (3 rules, `GatewayService.swift:33-51`): a
  mesh-learned event is never re-published/re-uplinked/rebroadcast; an
  uplinked/rebroadcast event ID is marked so it's each handled at most once;
  uplink is only ever initiated for locally-composed events.
- Mesh-only senders auto-uplink without the toggle when relays are unreachable
  and a gateway peer is available (`uplinkViaMesh`, `409-425`) — "fire and
  forget", no gateway ack (v1).

### 5b. BridgeService — cross-mesh-island rendezvous

Purpose (`bitchat/Services/Gateway/BridgeService.swift:14-26`): stitches
disjoint BLE mesh islands sharing a physical place. While enabled:
- **Outgoing**: every public mesh message the device sends is additionally
  signed with a derived, per-cell, unlinkable Nostr identity
  (`NostrIdentityBridge.deriveIdentity(forBridgeRendezvous:)`,
  `bitchat/Nostr/NostrIdentityBridge.swift:92-94`, HMAC-derived, distinct label
  from the geohash-chat identity) as a "rendezvous" event (kind 20000, `r` tag
  = geohash-precision-6 cell, `~1.2km`, `BridgeService.swift:99-100`) and
  published directly to relays, or deposited via a directed `toBridge` carrier
  to a bridge-gateway peer if mesh-only (`bridgeOutgoing`, `354-381`). A
  per-message "nearby only" flag skips composing the rendezvous copy entirely.
- **Incoming** (subscription/internet role): `handleRendezvousEvent`
  (`432-485`) verifies the event's `r` tag is within the subscribed cell ring
  (own cell + `Geohash.neighbors`, `319-321`), classifies it as a presence
  heartbeat (kind 20001) or message (kind 20000 with the `m`-tag radio-copy
  hint, `classify`, `836-880`), injects it into the mesh timeline marked as
  bridged, and (serving duty) rebroadcasts genuine remote events onto the
  local mesh as `fromBridge` carriers, jittered to let multiple online
  bridgers de-duplicate broadcasts (`scheduleDownlinkDrainIfNeeded(jitter:)`,
  `683-708`).
- **Incoming** (mesh/radio role): `fromBridge` broadcasts are always accepted
  (passive, no toggle gate) and injected the same way (`handleDownlinkBroadcast`,
  `712-736`).
- Radio-vs-bridge race handling: an authenticated radio-received copy of a
  message always wins over — and retroactively replaces — a bridge-relayed
  row that merely matched the untrusted `m`-tag hint
  (`handleAuthenticatedRadioMessage`, `492-512`); the `m` tag can merge a
  duplicate but never suppress the genuine signed event.
- A device with both Gateway and Bridge toggles on serves its island: accepts
  `toBridge` deposits, publishes them, and rebroadcasts remote events
  (`BridgeService.swift:23-26`).

### 5c. NostrCarrierPacket — the mesh<->Nostr wire format

`bitchat/Protocols/NostrCarrierPacket.swift:31-144`. TLV-encoded
(`direction` byte, `geohash` string, `eventJSON` — the complete signed Nostr
event), carried inside `MessageType.nostrCarrier` (0x28) on the BLE mesh.
`Direction` enum (`32-43`): `toGateway=0x01` (directed uplink),
`fromGateway=0x02` (broadcast downlink), `toBridge=0x03` (directed rendezvous
uplink), `fromBridge=0x04` (broadcast rendezvous downlink). Old clients that
don't recognize 0x03/0x04 fail the decode and silently drop the carrier —
bridge traffic degrades to invisible rather than being misinterpreted
(`39-41`). Callers must independently re-verify `event.isValidSignature()`
after decoding (`79-80`) — the carrier format itself carries no trust.

**Divergence found**: none of §5a/§5b/§5c (GatewayService, BridgeService,
BridgeCourierService, NostrCarrierPacket, the `r`/`m`/`x` tags, kinds
5/1401) exist in `permissionlesstech/bitchat-android` as of this research's
clone. The entire mesh<->Nostr bridging feature set — gateway uplink/downlink,
cross-island rendezvous bridging, and courier store-and-forward — is Swift/iOS
-only today.

## 6. Processed-event / dedup persistence (context, not bridge-specific)

`NostrProcessedEventStore` (`bitchat/Services/NostrProcessedEventStore.swift:23-107`)
persists processed private-envelope event IDs to disk because BitChat
randomizes envelope timestamps (`NostrProtocol.randomizedTimestamp`,
`NostrProtocol.swift:748-767`, ±15 min) so DM subscriptions must look back 24h,
and relays redeliver the same events on every relaunch.
`BridgeDropDedupStore` (`bitchat/Services/Gateway/BridgeDropDedupStore.swift:75-137`)
is the analogous persisted dedup for courier drops (sender-side
publish-dedup keys and receiver-side seen-event IDs), both with a 24h
lifetime matching the courier drop's NIP-40 expiration window. Neither has a
Kotlin counterpart (courier drops don't exist on Android; Android's own
event-dedup, `NostrEventDeduplicator.kt`, was not examined in depth as it is
out of scope for bridge semantics).

## Summary of divergences (flagged, not fixed, per #14)

1. Built-in default relay list differs: Android has `nostr21.com` instead of
   Swift's `nos.lol`, despite an Android code comment claiming parity
   (`NostrRelayManager.kt:43` vs `NostrRelayManager.swift:156-162`).
2. Geo-relay-directory trust model differs: Android fetches directly from
   `permissionlesstech/georelays` upstream; Swift fetches from this repo's own
   reviewed copy specifically to avoid that upstream-mutation trust gap
   (`RelayDirectory.kt:29` vs `GeoRelayDirectory.swift:70-73`). Android's CSV
   validation is also considerably lighter (no byte/row caps, no
   baseline-overlap check).
3. The entire Gateway/Bridge/Courier mesh<->Nostr bridging feature (kinds
   1401 and the bridge-tagged use of 20000/20001, `r`/`m`/`x` tags,
   `NostrCarrierPacket` directions 0x03/0x04, `GatewayService`,
   `BridgeService`, `BridgeCourierService`) is Swift/iOS-only; no Kotlin
   equivalent exists.
4. NIP-09 deletion (kind 5) has no Kotlin equivalent.
5. Android declares an extra kind, `FILE_MESSAGE = 15`
   (`NostrEvent.kt:214`), not present in the Swift `EventKind` enum.
6. Android's `NostrProtocol.kt` header/comments describe the private-message
   construction as "NIP-17 ... Compatible with iOS," which the Swift source
   explicitly disclaims (`NostrProtocol.swift:10-17`) — a documentation/comment
   divergence, not necessarily a wire-format one (the wire shapes for
   kinds 13/14/1059 do appear compatible based on the code read).
