# Conformance chapter research: hex test vectors (Swift vs. Kotlin)

Scope: this document covers only the two test-vector sub-investigations
assigned to issue #30 ("Conformance chapter: build checklist + reconcile hex
test vectors"). It is **not** the conformance checklist itself — that
depends on facts from the other six chapter tickets (issues #17-29), which
have not landed yet. This is fact-finding only, feeding a future
`spec/conformance.md`.

## 1. `bitchatTests/Noise/NoiseTestVectors.json` (Swift, this repo)

File: `bitchatTests/Noise/NoiseTestVectors.json` (71 lines).

Format: a top-level JSON array of two vector objects. Each object has:
- `protocol_name` (string) — `bitchatTests/Noise/NoiseTestVectors.json:3,39` — both vectors are `Noise_XX_25519_ChaChaPoly_SHA256`.
- `init_prologue`, `resp_prologue` (hex strings) — `bitchatTests/Noise/NoiseTestVectors.json:4,7` (vector 1) and `:40,44` (vector 2).
- `init_static`, `init_ephemeral`, `resp_static`, `resp_ephemeral` (hex private-key strings) — `bitchatTests/Noise/NoiseTestVectors.json:5-6,8-9` (vector 1), `:42-43,46-47` (vector 2).
- `handshake_hash` (hex string, optional — present only on vector 1) — `bitchatTests/Noise/NoiseTestVectors.json:10`.
- `init_psks` / `resp_psks` (empty arrays, present only on vector 2) — `bitchatTests/Noise/NoiseTestVectors.json:41,45`.
- `messages`: an array of `{payload, ciphertext}` hex-string pairs — vector 1 has 6 messages (`bitchatTests/Noise/NoiseTestVectors.json:11-36`: 3 handshake messages + 3 transport messages), vector 2 has 5 messages (`bitchatTests/Noise/NoiseTestVectors.json:48-69`: 3 handshake + 2 transport).

Provenance, per the doc comment at `bitchatTests/Noise/NoiseProtocolTests.swift:17-28`: vector 1 is the **cacophony** suite vector (`haskell-cryptography/cacophony` `vectors/cacophony.txt`) and vector 2 is the **snow** suite vector (`mcginty/snow` `tests/vectors/snow.txt`). Both target `Noise_XX_25519_ChaChaPoly_SHA256`, the exact pattern bitchat speaks (`NoiseProtocolName` / `NoisePattern.XX`).

Consumption: `bitchatTests/Noise/NoiseProtocolTests.swift` decodes the file via `Codable` struct `NoiseTestVector` (fields declared at `bitchatTests/Noise/NoiseProtocolTests.swift:29-41`, mirroring the JSON keys above 1:1) and loads it from the test bundle at `bitchatTests/Noise/NoiseProtocolTests.swift:676-686` (`Bundle...url(forResource: "NoiseTestVectors", withExtension: "json")`). The test loop logs `"Running test vector \(index + 1): ..."` at `bitchatTests/Noise/NoiseProtocolTests.swift:642` and asserts `testVector.protocol_name == appProtocolName` at `bitchatTests/Noise/NoiseProtocolTests.swift:644-645`, then feeds `init_static`/`init_ephemeral`/`resp_static`/`resp_ephemeral` as hex-decoded `Data` into the handshake at `bitchatTests/Noise/NoiseProtocolTests.swift:691-694`, replaying each `messages[]` entry through the real handshake/transport code and comparing to the vector's `ciphertext`.

## 2. `docs/courier-test-vectors.json` — does not exist

- Not present in this repo's working tree: `find . -iname "*courier-test-vectors*"` returns nothing, and `git log --all --oneline -- docs/courier-test-vectors.json` returns no history (file has never existed on any branch of this fork).
- Not present upstream either: `gh api repos/permissionlesstech/bitchat/contents/docs/courier-test-vectors.json` returns `404 Not Found`.
- The reference comes from jackjackbits's comment on upstream `permissionlesstech/bitchat#1448` (https://github.com/permissionlesstech/bitchat/issues/1448#issuecomment-5084759119), which lists a suggested spec-writing order and says: "test vectors as a follow-up (docs/courier-test-vectors.json and the Noise vectors show the house pattern)". This reads as **aspirational** — a file the maintainer expects someone to create following "the house pattern" set by the Noise vectors (i.e. a JSON array of hex-encoded fixtures like `NoiseTestVectors.json`), not a file that currently exists anywhere in the bitchat org.
- Closest existing courier-related test coverage in this repo, checked for embeddable hex vectors (none found — `grep -n "hex\|\"[0-9a-fA-F]\{20,\}\""` on each returns no matches, i.e. these tests use synthetic/repeated-byte fixtures like `Data(repeating: 0xAB, count: ...)`, not canonical hex vectors):
  - `bitchatTests/NoiseCourierTests.swift` (107 lines)
  - `localPackages/BitFoundation/Tests/BitFoundationTests/CourierEnvelopeTests.swift` (154 lines; e.g. `Data(repeating: 0xAB, count: CourierEnvelope.tagLength)` at line 16)
  - `bitchatTests/CourierStoreTests.swift` (482 lines)
  - `bitchatTests/Services/BridgeCourierServiceTests.swift`
  - `bitchatTests/EndToEnd/CourierEndToEndTests.swift`

**Conclusion for this sub-item**: `docs/courier-test-vectors.json` does not exist yet in either this fork or upstream. `spec/conformance.md` cannot cite it as a source; if a courier conformance chapter is written later, a new vectors file would need to be authored from scratch (following the `NoiseTestVectors.json` JSON-array-of-hex-fields convention jackjackbits alludes to), sourced from `CourierEnvelope`/`CourierStore` fixtures, not adopted from an existing file.

## 3. `permissionlesstech/bitchat-android`'s own Noise vectors

Shallow-cloned read-only to `/tmp/bitchat-android-research-conformance` (`git clone --depth 1 https://github.com/permissionlesstech/bitchat-android`, HEAD at clone time). File found at the exact name given in the ticket:

`app/src/test/kotlin/com/bitchat/android/noise/NoiseExternalVectorTest.kt` (261 lines).

Format divergence: unlike the Swift side, Android does **not** load vectors from an external JSON file — the vector bytes are hardcoded as Kotlin string literals directly in the test class (`VectorMessage` data class at lines 247-253, constructed inline at lines 18-50; static handshake keys/prologue inlined in `vectorState()` at lines 174-192). There is no `.json` fixture file anywhere in the bitchat-android tree matching `*[Nn]oise*[Vv]ector*` or similar.

Content comparison — byte-for-byte identical to Swift vector 1 (cacophony):
- `PROTOCOL = "Noise_XX_25519_ChaChaPoly_SHA256"` (`NoiseExternalVectorTest.kt:256`) matches `bitchatTests/Noise/NoiseTestVectors.json:3`.
- `prologue = hex("4a6f686e2047616c74")` (`NoiseExternalVectorTest.kt:176`) matches `init_prologue`/`resp_prologue` at `bitchatTests/Noise/NoiseTestVectors.json:4,7`.
- Initiator/responder static and ephemeral private keys (`NoiseExternalVectorTest.kt:178-187`) match `init_static`/`init_ephemeral`/`resp_static`/`resp_ephemeral` at `bitchatTests/Noise/NoiseTestVectors.json:5-6,8-9` exactly.
- All 6 `VectorMessage` payload/ciphertext hex pairs (`NoiseExternalVectorTest.kt:18-50`) match all 6 entries of vector 1's `messages` array (`bitchatTests/Noise/NoiseTestVectors.json:11-36`) exactly, including the split point between handshake messages (first 3, `messages.take(3)` at line 62) and transport messages (last 3, indices 3-5 at lines 72-86).

Divergence found: **the Kotlin test only carries the cacophony vector (Swift's vector 1). It has no equivalent of Swift's vector 2 (the snow-suite vector at `bitchatTests/Noise/NoiseTestVectors.json:38-70`, with empty PSK arrays and 5 messages).** So bitchat-android's Noise vector coverage is a strict subset of Swift's: same protocol, same first vector byte-for-byte, but missing the second independent vector suite. Additionally, `handshake_hash` (present on Swift vector 1, `bitchatTests/Noise/NoiseTestVectors.json:10`) is not checked against a fixed expected value on the Kotlin side — `NoiseExternalVectorTest.kt:68` only asserts `initiator.handshakeHash` equals `responder.handshakeHash` (mutual agreement), not equality to the vector's published hash.

Also note: Android additionally covers two things Swift's vector-driven test does not, in the same file: a tampered-ciphertext/invalid-state-transition test (`NoiseExternalVectorTest.kt:97-120`) and a standalone ChaChaPoly AEAD associated-data test (`NoiseExternalVectorTest.kt:122-172`). These aren't "vectors" in the reconciliation sense (no external hex fixture, no Swift counterpart file), just extra unit-test coverage co-located in the same Kotlin file.

### Recommendation (fact-finding only, not a final decision)

- **Adopt `bitchatTests/Noise/NoiseTestVectors.json` as the vector source of truth for `spec/conformance.md`**, since it is a strict superset of what Android currently checks (both of Android's checked values — protocol name, prologue, static/ephemeral keys, all 6 message pairs — are present in and identical to Swift's vector 1), and it is already an external, language-agnostic JSON format suitable for citing/copying into a spec doc, whereas Android's vector is Kotlin source code, not data.
- **Flag, not fix, the gap**: bitchat-android does not yet exercise Swift's vector 2 (the snow vector). The conformance chapter should note this as a known interop-test gap for the Android client (something for a future bitchat-android PR, out of scope for this repo) rather than silently treating the two suites as equivalent.
- Do not invent a new "unified" vector format — the existing JSON shape (`protocol_name`, `init_*`/`resp_*`, optional `handshake_hash`, optional `*_psks`, `messages[]` of `{payload, ciphertext}`) already round-trips cleanly through both a Swift `Codable` struct and (manually) through Kotlin literals; changing it would cost more than it buys.

## Sources consulted

- `bitchatTests/Noise/NoiseTestVectors.json` (this repo, worktree of `dsfx3d/bitchat`)
- `bitchatTests/Noise/NoiseProtocolTests.swift` (this repo)
- `bitchatTests/NoiseCourierTests.swift`, `localPackages/BitFoundation/Tests/BitFoundationTests/CourierEnvelopeTests.swift`, `bitchatTests/CourierStoreTests.swift` (this repo)
- `permissionlesstech/bitchat-android`, shallow clone (depth 1) at `/tmp/bitchat-android-research-conformance`, file `app/src/test/kotlin/com/bitchat/android/noise/NoiseExternalVectorTest.kt`
- `permissionlesstech/bitchat#1448` (issue, upstream), comment by jackjackbits: https://github.com/permissionlesstech/bitchat/issues/1448#issuecomment-5084759119
- `gh api repos/permissionlesstech/bitchat/contents/docs/courier-test-vectors.json` → 404 (confirms file does not exist upstream)
