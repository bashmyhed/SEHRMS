# Map & Geospatial Layer — Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Map Layer is the visual, geographic heart of the system. It is where everything the other modules track becomes visible in space — where hospitals are, where ambulances are right now, where emergencies are happening, and where blood banks are, all on one map that the right people can look at.

Specifically, this layer is responsible for:

- Showing a base map of the region the system covers — streets, areas, landmarks
- Placing our own data on top of that base map — hospitals, ambulances, blood banks, and active emergencies, each as markers or overlays
- Keeping those placements live — when an ambulance moves, its marker moves; when a hospital's ICU fills up, its colour changes
- Answering distance questions — which hospitals are within a radius of an emergency, which is nearest, how far away
- Presenting all of this differently to different people, based on their role — the dispatcher sees the full operational picture, the public sees a deliberately limited one

The map is not a separate module with its own data. It is a presentation and query layer that reads from the hospital, ambulance, blood bank, and incident/dispatch modules and shows their data in geographic form.

---

## 2. Are We Making a Map?

This is worth answering directly, because it is a common confusion.

**We are not building a map from scratch.** Making a map — surveying roads, drawing streets, naming areas — is the work of organisations like OpenStreetMap, Google, and national survey bodies. It is enormous, already-done work, and we do not need to repeat it.

**What we are doing is using an existing free map as the background, and adding our own data on top of it.** The base map — the streets, the area names, the geography — comes from OpenStreetMap, which is free and open. On top of that base, we place our markers and overlays: hospital markers colour-coded by availability, ambulance markers that move with live location, incident markers where emergencies are reported, blood bank markers with stock status.

So the answer to "are we making a map?" is: we are making **our view of the map** — the operational picture that shows where our resources are and what state they are in — on top of a base map we did not have to build. This is exactly how real-world systems like ambulance trackers, delivery apps, and ride-hailing apps work: none of them draw their own streets; they all overlay their own live data on a shared base map.

---

## 3. Can We Just Add Our Data to the Map?

Yes — that is precisely the approach, and it is the right one. Here is what "adding our data to the map" means in practice, and why it works well for this project.

### 3.1 Every Resource Gets a Position

Each thing we track has a geographic position:

- **Hospitals** have a fixed position — their address, converted to a latitude and longitude when they are registered on the platform
- **Blood banks** have a fixed position, the same way
- **Ambulances** have a changing position — their current location, which updates as they move (only during active calls, as decided in the ambulance module)
- **Emergencies** have a position — where the emergency was reported, from the caller's device location or from an address they provide

### 3.2 Positions Come From Our Own Data

We do not need any external live data service to place our markers. The positions come from the data our own modules already hold:

- Hospital and blood bank positions are stored once, at registration, in our own database
- Ambulance positions come from the ambulance module's live location updates
- Emergency positions come from the incident module's recorded location

All of it is our data, placed on the shared base map. The base map provides the geography; we provide everything that matters operationally.

### 3.3 Distance Questions Are Answered From Our Data

"Which hospital is nearest to this emergency?" and "which blood banks are within five kilometres?" are answered by comparing positions we already store. The system can rank hospitals and blood banks by distance from an emergency, filter them by what they have available, and show the results on the map or as a ranked list. This is the geographic engine behind the dispatch module's matching recommendations.

### 3.4 Mock Data for the Demo

In this project, the hospitals, blood banks, and ambulances are mock entities with positions we choose — either realistic locations in a chosen city, or plausible invented ones. The map layer works exactly the same way with mock data as it would with real data: positions in, markers out. This is what makes the demo convincing — a live map with moving ambulances and colour-changing hospitals — without depending on any real hospital or tracking service.

---

## 4. What the Map Shows

### 4.1 The Layers

The map is organised in layers, and each layer can be turned on or off depending on who is looking:

