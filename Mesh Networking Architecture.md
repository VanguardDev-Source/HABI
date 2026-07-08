# HABI
### Mesh Networking Architecture
**Document 2 of N**

Version 0.1 · Draft for Engineering Review
Depends on: Document 1 (Executive Summary, System Vision, PRD) — specifically the Message Priority Model (§3.4) and Operating Tiers (§2.4), which this document implements at the protocol level.

---

## 1. Design Goals and Constraints

Before any component design, the constraints that shape every decision here need to be explicit:

- **No central coordinator during Tier 0.** Every algorithm must function with only locally-available information (a device's own neighbor table and message store). Anything requiring global topology knowledge is disqualified.
- **Transport is dual-mode.** BLE is used for low-power discovery/advertisement and small control messages; Wi-Fi Direct (via the Nearby Connections API, which abstracts both) is used for higher-bandwidth payload transfer. The architecture must decide, per message, which transport to use.
- **Nodes are unreliable and mobile.** A neighbor present at t=0 may be gone at t=5s. All state must be soft (TTL'd, re-confirmed), never assumed durable.
- **Devices are heterogeneous.** Budget Android phones, 2–4 years old, with limited RAM and aggressive OS-level background restrictions (Doze mode, battery optimization killing services) are the *majority* target device, not the edge case.
- **Security cannot depend on the network being trusted**, because in Tier 0 there is no server to arbitrate trust. Identity and integrity must be self-certifying (signed at origin).

---

## 2. Layered Protocol Model

HABI's mesh stack is organized into five layers, deliberately mirroring OSI-style separation so each layer can be tested, reasoned about, and replaced independently (e.g., swapping Nearby Connections for a future LoRa transport later without redesigning routing).

```
┌─────────────────────────────────────────────┐
│  L5 — Application Layer                      │  Chat, SOS, Requests, Alerts, Bulletin
├─────────────────────────────────────────────┤
│  L4 — Message Layer                          │  Envelope, Priority, TTL, Signatures, Dedup
├─────────────────────────────────────────────┤
│  L3 — Routing Layer                          │  Neighbor table, forwarding decisions, ACKs
├─────────────────────────────────────────────┤
│  L2 — Session Layer                          │  Connection lifecycle, peer authentication
├─────────────────────────────────────────────┤
│  L1 — Transport Layer                        │  BLE (discovery/control), Wi-Fi Direct (payload)
└─────────────────────────────────────────────┘
```

Each layer's responsibility is isolated so failures are legible: a routing bug cannot corrupt message integrity (that's L4's job, protected by signatures that survive re-transmission across many L3 hops), and a transport dropout cannot corrupt session state (L2 owns reconnection).

---

## 3. Peer Discovery

### 3.1 Mechanism

HABI uses a two-phase discovery process built on Google's Nearby Connections API (which itself multiplexes BLE, Wi-Fi Direct, and classic Bluetooth depending on device capability and proximity):

**Phase 1 — Advertisement (passive/low-power):**
Every device, whenever the HABI foreground service is active, advertises a compact BLE beacon containing:

```
AdvertisementPayload {
  protocol_version: uint8
  device_short_id:  bytes[8]     // truncated hash of device public key
  tier:             uint2        // 0, 1, or 2 (current connectivity tier)
  role_flags:       uint8        // bitfield: RESPONDER, GOV_VERIFIED, RELAY_CAPABLE, LOW_BATTERY
  queue_pressure:   uint4        // 0-15, coarse signal of how full this node's outbound queue is
}
```

This is intentionally tiny (~14 bytes) to keep BLE advertisement cheap on battery. It is *not* the full identity handshake — it's a "here I am, this is roughly what I am" signal.

**Phase 2 — Negotiated Connection (active):**
When two devices' advertisements are mutually visible, either side may initiate a Nearby Connections `requestConnection`. This upgrades the link from BLE-only to a full bidirectional channel (Wi-Fi Direct where available, falling back to Bluetooth Classic), at which point the L2 Authentication Handshake (§4) runs.

### 3.2 Discovery Policy — Who Initiates?

To avoid every device in a crowded room trying to connect to every other device simultaneously (an O(n²) connection storm), initiation is deterministic and symmetry-broken:

- A device initiates a connection to a discovered peer only if `own_device_id < peer_device_id` (lexicographic comparison of the public-key-derived ID), **unless** the peer's advertisement shows `RESPONDER` or the local device has SOS-priority traffic queued for send — in which case initiation is immediate and bidirectional-attempt, because emergency traffic should not wait on tie-breaking convention.
- Devices cap concurrent active connections at a configurable `MAX_ACTIVE_PEERS` (default 6, tuned empirically against Nearby Connections' practical stability limits on mid-tier hardware). Beyond this cap, new discoveries are recorded in the neighbor table as "known but not connected" and are opportunistically connected to as existing connections churn.

### 3.3 Device Advertisement Content Rationale

The `queue_pressure` field lets a neighbor make a cheap local routing decision (§6) without a full handshake: if my queue is nearly full and a peer's advertised pressure is low, I prefer routing through them. This is a coarse, gossip-free load-balancing signal — deliberately imprecise, because precision here would cost more battery than it saves in routing efficiency.

---

## 4. Peer Authentication & Connection Lifecycle

### 4.1 Identity Model

Every HABI installation generates an ECDSA (P-256) keypair on first launch, stored in the Android Keystore (hardware-backed where available — detailed in the Security Architecture document). The public key, hashed, is the device's permanent `device_id`. There is no username/password identity at the mesh layer — identity is the key. Human-readable profile info (name, role) is bound to this key via a signed profile object, so it can't be forged by a relay.

Government/verified accounts (LGU, NDRRMC, Red Cross) carry an additional **role certificate**: a short-lived, backend-issued signed statement ("this public key is verified as Barangay X Tanod") that devices can verify offline against a pinned government root public key baked into the app — this is what makes `GOV_VERIFIED` badges and Government Alert priority tamper-resistant even with zero connectivity (detailed in Security Architecture §Role Certificates).

### 4.2 Handshake Sequence

```
Device A                                  Device B
   |──── ConnectionRequest(A_id) ─────────────>|
   |<─── ConnectionAccept/Reject(B_id) ────────|
   |──── Hello{A_pubkey, A_nonce, A_sig} ──────>|
   |<─── Hello{B_pubkey, B_nonce, B_sig} ───────|
   |         [both verify signature over nonce]  |
   |──── ECDH_KeyExchange(A_ephemeral_pub) ────>|
   |<─── ECDH_KeyExchange(B_ephemeral_pub) ─────|
   |    [derive session key via ECDH + HKDF]     |
   |──── SessionReady(A) ──────────────────────>|
   |<─── SessionReady(B) ────────────────────────|
   |==== Encrypted Session Established =========|
```

Key points:
- The `Hello` exchange authenticates *long-term identity* (proves each side controls the private key behind their advertised `device_id`).
- The ECDH exchange uses fresh ephemeral keys per session, giving forward secrecy — compromise of a device's long-term key later doesn't retroactively decrypt past session traffic (relevant given phones may be lost/damaged/looted during a disaster).
- Full technical detail (nonce construction, replay protection at the handshake level, HKDF parameters) lives in the Security Architecture document; this section covers only what's needed to understand the mesh lifecycle.

### 4.3 Connection States

```
       discover()
   ┌─────────────┐
   │  DISCOVERED  │──────────────┐
   └─────────────┘                │ connect()
          │ timeout/lost           ▼
          │                 ┌─────────────┐
          │                 │ CONNECTING   │
          │                 └─────────────┘
          │                        │ handshake success   │ handshake fail/timeout
          │                        ▼                       │
          │                 ┌─────────────┐                │
          │                 │ AUTHENTICATED│               │
          │                 └─────────────┘                │
          │                        │ idle > HEARTBEAT_MISS  │
          │                        ▼                        ▼
          │                 ┌─────────────┐          ┌─────────────┐
          └────────────────>│  DISCONNECTED│<─────────│   REJECTED   │
                             └─────────────┘          └─────────────┘
```

- **DISCOVERED**: seen via BLE advertisement, not yet connected. Entry in neighbor table with `last_seen` timestamp only.
- **CONNECTING**: Nearby Connections link + handshake in progress.
- **AUTHENTICATED**: session key established; this is the only state in which message forwarding occurs.
- **DISCONNECTED**: link lost (explicit disconnect, timeout, or out-of-range). Any messages in-flight to this peer are returned to the local outbound queue for re-routing.
- Heartbeats (small L2 keepalive, every `HEARTBEAT_INTERVAL` = 15s, tunable) detect silent connection death faster than relying solely on OS-level disconnect callbacks, which are not always timely on constrained hardware.

### 4.4 Neighbor Table

Each device maintains an in-memory (Room-backed for crash recovery) table:

| Field | Purpose |
|---|---|
| `device_id` | Peer identity |
| `state` | Current lifecycle state (§4.3) |
| `last_seen` | Freshness for eviction |
| `tier` | Peer's connectivity tier — critical for routing toward bridges |
| `role_flags` | Responder/Gov/RelayCapable/LowBattery |
| `queue_pressure` | Last-advertised load signal |
| `link_quality` | Rolling estimate (RSSI-derived + recent delivery success rate) used as a routing weight |
| `hop_distance_to_bridge` | Best known distance (in hops) from this neighbor to *any* Tier-2 device, gossiped opportunistically (§6.3) |

Entries older than `NEIGHBOR_TTL` (default 90s) without a refresh (heartbeat, advertisement, or successful message exchange) are evicted, and any queued messages routed via that neighbor are re-queued for re-routing.

---

## 5. Message Layer: Envelope, Priority, TTL, Deduplication

### 5.1 Message Envelope (Wire Format)

Every message on the mesh, regardless of application type, is wrapped in a canonical envelope. This is the single contract every layer above L3 depends on.

```
MessageEnvelope {
  message_id:        uuid128         // origin-generated, globally unique
  origin_device_id:  bytes[32]       // sender's public key hash
  msg_type:          enum            // CHAT, SOS, MEDICAL, FOOD, WATER, SHELTER,
                                      // VOLUNTEER, GOV_ALERT, MISSING_PERSON,
                                      // ROAD_BLOCK, HAZARD
  priority:          uint8           // derived from msg_type per PRD §3.4, see table below
  created_at:        int64           // origin device epoch millis
  ttl_hops:          uint8           // decremented per relay hop
  ttl_expiry:        int64           // absolute wall-clock expiry (belt-and-suspenders vs ttl_hops)
  destination:       enum{BROADCAST, UNICAST(device_id), GROUP(group_id)}
  payload:           bytes           // encrypted application payload (see Security doc)
  origin_signature:  bytes[64]       // ECDSA sig over (all fields above) by origin device
  route_history:     bytes[]         // append-only list of relay device_ids (bounded length, see §5.4)
}
```

Two independent TTL mechanisms are deliberate: `ttl_hops` bounds propagation in dense, low-mobility meshes where a message could otherwise loop for a long time; `ttl_expiry` bounds propagation in *time*, which matters because SOS/Medical messages should stop being relayed once they're stale regardless of hop count (an SOS from 6 hours ago that never got acknowledged needs different handling than fresh dispatch — see §9 for expiry-triggered escalation behavior).

### 5.2 Priority Mapping

Directly implementing PRD §3.4:

| msg_type | priority (0=highest) | default ttl_hops | default ttl_expiry |
|---|---|---|---|
| SOS | 0 | 32 | 24h |
| MEDICAL | 1 | 32 | 12h |
| GOV_ALERT | 2 | 24 | 48h |
| MISSING_PERSON | 3 | 20 | 7d |
| HAZARD, ROAD_BLOCK | 4 | 16 | 24h |
| FOOD, WATER, SHELTER, VOLUNTEER (Resource Requests) | 5 | 12 | 24h |
| CHAT, Bulletin | 6 | 8 | 6h |

These defaults are configuration, not hardcoded constants — they're pulled from a signed config object the backend can push once online (Tier 2) so NDRRMC can, e.g., extend GOV_ALERT TTL during a prolonged event, without an app update. Absent connectivity, devices use the last-known config or these compiled-in defaults.

### 5.3 Duplicate Detection

Because messages broadcast to multiple neighbors, who each relay to their own neighbors, the same `message_id` will arrive at a device many times via different paths. Naive re-broadcast of every arrival causes exponential traffic blowup (broadcast storm) that would saturate mesh bandwidth within seconds in a moderately dense area.

**Mechanism:** each device maintains a bounded **Seen-Message Cache** — a Bloom filter backed by a Room table of `(message_id, first_seen_at)` for exact confirmation on filter hits (Bloom filters have false positives; the DB table resolves them, but is only consulted when the filter says "maybe seen," keeping the hot path cheap).

```
on_receive(envelope):
    if bloom_filter.might_contain(envelope.message_id):
        if seen_table.exists(envelope.message_id):
            record_route_diversity(envelope)   // still useful for hop_distance_to_bridge gossip
            drop(envelope)                     // do not re-relay
            return
    bloom_filter.add(envelope.message_id)
    seen_table.insert(envelope.message_id, now())
    process_and_maybe_relay(envelope)
```

The Bloom filter is sized (and periodically rotated/cleared based on `ttl_expiry` of tracked messages) to keep false-positive rate low without unbounded memory growth — full sizing math is in Appendix A of this document series' implementation notes, but the operating principle is: filter capacity is provisioned for `MAX_EXPECTED_UNIQUE_MESSAGES_PER_TTL_WINDOW`, not for total device lifetime.

### 5.4 Route History & Loop Prevention

`route_history` records the last `N` (default 8) relay device_ids the message has passed through. Before relaying, a device checks: *is my own device_id already in route_history?* If so, it does not relay (I've already seen and forwarded this via a different path — relaying again would create a loop). This is a secondary loop guard on top of duplicate detection (§5.3), useful specifically during the brief window before a device's own Bloom filter/seen-table entry is durably committed.

Route history also feeds the `hop_distance_to_bridge` gossip described in §6.3, and — operationally — gives responders/dashboard a forensic trail of which physical path an emergency message took, useful for after-action analysis of mesh performance in a real event.

---

## 6. Routing Strategy

### 6.1 Why Not Classical MANET Protocols Wholesale

Standard mobile ad-hoc network protocols (AODV, OLSR, DSR) assume relatively frequent connectivity and optimize for shortest-path efficiency. HABI's actual objective function is different: **maximize probability of eventual delivery under partition-prone, intermittent topology**, which is the problem space of Delay-Tolerant Networking (DTN), not classical MANET routing. HABI's routing strategy is therefore a **priority-weighted epidemic/spray-and-wait hybrid**, informed by DTN literature (notably Vahdat & Becker's Epidemic Routing and Spyropoulos et al.'s Spray-and-Wait) but adapted to HABI's specific priority model and battery constraints.

### 6.2 Core Strategy: Priority-Weighted Controlled Flooding

- **SOS, Medical, Government Alert (priority 0–2):** Use near-pure epidemic routing — relay to *all* currently-connected AUTHENTICATED neighbors that haven't already seen the message (per §5.3/5.4), regardless of relay-battery policy (with the safety floor described in §9.4). Rationale: for life-critical traffic, maximizing propagation speed and path diversity outweighs bandwidth/battery cost. This is a deliberate, documented trade-off, not an oversight.
- **Missing Person, Hazard (priority 3–4):** Use **Spray-and-Wait**: the message is given a copy budget (`L` copies, default 10). The holding device splits its remaining copy budget with each newly-encountered neighbor (binary spray: give away half), until a device holds only 1 copy, at which point it switches to direct-delivery-only (waits for the destination or a bridge, does not relay further except to a strictly-better-positioned node per `hop_distance_to_bridge`). This bounds total network traffic while still achieving good propagation for area-relevant-but-not-instantly-life-critical content.
- **Resource Requests, Chat/Bulletin (priority 5–6):** Use **bridge-seeking unicast/limited-flood**: for UNICAST destinations, prefer routing toward the neighbor with lowest `hop_distance_to_bridge` combined with highest historical delivery success toward that destination (a simple locally-learned reputation score, not a full distance-vector table — full distance-vector routing is considered too chatty/battery-costly for this priority tier at mesh scale). For BROADCAST-scoped low-priority content (e.g., Bulletin Board posts), relay is throttled: a device only relays a priority-6 message if its own `queue_pressure` is below a threshold and it hasn't relayed more than `N` priority-6 messages in the current time window (rate limiting to protect higher-priority traffic's share of bandwidth).

### 6.3 Bridge-Seeking: `hop_distance_to_bridge` Gossip

Devices in Tier 2 (internet-connected) advertise `hop_distance_to_bridge = 0`. Every other device sets its own value to `min(neighbor.hop_distance_to_bridge) + 1` across its currently AUTHENTICATED neighbors, refreshed on every heartbeat. This is a minimal, single-scalar distributed Bellman-Ford-style relaxation — deliberately not a full routing table, just enough signal to let priority 3–6 traffic make a locally-greedy "move toward the internet" decision without expensive gossip overhead. Priority 0–2 traffic ignores this signal (it floods regardless, because waiting for an optimal path is the wrong trade-off for an SOS).

### 6.4 Priority Queue Implementation

Each device's outbound queue is a **multi-level priority queue**, not a single sorted structure, to make starvation of low-priority traffic bounded rather than total:

```
class MeshOutboundQueue {
    queues: Map<PriorityLevel, Deque<MessageEnvelope>>   // 7 levels, per PRD §3.4

    fun enqueue(msg: MessageEnvelope) {
        queues[msg.priority].addLast(msg)
    }

    fun nextForTransmission(availableBandwidthBudget: Int): List<MessageEnvelope> {
        // Weighted round-robin with strict priority for tiers 0-2 (SOS/Medical/Gov):
        // those tiers are always fully drained before any budget is given to lower tiers,
        // UNLESS starvation-prevention kicks in (see below).
        val batch = mutableListOf<MessageEnvelope>()
        var budget = availableBandwidthBudget

        for (level in 0..2) {                      // SOS, Medical, Gov: strict priority
            while (budget > 0 && queues[level].isNotEmpty()) {
                val msg = queues[level].removeFirst()
                batch.add(msg); budget -= msg.estimatedSize()
            }
        }
        // Starvation prevention: guarantee a minimum slice (e.g. 10%) of every
        // transmission window to levels 3-6 even under sustained 0-2 traffic,
        // because Missing Person / Hazard / Relief / Chat must never be
        // *permanently* starved during a long event.
        val reservedForLower = (availableBandwidthBudget * 0.10).toInt()
        var lowerBudget = min(budget, reservedForLower).coerceAtLeast(reservedForLower)
        for (level in 3..6) {
            while (lowerBudget > 0 && queues[level].isNotEmpty()) {
                val msg = queues[level].removeFirst()
                batch.add(msg); lowerBudget -= msg.estimatedSize()
            }
        }
        return batch
    }
}
```

The 10% starvation-prevention reservation is a deliberate policy choice worth calling out: a purely strict-priority queue is the "textbook correct" triage answer, but in a real multi-day event a Missing Person report or a Road Block hazard that never gets bandwidth because SOS traffic is sustained is itself a harm the system should avoid. This value is operator-configurable (pushed via signed config, per §5.2) so NDRRMC can tune it based on observed event severity.

### 6.5 Acknowledgement System

Two ACK types exist, serving different purposes:

- **Hop ACK (L3):** Lightweight, per-link, confirms a message was received by the immediate next hop (not that it reached its final destination). Used to trigger retry (§6.6) if missing. Not application-visible.
- **Delivery ACK (L4/L5):** End-to-end, generated by the actual destination (or, for BROADCAST messages, by the first responder/dashboard that "claims" the item — e.g., an SOS acknowledged by a Tanod). Delivery ACKs are themselves messages that get routed back through the mesh (typically at the same or one-lower priority as the original), and are what flips a message's status from "Relaying" to "Delivered" in the sender's UI (per PRD FR-1.3).

### 6.6 Retry Algorithm

```
on_send_attempt(msg, neighbor):
    schedule_timeout(msg.id, neighbor.id, HOP_ACK_TIMEOUT)  // e.g. 5s

on_hop_ack_timeout(msg, neighbor):
    attempt_count += 1
    if attempt_count >= MAX_RETRY_PER_NEIGHBOR (3):
        mark_neighbor_unreliable(neighbor)   // deprioritize in future routing decisions
        requeue_for_alternate_route(msg)
    else:
        backoff = BASE_BACKOFF * (2 ^ attempt_count) + jitter()   // exponential backoff + jitter
        schedule_retry(msg, neighbor, after=backoff)
```

Exponential backoff with jitter prevents synchronized retry storms across many devices that all lost the same neighbor simultaneously (a common event — e.g., a responder walks out of range of a cluster). SOS/Medical retries use a shorter base backoff and a higher retry ceiling than lower-priority traffic, reflecting their tolerance for redundant transmission cost.

---

## 7. Packet Serialization

Envelopes are serialized using **Protocol Buffers**, chosen over JSON for three concrete reasons relevant to this environment specifically: (1) compact binary encoding matters directly for BLE/Wi-Fi Direct throughput and battery cost, unlike a typical server-to-server API where bandwidth is cheap; (2) schema evolution support lets the wire format add fields across app version upgrades without breaking mesh interop between devices running slightly different HABI versions during a rolling nationwide deployment; (3) strong typing catches malformed-envelope bugs at compile time rather than at parse time in the field.

```protobuf
syntax = "proto3";
package habi.mesh.v1;

message MessageEnvelope {
  bytes message_id = 1;
  bytes origin_device_id = 2;
  MessageType msg_type = 3;
  uint32 priority = 4;
  int64 created_at = 5;
  uint32 ttl_hops = 6;
  int64 ttl_expiry = 7;
  Destination destination = 8;
  bytes payload = 9;              // encrypted; see Security Architecture doc
  bytes origin_signature = 10;
  repeated bytes route_history = 11;
}

enum MessageType {
  CHAT = 0; SOS = 1; MEDICAL = 2; FOOD = 3; WATER = 4;
  SHELTER = 5; VOLUNTEER = 6; GOV_ALERT = 7; MISSING_PERSON = 8;
  ROAD_BLOCK = 9; HAZARD = 10;
}

message Destination {
  oneof kind {
    bool broadcast = 1;
    bytes unicast_device_id = 2;
    bytes group_id = 3;
  }
}
```

A version byte precedes every serialized envelope on the wire (outside the protobuf message itself) so a device can detect and gracefully reject/quarantine envelopes from an incompatible future protocol version rather than crash-looping on unknown fields — important given the "rolling upgrade across a nationwide fleet during active disasters" deployment reality.

---

## 8. Offline Storage & Sync Queue

### 8.1 Local Persistence

All mesh state lives in Room (SQLite), encrypted at rest via SQLCipher (Security Architecture doc, §Local Encryption). Three tables are load-bearing for the mesh engine specifically (full ER diagram in the Database document):

- `mesh_messages` — the durable envelope store; every message this device has originated, relayed, or received, with local status (`QUEUED`, `RELAYING`, `DELIVERED`, `EXPIRED`, `FAILED`).
- `mesh_neighbors` — persisted mirror of the in-memory neighbor table, so a killed/restarted app doesn't lose routing context immediately (repopulated fast via re-advertisement on restart, but persistence avoids a cold-start gap).
- `sync_queue` — the subset of `mesh_messages` not yet confirmed synced to the backend, ordered by the same priority discipline as §6.4, consumed when the device transitions to Tier 1/2.

### 8.2 Sync Queue Behavior

On tier transition to Bridged/Connected (detected via `ConnectivityManager` callbacks + an active internet reachability probe, not just "Wi-Fi is on" which is an unreliable proxy for actual internet):

```
on_tier_upgrade(new_tier):
    if new_tier >= TIER_1_BRIDGED:
        sync_worker.enqueue(SyncJob(priority_ordered = true))

SyncJob.run():
    batch = sync_queue.nextBatch(orderedByPriority = true, maxBatchBytes = ADAPTIVE_BATCH_SIZE)
    for msg in batch:
        response = backend_api.pushMessage(msg)     // idempotent, keyed by message_id
        if response.success:
            local_db.markSynced(msg.id)
        else:
            exponential_backoff_and_retry(msg)
    pull_updates_since(last_sync_watermark)          // gov alerts, status changes, etc.
    redistribute_pulled_updates_into_mesh()           // bridge behavior: push pulled
                                                       // gov alerts back OUT into mesh
                                                       // neighbors who are still Tier 0
```

The last line matters architecturally: a bridge device doesn't just upload — it also **downloads and re-injects** (e.g., a fresh Government Alert or a "Missing Person Found" status update pulled from the backend gets broadcast back into the local mesh cluster at the appropriate priority), which is how Tier-0-only devices eventually receive information that originated or was resolved entirely outside their direct mesh reach. This closes the loop described in PRD §2.4's Tier 1 definition.

### 8.3 Idempotency & Duplicate-Safe Sync

Backend `pushMessage` is keyed on `message_id` (globally unique, origin-generated) with an upsert/conflict-ignore semantic, because the same message may be pushed by multiple independent bridge devices that each carried a copy through different mesh paths (a direct, desired consequence of the epidemic routing in §6.2 for high-priority traffic). This is why message_id is generated at origin, not at the backend — the backend must be able to deduplicate across N independent uploads of "the same" emergency without needing the uploader to coordinate.

---

## 9. Reconnection Logic & Degradation Handling

### 9.1 Reconnection Backoff

When a previously-AUTHENTICATED peer disconnects, the device attempts reconnection using the same exponential-backoff-with-jitter pattern as §6.6, capped at `MAX_RECONNECT_INTERVAL` (default 60s), rather than continuous polling — continuous BLE/Wi-Fi Direct reconnect attempts are one of the largest avoidable battery drains in poorly-designed mesh implementations.

### 9.2 Partition Handling

There is no special "partition detected" event — HABI's architecture treats every moment as a potential partition by design (§1, "no central coordinator"). What changes on partition is purely local: neighbor table entries expire (§4.4), `hop_distance_to_bridge` estimates go stale and increase (correctly reflecting reduced connectivity), and queued messages simply wait, re-attempting relay as new neighbors are discovered. This is a deliberate simplicity choice: rather than build explicit partition-detection/merge logic (as some MANET protocols do), HABI's store-and-forward model makes partition and normal operation the same code path, which reduces a significant source of distributed-systems bugs.

### 9.3 Stale-SOS Escalation

If an SOS message's local device never receives a Delivery ACK (§6.5) within a configurable window (default 10 minutes) despite continued re-broadcast, the UI escalates: prompts the user with guidance (move to higher ground/open area for better radio propagation, alternate emergency contact methods) rather than silently continuing to retry forever with no feedback — a direct application of the "human-centered under stress" principle from PRD §2.2.

### 9.4 Battery Optimization

Battery policy is the mesh engine's most safety-sensitive trade-off surface, so it's specified as an explicit, tiered obligation model rather than a single global toggle:

| Battery Level | Relay Obligation |
|---|---|
| > 30% | Full relay participation, all priority levels, per §6.2 |
| 15–30% | Relay priority 0–2 (SOS/Medical/Gov) always; priority 3–6 relay throttled to ~30% of normal rate |
| 5–15% | Relay priority 0–1 (SOS/Medical) only; all other relay suspended; device still fully usable for the *user's own* messaging |
| < 5% | Device stops relaying entirely (self-preservation — a dead phone helps no one); still permitted to *originate* an SOS for itself if the user needs it, as a final action |

This tiering directly resolves Risk R3 from the PRD: individually-rational battery conservation is allowed to reduce relay generosity, but SOS/Medical relay obligation is the last thing to degrade, not the first — encoding the ethical priority explicitly in the algorithm rather than leaving it to emergent behavior. The background scanning/advertising duty cycle itself also scales with battery level and with `role_flags` (a device the user has marked as `RESPONDER`, e.g. a Red Cross volunteer's phone plugged into a vehicle charger, can be configured to hold a more aggressive always-on relay posture regardless of this table, since it's expected to be power-provisioned).

Implementation-wise, this is enforced via Android `WorkManager` constraints plus a custom battery-tier evaluator that runs on every scheduled background mesh cycle (detailed further, including Doze-mode interaction and foreground service justification, in the Android Application Architecture document).

---

## 10. Summary: What Each Message Type Actually Does, End to End

To make the abstractions above concrete, here is the full lifecycle of a single SOS, tying every section together:

1. **Origination (L5→L4):** Maria taps SOS. App captures GPS (or last-known), battery level, optional note. L4 wraps it in a `MessageEnvelope` with `priority=0`, `ttl_hops=32`, `ttl_expiry=+24h`, signs it with her device key.
2. **Local persistence (§8.1):** Written to `mesh_messages` as `QUEUED` before any network attempt — survives app kill/reboot.
3. **Routing decision (§6.2):** Priority 0 → flood to all AUTHENTICATED neighbors immediately, bypassing normal battery-tier throttling except the >5% floor (§9.4).
4. **Transmission + Hop ACK (§6.5/6.6):** Sent to each neighbor; retried with backoff on missing Hop ACK.
5. **Relay hop-by-hop (§5.3/5.4):** Each receiving device checks Bloom filter/seen-table, checks route_history for loops, and — being priority 0 — floods onward to its own neighbors, decrementing `ttl_hops`.
6. **Bridge encounter (§6.3/§8.2):** Eventually a device with `hop_distance_to_bridge = 0` (Tier 2) or one that transitions to Tier 1/2 receives it; `sync_worker` pushes it to the backend (§8.2/8.3), idempotently.
7. **Backend → Dashboard:** NDRRMC/LGU dashboard displays it on the live map instantly (Admin Dashboard document), triaged into the appropriate responder queue.
8. **Acknowledgement (§6.5):** A responder (P2) claims it; a Delivery ACK routes back through the mesh (or direct via backend push if Maria's device has since reached Tier 1/2 itself) to her device, flipping status to Delivered in her UI.
9. **Escalation fallback (§9.3):** If no ACK arrives within 10 minutes, Maria's app surfaces guidance rather than failing silently.

---

**Next up:** the **Database Schema** document will formalize `mesh_messages`, `sync_queue`, `neighbors`, and the rest of the normalized PostgreSQL schema referenced throughout this document (Users, Devices, Messages, MessageRoutes, EmergencyRequests, MedicalRequests, Shelters, Resources, MissingPersons, SynchronizationLogs, AuditLogs, NotificationQueue), followed by the Android Application Architecture (Clean Architecture module breakdown + screen specs) and Backend Architecture (NestJS services + API contracts).
