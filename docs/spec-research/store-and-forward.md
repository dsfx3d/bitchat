# Store-and-Forward — Research Notes (for spec/store-and-forward.md)

**Ticket:** dsfx3d/bitchat#26 ("Store-and-forward chapter: verify caching/relay behavior (Swift + Kotlin)")
**Map:** dsfx3d/bitchat#16
**Status:** Research only. Facts below are cited file+line; kept internal to this ticket per #11 (someone else folds this into #16's Decisions-so-far).

Source of truth: this repo's Swift store-and-forward stack (`bitchat/Services/Courier/`,
`bitchat/Services/Gateway/BridgeCourierService.swift`, `bitchat/Services/MessageRouter.swift`,
`localPackages/BitFoundation/Sources/BitFoundation/CourierEnvelope.swift`) plus the BLE
relay/fanout/source-routing files (`bitchat/Services/BLE/BLE{FanoutSelector,
DirectedRelaySpool, RouteForwardingPolicy, ScheduledRelayStore,
SourceRouteOriginationPolicy, SourceRouteFailureCache}.swift`). Cross-checked (read-only)
against `permissionlesstech/bitchat-android` at the commit fetched by a `--depth 1` shallow
clone on 2026-08-01.

Also folds in upstream issue **permissionlesstech/bitchat#1473** ("Relay Selection &
Delivery Semantics for External Geohash Publishers"), per design map issue #6. That issue
is about a *different* subsystem than BLE-mesh store-and-forward — Nostr geohash-channel
relay selection — but its ask is answered below (§5) since the ticket instructs folding it
in.

---

## 1. What gets cached — two distinct mechanisms

BitChat's Swift client has **two separate "store-and-forward" mechanisms** that a spec
chapter needs to keep distinct:

### 1.1 `MessageOutboxStore` — sender-side retry queue for the router's own DMs

Holds the *sender's own* queued private messages (plaintext content, addressed to a specific
peer) when no transport can deliver promptly. Not carried by other peers; it is retried by
`MessageRouter` itself as the peer becomes reachable/connected.

- Struct: `bitchat/Services/Courier/MessageOutboxStore.swift:26-61` (`QueuedMessage`: content,
  nickname, messageID, timestamp, `sendAttempts`, `depositedCourierKeys`).
- Persisted to disk, ChaCha20-Poly1305-sealed with an After-First-Unlock-ThisDeviceOnly
  Keychain key (`bitchat/Services/Courier/MessageOutboxStore.swift:63-64, 620-655, 733-763`).
  Wiped on panic (`MessageOutboxStore.swift:597-616`).
- Limits, both in `bitchat/Services/MessageRouter.swift:129-137`:
  - `maxMessagesPerPeer = 100` (oldest evicted first).
  - `messageTTLSeconds = 24 * 60 * 60` (24h).
  - `maxSendAttempts = 8` (caps retried "connected but no secure session yet" / "reachable"
    sends; does not cap sends already handed to BLE mid-handshake, which BLE itself owns).
  - `maxCouriersPerMessage = 3` (distinct couriers a given queued message may be deposited
    with, tracked via `depositedCourierKeys`).

### 1.2 `CourierStore` — third-party mailbag ("physical courier" store-and-forward)

Holds **opaque, encrypted envelopes for third parties** that this device is physically
carrying, to be handed over on later encounter. This is the "spray-and-wait" mailbag system:
a device becomes a courier for someone else's message.

- Doc comment: "bounded count, bounded per-depositor count by trust tier, bounded size, and a
  24-hour lifetime aligned with the outbox retention policy" —
  `bitchat/Services/Courier/CourierStore.swift:26-32`.
- The store never learns sender, recipient, or content: only a rotating recipient tag
  (`recipientTag`) and opaque `ciphertext` (`CourierStore.swift:34-38`,
  `CourierEnvelope.swift:20-37`).
- Persisted to disk as JSON, iOS file-protection `completeUntilFirstUserAuthentication`
  (`CourierStore.swift:457-484`). Wiped on panic (`CourierStore.swift:419-428`).
