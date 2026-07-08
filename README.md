# HABI
### Connecting Communities When Everything Else Fails
**Document 1 of N — Executive Summary, System Vision, Product Requirements Document**

Version 0.1 · Draft for Engineering Review

---

## 1. Executive Summary

### 1.1 Problem Statement

The Philippines experiences an average of 20 tropical cyclones per year, sits on the Pacific Ring of Fire, and is regularly affected by earthquakes, volcanic activity, flooding, and landslides. In nearly every major disaster event of the last two decades — Yolanda (2013), the Bohol earthquake (2013), Odette (2021), Taal's eruption (2020) — the first infrastructure to fail is telecommunications: cell towers lose power, fiber backhaul is severed, and internet connectivity in the affected zone disappears for days to weeks.

This creates a critical gap during the exact window when communication matters most:

- Survivors cannot report their location or condition (SOS).
- Families cannot locate missing relatives.
- Local government units (LGUs) cannot coordinate relief distribution.
- First responders cannot triage medical emergencies by severity.
- National agencies (NDRRMC, DICT, Philippine Red Cross) have no real-time visibility into ground conditions until manual reports are physically relayed out of the affected area.

Existing solutions (SMS, satellite phones, ham radio, Starlink) each address part of the problem but share a common weakness: they either depend on infrastructure that fails alongside the disaster, or require specialized hardware unavailable to the general population.

### 1.2 Proposed Solution

HABI is an Android-first, offline-capable disaster communication platform that turns every participating smartphone into a mesh network node. Using Bluetooth Low Energy (BLE) for peer discovery and Wi-Fi Direct / Nearby Connections for higher-bandwidth message transport, devices within range of one another form ad-hoc clusters. Messages hop from phone to phone — store-and-forward, similar in principle to delay-tolerant networking (DTN) used in space communications — until they reach their destination or, opportunistically, a device that has regained internet or cellular connectivity, at which point the message is synchronized to the cloud backend and made visible to government dashboards.

No app-layer feature in HABI requires the internet to function at the local (barangay/community) level. Internet is treated as an optional accelerant, not a dependency.

### 1.3 Why This Is Hard (and why it matters)

This is not a chat app with a "no internet" banner. The engineering challenges are genuinely distributed-systems problems, compressed into constrained mobile hardware:

- **Network topology is unknown, unstable, and adversarial-by-nature.** Nodes appear and disappear as people move, phones die, and buildings block signal. There is no central coordinator during the mesh phase.
- **Battery is the scarcest resource.** A phone acting as a relay node while its owner is also trying to preserve battery for their own emergency use creates a direct tension the routing and power layers must resolve.
- **Data integrity and authenticity must hold without a trusted server in the loop at message-creation time.** A false SOS or a spoofed government alert during a disaster is not a bug — it is a life-safety incident.
- **Priority is not a UX nicety, it's triage.** An SOS from a trapped survivor must displace ordinary chat traffic on constrained mesh bandwidth, deterministically, under a formal scheduling policy.
- **The system must degrade gracefully across a spectrum of connectivity**, from fully offline mesh-only, to partial mesh with one bridging node online, to full internet availability — without the user having to think about which mode they're in.

### 1.4 Stakeholders

| Stakeholder | Primary Interest |
|---|---|
| NDRRMC | National situational awareness, resource allocation, alert dissemination |
| DICT | Technical standards compliance, infrastructure integration, data sovereignty |
| Philippine Red Cross | Medical request triage, volunteer coordination, missing persons |
| LGUs (Barangay → Provincial) | Local coordination, evacuation management, resource requests |
| Citizens / Survivors | SOS, chat with family/neighbors, information access |
| First Responders | Field triage, dispatch, situational data from the ground |

### 1.5 Success Criteria

HABI is successful if, in a disaster scenario with zero cellular/internet infrastructure across a barangay-sized area (roughly 1–5 km²), a message of type SOS or Medical originating from any device reaches either (a) a physically present responder, or (b) an internet-connected relay device, within a bounded, measurable time window that is a function of population device density — and does so without any centralized infrastructure in the affected zone.

---

## 2. System Vision

### 2.1 Vision Statement

> A disaster should never be able to fully silence a community. As long as two phones exist within radio range of each other, HABI ensures a message can move — and as long as one phone anywhere in the chain can see the internet, that message reaches the people who can act on it.

### 2.2 Guiding Design Principles

