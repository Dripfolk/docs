# Shadow Creatures: Multiplayer Shared-World Architecture
## Complete Production Technical Specification

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Network Architecture](#2-network-architecture)
3. [Entity State & Wire Protocol](#3-entity-state--wire-protocol)
4. [World Partitioning & Interest Management](#4-world-partitioning--interest-management)
5. [Client-Side Prediction & Interpolation](#5-client-side-prediction--interpolation)
6. [Creature AI Distribution](#6-creature-ai-distribution)
7. [Conversation System](#7-conversation-system)
8. [Action System](#8-action-system)
9. [Persistence & Database Design](#9-persistence--database-design)
10. [Scaling Architecture](#10-scaling-architecture)
11. [Security](#11-security)
12. [Infrastructure & Deployment](#12-infrastructure--deployment)
13. [Frontend Architecture](#13-frontend-architecture)
14. [Failure Modes & Mitigations](#14-failure-modes--mitigations)
15. [Cost Analysis](#15-cost-analysis)

---

## 1. System Architecture Overview

### Text-Based Architecture Diagram

```
                           CLIENTS (Browsers)
                    ┌──────────┐  ┌──────────┐  ┌──────────┐
                    │ Client A │  │ Client B │  │ Client N │
                    │ Canvas2D │  │ Canvas2D │  │ Canvas2D │
                    │ 60fps    │  │ 60fps    │  │ 60fps    │
                    └────┬─────┘  └────┬─────┘  └────┬─────┘
                         │WebSocket    │WS           │WS
                         │(binary)     │(binary)     │(binary)
                    ─────┴─────────────┴─────────────┴──────
                              │
                    ┌─────────┴──────────┐
                    │   LOAD BALANCER    │
                    │ (Sticky Sessions)  │
                    │  HAProxy / AWS ALB │
                    └─────────┬──────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────┴──────┐     ┌─────┴──────┐     ┌─────┴──────┐
    │  ZONE SVR  │     │  ZONE SVR  │     │  ZONE SVR  │
    │  Zone A    │     │  Zone B    │     │  Zone C    │
    │ (0,0-1000) │     │(1000-2000) │     │(2000-3000) │
    │            │     │            │     │            │
    │ Game Loop  │     │ Game Loop  │     │ Game Loop  │
    │ 20Hz tick  │     │ 20Hz tick  │     │ 20Hz tick  │
    │            │     │            │     │            │
    │ Creature   │     │ Creature   │     │ Creature   │
    │ Simulation │     │ Simulation │     │ Simulation │
    │ (AI+Phys)  │     │ (AI+Phys)  │     │ (AI+Phys)  │
    └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
          │                   │                   │
          └───────┬───────────┼───────────┬───────┘
                  │           │           │
           ┌──────┴───────────┴───────────┴──────┐
           │         REDIS CLUSTER               │
           │  Pub/Sub: cross-zone messaging      │
           │  Cache: hot creature state           │
           │  Sorted Sets: spatial index          │
           └──────────────────┬──────────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
    ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
    │ PostgreSQL  │   │ AI Worker   │   │ Conversation │
    │  Primary +  │   │ Pool (N)    │   │ Worker Pool  │
    │  Replicas   │   │             │   │              │
    │             │   │ Behavior    │   │ LLM Gateway  │
    │ Creatures   │   │ Tree Exec   │   │ (Claude API) │
    │ Users       │   │ Pathfinding │   │              │
    │ World State │   │ Decisions   │   │ Queue +      │
    │ Convos      │   │             │   │ Dedup        │
    │ Communities │   │ 50ms budget │   │              │
    └─────────────┘   └─────────────┘   └──────────────┘
```

### Design Principles

1. **Authoritative Server**: The server is the single source of truth. Clients are dumb renderers with prediction. No client can modify authoritative state directly.
2. **Zone-Based Architecture**: The world is divided into spatial zones. Each zone server owns the simulation for all creatures within its boundaries. This is the Agar.io / Slither.io model, not a peer-to-peer model.
3. **Interest Management**: Each client only receives data for creatures within its viewport plus a margin. A client viewing a 1920x1080 window at 3x zoom sees roughly 640x360 world units -- it only needs creatures in that rectangle plus ~200 unit margin.
4. **Delta Compression**: Only changed fields are sent. Full snapshots are sent on join and periodically for resync.
5. **AI is Server-Side**: All creature AI runs on the server. Clients never run AI logic. This prevents cheating and ensures consistency.

### Why Not Colyseus / Photon / PlayCanvas?

After evaluating the options:

- **Colyseus**: Good starting point for <500 entities. Its automatic state synchronization uses JSON diffing which becomes expensive at 5000 entities. Its schema system adds overhead per field. The real problem: Colyseus rooms are single-process, so you hit a ceiling at ~500-800 creatures per room before the 50ms tick budget is blown. However, Colyseus is an excellent choice for Phase 1 (MVP, <100 users) because it handles 90% of the networking boilerplate. **Recommendation: Start with Colyseus, migrate to custom when you hit scaling walls.**
- **Photon**: Cloud-hosted, expensive at scale, limited server-side logic. Not suitable for heavy AI simulation.
- **Custom WebSocket**: Maximum control, but 3-6 months of networking code. Reserve for Phase 3+.

**Phase strategy:**
- Phase 1 (MVP): Colyseus + single zone + PostgreSQL. Supports ~100 concurrent users.
- Phase 2 (Growth): Custom zone servers + Redis pub/sub + multiple zones. Supports ~1000 users.
- Phase 3 (Scale): Full distributed architecture with Kubernetes orchestration. Supports 10,000+.

---

## 2. Network Architecture

### 2.1 Connection Lifecycle

```
Client                          Load Balancer              Zone Server
  │                                  │                         │
  │──── HTTPS GET /api/auth ────────>│                         │
  │<─── JWT token + zone assignment──│                         │
  │                                  │                         │
  │──── WSS://zone-a.server ────────>│──── route by zone ─────>│
  │                                  │                         │
  │<──────────── FULL SNAPSHOT ──────────────────────────────── │
  │     (all creatures in viewport)                            │
  │                                  │                         │
  │<──────────── DELTA UPDATES (20Hz) ─────────────────────────│
  │                                  │                         │
  │──── INPUT COMMANDS ─────────────────────────────────────── >│
  │     (move-to, interact, etc)     │                         │
  │                                  │                         │
  │<──────────── ACK + CORRECTION ─────────────────────────────│
```

### 2.2 Server Tick Architecture

The server runs a fixed-timestep game loop at **20 ticks per second** (50ms per tick). This is the Gabriel Gambetta model used by virtually every successful browser multiplayer game.

```
┌─────────────────── 50ms TICK ───────────────────┐
│                                                  │
│  1. RECEIVE INPUTS          (~2ms)               │
│     - Dequeue all client commands                │
│     - Validate inputs (rate limit, bounds)       │
│     - Apply to input buffer                      │
│                                                  │
│  2. SIMULATE                (~15ms budget)       │
│     - Physics: move creatures, resolve collisions│
│     - AI: run behavior trees for due creatures   │
│       (staggered: ~1/10 of creatures per tick)   │
│     - Actions: process action queue              │
│     - Conversations: check pending responses     │
│                                                  │
│  3. BUILD SNAPSHOTS         (~5ms)               │
│     - For each connected client:                 │
│       - Query spatial index for viewport         │
│       - Diff against last-sent state             │
│       - Build binary delta message               │
│                                                  │
│  4. SEND                    (~3ms)               │
│     - Flush all client messages                  │
│     - Send cross-zone messages via Redis         │
│                                                  │
│  5. PERSIST (async, every 30s) (~0ms inline)     │
│     - Queue dirty creatures for DB write         │
│                                                  │
│  Remaining: ~25ms HEADROOM                       │
└──────────────────────────────────────────────────┘
```

**Why 20Hz?** At 20Hz, the client has 50ms between server updates. With interpolation, this is imperceptible for the slow-moving creature simulation (creatures move at 60 world-units/sec max -- that is 3 world-units per tick, roughly 9 pixels at 3x scale). Fighting or fast-action features might need 30Hz, but 20Hz is the correct starting point for this simulation-heavy game.

**Why not 60Hz?** CPU budget. At 5000 creatures, even simple physics takes ~5ms. AI behavior trees take ~10ms (staggered). Building per-client snapshots for 100 clients takes ~5ms. At 60Hz you have only 16.6ms total -- impossible. 20Hz gives 50ms, comfortable headroom.

### 2.3 Interest Management -- What Data Goes Where

This is the single most important optimization. Without it, 100 clients x 5000 creatures x 20Hz = 10 million entity-updates per second, which is absurd.

**Algorithm: Viewport + Margin AOI (Area of Interest)**

```
For each client C:
  viewport = C.camera_position +/- (C.screen_size / C.zoom / 2)
  aoi = viewport.expand(MARGIN = 300 world units)

  visible_creatures = spatial_index.query_rect(aoi)

  for each creature in visible_creatures:
    if creature.last_sent_state[C] != creature.current_state:
      queue_delta(C, creature)
    else:
      skip (no change = no send)

  for each creature in C.previously_visible - visible_creatures:
    queue_despawn(C, creature.id)

  for each creature in visible_creatures - C.previously_visible:
    queue_spawn(C, creature)  // full state for new creatures
```

**Spatial Index**: A grid-based spatial hash with 500x500 world-unit cells. At 5000 creatures in a 10,000x10,000 world, that is 20x20 = 400 cells, average 12.5 creatures per cell. A viewport query touches ~6-12 cells. This is O(1) lookup per cell.

```
class SpatialGrid {
  cellSize = 500;
  cells = new Map<string, Set<CreatureId>>();

  key(x, y) { return `${Math.floor(x/500)},${Math.floor(y/500)}`; }

  insert(id, x, y) { this.cells.get(this.key(x,y))?.add(id); }
  remove(id, x, y) { this.cells.get(this.key(x,y))?.delete(id); }

  queryRect(x1, y1, x2, y2): CreatureId[] {
    const results = [];
    for (let cx = Math.floor(x1/500); cx <= Math.floor(x2/500); cx++)
      for (let cy = Math.floor(y1/500); cy <= Math.floor(y2/500); cy++)
        results.push(...(this.cells.get(`${cx},${cy}`) ?? []));
    return results;
  }
}
```

**Result**: Each client receives updates for ~20-80 creatures (viewport contents), not 5000. At 20Hz with delta compression, this is ~2-8 KB/s per client. 100 clients = 200-800 KB/s total outbound, easily handled by a single server.

### 2.4 What Happens When User A's Creature Walks Toward User B's Creature

**Same Zone Server (common case):**
1. User A's client sends: `MOVE_TO {target: [1500, 2200]}`
2. Server validates: is this creature owned by User A? Is the target reachable? Is movement speed legal?
3. Server updates creature A's pathfinding target. Each tick, creature A moves toward the target at its speed.
4. Both User A and User B are connected to the same zone server. Both have creature A in their viewport.
5. The delta update for creature A (new position) is sent to both clients simultaneously.
6. Both clients interpolate creature A's position smoothly.
7. When creatures A and B are within interaction range (50 world units), the server triggers proximity events (enables "talk" action, shows interaction UI hints).

**Different Zone Servers (creature near zone boundary):**
1. Creature A is at position (990, 500) in Zone A. Zone B starts at x=1000.
2. Zone A's spatial query for clients in Zone B picks up creature A because it is within the 300-unit margin.
3. Zone A publishes creature A's state to Redis channel `zone:B:entities`.
4. Zone B receives the state and includes it in snapshots for its clients.
5. When creature A crosses x=1000, Zone A performs a **handoff**: it serializes creature A's full state, publishes it to Zone B, and Zone B takes ownership.
6. Zone A sends a `ZONE_TRANSFER` message to User A's client, which reconnects its WebSocket to Zone B's server.

```
Zone A                    Redis                    Zone B
  │                         │                        │
  │  creature at (990,500)  │                        │
  │─── PUB zone:B:border ──>│───> SUB zone:B:border──│
  │    {id, pos, state}     │    {id, pos, state}    │
  │                         │                        │
  │  creature crosses 1000  │                        │
  │─── PUB zone:B:handoff ─>│───> SUB zone:B:handoff─│
  │    {FULL creature state} │   {create creature}   │
  │                         │                        │
  │  delete creature locally │                       │
  │─── WS to client: ──────────────────────────────>│
  │    ZONE_TRANSFER zone-b  │                       │
  │                         │                        │
  │                         │  client reconnects ──> │
  │                         │  <── FULL SNAPSHOT ─── │
```

### 2.5 Delta Compression vs Full Snapshots

**Full Snapshot**: Sent on:
- Client first connects
- Client crosses zone boundary
- Every 60 seconds as a resync safety net
- After detecting client desync (client sends state hash that does not match server)

**Delta Update**: Sent every tick (20Hz). Contains ONLY fields that changed since the last acknowledged snapshot for that client.

**Delta compression approach** (not Quake-style XOR, but field-level diffing -- simpler and sufficient for this tick rate):

```
Per creature, track a "last sent" state per client.
Compare current state vs last-sent.
Only encode changed fields using a bitmask.
```

### 2.6 Clock Synchronization

Use the **NTP-lite** approach (same as Overwatch, Valve Source Engine):

1. Client sends `PING {client_time: T1}` every 2 seconds
2. Server responds `PONG {client_time: T1, server_time: T2}`
3. Client calculates: `RTT = now - T1`, `one_way = RTT / 2`, `clock_offset = T2 - (T1 + one_way)`
4. Client maintains a running average of `clock_offset` (exponential moving average, alpha=0.1)
5. All server timestamps are converted to client time for interpolation

**Why this matters**: Interpolation requires knowing "this snapshot is from server-time T". Without clock sync, interpolation jitters.

### 2.7 Handling Packet Loss and High Latency

**WebSocket runs over TCP**, so there is no packet loss per se -- but there is **head-of-line blocking** and **delayed delivery**.

**High latency (300ms):**
- Client-side prediction covers the gap. When the user clicks to move, the client immediately starts moving the creature locally.
- Server confirms or corrects 300ms later. If the correction is small (<5 world units), the client smoothly interpolates toward the corrected position over 200ms. If large (>20 units -- indicates cheating or major desync), the client snaps instantly.

**Connection stall (>2 seconds no data):**
- Client shows a "Reconnecting..." overlay
- Client attempts to reconnect with exponential backoff: 1s, 2s, 4s, 8s, max 30s
- On reconnect, server sends full snapshot
- Creatures the user owns continue their AI behavior on the server (they don't freeze)

**Ordering**: WebSocket guarantees ordered delivery. No need for sequence numbers for ordering, but we use them for delta compression (client ACKs last received sequence, server diffs against that).

---

## 3. Entity State & Wire Protocol

### 3.1 Complete Creature Data Structure

```typescript
// FULL creature state (stored in memory and DB)
interface Creature {
  // Identity (immutable after creation)
  id:           uint32;     // 4 bytes - unique creature ID
  ownerId:      uint32;     // 4 bytes - user who owns this creature
  name:         string;     // variable - creature's name
  createdAt:    uint32;     // 4 bytes - unix timestamp

  // Appearance (changes rarely -- on morph/evolution only)
  morphData:    MorphData;  // ~200 bytes - body shape variant index + parameters
  bodyScale:    float16;    // 2 bytes - overall size (0.5 - 3.0)
  colorHue:     uint8;      // 1 byte  - hue shift (0-255)
  eyeVariant:   uint8;      // 1 byte  - which eye shape
  legCount:     uint8;      // 1 byte  - number of leg pairs (2-6)

  // Position & Movement (changes every tick when moving)
  x:            float32;    // 4 bytes - world X
  y:            float32;    // 4 bytes - world Y
  angle:        uint16;     // 2 bytes - facing direction (0-65535 maps to 0-2pi)
  speed:        uint8;      // 1 byte  - current speed (0-255 maps to 0-MAX_SPEED)
  walkPhase:    uint8;      // 1 byte  - walk animation cycle (0-255 maps to 0-1)

  // State (changes on events)
  health:       uint16;     // 2 bytes - current HP (0-65535)
  maxHealth:    uint16;     // 2 bytes - max HP
  energy:       uint8;      // 1 byte  - stamina/energy (0-255)
  mood:         uint8;      // 1 byte  - enum: 0=neutral,1=happy,2=sad,3=angry,4=scared,5=curious,6=sleepy
  currentAction:uint8;      // 1 byte  - enum: 0=idle,1=walking,2=talking,3=fighting,4=researching,5=sleeping,6=eating,7=socializing
  actionTarget: uint32;     // 4 bytes - creature/object ID being interacted with (0=none)

  // Personality (immutable after generation)
  personality:  PersonalityVector; // 5 bytes - see below

  // Social
  communityId:  uint32;     // 4 bytes - which community (0=none)
  reputation:   int16;      // 2 bytes - social standing (-32768 to 32767)

  // AI internal (server-only, never sent to clients)
  behaviorState: BehaviorTreeState; // complex, server-only
  pathNodes:     Vec2[];            // current path, server-only
  memoryLog:     string[];          // recent memories, server-only
}

// Personality is 5 dimensions, each 0-255
interface PersonalityVector {
  aggression:   uint8;  // 1 byte - 0=pacifist, 255=berserker
  curiosity:    uint8;  // 1 byte - 0=incurious, 255=explorer
  sociability:  uint8;  // 1 byte - 0=hermit, 255=extrovert
  loyalty:      uint8;  // 1 byte - 0=fickle, 255=devoted
  intelligence: uint8;  // 1 byte - 0=simple, 255=genius
}

// Morph data references the shape system from the existing code
interface MorphData {
  baseShapeId:  uint8;   // which base body shape (index into shape library)
  compactness:  uint8;   // 0=elongated (base), 255=compact
  legsProfile:  uint8;   // leg length/thickness variant
  headProfile:  uint8;   // head shape variant
}
```

### 3.2 Wire Protocol -- Binary Message Format

All messages use a binary protocol over WebSocket. The first byte is always the message type.

**Message Types:**

```
CLIENT -> SERVER:
  0x01  MOVE_TO         { targetX: f32, targetY: f32 }
  0x02  STOP            { }
  0x03  INTERACT        { targetCreatureId: u32, actionType: u8 }
  0x04  SPEAK           { targetCreatureId: u32, message: utf8 }
  0x05  PING            { clientTime: f64 }
  0x06  VIEWPORT        { x: f32, y: f32, w: f32, h: f32 }
  0x07  ACK_SNAPSHOT    { sequenceNum: u32 }
  0x08  SPAWN_CREATURE  { name: utf8, personality: 5xu8 }
  0x09  SET_BEHAVIOR    { creatureId: u32, priority: u8, behaviorId: u8 }

SERVER -> CLIENT:
  0x81  FULL_SNAPSHOT   { seq: u32, serverTime: f64, count: u16, creatures: [...] }
  0x82  DELTA_UPDATE    { seq: u32, serverTime: f64, basedOnSeq: u32, deltas: [...] }
  0x83  CREATURE_SPAWN  { creature: FullCreatureData }
  0x84  CREATURE_DESPAWN{ creatureId: u32 }
  0x85  PONG            { clientTime: f64, serverTime: f64 }
  0x86  SPEECH_BUBBLE   { creatureId: u32, text: utf8, duration: u16 }
  0x87  ACTION_EVENT    { actorId: u32, targetId: u32, actionType: u8, data: bytes }
  0x88  ZONE_TRANSFER   { newZoneUrl: utf8, token: utf8 }
  0x89  ERROR           { code: u16, message: utf8 }
  0x8A  COMMUNITY_EVENT { type: u8, communityId: u32, data: bytes }
```

### 3.3 Delta Update Binary Layout

```
DELTA_UPDATE message:
  [0]     u8   messageType = 0x82
  [1-4]   u32  sequenceNumber
  [5-12]  f64  serverTime
  [13-16] u32  basedOnSequence (client's last ACK'd snapshot)
  [17-18] u16  entityCount (how many creatures have updates)

  For each entity:
    [+0..+3]  u32  creatureId
    [+4..+5]  u16  fieldBitmask (which fields are present)
    [+6..]    ...  field data (only fields set in bitmask)

Field bitmask bits:
  bit 0:  x (f32, 4 bytes)
  bit 1:  y (f32, 4 bytes)
  bit 2:  angle (u16, 2 bytes)
  bit 3:  speed (u8, 1 byte)
  bit 4:  walkPhase (u8, 1 byte)
  bit 5:  health (u16, 2 bytes)
  bit 6:  energy (u8, 1 byte)
  bit 7:  mood (u8, 1 byte)
  bit 8:  currentAction (u8, 1 byte)
  bit 9:  actionTarget (u32, 4 bytes)
  bit 10: communityId (u32, 4 bytes)
  bit 11: reputation (i16, 2 bytes)
  bit 12: bodyScale (f16, 2 bytes) -- rare
  bit 13: (reserved)
  bit 14: (reserved)
  bit 15: (reserved)
```

### 3.4 Bytes Per Creature Per Update

**Typical moving creature** (only position + angle + speed + walkPhase changed):
- creatureId: 4 bytes
- bitmask: 2 bytes
- x: 4, y: 4, angle: 2, speed: 1, walkPhase: 1 = 12 bytes
- **Total: 18 bytes per moving creature**

**Idle creature** (nothing changed): **0 bytes** (not included in delta)

**Creature changing action** (starts talking):
- creatureId: 4, bitmask: 2, currentAction: 1, actionTarget: 4, mood: 1
- **Total: 12 bytes**

**Worst case per update message** (50 creatures all moving in viewport):
- Header: 19 bytes
- 50 creatures x 18 bytes = 900 bytes
- **Total: ~919 bytes per tick**
- At 20Hz: **~18.4 KB/s per client**

**Typical case** (20 creatures visible, 5 moving):
- Header: 19 bytes
- 5 creatures x 18 bytes = 90 bytes
- **Total: ~109 bytes per tick**
- At 20Hz: **~2.2 KB/s per client**

This is extremely manageable. Even on mobile networks.

### 3.5 Speech / Conversation Data

Speech is sent as a separate message type, not in the delta stream, because:
1. It is variable-length (text)
2. It is event-driven, not every-tick
3. It needs to be reliably delivered (not just latest-state-wins)

```
SPEECH_BUBBLE (0x86):
  [0]     u8   messageType
  [1-4]   u32  creatureId (who is speaking)
  [5-6]   u16  textLength
  [7..]   utf8 text
  [+0..+1] u16 durationMs (how long to show bubble)
  [+2]     u8  emotionTag (0=neutral, 1=excited, 2=questioning, etc.)
```

### 3.6 Creature Spawning (Breeding)

When two creatures breed:
1. Server validates: both creatures are in proximity, both have sufficient energy, breeding cooldown elapsed
2. Server generates new creature: inherits personality traits (crossover + mutation), appearance blending
3. Server sends `CREATURE_SPAWN` to all clients who have the parents in their viewport
4. The new creature appears with a spawn animation (client-side only)
5. Ownership: if both parents have the same owner, new creature belongs to that owner. If different owners, ownership is... the user who initiated the breed action? Or random? (Product decision -- flag for PM.)

```
CREATURE_SPAWN (0x83):
  [0]      u8   messageType
  [1-4]    u32  creatureId
  [5-8]    u32  ownerId
  [9]      u8   nameLength
  [10..]   utf8 name
  [+0..+3] f32  x
  [+4..+7] f32  y
  [+8..+9] u16  angle
  [+10]    u8   bodyScale
  [+11]    u8   morphBaseShape
  [+12]    u8   morphCompactness
  [+13]    u8   eyeVariant
  [+14]    u8   legCount
  [+15..+19] personality (5 bytes)
  [+20..+21] u16 health
  [+22..+23] u16 maxHealth
  [+24]    u8   mood
  [+25]    u8   currentAction
  ... (full initial state)
```

---

## 4. World Partitioning & Interest Management

### 4.1 World Structure

```
WORLD: 10,000 x 10,000 world units
(at 3x pixel scale, this is 30,000 x 30,000 pixels)
(at 1920px viewport = 640 world units visible width)

┌──────────┬──────────┬──────────┬──────────┐
│ Zone A   │ Zone B   │ Zone C   │ Zone D   │
│ (0-2500, │ (2500-   │ (5000-   │ (7500-   │
│  0-5000) │  5000,   │  7500,   │  10000,  │
│          │  0-5000) │  0-5000) │  0-5000) │
├──────────┼──────────┼──────────┼──────────┤
│ Zone E   │ Zone F   │ Zone G   │ Zone H   │
│ (0-2500, │ (2500-   │ (5000-   │ (7500-   │
│  5000-   │  5000,   │  7500,   │  10000,  │
│  10000)  │  5000-   │  5000-   │  5000-   │
│          │  10000)  │  10000)  │  10000)  │
└──────────┴──────────┴──────────┴──────────┘

Each zone: 2500 x 5000 world units
At 5000 creatures / 8 zones = ~625 creatures per zone
At uniform distribution. Hotspots may have more.
```

### 4.2 Zone Boundary Handling

The hard problem: what happens at zone boundaries?

**Overlap regions**: Each zone extends 300 world units past its boundary into neighboring zones. Creatures in the overlap region are **simulated by the owning zone** but **visible to clients in the neighboring zone**.

```
Zone A boundary at x=2500:

Zone A simulates:  x = 0 to 2800 (300 unit overlap)
Zone B simulates:  x = 2200 to 5000 (300 unit overlap)

Overlap:           x = 2200 to 2800

Creatures in 2200-2500: owned by Zone A, mirrored to Zone B
Creatures in 2500-2800: owned by Zone B, mirrored to Zone A
```

**Mirroring** uses Redis pub/sub. Zone A publishes creature positions in the overlap region to channel `zone:A:border:right`. Zone B subscribes and includes those creatures in its spatial index as read-only "ghost" entities.

**Handoff**: When a creature's authoritative position crosses the boundary midpoint, ownership transfers. This is a two-phase commit:
1. Source zone publishes `HANDOFF_REQUEST` with full creature state
2. Destination zone confirms `HANDOFF_ACK`
3. Source zone deletes creature, marks it as transferred
4. If ACK not received in 500ms, source zone retains ownership (creature bounces back)

### 4.3 Spatial Indexing Within a Zone

Two-level spatial index:

**Level 1: Coarse Grid** (500x500 cells) for interest management queries
- Used every tick to determine which creatures each client can see
- O(1) insert/remove/query per cell

**Level 2: Fine Grid** (50x50 cells) for proximity detection
- Used for interaction ranges, collision detection, AI awareness
- "Which creatures are within 50 units of creature X?" -- query the fine grid cell + neighbors

```typescript
class TwoLevelSpatialIndex {
  coarse = new SpatialGrid(500);  // for viewport culling
  fine   = new SpatialGrid(50);   // for proximity

  update(id: number, oldX: number, oldY: number, newX: number, newY: number) {
    this.coarse.move(id, oldX, oldY, newX, newY);
    this.fine.move(id, oldX, oldY, newX, newY);
  }

  queryViewport(x1: number, y1: number, x2: number, y2: number): number[] {
    return this.coarse.queryRect(x1, y1, x2, y2);
  }

  queryProximity(x: number, y: number, radius: number): number[] {
    return this.fine.queryRect(x - radius, y - radius, x + radius, y + radius);
  }
}
```

### 4.4 Handling Hotspots

If 2000 creatures converge on one zone (community gathering, popular area):

1. **Dynamic zone splitting**: The overloaded zone splits into 2 or 4 sub-zones. The orchestrator (Kubernetes controller or a management service) spins up new zone server processes and redistributes creatures.
2. **Interest management throttling**: If a client has >200 creatures in viewport, reduce the update rate for distant/idle creatures to 5Hz while keeping nearby/active creatures at 20Hz. This is **priority-based AOI**.
3. **LOD (Level of Detail) for networking**: Creatures >500 units from camera get reduced updates (position only, no animation state). Creatures >1000 units get 2Hz updates.

```
Distance from camera center:
  0-200 units:   FULL update, 20Hz (all fields)
  200-500 units:  REDUCED update, 10Hz (position + action only)
  500-1000 units: MINIMAL update, 5Hz (position only)
  >1000 units:    Not sent (outside AOI)
```

---

## 5. Client-Side Prediction & Interpolation

### 5.1 The Core Problem

Server sends updates at 20Hz (every 50ms). Client renders at 60fps (every 16.7ms). Between server updates, what does the client draw?

### 5.2 Entity Interpolation (for OTHER users' creatures)

The client does NOT predict other users' creatures. Instead, it **interpolates between the two most recent server snapshots**, rendering the world as it was ~50-100ms in the past.

```
Timeline:
  Server tick 100 (t=5.000s): creature at (100, 200)
  Server tick 101 (t=5.050s): creature at (103, 202)

  Client receives tick 100 at client-time 5.080s (80ms latency)
  Client receives tick 101 at client-time 5.130s

  At client render time 5.110s:
    interpolation_time = 5.110 - INTERP_DELAY(100ms) = 5.010s
    t = (5.010 - 5.000) / (5.050 - 5.000) = 0.2
    rendered_x = lerp(100, 103, 0.2) = 100.6
    rendered_y = lerp(200, 202, 0.2) = 200.4
```

**INTERP_DELAY**: Set to 100ms (2 server ticks). This gives us a buffer of two snapshots to interpolate between. If a snapshot arrives late (jitter), we still have the previous pair. This is the standard approach from Valve's Source Engine.

**Jitter buffer**: The client maintains a buffer of the last 3 snapshots. Interpolation always uses the two snapshots that bracket `current_time - INTERP_DELAY`. If the buffer runs dry (severe packet delay), the client switches to **extrapolation** (dead reckoning) for up to 200ms, then freezes the creature.

```typescript
class InterpolationBuffer {
  snapshots: { time: number; state: CreatureState }[] = [];
  INTERP_DELAY = 0.1; // 100ms

  addSnapshot(serverTime: number, state: CreatureState) {
    this.snapshots.push({ time: serverTime, state });
    // Keep last 5 snapshots
    if (this.snapshots.length > 5) this.snapshots.shift();
  }

  getInterpolatedState(renderTime: number): CreatureState | null {
    const t = renderTime - this.INTERP_DELAY;

    // Find bracketing snapshots
    for (let i = 0; i < this.snapshots.length - 1; i++) {
      const a = this.snapshots[i];
      const b = this.snapshots[i + 1];
      if (a.time <= t && t <= b.time) {
        const frac = (t - a.time) / (b.time - a.time);
        return lerpCreatureState(a.state, b.state, frac);
      }
    }

    // If we're past all snapshots, extrapolate from last
    const last = this.snapshots[this.snapshots.length - 1];
    if (last && t - last.time < 0.2) {
      return extrapolate(last.state, t - last.time);
    }

    return null; // stale -- freeze creature
  }
}
```

### 5.3 Client-Side Prediction (for the user's OWN creatures)

When User A clicks to move their creature, the client does NOT wait for the server. It immediately starts moving the creature locally using the same movement logic the server uses.

```
1. User clicks at (500, 300)
2. Client sends MOVE_TO {target: [500, 300]} to server
3. Client IMMEDIATELY starts moving the creature toward (500, 300) at the creature's speed
4. ~80ms later, server processes the command and sends back the authoritative state
5. Client compares its predicted position vs server position
6. If difference < 5 units: smoothly correct over 200ms (invisible to user)
7. If difference > 20 units: snap to server position (indicates desync)
```

**Why this works for this game**: Creatures move slowly (60 units/sec max). In 80ms of latency, a creature moves 4.8 units. The prediction error is negligible. This is not a twitch shooter -- small corrections are invisible.

**Server reconciliation**: The server includes a `lastProcessedInput` sequence number in its responses. The client replays any unacknowledged inputs on top of the server state. This is the standard Gabriel Gambetta approach.

### 5.4 Walk Animation Sync

The client's creature renderer already uses `walkPhase` (0-1 cycle) and `speed` to drive the leg animation. The server sends `walkPhase` as a uint8, but the client does NOT directly use this for rendering. Instead:

1. Client receives `speed` from server
2. Client locally advances `walkPhase` at `WALK_CYCLE_SPEED * (speed/MAX_SPEED) * dt` -- the same formula the server uses
3. Every ~2 seconds, client resyncs `walkPhase` with the server value to prevent drift
4. This ensures smooth 60fps leg animation without needing 60Hz position data

### 5.5 Breathing Animation

Breathing is entirely client-side. The server does not send breathing state. All clients run `applyBreathWave()` independently with their own local time. This is fine because breathing is a cosmetic visual, not gameplay-relevant, and slight differences between clients are imperceptible.

---

## 6. Creature AI Distribution

### 6.1 Where AI Runs

**All AI runs on the server.** Period. No exceptions.

Rationale:
- Prevents cheating (client cannot make creatures behave illegally)
- Ensures consistency (all clients see the same creature behavior)
- Offline creatures must continue to behave -- only the server can do this
- AI involves reading other creatures' states, which only the server has complete access to

### 6.2 Behavior Tree Architecture

Each creature runs a behavior tree. The tree is evaluated at **2Hz** (every 500ms), not every tick. Movement and physics run at 20Hz, but decision-making runs at 2Hz to save CPU.

**Staggered evaluation**: With 625 creatures per zone, evaluating all at 2Hz means ~1250 evaluations per second. At 50ms tick, that is ~62 evaluations per tick. Each evaluation takes ~0.1-0.5ms. Budget: 6-31ms per tick just for AI -- within the 15ms budget if staggered.

**Staggering algorithm**: Creatures are assigned a tick offset based on `creatureId % 10`. Each tick, only creatures with `offset == (currentTick % 10)` get their behavior tree evaluated. This spreads the load evenly.

```typescript
// In the game loop:
for (const creature of zone.creatures) {
  if (creature.id % 10 === currentTick % 10) {
    creature.behaviorTree.evaluate(creature, zone.spatialIndex, currentTime);
  }
  // Physics/movement runs EVERY tick for ALL creatures
  creature.updatePhysics(dt);
}
```

### 6.3 Behavior Tree Structure

```
ROOT (Selector -- try each in priority order)
├── CRITICAL (Sequence)
│   ├── Health < 20%? --> Flee from threat
│   └── Energy < 10%? --> Find food / rest
│
├── SOCIAL (Selector, weight = personality.sociability)
│   ├── See friend nearby? --> Approach and greet
│   ├── See stranger? --> Evaluate (curiosity vs caution)
│   ├── In conversation? --> Continue conversation
│   └── Lonely timer > 60s? --> Seek nearest creature
│
├── TASK (Selector)
│   ├── Has active task? --> Continue task
│   ├── Community needs? --> Pick community task
│   └── Personality-driven: explore / research / patrol
│
├── WANDER (Sequence)
│   ├── Pick random nearby point
│   └── Walk to point
│
└── IDLE
    └── Stand still, breathe, look around
```

**CPU cost per evaluation**: ~0.1ms for simple decisions (already near food, no threats), ~0.5ms for complex (pathfinding + social evaluation). Average: ~0.2ms.

### 6.4 Cross-Zone AI Interactions

When creature A (Zone 1) wants to interact with creature B (Zone 2):

**Case 1: Creatures are near zone boundary.** Both zones have the other creature as a ghost entity. Zone 1 detects proximity via its spatial index. Zone 1 publishes an `INTERACTION_REQUEST` to Redis. Zone 2 receives it and both zones coordinate the interaction.

**Case 2: Creature A wants to walk to creature B who is far away.** Zone 1's AI decides "go find creature B". Zone 1 pathfinds creature A toward the zone boundary. When creature A crosses into Zone 2 (handoff), Zone 2's AI continues the "go find creature B" behavior. The behavior tree state is serialized as part of the handoff.

### 6.5 Offline Creature AI

When a user disconnects:
1. Their creatures remain in the world. The server continues running their AI.
2. Offline creatures switch to a "low priority" AI mode: evaluated at 0.5Hz instead of 2Hz.
3. Offline creatures prefer safe behaviors: wander near home, avoid combat, maintain community duties at minimum.
4. If the server detects >1000 offline creatures, it "hibernates" the least recently active ones: stops simulating them, records their state to DB, and respawns them when their owner returns or when another creature enters their area.

**Hibernation system:**
```typescript
class CreatureManager {
  activeCreatures: Map<uint32, Creature>;     // in-memory, simulated
  hibernatedCreatures: Set<uint32>;           // in DB only

  hibernate(creature: Creature) {
    this.persistToDB(creature);
    this.activeCreatures.delete(creature.id);
    this.hibernatedCreatures.add(creature.id);
    this.spatialIndex.remove(creature.id);
  }

  wake(creatureId: uint32) {
    const state = this.loadFromDB(creatureId);
    const creature = new Creature(state);
    this.activeCreatures.set(creatureId, creature);
    this.hibernatedCreatures.delete(creatureId);
    this.spatialIndex.insert(creature);
  }

  // When any creature enters a cell containing hibernated creatures, wake them
  onCellEnter(cellKey: string) {
    for (const id of this.getHibernatedInCell(cellKey)) {
      this.wake(id);
    }
  }
}
```

---

## 7. Conversation System

### 7.1 Architecture

Conversations are the most latency-sensitive feature after movement, and the most computationally expensive because they involve LLM calls.

```
┌───────────┐    ┌──────────────┐    ┌────────────────┐    ┌─────────┐
│ Zone      │───>│ Conversation │───>│ LLM Gateway    │───>│ Claude  │
│ Server    │    │ Queue        │    │ (rate limited)  │    │ API     │
│           │<───│ (Redis)      │<───│                │<───│         │
└───────────┘    └──────────────┘    └────────────────┘    └─────────┘
```

### 7.2 Conversation Flow

```
1. Creature A's AI decides to talk to Creature B (proximity + sociability check)
2. Zone server creates CONVERSATION_SESSION:
   {
     id: uuid,
     participants: [creatureA.id, creatureB.id],
     context: {
       location: "near the river",
       a_personality: creatureA.personality,
       b_personality: creatureB.personality,
       a_mood: creatureA.mood,
       b_mood: creatureB.mood,
       relationship: "strangers" | "friends" | "rivals",
       recent_events: [...],
     },
     history: [],
     state: "pending"
   }

3. Server pushes to Redis conversation queue:
   LPUSH conv:queue {sessionId, priority}

4. Conversation Worker picks up the job:
   - Builds prompt from context + personality + history
   - Calls Claude API with streaming enabled
   - As tokens arrive, publishes partial text to Redis channel conv:{sessionId}

5. Zone server subscribes to conv:{sessionId}:
   - Receives partial text
   - Sends SPEECH_BUBBLE to both User A and User B's clients
   - Both clients show the speech bubble appearing word-by-word

6. After response complete:
   - Update conversation history
   - If conversation should continue (AI decides), queue next turn
   - Otherwise, end session
```

### 7.3 Handling LLM Latency (1-3 seconds)

**Streaming**: The LLM response is streamed token by token. The client shows speech appearing progressively, like a typewriter effect. This masks the latency -- the user sees the creature "thinking" for ~500ms, then words start appearing.

**Visual cues during thinking**:
- Creature shows a "..." thought bubble
- Creature's mood shifts to "curious" (slightly different idle animation)
- This 500ms-1s "thinking" period feels natural for a creature pondering a response

**Turn-taking**: Conversations are turn-based. While creature A is speaking, creature B plays a "listening" animation. No need for real-time overlap.

### 7.4 Conversation Queueing

If many creatures want to talk simultaneously:

**Priority queue** based on:
1. Both creatures' owners are online (highest priority -- real users watching)
2. One owner online
3. Both offline (lowest priority -- background simulation)

**Rate limiting**: Max 10 concurrent LLM calls per zone server. With Claude API at ~30 tokens/sec, a typical 50-token response takes ~1.7s. 10 concurrent = 6 conversations/second throughput.

**Deduplication**: If creature A is already in a conversation, it cannot start another. Queue the request and process when current conversation ends.

### 7.5 Conversation Persistence

```sql
-- conversations table
CREATE TABLE conversations (
  id            UUID PRIMARY KEY,
  started_at    TIMESTAMPTZ NOT NULL,
  ended_at      TIMESTAMPTZ,
  location_x    REAL,
  location_y    REAL,
  zone_id       INTEGER
);

-- conversation_participants
CREATE TABLE conversation_participants (
  conversation_id  UUID REFERENCES conversations(id),
  creature_id      INTEGER REFERENCES creatures(id),
  PRIMARY KEY (conversation_id, creature_id)
);

-- conversation_messages
CREATE TABLE conversation_messages (
  id              BIGSERIAL PRIMARY KEY,
  conversation_id UUID REFERENCES conversations(id),
  creature_id     INTEGER REFERENCES creatures(id),
  content         TEXT NOT NULL,
  emotion_tag     SMALLINT DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for retrieving creature's conversation history
CREATE INDEX idx_conv_msg_creature ON conversation_messages(creature_id, created_at DESC);
```

**Memory**: Each creature's AI has access to its last 20 conversations (summarized to ~100 tokens each via a separate summarization pass). This provides long-term memory without unbounded context growth.

---

## 8. Action System

### 8.1 Entity-Component-System (ECS) Inspired Action Architecture

Every action in the game is represented as an `Action` that flows through a pipeline. This is designed for extensibility -- adding "fighting" or "research" later requires only implementing a new `ActionHandler`, not modifying the core loop.

```typescript
// Core action types -- extensible via enum
enum ActionType {
  IDLE       = 0,
  WALK       = 1,
  TALK       = 2,
  FIGHT      = 3,
  RESEARCH   = 4,
  SLEEP      = 5,
  EAT        = 6,
  SOCIALIZE  = 7,
  BUILD      = 8,
  TRADE      = 9,
  GATHER     = 10,
  TEACH      = 11,
  LEARN      = 12,
  PATROL     = 13,
  FLEE       = 14,
  GROUP_MOVE = 15,
}

// An action is a discrete unit of work a creature performs
interface Action {
  type:       ActionType;
  creatureId: uint32;        // who is performing
  targetId?:  uint32;        // target creature/object (optional)
  targetPos?: [number, number]; // target position (optional)
  startTime:  number;        // server time when started
  duration?:  number;        // expected duration (some actions are indefinite)
  state:      ActionState;   // pending | active | completing | done | cancelled
  data:       any;           // action-specific payload
}

enum ActionState {
  PENDING    = 0,  // queued, waiting for prerequisites
  ACTIVE     = 1,  // currently executing
  COMPLETING = 2,  // wrapping up (animation, etc)
  DONE       = 3,  // finished successfully
  CANCELLED  = 4,  // interrupted
}
```

### 8.2 Action Pipeline

```
  ┌──────────────┐
  │ AI Decision  │ "I want to talk to creature B"
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │  VALIDATION  │ Can the creature do this? (energy, cooldowns, proximity)
  └──────┬───────┘
         │ (reject if invalid)
  ┌──────▼───────┐
  │  QUEUEING    │ Add to creature's action queue (max 3 queued)
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │  EXECUTION   │ ActionHandler.start() -> tick() -> complete()
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │  BROADCAST   │ Send ACTION_EVENT to relevant clients
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │  EFFECTS     │ Apply results: damage, XP, relationship changes, items
  └──────────────┘
```

### 8.3 Action Handlers (Plugin Architecture)

```typescript
interface ActionHandler {
  type: ActionType;

  // Validate whether the action can start
  validate(creature: Creature, action: Action, world: WorldState): ValidationResult;

  // Called once when action starts
  start(creature: Creature, action: Action, world: WorldState): void;

  // Called every simulation tick while action is active
  tick(creature: Creature, action: Action, world: WorldState, dt: number): ActionTickResult;

  // Called when action completes or is cancelled
  complete(creature: Creature, action: Action, world: WorldState, reason: 'done' | 'cancelled'): ActionEffects;
}

// Example: FightActionHandler (future)
class FightActionHandler implements ActionHandler {
  type = ActionType.FIGHT;

  validate(creature, action, world) {
    const target = world.getCreature(action.targetId);
    if (!target) return { ok: false, reason: 'target_not_found' };
    if (distance(creature, target) > 80) return { ok: false, reason: 'too_far' };
    if (creature.energy < 20) return { ok: false, reason: 'too_tired' };
    if (creature.currentAction === ActionType.SLEEP) return { ok: false, reason: 'sleeping' };
    return { ok: true };
  }

  start(creature, action, world) {
    creature.currentAction = ActionType.FIGHT;
    creature.actionTarget = action.targetId;
    creature.mood = MoodType.ANGRY;
    action.data = { turnNumber: 0, lastAttackTime: 0 };
  }

  tick(creature, action, world, dt) {
    const target = world.getCreature(action.targetId);
    if (!target || target.health <= 0) return ActionTickResult.COMPLETE;

    // Turn-based: attack every 2 seconds
    action.data.lastAttackTime += dt;
    if (action.data.lastAttackTime >= 2.0) {
      action.data.lastAttackTime = 0;
      action.data.turnNumber++;

      // Calculate damage based on stats + personality
      const damage = calculateDamage(creature, target);
      target.health -= damage;

      // Broadcast hit event
      world.broadcastEvent({
        type: 'FIGHT_HIT',
        actorId: creature.id,
        targetId: target.id,
        damage,
        targetHealthRemaining: target.health
      });

      if (target.health <= 0) return ActionTickResult.COMPLETE;
    }
    return ActionTickResult.CONTINUE;
  }

  complete(creature, action, world, reason) {
    creature.currentAction = ActionType.IDLE;
    creature.actionTarget = 0;
    creature.energy -= 30;
    // Relationship effects, XP, etc.
    return { relationshipChange: -10, xpGained: action.data.turnNumber * 5 };
  }
}
```

### 8.4 Group Activities

Group activities use a **GroupAction** coordinator:

```typescript
interface GroupAction {
  id:          string;
  type:        ActionType; // GROUP_MOVE, RESEARCH, BUILD, etc.
  leaderId:    uint32;
  memberIds:   uint32[];
  state:       ActionState;
  sharedData:  any;        // shared progress, shared resources, etc.
}
```

When a creature initiates a group activity:
1. It sends invitations to nearby creatures (AI evaluates based on personality)
2. Accepting creatures join the group
3. The group has a shared action state that all members contribute to
4. Progress is pooled (e.g., research progress = sum of all members' intelligence * time)

### 8.5 Community Formation

Communities emerge organically from sustained social interactions:

```typescript
interface Community {
  id:           uint32;
  name:         string;
  foundedAt:    number;
  leaderId:     uint32;      // elected or founder
  memberIds:    Set<uint32>;
  territory:    { x: number, y: number, radius: number }; // home area
  values:       PersonalityVector; // averaged from members
  resources:    Map<string, number>;
  relationships: Map<uint32, number>; // community -> opinion (-100 to 100)
}
```

Community formation trigger: When 3+ creatures have mutual "friend" relationships (relationship score > 50) and are frequently in the same area (>50% of time within 500 units of each other over 10 real-time minutes), the AI proposes community formation. The creature with highest sociability becomes the leader.

---

## 9. Persistence & Database Design

### 9.1 Complete Database Schema

```sql
-- ═══════════════════════════════════════════
-- USERS
-- ═══════════════════════════════════════════
CREATE TABLE users (
  id              SERIAL PRIMARY KEY,
  external_id     VARCHAR(255) UNIQUE NOT NULL,  -- OAuth provider ID
  username        VARCHAR(50) UNIQUE NOT NULL,
  email           VARCHAR(255),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_login_at   TIMESTAMPTZ,
  creature_slots  INTEGER DEFAULT 50,            -- max creatures this user can have
  is_banned       BOOLEAN DEFAULT FALSE
);

-- ═══════════════════════════════════════════
-- CREATURES
-- ═══════════════════════════════════════════
CREATE TABLE creatures (
  id              SERIAL PRIMARY KEY,
  owner_id        INTEGER NOT NULL REFERENCES users(id),
  name            VARCHAR(50) NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- Position
  world_x         REAL NOT NULL DEFAULT 0,
  world_y         REAL NOT NULL DEFAULT 0,
  facing_angle    REAL NOT NULL DEFAULT 0,
  zone_id         INTEGER NOT NULL DEFAULT 0,

  -- Appearance
  morph_base      SMALLINT NOT NULL DEFAULT 0,
  morph_compact   SMALLINT NOT NULL DEFAULT 128,
  body_scale      REAL NOT NULL DEFAULT 1.0,
  color_hue       SMALLINT NOT NULL DEFAULT 0,
  eye_variant     SMALLINT NOT NULL DEFAULT 0,
  leg_count       SMALLINT NOT NULL DEFAULT 4,

  -- Stats
  health          INTEGER NOT NULL DEFAULT 100,
  max_health      INTEGER NOT NULL DEFAULT 100,
  energy          INTEGER NOT NULL DEFAULT 255,
  mood            SMALLINT NOT NULL DEFAULT 0,

  -- Personality (immutable after creation)
  p_aggression    SMALLINT NOT NULL,
  p_curiosity     SMALLINT NOT NULL,
  p_sociability   SMALLINT NOT NULL,
  p_loyalty       SMALLINT NOT NULL,
  p_intelligence  SMALLINT NOT NULL,

  -- Social
  community_id    INTEGER REFERENCES communities(id),
  reputation      INTEGER NOT NULL DEFAULT 0,

  -- AI State (serialized)
  behavior_state  JSONB,
  memory_summary  TEXT,     -- LLM-summarized recent memories

  -- Lifecycle
  is_alive        BOOLEAN NOT NULL DEFAULT TRUE,
  generation      INTEGER NOT NULL DEFAULT 1,
  parent_a_id     INTEGER REFERENCES creatures(id),
  parent_b_id     INTEGER REFERENCES creatures(id),

  -- Metadata
  last_active_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  is_hibernated   BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_creatures_owner ON creatures(owner_id);
CREATE INDEX idx_creatures_zone ON creatures(zone_id);
CREATE INDEX idx_creatures_position ON creatures USING gist (
  point(world_x, world_y)
);  -- PostGIS-style spatial index for DB-level queries
CREATE INDEX idx_creatures_community ON creatures(community_id);
CREATE INDEX idx_creatures_hibernated ON creatures(is_hibernated) WHERE is_hibernated = true;

-- ═══════════════════════════════════════════
-- COMMUNITIES
-- ═══════════════════════════════════════════
CREATE TABLE communities (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(100) NOT NULL,
  founded_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  leader_id       INTEGER REFERENCES creatures(id),
  territory_x     REAL,
  territory_y     REAL,
  territory_radius REAL DEFAULT 500,
  avg_aggression  SMALLINT,
  avg_curiosity   SMALLINT,
  avg_sociability SMALLINT,
  avg_loyalty     SMALLINT,
  avg_intelligence SMALLINT,
  member_count    INTEGER NOT NULL DEFAULT 0,
  is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- ═══════════════════════════════════════════
-- RELATIONSHIPS (between creatures)
-- ═══════════════════════════════════════════
CREATE TABLE creature_relationships (
  creature_a_id   INTEGER NOT NULL REFERENCES creatures(id),
  creature_b_id   INTEGER NOT NULL REFERENCES creatures(id),
  score           INTEGER NOT NULL DEFAULT 0,  -- -100 (enemy) to +100 (best friend)
  interaction_count INTEGER NOT NULL DEFAULT 0,
  last_interaction TIMESTAMPTZ,
  relationship_type SMALLINT DEFAULT 0, -- 0=neutral, 1=friend, 2=rival, 3=mate, 4=family
  PRIMARY KEY (creature_a_id, creature_b_id)
);

CREATE INDEX idx_relationships_b ON creature_relationships(creature_b_id);

-- ═══════════════════════════════════════════
-- CONVERSATIONS (see Section 7.5 above)
-- ═══════════════════════════════════════════
-- conversations, conversation_participants, conversation_messages
-- (already defined in Section 7.5)

-- ═══════════════════════════════════════════
-- COMMUNITY RELATIONSHIPS
-- ═══════════════════════════════════════════
CREATE TABLE community_relationships (
  community_a_id  INTEGER NOT NULL REFERENCES communities(id),
  community_b_id  INTEGER NOT NULL REFERENCES communities(id),
  opinion         INTEGER NOT NULL DEFAULT 0,  -- -100 to +100
  last_interaction TIMESTAMPTZ,
  PRIMARY KEY (community_a_id, community_b_id)
);

-- ═══════════════════════════════════════════
-- WORLD EVENTS (audit log of significant events)
-- ═══════════════════════════════════════════
CREATE TABLE world_events (
  id              BIGSERIAL PRIMARY KEY,
  event_type      VARCHAR(50) NOT NULL,  -- 'birth', 'death', 'fight', 'community_formed', etc.
  timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  zone_id         INTEGER,
  actor_id        INTEGER REFERENCES creatures(id),
  target_id       INTEGER REFERENCES creatures(id),
  data            JSONB,
  world_x         REAL,
  world_y         REAL
);

CREATE INDEX idx_events_type_time ON world_events(event_type, timestamp DESC);
CREATE INDEX idx_events_actor ON world_events(actor_id, timestamp DESC);

-- ═══════════════════════════════════════════
-- RESEARCH / KNOWLEDGE (future feature)
-- ═══════════════════════════════════════════
CREATE TABLE knowledge_topics (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(100) NOT NULL,
  description     TEXT,
  prerequisites   INTEGER[],  -- topic IDs required first
  difficulty      INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE creature_knowledge (
  creature_id     INTEGER NOT NULL REFERENCES creatures(id),
  topic_id        INTEGER NOT NULL REFERENCES knowledge_topics(id),
  progress        REAL NOT NULL DEFAULT 0,  -- 0.0 to 1.0
  completed_at    TIMESTAMPTZ,
  PRIMARY KEY (creature_id, topic_id)
);
```

### 9.2 Persistence Strategy

**In-Memory State** (authoritative, held by zone servers):
- All active creature positions, stats, actions
- Spatial indexes
- Active conversations
- Behavior tree states

**Persistence Schedule:**

| Data | Frequency | Method |
|------|-----------|--------|
| Creature position | Every 30 seconds | Batch UPDATE |
| Creature stats (health, energy, mood) | Every 30 seconds | Batch UPDATE |
| Creature behavior state | Every 60 seconds | JSONB UPDATE |
| Relationships | On change, debounced 10s | Upsert |
| Conversations | On message completion | INSERT |
| World events | On occurrence | INSERT (async) |
| Community state | Every 60 seconds | UPDATE |

**Write batching**: Dirty creatures are accumulated in a Set. Every 30 seconds, a background task builds a bulk UPDATE statement:

```sql
-- Batch update 500 creatures in one query using unnest
UPDATE creatures SET
  world_x = data.x,
  world_y = data.y,
  facing_angle = data.angle,
  health = data.health,
  energy = data.energy,
  mood = data.mood,
  last_active_at = NOW()
FROM (
  SELECT unnest($1::int[]) AS id,
         unnest($2::real[]) AS x,
         unnest($3::real[]) AS y,
         unnest($4::real[]) AS angle,
         unnest($5::int[]) AS health,
         unnest($6::int[]) AS energy,
         unnest($7::smallint[]) AS mood
) AS data
WHERE creatures.id = data.id;
```

This is one DB round-trip for 500 creatures. At 5000 creatures, that is 10 queries every 30 seconds -- negligible load.

### 9.3 Server Crash Recovery

**Maximum data loss**: 30 seconds of creature position/state. The last DB persist is the recovery point.

**Recovery procedure**:
1. Zone server crashes.
2. Health check (every 5s) detects failure.
3. Orchestrator (Kubernetes / supervisor) launches replacement zone server.
4. New server loads all creatures for its zone from PostgreSQL.
5. New server rebuilds spatial index.
6. Clients reconnect (they retry automatically with exponential backoff).
7. Clients receive full snapshot of the (30-seconds-stale) state.
8. Within 1 tick, everything is back to normal. Users notice a brief freeze (5-15 seconds total).

**Redis as WAL acceleration**: For more critical data, write creature states to Redis every 5 seconds as well. On crash, recover from Redis (5s stale) instead of PostgreSQL (30s stale). Redis BGSAVE ensures Redis itself persists across restarts.

---

## 10. Scaling Architecture

### 10.1 Phase 1: Single Server (0-100 users, 0-5000 creatures)

```
┌──────────────────────────────────┐
│        Single VPS / VM           │
│                                  │
│  ┌───────────────────────────┐   │
│  │ Node.js (Colyseus)       │   │
│  │ - Game loop (20Hz)        │   │
│  │ - WebSocket server        │   │
│  │ - AI simulation           │   │
│  │ - All zones in-process    │   │
│  └────────────┬──────────────┘   │
│               │                  │
│  ┌────────────▼──────────────┐   │
│  │ PostgreSQL (same machine) │   │
│  └────────────┬──────────────┘   │
│               │                  │
│  ┌────────────▼──────────────┐   │
│  │ Redis (same machine)     │   │
│  └───────────────────────────┘   │
└──────────────────────────────────┘
```

**Estimated resources**: 4 vCPU, 8GB RAM. Colyseus can handle ~500 WebSocket connections on a single process. With 5000 creature entities, memory usage ~50MB for all entity state. CPU bottleneck is AI evaluation -- staggered at 2Hz, this is ~30% of one core.

### 10.2 Phase 2: Multi-Server (100-1000 users, 5000-50000 creatures)

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Zone Svr 1  │  │ Zone Svr 2  │  │ Zone Svr 3  │  │ Zone Svr 4  │
│ (NW quadrant)│  │ (NE quadrant)│  │ (SW quadrant)│  │ (SE quadrant)│
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────────────┼────────────────┼────────────────┘
                        │
                ┌───────▼───────┐
                │ Redis Cluster │ (pub/sub + cache)
                └───────┬───────┘
                        │
                ┌───────▼───────┐
                │  PostgreSQL   │ (primary + 1 replica)
                └───────┬───────┘
                        │
                ┌───────▼───────┐
                │ AI Workers x2 │ (conversation LLM calls)
                └───────────────┘

                ┌───────────────┐
                │ Gateway / LB  │ (sticky sessions to zone server)
                └───────────────┘
```

**Key change**: Zone servers are separate processes (possibly separate machines). Redis is the communication backbone. Each zone server is a Node.js process handling one spatial zone.

**Sticky sessions**: The load balancer routes each client to the zone server that owns their current viewport center. If the user pans to a different zone, the client reconnects.

### 10.3 Phase 3: Full Scale (1000-10000+ users, 50000+ creatures)

```
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                    │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Zone Server Pods (auto-scaling)                  │  │
│  │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ... (N)   │  │
│  │ │Zone 1│ │Zone 2│ │Zone 3│ │Zone 4│            │  │
│  │ └──────┘ └──────┘ └──────┘ └──────┘            │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ AI Worker Pods (auto-scaling by queue depth)     │  │
│  │ ┌──────┐ ┌──────┐ ┌──────┐ ... (M)             │  │
│  │ │AI Wk1│ │AI Wk2│ │AI Wk3│                     │  │
│  │ └──────┘ └──────┘ └──────┘                     │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─────────────────┐  ┌────────────────────────────┐  │
│  │ Redis Cluster   │  │ PostgreSQL (Citus or Aurora)│  │
│  │ 3 nodes         │  │ sharded by zone_id          │  │
│  └─────────────────┘  └────────────────────────────┘  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Agones GameServer Manager                        │  │
│  │ (allocates zone servers, manages health checks)  │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Ingress (WebSocket-aware: Nginx / Envoy)        │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

**Agones** (Google's open-source game server orchestration for Kubernetes) manages zone server lifecycle: scaling, health checking, graceful shutdown with state migration.

**Database sharding**: At 50,000+ creatures, single PostgreSQL is still fine (it can handle millions of rows). The bottleneck is write throughput. Using connection pooling (PgBouncer) and write batching, a single primary can handle ~10,000 writes/second, which is far above the ~170 writes/second from batch persistence (5000 creatures / 30 seconds).

### 10.4 Cross-Server Creature Interaction at Scale

At Phase 3, creatures on different servers interact via Redis pub/sub with structured message channels:

```
Redis channels:
  zone:{zoneId}:border:{direction}     -- border entity mirrors
  zone:{zoneId}:handoff                -- creature ownership transfer
  zone:{zoneId}:interaction            -- cross-zone interactions
  conv:queue                           -- conversation job queue
  conv:{sessionId}                     -- conversation streaming output
  global:events                        -- world-wide event broadcast
```

**Latency**: Redis pub/sub adds ~1ms per message. Cross-zone interactions have ~2ms extra latency (publish + subscribe). This is imperceptible.

---

## 11. Security

### 11.1 Authoritative Server Model

The #1 security principle: **the client is never trusted.** The client sends INPUTS (intentions), not STATE (results).

**What the client CAN send:**
- "Move my creature toward (X, Y)" -- server validates speed, pathfinding
- "My creature wants to talk to creature #123" -- server validates proximity, cooldowns
- "I want to spawn a new creature" -- server validates slot count, cooldowns

**What the client CANNOT send:**
- "My creature is now at position (X, Y)" -- REJECTED. Server calculates position.
- "My creature's health is now 9999" -- REJECTED. Server owns health.
- "Creature #456 (not mine) should move here" -- REJECTED. Can only control own creatures.

### 11.2 Input Validation

```typescript
function validateMoveCommand(client: Client, cmd: MoveToCommand): boolean {
  // 1. Does this client own any creatures?
  const creatures = getCreaturesByOwner(client.userId);
  if (creatures.length === 0) return false;

  // 2. Is the target position within the world bounds?
  if (cmd.targetX < 0 || cmd.targetX > WORLD_WIDTH ||
      cmd.targetY < 0 || cmd.targetY > WORLD_HEIGHT) return false;

  // 3. Rate limit: max 10 move commands per second
  if (client.moveCommandCount > 10) return false;
  client.moveCommandCount++;

  // 4. Target must be reachable (not inside a wall/obstacle)
  if (isBlocked(cmd.targetX, cmd.targetY)) return false;

  return true;
}
```

### 11.3 Rate Limiting

| Action | Rate Limit | Penalty for Exceed |
|--------|-----------|-------------------|
| MOVE_TO | 10/sec | Ignore excess |
| INTERACT | 2/sec | Ignore excess |
| SPEAK | 1/5sec | Ignore excess, warn client |
| SPAWN_CREATURE | 1/min | Reject with error |
| VIEWPORT updates | 5/sec | Throttle to last |
| Any message | 100/sec total | Disconnect client |

### 11.4 Anti-Cheat Measures

1. **Speed hack detection**: Server tracks creature positions. If a creature moves faster than MAX_SPEED * 1.1 between ticks, it is snapped back to the legal position. (The 1.1 multiplier accounts for floating-point drift.)

2. **Spawn spam**: Users have a creature slot limit (default 50). Creating a creature costs "energy" that regenerates over real time (1 slot per 10 minutes).

3. **Conversation spam**: Each creature can only be in one conversation at a time. Conversations have a minimum 10-second cooldown between sessions.

4. **Memory/CPU DoS prevention**: Client messages have a maximum size (4KB). Messages exceeding this are dropped. The server processes at most 100 messages per client per second.

5. **WebSocket authentication**: JWT token is validated on WebSocket upgrade. Token contains userId, issued-at, expiry. Token is refreshed every 15 minutes via HTTP. If token expires during a WebSocket session, the server sends a REAUTH_REQUIRED message and gives the client 30 seconds to provide a new token.

### 11.5 Server-Side Validation for Every Action

```typescript
// EVERY incoming message goes through this pipeline
function processClientMessage(client: Client, rawBytes: ArrayBuffer) {
  // 1. Size check
  if (rawBytes.byteLength > MAX_MESSAGE_SIZE) {
    client.warn('message_too_large');
    return;
  }

  // 2. Rate limit
  if (!client.rateLimiter.allow()) {
    client.warn('rate_limited');
    return;
  }

  // 3. Deserialize
  const msg = deserialize(rawBytes);
  if (!msg) {
    client.warn('malformed_message');
    return;
  }

  // 4. Auth check
  if (!client.isAuthenticated) {
    client.disconnect('not_authenticated');
    return;
  }

  // 5. Route to handler with validation
  const handler = messageHandlers[msg.type];
  if (!handler) {
    client.warn('unknown_message_type');
    return;
  }

  handler.validate(client, msg);
  handler.execute(client, msg);
}
```

---

## 12. Infrastructure & Deployment

### 12.1 Hosting Recommendation by Phase

**Phase 1 (MVP, <100 users): Fly.io or Railway**
- Single machine deployment
- Fly.io: Excellent WebSocket support, global edge network, $0.0000573/s per shared CPU ($15/mo for always-on)
- Railway: Simpler DX, slightly more expensive, good for prototyping
- **Recommendation: Fly.io**

**Phase 2 (Growth, 100-1000 users): Fly.io Multi-Region or AWS**
- Fly.io: Scale to multiple machines easily, built-in private networking
- AWS: ECS/Fargate for zone servers, ElastiCache for Redis, RDS for PostgreSQL
- **Recommendation: Stay on Fly.io until WebSocket scaling becomes painful, then migrate to AWS**

**Phase 3 (Scale, 1000+ users): AWS EKS or GCP GKE**
- Kubernetes with Agones for game server management
- AWS ALB with WebSocket support (sticky sessions via cookies)
- **Recommendation: AWS EKS + Agones**

### 12.2 WebSocket Hosting Considerations

**Critical**: Most serverless platforms (Cloudflare Workers, Vercel, Lambda) do NOT support long-lived WebSocket connections natively. Cloudflare Durable Objects is the exception, but it has per-object memory limits (128MB) and is expensive at scale for this use case.

**Requirements for WebSocket hosting:**
- Long-lived connections (sessions last minutes to hours)
- Sticky sessions (client must reconnect to the same zone server)
- No cold starts (game server must be always-hot)
- Direct TCP/WebSocket routing (no HTTP-only proxies)

**Fly.io** handles all of these natively with `fly-replay` header for regional routing.

### 12.3 Infrastructure Diagram

```
                    ┌──────────────────┐
                    │   Cloudflare     │
                    │   CDN / DNS      │
                    │   (static assets,│
                    │   DDoS protect)  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   Fly.io App     │
                    │   (or AWS ALB)   │
                    │                  │
                    │  ┌────────────┐  │
                    │  │ HTTP API   │  │  ← Auth, user management, REST endpoints
                    │  │ (Express)  │  │
                    │  └────────────┘  │
                    │                  │
                    │  ┌────────────┐  │
                    │  │ WS Gateway │  │  ← Routes WebSocket to correct zone server
                    │  │            │  │
                    │  └─────┬──────┘  │
                    │        │         │
                    │  ┌─────▼──────┐  │
                    │  │Zone Server │  │  ← Game simulation
                    │  │(1 or more) │  │
                    │  └─────┬──────┘  │
                    │        │         │
                    │  ┌─────▼──────┐  │
                    │  │Redis       │  │  ← Fly.io Upstash Redis or self-hosted
                    │  └─────┬──────┘  │
                    │        │         │
                    │  ┌─────▼──────┐  │
                    │  │PostgreSQL  │  │  ← Fly.io Postgres or Neon or Supabase
                    │  └────────────┘  │
                    └──────────────────┘
```

### 12.4 Monitoring & Observability

**Metrics to track:**

| Metric | Tool | Alert Threshold |
|--------|------|-----------------|
| Tick duration (ms) | Prometheus + Grafana | >40ms (80% of 50ms budget) |
| Active creatures per zone | Prometheus | >2000 (consider splitting) |
| WebSocket connections | Prometheus | >800 per process |
| Message throughput (msg/s) | Prometheus | >50,000/s per zone |
| Client RTT (ms) | Custom (from PING/PONG) | p95 > 200ms |
| DB write latency | pg_stat_statements | p95 > 100ms |
| Redis pub/sub latency | Redis MONITOR | >5ms |
| LLM API latency | Custom | p95 > 5s |
| Memory usage per zone | OS metrics | >80% of allocated |
| Error rate | Sentry | >1% of messages |

**Logging**: Structured JSON logs. Log every zone server tick that exceeds 40ms with a breakdown of where time was spent (AI, physics, snapshot building, I/O).

**Distributed tracing**: OpenTelemetry for cross-zone interactions. Trace a conversation from "AI decides to talk" through "LLM call" to "speech bubble displayed on client."

---

## 13. Frontend Architecture

### 13.1 Rendering 100+ Creature Sprites

The current prototype uses Canvas2D with 200-point Bezier curves per creature. This is expensive. At 100 creatures on screen:

**Problem**: 200 quadraticCurveTo calls x 100 creatures x 60fps = 1,200,000 curve draws per second. Canvas2D will choke.

**Solution: Multi-tier rendering**

```
Tier 1: FULL RENDER (0-20 creatures closest to camera center)
  - Full 200-point body outline with walk deformation
  - Eye animations with blink
  - Breath wave
  - Speech bubbles
  - Shadow / glow effects
  → This is the current rendering code, unchanged

Tier 2: SIMPLIFIED RENDER (20-50 creatures at medium distance)
  - Reduced 50-point body outline (every 4th point from BASE)
  - Eyes rendered as simple ovals (no deformation)
  - No breath wave
  - Action indicator icon only (no speech text)
  → ~4x cheaper to render

Tier 3: ICON RENDER (50-200 creatures at far distance)
  - Pre-rendered sprite image (cached to offscreen canvas)
  - Single drawImage() call
  - Color tint applied via globalCompositeOperation
  - Facing direction shown by sprite rotation
  → ~50x cheaper to render

Tier 4: DOT RENDER (200+ creatures at extreme distance)
  - Single fillRect() call (2x2 pixel dot)
  - Color only (owner's color or community color)
  → ~200x cheaper
```

### 13.2 Viewport Culling

```typescript
class ViewportCuller {
  viewX: number;  // camera center in world coords
  viewY: number;
  viewW: number;  // visible world width
  viewH: number;  // visible world height

  isVisible(creatureX: number, creatureY: number, margin: number = 100): boolean {
    return creatureX >= this.viewX - this.viewW/2 - margin &&
           creatureX <= this.viewX + this.viewW/2 + margin &&
           creatureY >= this.viewY - this.viewH/2 - margin &&
           creatureY <= this.viewY + this.viewH/2 + margin;
  }

  getRenderTier(creatureX: number, creatureY: number): 1 | 2 | 3 | 4 {
    const dx = creatureX - this.viewX;
    const dy = creatureY - this.viewY;
    const dist = Math.sqrt(dx*dx + dy*dy);
    if (dist < 200) return 1;
    if (dist < 500) return 2;
    if (dist < 1000) return 3;
    return 4;
  }
}
```

### 13.3 Object Pooling

Creature renderers are pooled. When a creature enters the viewport, a renderer is allocated from the pool. When it exits, the renderer is returned.

```typescript
class CreatureRendererPool {
  available: CreatureRenderer[] = [];
  active: Map<number, CreatureRenderer> = new Map(); // creatureId -> renderer

  acquire(creatureId: number): CreatureRenderer {
    let renderer = this.available.pop();
    if (!renderer) {
      renderer = new CreatureRenderer();
    }
    renderer.reset();
    this.active.set(creatureId, renderer);
    return renderer;
  }

  release(creatureId: number) {
    const renderer = this.active.get(creatureId);
    if (renderer) {
      this.active.delete(creatureId);
      renderer.reset();
      this.available.push(renderer);
    }
  }
}
```

### 13.4 Memory Management

**Pre-computed shape caches**: The 200-point BASE shape and its reduced versions (50-point, 20-point) are computed once on startup and stored as Float32Arrays for fast iteration.

**Offscreen canvas for sprites**: Tier 3 renders use pre-rendered sprites. Each unique creature appearance (morph + color) is rendered once to an offscreen canvas (128x128 pixels), then reused via drawImage(). Cache up to 200 sprites. LRU eviction when cache is full.

**Texture atlas (future)**: If moving to WebGL (PixiJS), pack all creature sprites into a single texture atlas for batched rendering. This is the standard approach for rendering thousands of sprites.

### 13.5 UI Layer Architecture

```
┌─────────────────────────────────────────┐
│            HTML/CSS Overlay             │  ← Menus, HUD, community panels
│  (position: absolute, pointer-events)   │
├─────────────────────────────────────────┤
│          Speech Bubble Layer            │  ← DOM elements positioned over canvas
│  (CSS transforms for world-to-screen)   │
├─────────────────────────────────────────┤
│          Canvas (Game World)            │  ← Creatures, terrain, effects
│  (Camera transform applied)             │
└─────────────────────────────────────────┘
```

**Speech bubbles**: Rendered as HTML `<div>` elements positioned absolutely above the canvas. This gives proper text rendering, word wrapping, and CSS animations. Position is calculated each frame:

```typescript
function updateSpeechBubblePosition(bubble: HTMLElement, creature: CreatureState, camera: Camera) {
  const screenX = (creature.x - camera.x) * camera.zoom + canvas.width / 2;
  const screenY = (creature.y - camera.y) * camera.zoom + canvas.height / 2 - 80; // above head
  bubble.style.transform = `translate(${screenX}px, ${screenY}px) translate(-50%, -100%)`;
}
```

**Health bars / name tags**: Also DOM elements for crisp text. Batch position updates by only recalculating for visible creatures.

**Minimap**: A small `<canvas>` element (200x200 pixels) in the corner showing all creatures as colored dots. Updated at 2Hz (not 60fps). Uses the server's full creature list (all 5000), not just the viewport.

---

## 14. Failure Modes & Mitigations

### 14.1 Complete Failure Catalog

| # | Failure | Impact | Detection | Mitigation |
|---|---------|--------|-----------|------------|
| 1 | Zone server crash | All creatures in zone freeze for 5-15s | Health check (5s interval) | Auto-restart, load state from DB/Redis. Clients auto-reconnect. |
| 2 | Redis crash | Cross-zone interactions stop. Conversations queue fails. | Connection error in zone servers | Redis Sentinel auto-failover (< 5s). Zone servers buffer messages in memory during outage. |
| 3 | PostgreSQL crash | No persistence. Active game continues from memory. | Connection pool errors | PG streaming replication + auto-failover. Active game is unaffected (memory is authoritative). |
| 4 | LLM API outage | Conversations stop | HTTP 5xx / timeout | Fallback to canned response templates based on personality. "Creature seems lost in thought..." |
| 5 | Client disconnects | User's creatures continue via AI | WebSocket close event | Creatures switch to offline AI mode. Reconnect restores control. |
| 6 | Zone overload (too many creatures) | Tick time exceeds 50ms budget | Tick duration monitoring | Dynamic zone splitting. Temporarily reduce AI frequency for non-essential creatures. |
| 7 | Memory leak in zone server | OOM crash | RSS monitoring, heap snapshots | Auto-restart + state recovery. Memory profiling in staging. |
| 8 | Network partition between zones | Cross-zone creatures desync | Heartbeat failure on Redis channels | Zones operate independently. Border creatures freeze. Reconnect triggers full resync. |
| 9 | Slow client (old device) | Client falls behind server | Client-side frame time monitoring | Automatically reduce render tier thresholds. Send fewer entities. Degrade gracefully. |
| 10 | Clock drift | Interpolation jitter | NTP sync divergence > 50ms | More aggressive clock sync (every 1s instead of 2s). Clamp interpolation to prevent time travel. |
| 11 | WebSocket flood (DDoS) | Server overwhelmed | Connection rate monitoring | Cloudflare DDoS protection at edge. Per-IP connection limits (max 3). |
| 12 | Corrupted database row | Creature loads with invalid state | Validation on DB load | Schema constraints. Application-level validation. Log + quarantine invalid rows. |
| 13 | Runaway AI loop | Creature behavior tree infinite loop burns CPU | Per-creature tick time limit (1ms) | Behavior tree evaluation has a max-node-visits counter (1000). Exceeding = force IDLE. |
| 14 | Stale client version | Protocol mismatch | Version handshake on connect | Server sends protocol version on connect. Client checks and force-reloads if mismatched. |
| 15 | Cross-zone handoff failure | Creature duplicated or lost | Handoff ACK timeout | Two-phase commit with timeout. On failure, source zone retains creature. Dedup by creature ID. |
| 16 | Conversation deadlock | Two creatures waiting for each other | Conversation state timeout (30s) | Timeout + forced conversation end. One-sided conversation permitted. |
| 17 | Breed spam | User creates creatures to overload server | Creature count per user monitoring | Hard cap: 50 creatures per user. Breeding cooldown: 5 minutes per creature. |
| 18 | Data center failure | Everything down | External health check (UptimeRobot) | Multi-region deployment (Phase 3). Or accept downtime for Phase 1-2 and communicate via status page. |

### 14.2 Graceful Degradation Tiers

```
NORMAL:     Everything works. 20Hz ticks. Full AI. Full conversations.
DEGRADED-1: Tick budget tight. Reduce AI to 1Hz. Reduce conversation concurrency.
DEGRADED-2: Zone overloaded. Hibernate idle creatures. Reduce send rate to 10Hz.
DEGRADED-3: Critical. Stop new connections. Reduce to position-only updates. No new conversations.
EMERGENCY:  Server near OOM. Persist all state to DB. Restart.
```

---

## 15. Cost Analysis

### 15.1 At 100 Concurrent Users (~5000 creatures)

| Resource | Spec | Monthly Cost |
|----------|------|-------------|
| Zone Server (Fly.io) | 1x shared-cpu-4x, 4GB RAM | $62/mo |
| PostgreSQL (Fly.io Postgres) | 1x shared-cpu-1x, 1GB RAM, 10GB disk | $16/mo |
| Redis (Upstash) | Pay-per-request, ~10K commands/day | $10/mo |
| LLM API (Claude) | ~50K conversations/mo, avg 200 tokens each = 10M tokens | ~$30/mo (Haiku) or ~$150/mo (Sonnet) |
| Cloudflare (CDN/DNS) | Free tier | $0 |
| Domain | .com | $12/yr = $1/mo |
| **TOTAL** | | **$119-$239/mo** |

### 15.2 At 1000 Concurrent Users (~50,000 creatures)

| Resource | Spec | Monthly Cost |
|----------|------|-------------|
| Zone Servers (4x) | 4x dedicated-cpu-2x, 4GB RAM each (Fly.io) | $248/mo |
| AI Workers (2x) | 2x shared-cpu-2x, 2GB RAM | $62/mo |
| PostgreSQL | dedicated-cpu-2x, 8GB RAM, 100GB disk | $124/mo |
| Redis (Upstash Pro) | 10GB, 10K commands/sec | $100/mo |
| LLM API (Claude) | ~500K conversations/mo = 100M tokens | ~$300/mo (Haiku) |
| Load Balancer | Fly.io built-in | $0 |
| Monitoring (Grafana Cloud) | Free tier or $29/mo | $29/mo |
| **TOTAL** | | **$863/mo** |

### 15.3 At 10,000 Concurrent Users (~500,000 creatures)

| Resource | Spec | Monthly Cost |
|----------|------|-------------|
| Zone Servers (16-32x) | AWS ECS Fargate, 2vCPU 4GB each | $1,500-$3,000/mo |
| AI Workers (8x) | ECS Fargate, 2vCPU 4GB | $600/mo |
| Conversation Workers (4x) | ECS Fargate, 1vCPU 2GB | $200/mo |
| PostgreSQL (RDS) | db.r6g.xlarge, Multi-AZ | $800/mo |
| Redis (ElastiCache) | cache.r6g.large, 2 nodes | $400/mo |
| LLM API (Claude) | 5M conversations/mo = 1B tokens | ~$2,500/mo (Haiku) |
| ALB + NAT Gateway | AWS networking | $200/mo |
| Kubernetes (EKS) | Cluster fee + nodes | $500/mo |
| Monitoring (Datadog) | APM + Logs | $300/mo |
| **TOTAL** | | **$7,000-$8,500/mo** |

### 15.4 Cost Optimization Strategies

1. **LLM costs dominate at scale.** Use Haiku for most conversations. Escalate to Sonnet only for "important" conversations (first meeting, community decisions, conflicts). This cuts LLM costs by 5-10x.

2. **Hibernate aggressively.** At 10K users, most users are offline at any given time. If average online rate is 10%, then 9000 users are offline. Their 450,000 creatures can be hibernated, reducing active simulation to 50,000 creatures.

3. **Conversation caching.** If two creatures with similar personalities discuss a common topic, cache the response template and inject variation. Reduces LLM calls by ~30%.

4. **Spot instances** for AI workers (AWS). 60-80% cost savings. Workers are stateless and can tolerate interruption.

5. **Time-based scaling.** If user base is US-centric, scale down to 25% capacity during 2-6 AM Pacific. Creatures hibernate during off-peak.

---

## 16. Implementation Roadmap

### Phase 1: MVP (Weeks 1-8)

**Goal**: 2 users can see each other's creatures in a shared world.

1. Set up Colyseus server with a single room
2. Define creature schema in Colyseus
3. Port the Canvas2D renderer to handle multiple creatures
4. Implement camera system (pan to follow owned creature)
5. WebSocket connection + full snapshot on join
6. Delta updates for position/angle/speed
7. Client-side interpolation for remote creatures
8. Basic creature AI (wander, approach nearby creatures)
9. PostgreSQL persistence (creature state on disconnect)
10. Basic conversation system (canned responses, then LLM)

**Deliverable**: A URL you can share with a friend. Both see creatures walking around in the same world.

### Phase 2: Social Features (Weeks 9-16)

1. Personality system affecting AI behavior
2. Relationship tracking between creatures
3. Full LLM conversation system with streaming
4. Community formation
5. User authentication (OAuth)
6. Creature breeding/spawning
7. Multi-zone support (custom server, migrate off Colyseus)
8. Interest management / viewport culling

### Phase 3: Scale & Features (Weeks 17-24)

1. Fighting system
2. Research / knowledge system
3. Group activities
4. Dynamic zone splitting
5. Kubernetes deployment
6. Monitoring and observability
7. Performance optimization (WebGL consideration)
8. Mobile browser support

---

## Appendix A: Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Client Runtime | Vanilla JS / TypeScript | No framework overhead. Canvas2D is native. |
| Client Rendering | Canvas2D (Phase 1-2), PixiJS/WebGL (Phase 3) | Canvas2D works now. WebGL when >100 creatures on screen. |
| Client Build | Vite | Fast dev server, modern bundling |
| Server Runtime | Node.js (TypeScript) | Same language as client. Colyseus is Node-native. |
| Game Framework (Phase 1) | Colyseus | Handles WebSocket, rooms, state sync, matchmaking |
| Game Framework (Phase 2+) | Custom | Outgrow Colyseus at ~500 entities per zone |
| Binary Protocol | Custom (ArrayBuffer/DataView) | Minimal overhead. No protobuf dependency for Phase 1. |
| Binary Protocol (Phase 3) | FlatBuffers | Zero-copy deserialization for performance |
| Database | PostgreSQL 16 | JSONB for flexible schema. GiST for spatial queries. Proven at scale. |
| Cache / PubSub | Redis 7 | Sub-millisecond pub/sub. Sorted sets for leaderboards. |
| LLM | Claude API (Haiku default, Sonnet escalation) | Best personality simulation. Streaming support. |
| CDN | Cloudflare | Free tier. DDoS protection. Edge caching for static assets. |
| Hosting (Phase 1) | Fly.io | WebSocket-native. Simple deployment. Affordable. |
| Hosting (Phase 3) | AWS EKS + Agones | Full Kubernetes orchestration for game servers. |
| Monitoring | Prometheus + Grafana (self-hosted) or Datadog | Game-specific metrics (tick time, entity count). |
| Error Tracking | Sentry | Client and server error capture. |
| Auth | Clerk or Auth0 | OAuth providers. JWT tokens. |

---

## Appendix B: Key Technical Decisions & Rationale

**Q: Why not peer-to-peer?**
A: P2P cannot handle persistent AI creatures. When User A goes offline, who simulates their creatures? Only a server can. Also, P2P has no authoritative state, making anti-cheat impossible.

**Q: Why not ECS (Entity Component System) for the server?**
A: ECS (like bitECS) is optimal when you have 100K+ entities with uniform behavior. At 5000 entities with complex, heterogeneous AI, a traditional OOP creature class is simpler and fast enough. If profiling shows the game loop bottleneck is entity iteration, migrate to ECS then.

**Q: Why not WebRTC data channels instead of WebSocket?**
A: WebRTC is for peer-to-peer or when you need UDP-like unreliable delivery. Since this game uses an authoritative server and TCP is fine for 20Hz ticks with slow-moving creatures, WebSocket is simpler and sufficient. WebRTC adds ICE/STUN/TURN complexity for no benefit.

**Q: Why 20Hz and not 10Hz or 30Hz?**
A: 10Hz (100ms between updates) causes visible jerking even with interpolation when creatures change direction. 30Hz (33ms) doubles CPU cost for minimal visual improvement given the creature movement speed. 20Hz is the sweet spot -- this is also what Agar.io uses.

**Q: Why PostgreSQL and not MongoDB?**
A: The data is highly relational (creatures belong to users, have relationships with other creatures, belong to communities). MongoDB would require denormalization and lose transactional consistency on cross-entity operations (breeding, community formation). PostgreSQL's JSONB gives schema flexibility where needed (behavior_state, AI memory) while maintaining relational integrity everywhere else.

**Q: Why not Cloudflare Durable Objects?**
A: Durable Objects have a 128MB memory limit per object and charge per request. A zone server holding 625 creatures with full behavior trees and spatial indexes needs ~200-500MB. Also, Durable Objects cannot run a continuous game loop -- they are request-driven. The game needs a persistent 20Hz tick loop that runs regardless of client requests.

---

## Appendix C: Message Protocol Quick Reference

### Client to Server

| Code | Name | Payload | Size |
|------|------|---------|------|
| 0x01 | MOVE_TO | targetX(f32) targetY(f32) | 9B |
| 0x02 | STOP | (none) | 1B |
| 0x03 | INTERACT | targetId(u32) actionType(u8) | 6B |
| 0x04 | SPEAK | targetId(u32) msgLen(u16) msg(utf8) | 7B + msg |
| 0x05 | PING | clientTime(f64) | 9B |
| 0x06 | VIEWPORT | x(f32) y(f32) w(f32) h(f32) | 17B |
| 0x07 | ACK_SNAPSHOT | seqNum(u32) | 5B |
| 0x08 | SPAWN_CREATURE | nameLen(u8) name(utf8) personality(5xu8) | 7B + name |
| 0x09 | SET_BEHAVIOR | creatureId(u32) priority(u8) behaviorId(u8) | 7B |

### Server to Client

| Code | Name | Key Fields | Typical Size |
|------|------|------------|-------------|
| 0x81 | FULL_SNAPSHOT | seq, time, creatures[] | 1-10KB |
| 0x82 | DELTA_UPDATE | seq, time, deltas[] | 50-500B |
| 0x83 | CREATURE_SPAWN | full creature data | ~60B |
| 0x84 | CREATURE_DESPAWN | creatureId | 5B |
| 0x85 | PONG | clientTime, serverTime | 17B |
| 0x86 | SPEECH_BUBBLE | creatureId, text, duration | 10B + text |
| 0x87 | ACTION_EVENT | actorId, targetId, type, data | 10-50B |
| 0x88 | ZONE_TRANSFER | url, token | 10B + strings |
| 0x89 | ERROR | code, message | 3B + msg |
| 0x8A | COMMUNITY_EVENT | type, communityId, data | 6B + data |