- **Hospital layer** — one marker per hospital, colour-coded by availability. For example, green for "has availability in the category you care about," amber for "running low," red for "full." Clicking a marker shows what that role is allowed to see — coarse status for the public, full category counts for a dispatcher.
- **Ambulance layer** — one marker per ambulance that is currently active, at its live position. Markers can show status — en-route, on-scene, transporting. This layer is restricted; it is not shown to the public, as decided in the ambulance module.
- **Blood bank layer** — one marker per blood bank, colour-coded by whether it has the group being asked about in stock. Clicking shows coarse availability for the public, fuller detail for hospital staff with an active request.
- **Incident layer** — markers for active emergencies. This layer is for the dispatch desk only. The public never sees other people's emergencies on a map.
- **Request origin layer** — when a dispatcher is matching an emergency to resources, the map can show the emergency's position and the candidate hospitals/ambulances/blood banks around it, with distance lines or ranked highlights. This is the visual form of the matching recommendation.

### 4.2 Live Behaviour

The map is not a static picture. It updates as the system's state changes:

- When a hospital's bed count changes, its marker colour can change — from green to amber to red — without anyone refreshing the page
- When an ambulance moves during an active call, its marker moves
- When a new emergency is reported, it appears on the dispatch desk's map immediately
- When a resource is locked for an emergency, the map can show that state — the hospital marker marked as "has a locked bed for an incoming patient," visible to the roles allowed to see it

This live behaviour is what makes the map a working operational tool rather than a decorative picture.

---

## 5. Who Gets Access to the Map and for What

This is the most important section, because the map is where the "don't dox hospitals, don't cause public chaos" principle is applied most visibly. Different roles see different maps — not just different detail, but genuinely different layers.

### 5.1 Dispatcher — The Full Operational Map

The dispatcher sees everything, because coordination requires it:

- All hospitals, colour-coded by live availability, with full category counts on click
- All active ambulances, at their live positions, with status
- All active emergencies, at their locations
- All blood banks, with stock status
- When matching an emergency: the emergency position, candidate resources around it, ranked by distance and suitability

This is the command view. It is the single shared picture the dispatch desk works from. Every marker on it is actionable — clicking can lead to assigning an ambulance, choosing a hospital, or placing a blood request.

**Purpose:** coordinate responses with the complete live picture.

### 5.2 Platform Administrator — The Oversight Map

The platform administrator sees the same full map as the dispatcher, but for a different purpose: oversight and review, not day-to-day coordination. They can look at the live picture, review how coverage is distributed across the region, and see where capacity pressure is building.

**Purpose:** oversight, review, and planning context.

### 5.3 Hospital Staff — Their Own Facility and Incoming Traffic

Hospital staff see a limited map:

- Their own hospital's position and its own availability — their own facility, which they already know
- Incoming ambulances that are headed to their hospital — so they can see what is coming and prepare
- Optionally, a coarse view of the region — hospital markers without exact counts — so they understand the general area context

They do not see other hospitals' exact availability, other ambulances' positions, or emergencies that are not coming to them. This matches the hospital module's rule: hospitals do not browse each other's live inventory.

**Purpose:** prepare for incoming patients and see their own facility's state in context.

### 5.4 Ambulance Crew — Their Own Assignment

An ambulance crew sees a minimal map focused on their own work:

- Their own unit's position
- The emergency location they are assigned to
- The destination hospital they are headed to

They do not see the full operational picture — other emergencies, other ambulances' assignments, or hospitals beyond their destination. They see what they need to do their job: where to go, and where to take the patient.

**Purpose:** navigate their own assignment.

### 5.5 Blood Bank Staff — Their Own Bank and Requests

Blood bank staff see their own bank's position and stock status. When there is an active request involving them, they see the requesting hospital's position in the context of that request. They do not see the operational map.

**Purpose:** see their own bank and fulfil requests.

### 5.6 Public — The Deliberately Limited Map

The public sees a map, but a carefully limited one. This is the citizen-facing map from the public interface module:

- **Nearest hospital finder:** the citizen's location (or a location they enter), with nearby hospitals shown as markers. Each marker shows the hospital's name, type, distance, and a **coarse availability indicator** — "available," "low," or "full" — never exact bed counts. The citizen can filter by what they need — "ICU," "general bed," "maternity."
- **Nearest blood bank finder:** nearby blood banks with a coarse indicator of whether the asked-for blood group is in stock — "in stock" or "low/out" — never exact unit counts.
- **Static information:** emergency numbers and basic guidance.