1. **Offline-first, not offline-tolerant.** Every core feature is designed assuming zero connectivity as the default state. Internet sync is an enhancement layer, architecturally bolted on rather than assumed.
2. **Store-and-forward over synchronous delivery.** HABI never assumes the recipient is reachable now. Every message has a lifecycle, a TTL, and a persistence guarantee independent of any single connection.
3. **Priority is a first-class architectural concern**, not a UI sort order. It governs queueing, retry, bandwidth allocation, and battery trade-offs throughout the stack.
4. **Zero-trust mesh.** No peer is inherently trusted. All messages are signed at origin; all relays are transparent (they forward, they do not need to read) where end-to-end encryption is in effect; identity is cryptographically anchored, not just self-asserted.
5. **Battery-aware by design.** Every background operation — scanning, advertising, relaying — is scheduled against a battery/priority cost model, not run unconditionally.
6. **Progressive enhancement across connectivity tiers.** The same UI and data model must work identically whether the device is mesh-only, mesh+intermittent internet, or fully online — only the sync behavior changes underneath.
7. **Human-centered under stress.** UI decisions assume the user may be panicked, injured, in low light, with low battery, and possibly not the primary user of the device (a rescuer using a survivor's phone). Design accordingly: large touch targets, minimal required input for SOS, offline maps, high contrast, no unnecessary confirmation friction on emergency actions.
8. **Government-grade auditability.** Every state-changing action in the system — message routing decisions, alert broadcasts, resource dispatch — is logged in a tamper-evident audit trail suitable for after-action review and legal/regulatory scrutiny.

### 2.3 System Boundaries (What HABI Is / Is Not)

**In scope:**
- Peer-to-peer offline messaging and emergency broadcast between Android devices
- Store-and-forward relay across multi-hop mesh topologies
- Opportunistic synchronization to a central backend once any node regains connectivity
- Government/NDRRMC-facing monitoring dashboard fed by synchronized mesh data
- iOS is explicitly **out of scope for v1** due to Apple's background BLE/Wi-Fi Direct platform restrictions (discussed in the Android Requirements document) — this is a material constraint, not an oversight, and is called out because it affects national deployment strategy (dual-OS households, iOS-only users are mesh-invisible in v1).
- HABI is not a replacement for professional emergency dispatch (911/143), ham radio networks, or satellite communication — it is a complementary layer that functions when those are degraded or overwhelmed, and it explicitly should integrate with, not replace, existing NDRRMC protocols.

### 2.4 Operating Modes

HABI's UI and services expose exactly one mental model to the user, but the system internally operates across three connectivity tiers that determine sync behavior:

| Tier | Condition | Behavior |
|---|---|---|
| **Tier 0 — Mesh Isolated** | No device in the local mesh has internet | Pure store-and-forward over BLE/Wi-Fi Direct. All data local (Room DB) until a bridge appears. |
| **Tier 1 — Bridged** | At least one mesh peer has intermittent internet | That peer acts as a gateway node, opportunistically syncing queued messages to the backend and pulling down government alerts to redistribute into the mesh. |
| **Tier 2 — Connected** | The device itself has internet | Direct sync with backend; mesh continues operating in parallel for resilience and redundancy, and to keep offline neighbors updated. |

This tiering, and the algorithms that manage transition between tiers, are detailed in the Mesh Networking Architecture document.

---

## 3. Product Requirements Document (PRD)

### 3.1 Personas

**P1 — Maria, Barangay Resident (Primary/Survivor)**
Non-technical smartphone user, mid-range Android device, may be injured or under stress during use. Needs: one-tap SOS, ability to tell family she's safe, see nearby hazard/road information.

**P2 — Responder Jun, Barangay Tanod / Red Cross Volunteer**
Semi-technical, uses HABI actively during operations. Needs: triaged incoming requests by severity/proximity, ability to mark requests resolved, situational map, coordination chat with other responders.

**P3 — LGU Disaster Officer**
Coordinates at municipal/provincial level. Needs: aggregated view of all requests, resource allocation tools, ability to push verified alerts into affected mesh clusters, exportable reports for NDRRMC.

**P4 — NDRRMC/DICT Analyst (Dashboard User)**
Desk-based, only online. Needs: national/regional map view, mesh network health metrics, analytics, historical data for after-action review.

### 3.2 Functional Requirements

Requirements are tagged by priority using MoSCoW (Must/Should/Could/Won't — for v1) and mapped to the feature list from the brief.

#### FR-1: Offline Mesh Messaging (Must)
- FR-1.1 Users within mesh range can exchange 1:1 and group text messages without internet.
- FR-1.2 Messages persist locally and are automatically retried/forwarded until delivered or TTL-expired.
- FR-1.3 Message delivery status is surfaced to the sender (Queued → Relaying → Delivered → Failed/Expired), acknowledging that "Delivered" in a mesh context means "received by a node that can reach the destination," detailed further in the routing spec.
- FR-1.4 Messages larger than a configured threshold (images, in v1.x) are chunked and reassembled across hops.

#### FR-2: SOS Broadcast (Must)
- FR-2.1 A single, unmistakable UI action broadcasts an SOS containing: GPS coordinates (if available) or last-known location, timestamp, device ID, optional voice note, optional free-text detail, and battery level of the sending device.
- FR-2.2 SOS messages are assigned the highest transport priority in the mesh (see Message Priority, Section 3.4) and are broadcast (not unicast) to all reachable neighbors, who are obligated to relay regardless of their own relay-battery policy thresholds (SOS overrides battery conservation, within safety limits — see Battery Optimization in the mesh doc).
- FR-2.3 SOS remains re-broadcasting at a decaying interval until acknowledged by a responder or the device's Cancel-SOS action is used (with are-you-sure friction specifically on *cancel*, not on send).

#### FR-3: Emergency Requests (Must)
- FR-3.1 Structured request types: Food, Water, Shelter, Medical, Volunteer, Road Block, Hazard (extensible enum, not free text, so it can be machine-aggregated on the dashboard).
- FR-3.2 Each request carries location, quantity/severity fields specific to its type, timestamp, and status (Open, Acknowledged, In Progress, Resolved).
- FR-3.3 Responders (P2/P3) can claim and update request status; status changes propagate back through the mesh to the original requester.

#### FR-4: Missing Person Broadcast (Must)
- FR-4.1 Users can post a missing-person record: name, description, photo (optional, size-capped for mesh transport), last-seen location/time, reporter contact.
- FR-4.2 Missing-person records propagate mesh-wide with lower priority than SOS/Medical but higher than normal chat, and persist/re-broadcast over a longer TTL than transient chat messages.
- FR-4.3 Records can be marked Found by any authenticated user, which propagates a status update.

#### FR-5: Community Bulletin Board (Should)
- FR-5.1 Location-scoped, append-only board for community-relevant posts (evacuation center status, water distribution schedule, general announcements) not tied to an individual emergency.
- FR-5.2 Posts from verified LGU/Government accounts are visually distinguished (badge) and cannot be spoofed by regular users (enforced by the signature/role verification described in Security Architecture).

#### FR-6: Medical Assistance Requests (Must)
- Modeled as a specialization of FR-3 with additional structured fields: symptom category, severity (self-reported triage color where applicable), number of patients, mobility status. Highest non-SOS priority.

#### FR-7: Disaster Information Sharing (Must)
- FR-7.1 Government/verified accounts can push structured alerts (typhoon signal changes, evacuation orders, volcanic alert level changes) that propagate through the mesh at Government-tier priority.
- FR-7.2 Alerts display prominently and persistently (not just in a chat thread) and are logged distinctly from user-generated content.

#### FR-8: Resource Requests (Should)
- Distinct from Emergency Requests in that these represent LGU/responder-side logistics asks (e.g., "need 50 more relief packs at Evac Center 3") rather than citizen-side needs — feeding the dashboard's resource allocation view.

#### FR-9: Automatic Synchronization (Must)
- FR-9.1 Any device transitioning from Tier 0/1 to Tier 2 automatically and silently syncs its outbound queue and pulls relevant updates, without requiring user action.
- FR-9.2 Sync is incremental and idempotent (safe to interrupt/resume), using the duplicate-detection mechanism defined in the mesh spec.
- FR-9.3 Sync prioritizes by the same priority ordering as mesh transport (SOS/Medical first).

#### FR-10: Government Monitoring Dashboard (Must)
- Live map of active SOS, requests, and mesh device density; detailed in Section 3.5 and the dedicated Dashboard document.

### 3.3 Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Mesh message hop latency | < 2s per hop under nominal BLE/Wi-Fi Direct conditions |
| NFR-2 | SOS end-to-end propagation | Reach a bridging/online node within a bounded time that scales with device density (modeled, not fixed — see Mesh doc §Simulation) |
| NFR-3 | Battery impact of background mesh service | Configurable power profiles; default profile must not exceed a defined %/hr drain budget, detailed in Battery Optimization |
| NFR-4 | Data at rest encryption | AES-256 on local Room DB via SQLCipher |
| NFR-5 | Data integrity | All messages signed (ECDSA); tamper detection at every relay hop |
| NFR-6 | Availability (backend) | 99.9% outside disaster windows; horizontally scalable during surge (disaster onset causes reconnection spikes) |
| NFR-7 | Offline durability | No message loss on app kill/device reboot; Room DB + WorkManager guarantee delivery attempts resume |
| NFR-8 | Accessibility | WCAG 2.1 AA equivalent for mobile; large-touch-target emergency mode |
| NFR-9 | Device support | Android 8.0 (API 26)+ to maximize reach on mid/low-tier devices common in LGU-target demographics |
| NFR-10 | Localization | Filipino + English at minimum, with architecture supporting additional regional languages (Cebuano, Ilocano, etc.) |

### 3.4 Message Priority Model (System-Wide Contract)

This ordering is authoritative and is referenced by the routing, queueing, and battery-optimization subsystems described in later documents:

1. **SOS** — life-critical, immediate
2. **Medical** — life-critical, time-sensitive
3. **Government Alert** — public-safety-critical, authoritative
4. **Missing Person** — high social value, moderate urgency
5. **Hazard** (e.g., Road Block, structural danger) — safety-relevant, area-wide value
6. **Relief** (Food, Water, Shelter, Volunteer, Resource Requests) — logistics-critical, not immediately life-threatening
7. **Normal Chat / Bulletin Board** — best-effort

This is not merely a UI sort order — it is the scheduling discipline the mesh engine's Priority Queue (detailed in the Mesh Networking Architecture document) uses to decide what gets airtime, what survives store-and-forward eviction under storage pressure, and what gets synced first when a bridge becomes available.

### 3.5 Government Dashboard — Requirements Summary

(Full design in the dedicated Admin Dashboard document; summarized here as part of the PRD contract.)

- Live map (Leaflet/OpenLayers) plotting SOS, requests, missing persons, and mesh device density by geography
- Mesh network health view: connected device counts, tier distribution (0/1/2), bridge node identification
- SOS heatmap with time-decay visualization
- Filterable/exportable request queues (Medical, Resource, Missing Person)
- Role-based access (National/Regional/Provincial/Municipal/Barangay scoping) — detail in Security Architecture
- Audit log viewer for after-action review

### 3.6 Out of Scope for v1 (Explicit)

- iOS application (platform BLE/background restrictions make true peer mesh infeasible without a companion protocol change; flagged for v2 investigation, possibly via a lightweight iOS "receiver-only" mode)
- Satellite/LoRa hardware integration (architecturally anticipated as a future relay-node type, not built in v1)
- Voice/video calling
- Public non-emergency social features beyond the Bulletin Board

### 3.7 Risks & Open Questions Carried Into Architecture

These are flagged now because they materially shape the technical design in subsequent documents, and are addressed explicitly there rather than glossed over:

- **R1 — Mesh scalability ceiling.** Nearby Connections / BLE mesh topologies have practical neighbor-count and throughput limits. The Mesh doc must define expected performance envelopes (e.g., dense urban barangay vs. sparse rural sitio) and degrade predictably beyond them.
- **R2 — Malicious/false SOS at scale.** Without a central authority in the loop at creation time, the system must rely on signed identity + post-hoc reputation/rate-limiting rather than pre-validation, and this trade-off is made explicit in Security Architecture.
- **R3 — Battery-vs-altruism tension.** Users may disable relay behavior to save battery for themselves, which is rational individually but degrades the network. The Battery Optimization design must address this with tiered relay obligations (e.g., SOS relay is never optional; general chat relay is throttled first).
- **R4 — Government verification/onboarding process** for LGU and NDRRMC accounts is an operational, not purely technical, workstream and is called out for the Deployment Guide.

---

**Next documents in this series** (to follow, each produced with the same level of depth rather than compressed into one response):

1. Mesh Networking Architecture (peer discovery, routing, store-and-forward, security)
2. Database Schema (PostgreSQL, normalized, with full DDL)
3. Android Application Architecture (Clean Architecture layering, module structure, screen-by-screen spec)
4. Backend Architecture (NestJS services, REST/WebSocket API contracts)
5. Admin Dashboard Design
6. Security Architecture (deep dive: key exchange, signing, replay protection)
7. AI Modules (relay prediction, congestion prediction, battery-aware routing)
8. Deployment Guide & Diagrams (architecture, ER, sequence, class, folder structure)

Say the word and I'll move to whichever one you want first — I'd suggest **Mesh Networking Architecture** next, since the database schema, Android module structure, and backend API all take their shape from the routing and message model defined there.
