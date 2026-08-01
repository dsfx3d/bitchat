# BLE Transport — Research Notes (for spec/ble-transport.md)

**Ticket:** dsfx3d/bitchat#20 ("BLE transport chapter: verify GATT UUIDs, fragmentation, advertising (Swift + Kotlin)")
**Map:** dsfx3d/bitchat#16
**Status:** Research only. Facts below are cited file+line; kept internal to this ticket per #11 (someone else folds this into #16's Decisions-so-far).

Source of truth: this repo's Swift BLE stack (`bitchat/Services/BLE/`). Cross-checked
(read-only) against `permissionlesstech/bitchat-android` at the commit fetched by a
`--depth 1` shallow clone on 2026-08-01.

---

## 1. GATT Service / Characteristic UUIDs

| Item | Value | Swift | Kotlin |
|---|---|---|---|
| Service UUID (mainnet/Release) | `F47B5E2D-4A9E-4C5A-9B3F-8E1D2C3A4B5C` | `bitchat/Services/BLE/BLEService.swift:219` | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:30` |
| Service UUID (DEBUG/testnet only, iOS-only build flavor) | `F47B5E2D-4A9E-4C5A-9B3F-8E1D2C3A4B5A` | `bitchat/Services/BLE/BLEService.swift:217` | — (no Android equivalent found; Android ships a single UUID) |
| Characteristic UUID | `A1B2C3D4-E5F6-4A5B-8C9D-0E1F2A3B4C5D` | `bitchat/Services/BLE/BLEService.swift:221` | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:31` |
| Characteristic properties | `.notify, .write, .writeWithoutResponse, .read`; permissions `.readable, .writeable` | `bitchat/Services/BLE/BLEService.swift:4123-4128` | `app/src/main/java/com/bitchat/android/mesh/BluetoothGattServerManager.kt:331,346` (not read in this pass — verify properties bitmask if spec needs exact parity) |
| CCCD (Client Characteristic Configuration Descriptor) UUID | `00002902-0000-1000-8000-00805f9b34fb` (standard BT SIG descriptor) | not defined as a named constant in Swift (CoreBluetooth manages it implicitly via `.notify`) | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:32` |

**Verdict: match.** Both platforms use the same 128-bit service UUID (mainnet) and the same
characteristic UUID. Single GATT service, single read/write/notify characteristic — this is
BitChat's sole BLE data channel (no separate TX/RX characteristics as in some BLE UART
schemes).

The DEBUG-only testnet service UUID (`...4B5A` vs `...4B5C`, differing in the last hex digit)
is an iOS-side build-configuration detail (`#if DEBUG`/`#else` at
`bitchat/Services/BLE/BLEService.swift:216-220`) and is not part of the shipped protocol
surface — the spec should document only the mainnet UUID as canonical.

---

## 2. Fragmentation Scheme

### 2.1 Wire format (fragment payload header)

Both platforms use an identical 13-byte fragment header followed by the fragment's data
bytes, carried as the `payload` of a `BitchatPacket` with `type = MessageType.fragment`:

```text
+----------------+-------------+-------------+------------+----------------+
| Fragment ID    | Index       | Total       | Orig. Type | Fragment Data  |
| 8 bytes        | 2 bytes BE  | 2 bytes BE  | 1 byte     | variable       |
+----------------+-------------+-------------+------------+----------------+
```

- Swift construction: `bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:112-117`
  (`payload.append(fragmentID)` then big-endian `UInt16(index)`, big-endian `UInt16(total)`,
  then `packet.type`, then `fragmentData`).
- Swift parsing / minimum-size guard (13 bytes: 8+2+2+1): `bitchat/Services/BLE/BLEFragmentAssemblyBuffer.swift:21-53`
  (`BLEFragmentHeader.init?(packet:)`).
- Kotlin construction/parsing: `app/src/main/java/com/bitchat/android/model/FragmentPayload.kt:20-111`
  (`HEADER_SIZE = 13`, `FRAGMENT_ID_SIZE = 8`, explicit big-endian encode/decode of index and
  total at lines 46-51 and 94-100). The Kotlin file's own doc comment states it is "100%
  iOS-compatible" and cites the iOS source it mirrors.
