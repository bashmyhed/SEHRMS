# Smart Emergency Healthcare Resource Management System

**Project Overview and Technical Report**

---

## 1. What This Project Is

An integrated platform for real-time emergency healthcare resource management. It brings hospitals, ambulances, blood banks, a dispatch centre, and the public onto one shared system so that emergency resources — beds, ICU capacity, ambulances, and blood — can be found, matched, and allocated quickly instead of through phone calls between disconnected organizations.

**The problem being solved:** In emergencies — road accidents, cardiac events, trauma, outbreaks — critical time is lost because:

- There is no single source of truth for bed and ICU availability across hospitals
- Ambulance dispatch is manual, local, and relationship-driven
- Blood bank inventory is siloed — each bank knows only its own stock
- Hospitals cannot see or borrow resources from neighbouring facilities quickly
- The public has no way to find the nearest hospital with an available ICU or the nearest blood bank with their group
- Nothing is recorded — no audit trail, no analytics, no basis for improvement

**The core idea:** Replace the phone-call protocol between silos with a shared, live, lockable, auditable platform. Every stakeholder sees the same state, recommendations are computed automatically, resources are locked against double-booking, every action is logged, and the public gets coarse but actionable visibility.

---

## 2. The Modules

The system is described module by module, each in its own report:

| Module | What it does |
|--------|--------------|
| **Universal Auth** | One centralized place for identity, login, roles, and permissions. Every other module checks with it before allowing anyone to see or change anything. |
| **Hospital** | Registers hospitals, tracks live bed and ICU availability and critical equipment, records every change, and serves this data — with role-based limits — to the rest of the system. |
| **Ambulance** | Registers ambulance units, tracks their status and location during active calls, and makes them dispatchable resources. |
| **Blood Bank** | Tracks blood stock by group across all banks, with expiry awareness, low-stock alerts, and a request-and-fulfilment workflow. |
| **Incident & Request Management** | The entry point for emergencies: records what happened, where, how urgent, and what resources are needed, and tracks the incident from report to resolution. |
| **Dispatch** | The orchestration layer: receives emergencies, matches resources across hospitals, ambulances, and blood banks, assigns them, and tracks the full response. |
| **Notification Service** | Delivers the right message to the right person at the right time — in-app, email, or SMS — with priority handling and delivery records. |
| **Public Emergency Interface** | The citizen-facing surface: report an emergency, find the nearest hospital with availability, find the nearest blood bank with your group. |
| **Map & Geospatial Layer** | The visual layer: our data — hospitals, ambulances, blood banks, emergencies — placed live on a shared base map, shown differently per role. |
| **Analytics** | Turns everything the system records into measured understanding: response times, demand patterns, capacity pressure, blood supply and demand. |

---

## 3. Feature Inventory

### 3.1 Core Features (the working system)

1. **Hospital bed and ICU tracking** — live counts of available beds by category (general, ICU, CCU, PICU, OT, isolation) plus critical equipment, updated by hospital staff and visible to authorized roles instantly.
2. **Ambulance dispatch and tracking** — registered units with live status and location during active calls, assignable from the dispatch view.
3. **Blood bank inventory** — stock by group with expiry tracking, low-stock alerts, and cross-bank visibility through requests.
4. **Incident registration** — emergencies enter from any channel (public form, dispatcher, hospital referral, crew report) with location, type, urgency, and resource requirements.
5. **Resource matching and allocation** — the system recommends suitable hospitals, ambulances, and blood banks by distance and availability; allocation is confirmed and the resource is locked against double-booking.
6. **Live updates everywhere** — connected users see changes as they happen, without refreshing: bed counts change, ambulances move, incidents appear.
7. **Role-based access** — every user sees only what their role allows, from full dispatcher view down to the public's coarse view.
8. **Notifications** — critical events reach the right people: assignment to a crew, incoming patient to a hospital, blood request to a bank, status to a reporter.
9. **Inter-hospital resource sharing** — formal request workflow between hospitals with status tracking and audit.
10. **Public resource finders** — nearest hospital with availability, nearest blood bank with the needed group, deliberately coarse to avoid public chaos.
11. **Audit trail** — every allocation, dispatch, inventory change, and request is logged with who, when, and before/after state.
12. **Analytics** — response times, occupancy trends, demand patterns, blood utilization, incident heatmaps.