- What's cached: `CourierEnvelope` TLV — `recipientTag` (16 bytes), `expiry` (ms epoch),
  `ciphertext` (opaque one-way Noise-X seal, ≤16 KiB), `copies` (spray-and-wait budget,
  1–8), optional `prekeyID` (v2 forward-secret envelopes) —
  `localPackages/BitFoundation/Sources/BitFoundation/CourierEnvelope.swift:20-46`.
- Recipient addressing: `recipientTag = HMAC-SHA256(recipient Noise static key, "bitchat-courier-tag-v1" || UTC-epoch-day)[0..16]`,
  rotating daily so a courier can't correlate mail to the same recipient across days without
  already knowing their static key (`CourierEnvelope.swift:169-193`). Candidate tags checked
  cover the current day ±1 to tolerate clock skew (`CourierEnvelope.swift:188-193`).

---

## 2. Cache size / TTL limits (all Swift; see §6 for Kotlin comparison)

| Limit | Value | Location |
|---|---|---|
| Courier envelope max ciphertext size | 16 KiB (`16 * 1024`) | `CourierEnvelope.swift:41` |
| Courier envelope max lifetime | 24h (`24 * 60 * 60`) | `CourierEnvelope.swift:43` |
| Courier envelope max spray-copy budget | 8 (`maxCopies`) | `CourierEnvelope.swift:46` |
| Recipient tag length | 16 bytes | `CourierEnvelope.swift:39` |
| `CourierStore` total envelope cap | 40 (`maxEnvelopes`) | `CourierStore.swift:102` |
| `CourierStore` verified-tier cap (subset of total) | 20 (`maxVerifiedEnvelopes`) | `CourierStore.swift:104` |
| `CourierStore` per-favorite-depositor cap | 5 (`maxPerFavoriteDepositor`) | `CourierStore.swift:105` |
| `CourierStore` per-verified-depositor cap | 2 (`maxPerVerifiedDepositor`) | `CourierStore.swift:106` |
| `CourierStore` expiry clock-skew slack | 1h (`maxExpirySlack`) | `CourierStore.swift:108` |
| Eviction order when `CourierStore` is full | oldest-first; verified-tier evicted before favorite-tier; a verified deposit is rejected outright rather than displacing favorite mail | `CourierStore.swift:216-231` |
| `MessageOutboxStore` per-peer cap | 100 messages | `MessageRouter.swift:129` |
| `MessageOutboxStore` message TTL | 24h | `MessageRouter.swift:130` |
| `MessageOutboxStore` max send attempts | 8 | `MessageRouter.swift:135` |
| `MessageOutboxStore` max couriers per message | 3 | `MessageRouter.swift:137` |
| Bridge (Nostr relay) pending-drop queue | 20 (`maxPendingDrops`, drop-oldest) | `bitchat/Services/Gateway/BridgeCourierService.swift:44` |
| Bridge drop republish cooldown (per envelope) | 30 min (`heldEnvelopePublishCooldown`) | `BridgeCourierService.swift:46` |
| Bridge encoded-drop byte cap | 20 KiB (`maxDropEnvelopeBytes`; 16 KiB ciphertext + TLV slack) | `BridgeCourierService.swift:54` |
| Bridge gateway watched-peer cap | 16 local peers × 3 candidate tags each | `BridgeCourierService.swift:48` |
| Bridge tag-refresh cadence | 30 min (also covers UTC day rollover) | `BridgeCourierService.swift:50` |
| Directed BLE relay spool: dedup key | `(recipient, messageID)` — one packet per recipient/message, no size cap other than natural drain | `bitchat/Services/BLE/BLEDirectedRelaySpool.swift:15,26-40` |
| Source-route confirmation window | 10s (`bleSourceRouteConfirmationWindowSeconds`) | `bitchat/Services/TransportConfig.swift:297` |
| Source-route failure suppression window | 60s (`bleSourceRouteSuppressionSeconds`) | `TransportConfig.swift:300` |
| Default mesh-flood TTL | 7 hops (`messageTTLDefault`) | `TransportConfig.swift:8` |
| Nostr geohash relay-selection fan-out | 5 relays (`nostrGeoRelayCount`) | `TransportConfig.swift:215` |