- Fragment ID generation: random 8 bytes on both sides —
  `bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:141-143` (`randomFragmentID()`) vs.
  `app/src/main/java/com/bitchat/android/model/FragmentPayload.kt:74-78` (`generateFragmentID()`).

**Verdict: match**, byte-for-byte compatible header layout.

### 2.2 Fragment versions / route-aware sizing (Swift-only detail, v2 packets)

Swift's planner emits `fragmentVersion = 2` (vs. default `1`) when the original packet
carries a source route (`packet.route`), and shrinks the per-fragment chunk size to leave
room for route overhead: `overhead = 16 (fixed header) + 8 (sender) + 8 (recipient) +
routeSize (1 + hops*8) + 13 (fragment header) + 16 (auth tag)`, floored at
`minimumChunkSize = 64` bytes —
`bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:81-100`. This is part of the v2
Source-Based Routing extension (see `docs/SOURCE_ROUTING.md`) layered on top of plain
fragmentation, not a divergence from Android — Android's source-routing support should be
checked separately if/when #14-style flag work touches routing; out of scope here.

### 2.3 Fragment size limits

| Constant | Swift value | Swift location | Kotlin value | Kotlin location |
|---|---|---|---|---|
| Default per-fragment chunk size | 469 bytes | `bitchat/Services/TransportConfig.swift:7` (`bleDefaultFragmentSize`, "~512 MTU minus protocol overhead") | 469 bytes | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:42` (`MAX_FRAGMENT_SIZE`) |
| BLE MTU ceiling used for sizing | 512 | `bitchat/Services/BLE/BLEService.swift:227` (`bleMaxMTU`) | 512 (threshold) | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:41` (`FRAGMENT_SIZE_THRESHOLD`) |
| Minimum chunk size (route-overhead floor) | 64 bytes | `bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:23` | not found as a matching named constant in this pass | — |
| Max fragments per stream (cross-platform deployed ceiling) | 256 (`privateMediaV1MaxFragments`; comment: "Current Android receivers reject fragment sets above 256") | `bitchat/Services/BLE/BLEOutboundFragmentPlanner.swift:20-22` | 256 (`MAX_FRAGMENTS_PER_ID`) | `app/src/main/java/com/bitchat/android/util/AppConstants.kt:45` |
| Header sanity ceiling on `total` field | 10,000 (rejects clearly-bogus headers before the 256 policy cap applies) | `bitchat/Services/BLE/BLEFragmentAssemblyBuffer.swift:38` | N/A — Kotlin enforces the 256 cap directly at receive time (`FragmentingPacketSender.kt:192`, `maxFragments`) | `app/src/main/java/com/bitchat/android/mesh/FragmentingPacketSender.kt:192` |

**Verdict: match** on the numbers that matter for interop (469-byte default chunk, 512 MTU
ceiling, 256-fragment cap). The Swift comment at
`BLEOutboundFragmentPlanner.swift:20-22` explicitly documents that this 256 cap exists
*because* Android rejects larger fragment sets — i.e. this constant is already
cross-platform-contract-driven, not just parallel coincidence.

### 2.4 Reassembly

- Swift: `BLEFragmentAssemblyBuffer` (`bitchat/Services/BLE/BLEFragmentAssemblyBuffer.swift:56-214`)
  stores fragments keyed by `(sender, fragmentID)` in a dictionary of `index -> Data`,
  completes when `fragments.count == header.total` (line 120-121), concatenates in index
  order (lines 125-129). Caps concurrent in-flight assemblies at
  `TransportConfig.bleMaxInFlightAssemblies = 128` (`bitchat/Services/TransportConfig.swift:9`,
  evicting the oldest by timestamp when full — `BLEFragmentAssemblyBuffer.swift:144-148`).
  Byte-size ceiling per assembly is type-dependent: file-transfer/Noise-encrypted payloads get
  `FileTransferLimits.maxFramedFileBytes`, everything else gets
  `FileTransferLimits.maxPayloadBytes` (`BLEFragmentAssemblyBuffer.swift:203-213`).
  Stalled broadcast reassemblies (no new fragment for `bleFragmentResyncStallSeconds = 5.0s`,
  `bitchat/Services/TransportConfig.swift:305`) trigger a targeted `REQUEST_SYNC` naming the
  fragment stream (`BLEFragmentAssemblyBuffer.swift:160-201`).