What the public **never** sees:

- Live ambulance positions. Publishing real-time ambulance locations to the public is a tracking and safety risk — it reveals where emergencies are happening, and it could be misused. Ambulances are not shown on the public map at all.
- Active emergencies. One person's emergency is another person's crisis; publishing emergency locations would cause panic, invite crowds, and violate the privacy of the people involved.
- Exact counts. "Hospital X has 3 ICU beds left" published to the public can cause a rush to that hospital, or public alarm when it hits zero. Coarse indicators give citizens actionable information without that risk.
- Other hospitals' internal detail. Contacts, staff, internal operations — none of it.

**Purpose:** help a citizen find the nearest place with the resource they need, with enough information to act and not enough to cause chaos.

### 5.7 Summary Table

| Role | Sees on the map | Purpose |
|------|----------------|---------|
| Dispatcher | Everything — all hospitals with full counts, all active ambulances, all emergencies, all blood banks, matching visualisation | Coordinate responses |
| Platform administrator | Everything, same as dispatcher | Oversight and planning |
| Hospital staff | Their own facility, incoming ambulances headed to them, coarse regional context | Prepare for incoming patients |
| Ambulance crew | Their own unit, their assigned emergency, their destination hospital | Navigate their assignment |
| Blood bank staff | Their own bank, requesting hospital in the context of an active request | Fulfil blood requests |
| Public | Nearby hospitals with coarse availability, nearby blood banks with coarse stock status, emergency guidance. No ambulances, no emergencies, no exact counts | Find the nearest resource they need |

---

## 6. How the Map Shares Data

### 6.1 It Reads From the Operational Modules

The map layer reads positions and statuses from the modules that own them:

- Hospital positions and availability from the hospital module
- Ambulance positions and statuses from the ambulance module (live positions only during active calls, per that module's decision)
- Blood bank positions and stock from the blood bank module
- Emergency locations from the incident module

The map owns none of this data. If a module's data changes, the map reflects it. If the map layer fails, the operational modules keep working — the data is still tracked; it is just not being shown geographically.

### 6.2 It Applies the Same Access Rules

The map does not invent its own access rules. It applies the same role-based scoping every other module uses, through the universal auth module. The dispatcher's map is full because the dispatcher role has full operational read access. The public map is coarse because the public role has coarse read access. The map is simply the visual form of access decisions made elsewhere — which is exactly why it is safe: it cannot show what the underlying access rules do not allow.

### 6.3 Distance and Matching Support

The map layer supports the dispatch module's matching by answering geographic questions: given an emergency position, which hospitals with the needed resources are within range, ranked by distance? This is computed from the positions the modules store. The dispatch module uses the answer to build recommendations; the map shows those recommendations visually.

---

## 7. Gaps in the Current System and How This Layer Changes That

### 7.1 What Exists Today

- **No shared geographic picture.** When an emergency happens today, there is no map anyone can look at to see where resources are. A dispatcher's mental map is built from memory and phone calls.
- **No nearest-available answer.** "Which hospital with an ICU bed is nearest to this accident?" has no fast answer. It is found by calling hospitals one by one.
- **No public way to find resources.** A citizen needing an ICU or a blood bank has no map-based way to find the nearest one with availability. They call around or rush to the nearest hospital and hope.
- **No visual coordination.** Multi-agency responses — police, ambulances, hospitals — have no shared visual picture of where everything is.

### 7.2 What This Layer Introduces

| Current gap | What this layer introduces |
|-------------|---------------------------|
| No shared geographic picture | One live map with all resources placed and colour-coded, shared by everyone at the dispatch desk |
| No nearest-available answer | Distance-ranked matching — the nearest hospital with the needed resource, computed in seconds |
| No public resource finder | A public map where a citizen can find the nearest hospital with coarse availability and the nearest blood bank with their group |
| No visual coordination | A visual operational picture — emergencies, ambulances, hospitals, and blood banks in one view, live |

### 7.3 How It Is Better

- **For the dispatcher:** the geographic picture is instant and complete. Instead of remembering where resources are, they see them — and the system ranks candidates for them.
- **For the citizen:** the nearest-resource question has a real answer, available immediately, without calling anyone.
- **For everyone coordinating:** a shared visual picture replaces fragmented mental maps. When two dispatchers look at the same map, they are literally looking at the same situation.

---

## 8. How This Layer Integrates With the Rest of the System

- **It depends on the auth module** for role-based access, like everything else. What appears on a user's map is determined by their role.
- **It reads from the hospital, ambulance, blood bank, and incident modules** for positions and live status.
- **It supports the dispatch module's matching** with distance and radius queries.
- **It presents through the public interface** for the citizen-facing finders.
- **It updates live** through the same real-time channel the other modules use for updates — when a module's state changes, the map reflects it without a refresh.

---

## 9. Privacy and Safety Considerations

### 9.1 The Public Map Is Deliberately Coarse

This is the core privacy decision for the map, and it deserves to be stated plainly: **the public map never shows exact counts, never shows ambulances, and never shows emergencies.** It shows hospitals and blood banks with coarse availability indicators — enough for a citizen to act, not enough to cause a rush, panic, or misuse. This is the map-level expression of the "we do not dox hospitals or cause public chaos" principle.

### 9.2 Ambulance Positions Are Restricted

Live ambulance positions are visible only to operational roles — the dispatcher, the platform administrator, and the crew's own unit view. They are never public. Real-time ambulance tracking visible to anyone would reveal where emergencies are happening across the city, which is both a privacy violation for the people involved and a safety risk.

### 9.3 Emergency Locations Are Restricted

Emergency markers appear only on the dispatch desk's map. They are never public, never visible to roles without operational need. An emergency's location is sensitive — it is where someone is in crisis.

### 9.4 Hospitals Are Not Compared Publicly

The map never presents a public ranking of hospitals by availability that could shame or pressure a hospital. The public finder shows nearby options with coarse status — a helpful list, not a league table. Per-hospital comparisons stay internal, and even internally, each hospital sees only its own detail.

---

## 10. What We Are Explicitly Not Doing

- **Building our own base map.** We use OpenStreetMap's free map as the background. We do not draw streets or maintain geography.
- **Turn-by-turn navigation.** The map shows positions and distances, but it does not provide driving directions or live routing for ambulance crews. That is a separate capability; in this project, positions and nearest-matching are the scope.
- **Live public ambulance tracking.** Deliberately excluded for privacy and safety, as above.
- **Traffic-aware routing or live traffic data.** Distance matching in this project is geographic distance, not travel-time-through-traffic. Travel-time routing is a natural future extension.
- **Indoor mapping.** The map shows facility locations, not ward-level or floor-level layouts inside a hospital.
- **Offline maps.** The map requires a connection. Offline capability is out of scope for this project.

---

## 11. Summary

| Aspect | What we are doing |
|--------|------------------|
| Are we making a map? | Not from scratch. We use OpenStreetMap's free base map as the background and build our operational picture on top of it — this is how all real-world tracking and dispatch systems work |
| Can we add our data to the map? | Yes — that is exactly the approach. Hospitals, blood banks, ambulances, and emergencies all have positions from our own data, and they are placed as live, colour-coded markers on the shared base map |
| Who gets access | Dispatcher and platform admin: the full operational map for coordination and oversight. Hospital staff: their own facility and incoming ambulances. Ambulance crew: their own assignment. Blood bank staff: their own bank. Public: nearby hospitals and blood banks with coarse availability only — no ambulances, no emergencies, no exact counts |
| What it does not do | No base map building, no turn-by-turn navigation, no public ambulance tracking, no traffic routing, no indoor maps, no offline maps |
| Privacy stance | The public map is deliberately coarse; ambulance and emergency positions are restricted to operational roles; hospitals are never publicly compared or ranked; the map applies the same role-based access as every other module, so it cannot show what access rules do not allow |

---