---

## 3. Relay / fanout semantics — how a Swift node picks which peers to relay to

Two independent mechanisms, both link-level (per-connection), not message-content-level:

### 3.1 Broadcast fanout subset — `BLEFanoutSelector`

For a **broadcast** packet (no directed-peer hint) that isn't a fragment, announce, or
`REQUEST_SYNC` (`bitchat/Services/BLE/BLEFanoutSelector.swift:205-210`, `shouldSubset`), the
node does **not** flood every connected link. It picks a deterministic pseudo-random subset:

- Subset size `k = ceil(log2(count)) + 1`, clamped to `[1, count]`, with `count ≤ 2` sent to
  everyone (`BLEFanoutSelector.swift:212-223`, `subsetSize`).
- Which links make the subset is chosen by SHA-256-hashing `"{messageID}::{linkID}"` for
  every candidate link and taking the `k` lowest-hashing IDs
  (`BLEFanoutSelector.swift:225-245`, `deterministicSubset`) — deterministic per message so
  duplicate relays of the *same* packet by different intermediate nodes don't diverge
  wildly, but different messages fan out to different subsets over time.
- Fragments, announces, and `REQUEST_SYNC` bypass subsetting entirely and go to every
  allowed link (`BLEFanoutSelector.swift:205-210`) — fragments need every link a large
  transfer might be split across; announces bind links to peers and are cheap/throttled;
  `REQUEST_SYNC` is link-local and never forwarded at all (see §3.3).
- Before subsetting, the ingress link and any explicitly excluded links are removed
  (`BLEFanoutSelector.swift:92-110`, `allowedLinks`), and duplicate links to the *same* bound
  peer are collapsed to one (preferring the peripheral/write side) so a dual-role connection
  doesn't get double airtime (`BLEFanoutSelector.swift:158-203`,
  `collapseDuplicateLinksPerPeer`). Note: **no probability/network-size input** — this is a
  purely deterministic k-of-n pick, unlike Kotlin's approach (§6).

### 3.2 Directed delivery — direct peer link, else source route, else flood

When a packet has a **directed-peer hint** (unicast, e.g. a private message):

- If a link is already bound to that peer, `BLEFanoutSelector` restricts fanout to just that
  peer's link(s) (`directLinks`, `BLEFanoutSelector.swift:112-137`), optionally requiring a
  direct link and returning empty otherwise (`requireDirectPeerLink`,
  `BLEFanoutSelector.swift:42-44`).