- Kotlin: `FragmentingPacketSender.kt:171-293` (receive path) stores fragments in
  `incomingFragments: MutableMap<index, ByteArray>` keyed by fragment-ID hex string, tracks
  cumulative size per set and globally (`MAX_FRAGMENT_TOTAL_BYTES = 1_048_576`,
  `MAX_GLOBAL_FRAGMENT_TOTAL_BYTES = 4 MiB`, `MAX_ACTIVE_FRAGMENT_SETS = 64`, all
  `AppConstants.kt:43-48`), completes when the received count reaches the expected total
  (`FragmentingPacketSender.kt:260-277`), and has its own timeout
  (`FRAGMENT_TIMEOUT_MS = 30_000L`, `AppConstants.kt:43`) matching Swift's
  `bleFragmentLifetimeSeconds = 30.0` (`TransportConfig.swift:176`).

**Verdict: conceptually matched** (same key scheme, same completion condition, same
30-second fragment lifetime), but the buffering-limit *numbers* differ in shape: Swift bounds
by assembly *count* (128) plus a payload-type-dependent byte ceiling; Kotlin bounds by
per-set bytes (1 MiB), global bytes (4 MiB), and set count (64). These are independent
DoS/memory defenses, not wire-protocol fields, so they don't need to match exactly for
interop — flagging as a minor divergence for whoever drafts the spec's "implementation
limits are advisory, not normative" section.

### 2.5 Inter-fragment pacing (transport-level, not wire format)

- Swift: `bleFragmentSpacingMs = 30` (broadcast) / `bleFragmentSpacingDirectedMs = 25`
  (directed), `bitchat/Services/TransportConfig.swift:192-193`, selected in
  `BLEOutboundFragmentPlanner.swift:133-139`.
- Kotlin: `interFragmentDelayMs: Long = 20L` default parameter,
  `app/src/main/java/com/bitchat/android/mesh/FragmentingPacketSender.kt:24` (single value,
  no broadcast/directed distinction found in this pass).

**Divergence noted:** pacing values differ (30/25ms Swift vs 20ms Kotlin flat) and Swift
distinguishes broadcast vs. directed sends while Kotlin's constructor default does not (a
distinguishing value may be supplied by a caller elsewhere in bitchat-android; not verified
in this pass — would need a deeper Kotlin read to confirm). This is timing/QoS tuning, not a
wire-format incompatibility, since fragments are self-describing (index/total) and neither
side depends on fixed inter-packet timing for correctness — flagging per #14 as a
divergence, not fixing it.

---

## 3. Advertising Behavior

### 3.1 What's advertised

- **Swift (peripheral role):** `buildAdvertisementData()` advertises **only** the service
  UUID list (`CBAdvertisementDataServiceUUIDsKey: [BLEService.serviceUUID]`) — no local name
  ("No Local Name for privacy" comment) and no service data —
  `bitchat/Services/BLE/BLEService.swift:4474-4483`. Advertising (re)starts whenever the
  peripheral manager powers on and isn't already advertising, and after each service
  add/central-subscribe cycle: `bitchat/Services/BLE/BLEService.swift:4214-4215`,
  `4233-4234`, `4283-4285`, `8029-8030`. Peer identity is never carried in the advertisement
  itself; peers only learn each other's peer ID after connecting and exchanging an announce
  packet over the characteristic (`sendAnnounce`, referenced at
  `bitchat/Services/BLE/BLEService.swift:4097-4098` on subscribe).