### 3.2 Deliberate Scope Limits

Stated honestly in every module report: no patient records or personal health information anywhere, no replacement of hospital internal systems, no real hospital feed integration in this project, no AI-based dispatch (rule-based matching), no mass-casualty mode, no turn-by-turn navigation. Each is a documented future extension, not an oversight.

---

## 4. Preferred Tools and How They Are Used

The tool choices follow one principle: use mature, free, open-source tools that fit each job, and keep the architecture simple enough to build and demonstrate solo.

### 4.1 Backend

**Preferred: a Python web framework with async support (FastAPI).** Chosen because it is fast to develop with, handles many simultaneous connections naturally (which the live-update layer needs), and shares a language with the analytics and any future machine-learning work. A Go backend is the alternative if raw throughput ever becomes the priority; for this project's scale, Python is the pragmatic choice.

### 4.2 Data Storage

**Preferred: PostgreSQL.** All the system's records — hospitals, beds, ambulances, blood stock, incidents, dispatches, users, audit logs — are relational data with relationships and integrity rules, which is exactly what a relational database is for. Its geospatial extension answers the distance questions ("nearest hospital with an ICU within 5 km") directly in the database rather than in application code.

**An in-memory store (Redis) alongside it** handles the fast, short-lived work: spreading live updates to all connected users, rate-limiting public endpoints, and caching hot data.

### 4.3 Live Updates

**Preferred: persistent two-way connections (WebSockets) from the backend to every dashboard.** When a module's state changes — a bed is freed, an ambulance moves, an incident is reported — the change is pushed to every subscribed screen immediately. A one-way push (server-sent events) is the fallback for clients that cannot hold a two-way connection. This is what makes the system feel alive in a demo and what makes it operationally useful in practice.

### 4.4 Frontend and Dashboards

**Preferred: a component-based web frontend (React) with an open-source map library (Leaflet).** The dashboards — dispatcher command view, hospital dashboard, blood bank view, public finders — are interactive and live, which suits a component-based frontend. Leaflet renders the map layer on top of free OpenStreetMap tiles at no cost. A lighter server-rendered approach is the fallback if build time becomes tight.

### 4.5 Maps and Location

**Preferred: OpenStreetMap as the free base map, with our data layered on top.** We do not build a map; we place our markers on an existing one. A free geocoding service (Nominatim) converts addresses to coordinates. Distance and nearest-resource queries run in the database's geospatial engine. Everything in this layer is free and self-contained — no per-request map API costs.

### 4.6 Notifications

**Preferred: in-app and email first, SMS optional.** In-app alerts and email cover all critical flows at no cost. SMS is reserved for high-priority dispatch confirmations where the recipient may not have the app open, and is treated as an add-on because it costs per message. Push notifications become available if a mobile installable web app is built.

### 4.7 Packaging and Deployment

**Preferred: containerized deployment (Docker) with everything the system needs in one reproducible setup** — database, cache, backend, frontend. The demo can run on a single modest server, or be exposed for demonstration through a tunnel. This keeps the system runnable by anyone with one command, which matters both for evaluation and for honest reproducibility.

### 4.8 Testing

Standard automated tests for the matching logic, access rules, and state transitions — the parts where a bug would matter operationally — plus a scripted demonstration scenario that exercises the full flow from incident report to resolution.

---

## 5. Feasibility Analysis

### 5.1 Technical Feasibility — High

- Every technology in the stack is mature, well-documented, free, and open-source
- Live dashboards over persistent connections with a relational database and a cache are a proven, well-understood pattern
- All data is under our control — realistic mock hospitals, ambulances, blood banks, and incidents are generated for the demonstration; no external dependency is needed for the core loop
- The map layer is the only externally-dependent piece, and it uses free services throughout

**Main risks and how they are managed:** live-update complexity grows with scale, so the system stays single-process until proven otherwise; map accuracy depends on data quality, so mock facilities are seeded with plausible real-world locations; SMS costs money, so it stays optional.

### 5.2 Data Feasibility — Medium, Handled Honestly