- If not directly connected, Swift may attach a **v2 source route** instead of falling back
  to flood-broadcast. Gated by `BLESourceRouteOriginationPolicy.route(...)`
  (`bitchat/Services/BLE/BLESourceRouteOriginationPolicy.swift:22-39`): only when (a) the
  packet is authored locally (not a relay re-signing someone else's packet), (b) it's
  single-peer directed, (c) TTL > 1, (d) the recipient isn't already directly connected, (e)
  `BLESourceRouteFailureCache.shouldAttemptRoute` says the peer isn't currently in failure
  suppression, and (f) a complete path exists via the mesh topology graph (documented further
  in `docs/SOURCE_ROUTING.md` — verified against source in §3 there: BFS with ≤4 intermediate
  hops over confirmed, mutually-announced edges, restricted to peers observed speaking v2).
- A packet carrying a route is followed hop-by-hop by `BLERouteForwardingPolicy.plan(...)`
  (`bitchat/Services/BLE/BLERouteForwardingPolicy.swift:30-101`): find local peer ID's index
  in the route; if not present, forward toward `route[0]`; if present and not last, forward
  to `route[index+1]`; if last, forward to the final `recipientID`. TTL is decremented on
  each forwarded hop (`relayed(_:)`, lines 96-100). If the computed next hop isn't connected,
  it falls back to normal flood relay rather than dropping
  (`BLERouteForwardingPolicy.swift:58-60, 88-91`).
- Route-failure health tracking: `BLESourceRouteFailureCache` marks a recipient as "route
  failed" if no inbound packet from them arrives within 10s of a routed send
  (`bleSourceRouteConfirmationWindowSeconds`), then suppresses routing (falls back to flood)
  toward that recipient for 60s (`bleSourceRouteSuppressionSeconds`) —
  `BLESourceRouteFailureCache.swift:36-56` / `TransportConfig.swift:297,300`.
- `BLEDirectedRelaySpool` buffers directed-relay packets per recipient (deduped by
  messageID) until a drain window elapses, then flushes once
  (`BLEDirectedRelaySpool.swift:25-53`, `drainUnexpired`/`pruneExpired`).
  `BLEScheduledRelayStore` tracks the `DispatchWorkItem`s backing per-message scheduled relay
  attempts and can be bulk-cancelled or reset if it grows past a capacity
  (`BLEScheduledRelayStore.swift:38-42`, `removeAllIfOverCapacity`).

### 3.3 What never gets relayed

`REQUEST_SYNC` and any packet whose `recipientID` matches the local peer are always
suppressed from further relay, both on the flood path and the source-routed path —
`BLERouteForwardingPolicy.swift:42-48`. `REQUEST_SYNC` is explicitly link-local: forwarding it
even with route/TTL headroom would let a crafted request replay a full store-sync onto a
next hop (comment at `BLERouteForwardingPolicy.swift:38-41`).

### 3.4 Courier spray-and-wait (mailbag re-deposit, not link relay)

Separate from BLE-link fanout: when one courier encounters *another* courier (not the
recipient), `CourierStore.transferSprayCopies`/`takeSprayCopies`
(`CourierStore.swift:359-414`) offers each eligible envelope with **half its remaining copy
budget** (binary spray, `copies / 2`), skipping envelopes the encountered courier already
deposited, envelopes addressed to them, carry-only envelopes (`copies == 1`), and couriers
already sprayed to (`sprayedTo` set) — so a given envelope only ever sprays to each courier
once and the total in-flight copy count is capped by the original depositor's `copies` value
(≤ `maxCopies` = 8).

---

## 4. Handover / delivery semantics

- **Direct handover** (`CourierStore.handoverEnvelopes`, `CourierStore.swift:267-297`):
  non-destructive at the offer stage — envelope is only removed from the local store after
  the transport's `accepting` closure returns true (i.e. delivered onto a live physical link),
  so a failed send leaves the envelope intact for the next encounter.
- **Relayed/speculative handover** (`envelopesForRemoteHandover`, `CourierStore.swift:303-321`):
  triggered when hearing a *relayed* (not direct) announce for the recipient. Non-destructive
  — the envelope stays carried (a multi-hop send is speculative) — and is rate-limited by a
  caller-supplied cooldown per envelope (`lastRemoteHandoverAt`) so repeated announces don't
  re-flood the mesh with the same mail.
- **Bridge (Nostr relay) publish path** (`BridgeCourierService`,
  `bitchat/Services/Gateway/BridgeCourierService.swift`): a courier envelope can also be
  parked on Nostr relays as a kind-1401 "drop" tagged with the rotating recipient tag
  (comment block, lines 18-39), so delivery doesn't require a physical courier encounter.
  Each publish uses a **fresh throwaway signing key** (line 27-28) so a stable publisher key
  can't fingerprint courier traffic. Completion of a publish is only counted once a relay
  sends a NIP-01 `OK` (line 70-71) — not merely "queued in RAM" or "written to a socket".
  A gateway (bridge+gateway toggles on) additionally watches up to 16 local verified peers'
  tags and hands matching drops to them directly (§2 table for the 16-peer/30-min limits).
- **Privacy metrics only** (`StoreAndForwardMetrics.swift:16-59`): bare event counters
  (queued/resent/delivered/dropped/deposited/accepted/handedOver/remoteHandover/
  sprayed/opened) with no message IDs, peer identities, or timestamps — log-only, never
  transmitted (`StoreAndForwardMetrics.swift:12-15`).

---

## 5. Nostr geohash relay selection (upstream #1473's ask)

Upstream issue **permissionlesstech/bitchat#1473** ("Relay Selection & Delivery Semantics for
External Geohash Publishers") asked three things about *geohash-channel* Nostr relay
selection (this is unrelated to BLE-mesh courier relaying above — it's how a client picks
which Nostr relays to subscribe/publish to for a public geohash channel). The upstream
maintainer (`jackjackbits`) confirmed all three via the comment fetched for this ticket
(`gh issue view 1473 --repo permissionlesstech/bitchat --comments`):

1. **Relay selection is nearest-5 by haversine distance** from a bundled ~300-relay
   directory, keyed to the geohash's decoded center coordinate, with **deterministic
   tie-breaking by host** so every device with the same directory converges on the same
   relay set (publishers and subscribers must agree) —
   `bitchat/Nostr/GeoRelayDirectory.swift:233-241` (`closestRelays(toLat:lon:count:)`,
   default `count = 5`), called with `TransportConfig.nostrGeoRelayCount = 5`
   (`TransportConfig.swift:215`) at call sites e.g.
   `bitchat/Services/Board/BoardManager.swift:162,186`,
   `bitchat/ViewModels/GeohashSubscriptionManager.swift:144,240`.
2. **Sparse/ocean geohash cells silently degrade** — `closestRelays` just returns whatever is
   nearest even if very far away; there is no distance ceiling or "undeliverable" signal in
   this code path (confirmed by the maintainer's reply; not independently found as a check in
   `GeoRelayDirectory.swift`).
3. **Kind-20000 ephemerality** (ephemeral Nostr events, ~NIP-16/NIP-01 semantics) means a late
   subscriber can't backfill a recent message — this is a Nostr-protocol-level property, not
   something this repo's code overrides.

An external publisher that only knows a *fixed* relay set (rather than replicating this
geo-selection) can therefore miss a client entirely, which is exactly #1473's concern. This
is a distinct fact from the BLE-mesh relay-selection facts in §3 and should likely get its
own subsection (or a forward-reference) in `spec/store-and-forward.md` or in the Nostr-bridge
chapter (`spec/nostr-bridge.md`, per dsfx3d/bitchat#28's research) — whichever the drafting
task decides is the better home; both chapters exist per the map's fixed chapter order.

---

## 6. Kotlin cross-check (`permissionlesstech/bitchat-android`)

**Verdict: significant divergence.** Android has no equivalent of Swift's physical-courier
mailbag (`CourierStore`/`CourierEnvelope`/spray-and-wait/bridge-drop-publish) at all. Its
"store-and-forward" is an older, simpler, same-link-only cache. Details:

### 6.1 No courier/spray-and-wait/bridge-drop system in Kotlin

Exhaustive search of `bitchat-android` (`grep -rli "courier\|spray.*wait"` across
`app/src/main/java`) found **zero matches**. There is no Kotlin analog to `CourierEnvelope`,
`CourierStore`, or `BridgeCourierService`'s Nostr kind-1401 drop mechanism.

### 6.2 Kotlin's actual store-and-forward: `StoreForwardManager` (different design)

`app/src/main/java/com/bitchat/android/mesh/StoreForwardManager.kt` caches messages **this
node overheard for other peers** and flushes them when that *specific peer* connects
directly to *this* node — i.e., a same-hop cache, not a carried mailbag that moves between
devices:

| Aspect | Swift (`CourierStore`) | Kotlin (`StoreForwardManager`) |
|---|---|---|
| Persistence | Encrypted, disk-persisted, survives app kill | **In-memory only** (`Collections.synchronizedList`/`ConcurrentHashMap`); lost on process death — `StoreForwardManager.kt:36-39` |
| Content visibility | Opaque ciphertext only (store never sees plaintext/sender/recipient) | Caches the **raw `BitchatPacket`** — `StoreForwardManager.kt:28-33` |
| Cache trigger | Explicit courier deposit protocol (Noise-X sealed envelope handed to a trusted carrier) | Any non-broadcast, non-handshake/announce/leave packet addressed to an offline peer is auto-cached — `StoreForwardManager.kt:54-68` |
| Regular-peer cache size | N/A (all `CourierStore` slots are trust-tiered, 40 total) | 100 messages (`MAX_CACHED_MESSAGES`) — `AppConstants.kt:78` |
| Favorite-peer cache size | 5 per favorite depositor (`maxPerFavoriteDepositor`) | **1,000** messages per favorite (`MAX_CACHED_MESSAGES_FAVORITES`) — `AppConstants.kt:79` — a 25x larger favorite allowance than Swift's per-depositor cap (note: different denominators — Swift's cap is per-depositor within a 40-envelope global pool; Kotlin's is per-favorite-recipient with no stated global pool cap) |
| Regular-message TTL | 24h (aligned across both Swift stores) | **12h** (`MESSAGE_CACHE_TIMEOUT_MS = 43_200_000L`) — `AppConstants.kt:77` — half of Swift's window |
| Favorite-message TTL | 24h + 1h skew slack | **No expiry** for favorite queue entries (only regular cache is time-pruned; `cleanupMessageCache` filters `!it.isForFavorite`) — `StoreForwardManager.kt:255-258` |
| Delivery mechanism | Spray-and-wait carried between devices, or a Nostr relay "drop" | Direct broadcast the moment the addressed peer reconnects; no inter-courier spray at all — `StoreForwardManager.kt:121-179` |
| Multi-hop / relay to a peer not yet seen | Yes — spray-and-wait circulates copies to other couriers who may meet the recipient later, plus Nostr-relay drop for internet-connected delivery | No — only this node's own encounter with the recipient triggers delivery |

### 6.3 Sender-side outbox (Swift `MessageOutboxStore` vs Kotlin `MessageRouter.outbox`)

This part **matches on the numbers, diverges on persistence**:

| Constant | Swift | Kotlin |
|---|---|---|
| Outbox TTL | 24h — `MessageRouter.swift:130` | 24h (`OUTBOX_MESSAGE_TTL_MS = 86_400_000L`) — `AppConstants.kt:140` |
| Max queued per peer | 100 — `MessageRouter.swift:129` | 100 (`OUTBOX_MAX_PER_PEER`) — `AppConstants.kt:141` |
| Persistence | ChaChaPoly-sealed, Keychain-key-protected disk file, survives app kill — `MessageOutboxStore.swift:63-64` | **In-memory `ConcurrentHashMap` only** — `services/MessageRouter.kt:83-84` — no disk persistence found; a killed Android process loses its queued DMs |
| Retry cap | 8 attempts (`maxSendAttempts`) — `MessageRouter.swift:135` | No attempt cap found; retried every scheduler tick until TTL expiry. Handshake retries use a fixed backoff ladder `[5s, 15s, 30s, 60s]` (`HANDSHAKE_RETRY_BACKOFF_MS`) — `AppConstants.kt:142`, `services/MessageRouter.kt:260-267` — a different mechanism (backoff vs. hard attempt cap) |

**Verdict: matching TTL/cap numbers, diverging durability model.** The 24h/100-per-peer
figures line up exactly, which suggests they're an intentionally shared cross-platform
contract; the persistence gap (Swift durable, Kotlin volatile) and the retry-limiting
strategy (hard cap vs. backoff-without-cap) are real behavioral divergences worth flagging
if the spec asserts message durability across app restarts as a cross-platform guarantee.

### 6.4 Relay/fanout selection — divergent strategy

**Verdict: diverges significantly.** Kotlin's `PacketRelayManager`
(`app/src/main/java/com/bitchat/android/mesh/PacketRelayManager.kt`) relays to **every**
connected link (full broadcast), gated by a **probability**, not Swift's deterministic
k-of-n subset (§3.1):

- Always relay if `packet.ttl >= 4` or network size ≤ 3 — `PacketRelayManager.kt:144-156`.
- Otherwise probability by network size: ≤10 peers → 1.0, ≤30 → 0.85, ≤50 → 0.7, ≤100 → 0.55,
  else → 0.4 — `PacketRelayManager.kt:159-167`. A single coin-flip per packet, not a
  per-link subset selection.
- No `BLEFanoutSelector`-style deterministic hashing, no per-link duplicate-peer collapsing,
  no ingress-link/messageID-seeded subset logic in Kotlin.
- **Source routing does match**: Kotlin's `PacketRelayManager.handlePacketRelay`
  (`PacketRelayManager.kt:76-106`) follows a `route` field the same way as Swift's
  `BLERouteForwardingPolicy` — find self in the route, forward to `route[index+1]` or to
  `recipientID` if last, falling back to flood broadcast if the next hop isn't connected —
  consistent with `docs/SOURCE_ROUTING.md`'s claim that source routing is "Implemented in
  Android and iOS: both decode routed packets, forward along routes, and originate routes."
  Kotlin additionally rejects a route containing duplicate hops as a loop-prevention check
  (`PacketRelayManager.kt:79-83`) not explicitly seen as a separate guard in the Swift files
  read for this ticket (Swift's TTL-based loop protection is implicit via
  `BLERouteForwardingPolicy`'s TTL decrement — not verified as an explicit duplicate-hop
  check in this pass).

### 6.5 Geohash/Nostr relay selection (§5) — matches

Android's `RelayDirectory.closestRelaysForGeohash(geohash, nRelays)`
(`app/src/main/java/com/bitchat/android/nostr/RelayDirectory.kt:88-106`) sorts bundled relays
by haversine distance to the geohash center and takes the nearest `nRelays`, called with
`nRelays = 5` at both call sites (`nostr/LocationNotesManager.kt:235`,
`nostr/NostrRelayManager.kt:155`) — matching Swift's nearest-5 selection (§5). **Minor
divergence:** Android's `sortedBy` has no explicit tie-break (Kotlin's `sortedBy` is stable,
so ties keep directory-load order, not a deterministic host-based tiebreak like Swift's
`($0.distance, $0.entry.host) < ...`, `GeoRelayDirectory.swift:238`) — a corner case, only
observable when two relays are exactly equidistant.

---

## 7. Summary for spec drafting

- `spec/store-and-forward.md` should describe **two Swift mechanisms** distinctly: the
  sender's own outbox retry queue (`MessageOutboxStore`) and the third-party courier mailbag
  (`CourierStore` + `CourierEnvelope` + spray-and-wait + Nostr-relay drop via
  `BridgeCourierService`). Kotlin only has rough analogs to the former (`MessageRouter`
  outbox) and a much simpler, non-carried version of the latter (`StoreForwardManager`) — the
  spec should mark the courier/spray-and-wait/relay-drop system as **iOS/Swift-only** unless
  and until Android implements it, per #14's "flag, don't fix" instruction.
- BLE relay/fanout selection is **iOS deterministic subset vs. Android probabilistic
  broadcast** — genuinely different algorithms achieving similar goals (bound rebroadcast
  cost in dense meshes). If the spec is meant to describe required cross-platform interop
  behavior (rather than each platform's implementation choice), this is the single largest
  divergence found in this pass and should be called out explicitly, since a spec reader
  might otherwise assume one canonical fanout algorithm.
- Source-based routing (v2) is a genuine cross-platform match and can be specified as a
  shared contract (already substantially documented in `docs/SOURCE_ROUTING.md`, verified
  against Swift source in this pass; Android's route-following logic in
  `PacketRelayManager.kt` also matches).
- Geohash/Nostr relay selection (nearest-5 by haversine) is a cross-platform match and
  directly answers upstream #1473; note the untreated sparse-cell/ephemerality limitations
  as known, working-as-designed behavior rather than a bug.