- **Kotlin (peripheral role):** Advertises the service UUID in the primary advertisement
  packet, plus a **scan-response packet carrying the first 8 bytes of the local peer ID** as
  BLE service data (`addServiceData(ParcelUuid(SERVICE_UUID), peerIDBytes)`), explicitly to
  let scanners deduplicate a device across MAC-address rotations —
  `app/src/main/java/com/bitchat/android/mesh/BluetoothGattServerManager.kt:387-407`. Device
  name is excluded on both packets (`setIncludeDeviceName(false)`, lines 392 and 406), TX
  power level excluded on both (`setIncludeTxPowerLevel(false)`, lines 391 and 405).

**Divergence found:** Android's scan response leaks a stable 8-byte peer-ID fragment for
MAC-rotation dedup; iOS's CoreBluetooth-based peripheral never puts any peer-identifying
bytes in the advertisement/scan-response at all — only the service UUID. This is a real
privacy-surface difference worth flagging explicitly for whoever drafts the privacy section
of spec/ble-transport.md: an Android BitChat peer's identity fragment is visible to any BLE
scanner in range even before pairing; an iOS peer's is not.

### 3.2 Advertising / scan interval

- **Swift:** CoreBluetooth gives the app no direct control over the physical advertising
  interval — `CBPeripheralManager.startAdvertising(_:)` takes only advertisement *data*
  (service UUIDs), not a timing parameter
  (`bitchat/Services/BLE/BLEService.swift:4214-4215` etc.). The app-level "advertising
  behavior" that *is* controllable is scan **duty cycling** (central role), governed by
  `BLEScanDutyPolicy` (`bitchat/Services/BLE/BLEScanDutyPolicy.swift:8-35`): continuous
  scanning is forced when `connectedCount <= 2` or there's been recent traffic; otherwise it
  duty-cycles on/off. Duration constants:
  `bleDutyOnDuration = 5.0s` / `bleDutyOffDuration = 10.0s` (sparse),
  `bleDutyOnDurationDense = 3.0s` / `bleDutyOffDurationDense = 15.0s` (dense, i.e.
  `connectedCount >= bleHighDegreeThreshold = 6`) — all in
  `bitchat/Services/TransportConfig.swift:74-75,195-196,10`.
- **Kotlin:** Explicit `AdvertiseSettings`/`ScanSettings` modes chosen by a power profile —
  `AdvertiseSettings.ADVERTISE_MODE_LOW_LATENCY` / `_BALANCED` / `_LOW_POWER`
  (`app/src/main/java/com/bitchat/android/mesh/PowerManager.kt:211-225`) and
  `ScanSettings.SCAN_MODE_LOW_LATENCY` / `_BALANCED` / `_LOW_POWER`
  (`PowerManager.kt:188-206`) — i.e. Android *can* directly influence the OS advertising
  interval via the Android BLE API's named modes, whereas iOS cannot (CoreBluetooth hides
  this from the app). Applied at
  `app/src/main/java/com/bitchat/android/mesh/BluetoothGattServerManager.kt:387` (advertise)
  and `BluetoothGattClientManager.kt:309` (scan).

**Divergence found (platform-API-driven, not a protocol bug):** no shared "advertising
interval" concept exists at the wire level — each platform's OS BLE stack manages the actual
interval, and the two apps expose different knobs (Android: named `AdvertiseSettings`/
`ScanSettings` modes; iOS: scan duty-cycle on/off timers only, advertising interval is opaque
to the app). spec/ble-transport.md should describe advertising interval as
**implementation-defined / not normative**, and describe duty-cycling as an
application-level power optimization, not a protocol requirement.

### 3.3 Central (scanning) service filter

- Swift: `centralManager.scanForPeripherals(withServices: [BLEService.serviceUUID], ...)`
  — `bitchat/Services/BLE/BLEService.swift:957,3227`. Also re-discovers services
  post-connect via `discoverServices([BLEService.serviceUUID])`
  (lines 3159, 3300, 4085) and characteristic via
  `discoverCharacteristics([BLEService.characteristicUUID], for: service)` (line 3889).
