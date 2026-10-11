# Replay profile proposal

Status: experimental fork proposal, not part of the stable v3 match contract.
Schema identifier: `cs2-demo-format/replay/0.1`. No changes to required v3 files,
statistics or existing `parseDemoPackage()` behavior are proposed here.

## Purpose

Exchange replay facts from complete matches, client recordings and standalone
clips without fabricating a score, match aggregate or round winner. A producer
can retain high precision preparation/input frames, actual smoke/fire state and
compressed voice packets when its source contains them.

## Package

`manifest.json` declares `metadata`, `players`, `events`, optional `analysis`,
and an ordered list of segments. Each segment has its own JSON file, so a
consumer can load one segment at a time. ZIP compression is a transport detail.
`parseReplayPackage()` returns metadata plus lazy segment and analysis readers.
Hosts must separately bound archive size, decompression and source trust.

The canonical Zod definitions are in `schemas/index.ts`; generated schemas are
`spec/replayManifest.schema.json`, `replayMetadata`, `replayPlayers`,
`replayEvents`, `replayFrames` and `replaySegment`.

## Time and coordinates

- Tick rate and sample rate are separate manifest fields. Do not assume 64 tick
  or 8 Hz. Frames carry explicit finite ticks, including fractional voice times
  and negative pre-game ticks where the source provides them.
- Segment `startTick` is the earliest normal playback time; `activeStartTick`
  is the default playback entry (freeze end for a regular match). `endTick` is
  the end of playback. Context/preparation samples may precede `startTick`.
- Coordinates are finite Source 2 world units in x/y/z order. They are not
  renderer metres. `position: null` means no observed position, not (0,0,0).
- This draft does not prescribe interpolation, physics or view-engine axes.

## Identity and state

Player IDs are package-local stable strings. Steam ID is nullable; bots,
reconnects and recordings without account information must be representable.
Names are labels, not guaranteed unique identities. A frame may contain any
number of players. Side is a per-frame value, not a fixed account property.

Frames expose position, yaw/pitch, health/alive, armor and equipment, current
weapon/inventory, crouch/airborne/walking state, flash duration in seconds,
scoped state, money and place name. Unobserved values are null. Input is null
when unavailable; otherwise it exposes movement, jump, primary/secondary fire,
walking, pin state and throw strength. Empty inventory is distinct from unknown
inventory.

## Events and effects

Events have stable IDs, type, tick, actor/target/assistant references, observed
position, weapon and health damage. The draft preserves producer event names;
a future common event-type registry can be additive.

Projectiles carry position, owner, entity/lifecycle ID and type. Smoke frames
carry origin, sequence and packed occupancy (`x | y<<5 | z<<10`, five bits per
axis). Fire cells are flat groups of six Source coordinates/normal components.
Voice packets contain Opus bytes encoded as Base64, speaker reference, tick and
duration in seconds; no PCM or inference from a speaking icon is required.

## Capability and forward compatibility

Capabilities distinguish `available`, `empty`, `unavailable`, and `unknown`,
with an optional reason. Missing a feature must not be interpreted as proof
that its event never happened. Consumers can decline or downgrade playback
based on capabilities instead of manufacturing replacement facts.

`producerFields` is a JSON-only, namespaced producer escape hatch for data not
yet in the neutral contract. Consumers must not execute it or treat it as
permission. All fields needed for ordinary 3D replay are independently exposed
in the neutral core. Optional future capabilities should preserve the ordinary
consumer's ability to view the base replay. Application commands, AI prompts,
sessions and collaboration permissions are outside this data format.

## Open decisions for upstream review

- Whether a replay profile should use a separate schema ID, or a future shared
  manifest/profile extension point.
- Event-name normalization and a generic extension registry.
- Binary/chunked encodings for high-volume tracks and large effect data.
- Cryptographic provenance and optional raw-source facts.

This proposal deliberately does not alter v3 match semantics or export invented
statistics merely to satisfy the v3 required-file set.