Real hospital information systems are not accessible to a student project, and pretending otherwise would be dishonest. The system therefore runs on realistic self-reported and simulated data, and this is stated plainly in the documentation. The architecture does not depend on mock data — it would accept real feeds through adapters when real organizations onboard. The project is framed as a pilot platform ready for adoption, not as a deployed production system.

### 5.3 Time Feasibility — Achievable With Scope Discipline

The core system — hospital tracking, ambulance dispatch, incident flow, matching, live dashboards, map, role-based access — fits within a solo developer's final-year timeline. The extended features (blood bank workflow, inter-hospital sharing, notifications, analytics) are phased after the core, and the advanced ideas (prediction, routing, public map at full fidelity) are documented as future work. The dominant cost is the dashboard and map work, which is exactly why scope discipline matters.

### 5.4 Scope Risk — The Main Risk, Actively Managed

The project touches many domains: geospatial, real-time, inventory, dispatch, access control, notifications. The mitigation is a hard core feature set, everything else explicitly phased, and no mobile application — a responsive web dashboard covers ambulance crews without doubling the interface work.

---

## 6. Impact Assessment

### 6.1 Real-World Impact — High

- Emergency response time is largely a function of information availability. Cutting the time to find an available ICU bed, the right blood group, or the nearest ambulance has genuine life-saving potential
- Blood visibility prevents wasted trips and delays in trauma care
- Inter-hospital sharing addresses the "empty bed in Hospital B while Hospital A is full" problem
- The same architecture applies to city emergency command centres, disaster coordination, and regions where centralized dispatch does not exist

### 6.2 Academic Impact — High

- Integrates multiple domains in one coherent system: databases, real-time communication, geospatial queries, access control, notifications, analytics
- Fully demonstrable — a live map with moving ambulances, bed counts changing across two open windows, an allocation locking a resource, an inter-hospital request flowing through
- Measurable outcomes: resource discovery in seconds versus tens of minutes of phone calls; structural prevention of double-booking; complete audit coverage
- Honest scoping — clearly stating what is mock and what is real, what is built and what is future work — which reviewers respect

### 6.3 Limitations Acknowledged

- Simulated data, not live hospital feeds
- Rule-based matching, not AI-optimized dispatch
- No patient-level data and therefore no clinical analytics
- A coordination layer that complements, not replaces, emergency call infrastructure

Each limitation is documented in the relevant module report as a deliberate scope decision with a stated future path.

---

## 7. Phasing

**Phase 1 — Core demonstrator:** hospital bed/ICU tracking, ambulance registration and dispatch, incident registration, resource matching with locking, live dashboards, the map with hospital and ambulance layers, role-based access. Deliverable: a dispatcher sees available beds, dispatches an ambulance, and watches the response progress live.

**Phase 2 — Extended features:** blood bank module, inter-hospital resource sharing, notification flows, audit viewing, basic analytics.

**Phase 3 — Advanced, if time permits:** demand forecasting, route optimization, full public emergency map, load testing.

---

## 8. Documentation Index

| Report | File |
|--------|------|
| Project overview (this document) | README.md |
| Universal Auth Module | [auth_module_report.md](auth_module_report.md) |
| Hospital Module | [hospital_module_report.md](hospital_module_report.md) |
| Ambulance Module | [ambulance_module_report.md](ambulance_module_report.md) |
| Blood Bank Module | [blood_bank_module_report.md](blood_bank_module_report.md) |
| Incident & Request Management Module | [incident_module_report.md](incident_module_report.md) |
| Dispatch Module | [dispatch_module_report.md](dispatch_module_report.md) |
| Notification Service Module | [notification_service_module_report.md](notification_service_module_report.md) |
| Public Emergency Interface Module | [public_interface_module_report.md](public_interface_module_report.md) |
| Map & Geospatial Layer | [map_module_report.md](map_module_report.md) |
| Analytics Module | [analytics_module_report.md](analytics_module_report.md) |

Every module report follows the same structure: what the module does and what data it tracks, who gets access to what, how data is shared, real-time behaviour, gaps in the current system and how the module addresses them, integration with the other modules, privacy and safety considerations, and what is explicitly out of scope.