- Kotlin: `.setServiceUuid(ParcelUuid(SERVICE_UUID))` scan filter —
  `app/src/main/java/com/bitchat/android/mesh/BluetoothGattClientManager.kt:253`, and
  matches on `scanRecord?.serviceUuids?.any { it.uuid == SERVICE_UUID }`
  (`BluetoothGattClientManager.kt:419`) plus reads the scan-response service data for the
  peer-ID dedup fragment (`BluetoothGattClientManager.kt:429`).

**Verdict: match** on the core filter (only the shared service UUID is scanned for);
Android additionally consumes the peer-ID service data described in §3.1, which iOS has
nothing equivalent to produce or read.

---

## 4. Summary Table

| Fact | Swift source (this repo) | Kotlin source (bitchat-android) | Match? |
|---|---|---|---|
| Service UUID (mainnet) `F47B5E2D-4A9E-4C5A-9B3F-8E1D2C3A4B5C` | `BLEService.swift:219` | `AppConstants.kt:30` | Yes |
| Characteristic UUID `A1B2C3D4-E5F6-4A5B-8C9D-0E1F2A3B4C5D` | `BLEService.swift:221` | `AppConstants.kt:31` | Yes |
| Characteristic supports notify/write/writeWithoutResponse/read | `BLEService.swift:4123-4128` | `BluetoothGattServerManager.kt:331,346` | Yes (properties not bit-verified) |
| 13-byte fragment header (8 ID + 2 idx BE + 2 total BE + 1 type) | `BLEFragmentAssemblyBuffer.swift:21-53`, `BLEOutboundFragmentPlanner.swift:112-117` | `FragmentPayload.kt:20-111` | Yes |
| Default fragment chunk size 469B / MTU ceiling 512 | `TransportConfig.swift:7`, `BLEService.swift:227` | `AppConstants.kt:41-42` | Yes |
| Max 256 fragments per stream | `BLEOutboundFragmentPlanner.swift:20-22` | `AppConstants.kt:45` | Yes (explicitly cross-platform-contract-driven in Swift comment) |
| Fragment lifetime / timeout 30s | `TransportConfig.swift:176` | `AppConstants.kt:43` | Yes |
| Assembly buffering limits (count/byte caps) | `TransportConfig.swift:9`, `BLEFragmentAssemblyBuffer.swift:203-213` | `AppConstants.kt:43-48` | Conceptual match, different limit shapes (not wire-relevant) |
| Inter-fragment pacing 30/25ms (broadcast/directed) vs 20ms flat | `TransportConfig.swift:192-193` | `FragmentingPacketSender.kt:24` | Diverges (timing only, not wire format) |
| Advertisement contents: service UUID only, no peer ID, no name | `BLEService.swift:4474-4483` | — | iOS only |
| Advertisement/scan-response: service UUID + 8-byte peer-ID service data | — | `BluetoothGattServerManager.kt:387-407` | Android only — **privacy divergence** |
| Advertising interval control | Not exposed by CoreBluetooth (opaque) | Explicit `AdvertiseSettings`/`ScanSettings` modes | Diverges (platform API constraint, not a bug) |
| Scan filtered to service UUID | `BLEService.swift:957,3227` | `BluetoothGattClientManager.kt:253,419` | Yes |
| Scan duty-cycling (on/off windows, sparse vs dense) | `BLEScanDutyPolicy.swift:8-35`, `TransportConfig.swift:74-75,195-196` | Power-profile-based scan mode instead (no on/off duty cycle found in this pass) | Diverges (different power-management strategy, not wire-relevant) |

**Overall: no wire-protocol-breaking divergence found.** GATT UUIDs, characteristic
properties, and the fragment header format are byte-compatible across Swift and Kotlin. The
divergences found are all in operational/power-management behavior (advertising contents,
advertising/scan interval control, duty-cycling strategy, inter-fragment pacing) — real
differences worth documenting in the spec as implementation-defined, but none of them break
interop between an iOS and an Android BitChat node.
