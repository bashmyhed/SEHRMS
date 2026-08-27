# Hospital Module - Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft - subject to revision

---

## 1. Purpose and Scope

The Hospital Module is the core data source for the entire platform. It is where the system keeps track of what every hospital has and does not have right now - beds, ICU capacity, equipment, and contact people - and makes that information available to the rest of the system in a controlled way.

Specifically, this module is responsible for:

- Registering hospitals on the platform, along with their organizational identity (which organization they belong to, what type of hospital they are, where they are located, how to contact them)
- Tracking bed inventory in real time - general beds, ICU, CCU, PICU, operation theatres, isolation wards, and any specialty units the hospital chooses to expose
- Tracking critical equipment counts - ventilators, oxygen supply, dialysis machines, defibrillators - as resources that matter just as much as beds during an emergency
- Recording every change to bed or equipment counts as an auditable event, so there is always a record of who changed what and when
- Serving this data to the dispatch centre, ambulance crews, other hospitals (when they need to share resources), and the public map (in a limited, coarse form)
- Receiving and acting on resource requests from other hospitals - for example, when Hospital A needs ICU beds and asks Hospital B if it can spare any

This is the module where the system's "single source of truth" for hospital capacity lives. Everything downstream - dispatch decisions, matching recommendations, what the public map shows - reads from here.

---

## 2. What Data We Track

### 2.1 Hospital Identity

For each hospital registered on the platform, we store:

- The hospital's name and the organization it belongs to (a hospital group may have multiple facilities; a single hospital is also fine)
- What type of hospital it is - general, trauma centre, cardiac, maternity, pediatric, multi-specialty, and so on
- Its address, city, state, and pincode
- Its location as latitude and longitude, so it can appear on the map and so the system can compute distances from incident locations or from other hospitals
- Its main phone number and alternate phone number
- Its email address
- The name and phone number of the emergency contact - the duty medical officer or administrator who is reachable when something urgent happens
- Whether it operates 24 hours a day, whether its ICU is open 24 hours, and any other operating-hour details
- Whether it is NABH-accredited, and if so, when that accreditation expires
- Its declared total capacity in each bed category - how many general beds, ICU beds, CCU beds, PICU beds, operation theatres, and isolation beds it says it has in total. These are reference numbers set by the hospital, not live counts - the live counts are tracked separately

We also store a few operational notes the hospital may want to add, and standard created/updated timestamps and an active/inactive flag.

We do not store patient information, financial data, or anything beyond what is needed for emergency resource coordination and contact.

### 2.2 Bed Categories

The system defines a set of bed types that hospitals can track. Examples include general beds, ICU, CCU, PICU, NICU, operation theatres, isolation wards, and any other specialty unit a hospital wants to declare. Each category has a name, a description, and a flag indicating whether it is an intensive-care type.

Having a defined set of categories means the system can treat all hospitals consistently - "ICU" means the same thing across all hospitals - and can show the right things on the dashboard and the map. A hospital does not have to track every category; it only tracks the ones that apply to it.

### 2.3 Live Bed Inventory

This is the heart of the module. For each hospital and each bed category it tracks, we store a live snapshot:

- The total number of beds of that type the hospital has (the reference number the hospital set)
- The number currently occupied
- The number currently available (derived from total minus occupied)
- Who last updated this count and when
- A version number that goes up every time the count is updated - this is how the system prevents two people from overwriting each other's changes at the same time
- Whether this inventory is currently locked - meaning a resource has been allocated or reserved for a specific incident, and the bed should not be offered to anyone else until the lock is released
- If locked, the reason for the lock (which incident it is for) and, optionally, when the lock expires

The "available" number is what everyone else in the system sees - the dispatcher, other hospitals, and (in coarse form) the public. When a patient is admitted, the occupied count goes up and available goes down. When a patient is discharged or transferred out, occupied goes down and available goes up. These changes happen immediately and are broadcast to anyone who is watching.

### 2.4 Bed Status Change Log

Every change to a hospital's bed inventory is recorded in a log that is never edited or deleted. Each entry records:

- Which hospital and which bed category changed
- What kind of change it was - a patient admitted, a patient discharged, a transfer in, a transfer out, a correction, a lock placed, a lock removed, or a capacity update
- The count before the change and the count after
- The reason for the change, in free text - for example, "road accident admission, 24-year-old male, blunt trauma"
- If the change is part of an incident workflow, which incident it is linked to
- Who made the change (which user)
- When the change was made
- Whether the change was made manually by a person, through the system automatically, or came from an import or API

This log is append-only. It is the audit trail for the hospital module. It is what you look at when you need to know who changed a bed count, when, and why. It also feeds the analytics - occupancy trends, admission and discharge rates, and so on.

### 2.5 Equipment Inventory

Beds are not the only thing that matters during an emergency. A hospital may have ICU beds available but no ventilators to put a patient on. This module also tracks equipment counts in the same way it tracks bed counts:

- Ventilators
- Oxygen supply (tanks or pipeline capacity)
- Dialysis machines
- Defibrillators
- And any other equipment category the hospital wants to declare

For each equipment type, the hospital sets a total and a live available count. Equipment counts are updated in the same way as bed counts, with the same locking and version mechanism, because the matching engine treats both beds and equipment as resources that can be allocated.

### 2.6 Hospital Contacts

For each hospital, we store a list of contacts - people who can be reached for specific purposes:

- A duty medical officer
- The ICU in-charge
- The administrator
- The bed manager
- Any other relevant contact

Each contact has a name, a role, a phone number, and an email. This matters because when the dispatch centre needs to talk to a hospital about an urgent admission, it helps to know exactly who to call rather than dialling a general number and hoping.

A hospital can have multiple contacts, and it can mark some as primary. Contacts are not shared publicly - they are visible to the hospital's own staff and to the dispatcher, who may need them for operational coordination, but not to other hospitals or to the public.

### 2.7 Blocked Time Slots (optional)

A hospital may occasionally have a period during which some of its capacity is not available - for example, ICU maintenance that takes four beds offline for a few hours, or a planned closure. This module can record those blocked periods: which hospital, which bed category (or the whole hospital if null), when the block starts and ends, and the reason.

This is an optional feature. For the initial version, the main point is that the data structure exists so that, if needed, the matching engine can avoid offering beds that are temporarily blocked. For now, it is fine to include this in the design without building the full matching logic around it immediately.

---

## 3. How the Data Is Stored and Organized

### 3.1 One Shared Platform Store

All hospital data - profiles, bed inventory, equipment inventory, contacts, change logs - lives in one shared data store managed by the platform. Each hospital's data is kept separate from every other hospital's data by tagging it with the organization it belongs to, and access is controlled so that a hospital's staff can only see their own hospital's data (unless they also have a platform-wide role like dispatcher).

This means:

- The dispatcher can see every hospital's data in one place, without having to connect to multiple separate systems
- Inter-hospital resource sharing is straightforward - Hospital A can be shown Hospital B's availability through the platform, with the right access controls
- The public map can show a coarse picture of all hospitals without needing permission from each one individually (because the coarse data is what the platform chooses to expose, not raw hospital data)
- Mock data for the demo is easy to set up - one set of realistic hospitals, one data store, everything consistent

### 3.2 Why Not Keep Each Hospital's Data Separate

In a production deployment, some hospitals might want to keep their data entirely within their own systems and only share it through standardized feeds. That is a reasonable production requirement, and the report acknowledges it as the future path. But for this project, having all the data in one place is the right choice because:

- It is what makes the system demonstrable - the dashboard, the map, the dispatch view all work from one consistent data source
- It is what makes inter-hospital sharing real rather than theoretical
- It matches the reality that, right now, there is no connected hospital data network to integrate with - the platform is the first system that brings this data together
- The alternative - each hospital running its own database and the platform federating across all of them - adds a huge amount of complexity for no benefit at this stage

The project is honest about this: the platform holds the data for now, and in a production rollout, hospitals would eventually own their own data and push it to the platform through standardized feeds. That is a future step, not a current limitation.

---

## 4. Who Gets Access to What

### 4.1 The Access Model in Plain Terms

Access to hospital data is controlled along two lines:

First, **platform-wide versus organization-scoped.** Some roles can see everything across all hospitals - the dispatcher and the platform administrator. Other roles - hospital administrators, bed managers, duty officers - can only see the hospitals that belong to their own organization.

Second, **what the role is allowed to do.** Within their scope, each role has specific permissions. A hospital administrator can change the hospital's profile and set bed capacities. A bed manager can update bed and equipment counts. A duty officer may only be able to view the dashboard. A read-only viewer can see everything but change nothing.

### 4.2 Hospital Administrator

A hospital administrator can do the following for their own organization's hospitals:

- Register and update the hospital's profile - name, address, type, contact details, operating hours, accreditation, and declared bed capacities
- Manage the list of contacts for the hospital
- View the full bed and equipment inventory dashboard for their hospitals
- Accept or decline resource requests that come from other hospitals
- View the change history for their hospitals
- Export reports for their own hospitals
- Manage staff accounts within their organization - create user accounts, assign roles, deactivate accounts

A hospital administrator cannot see other hospitals' inventory or profiles. They can only see other hospitals through the matching recommendations that appear when they request a resource - for example, "Hospital B, 3 kilometres away, has ICU availability" - without seeing Hospital B's full data.

### 4.3 Bed Manager / Duty Nurse

A bed manager or duty nurse, within their own hospital, can:

- View the current bed and equipment inventory
- Update the occupied count when a patient is admitted or discharged, or when a bed changes status
- View the recent change history for their hospital
- See active locks on their inventory, so they know when a bed has been reserved for an incoming patient

They cannot change the hospital's profile, manage contacts, or manage user accounts. Their access is scoped to operational inventory updates, not administrative changes.

### 4.4 Read-Only Viewer

A read-only viewer - for example, a medical superintendent or an auditor who needs to see what is happening but should not be able to change bed counts - can view the hospital's dashboard and data but cannot make any changes. This is a deliberate role, because in real hospitals there are people who need visibility without edit access.

### 4.5 Dispatcher

The dispatcher can see everything operationally relevant across all hospitals:

- All hospitals on the map, with their inventory counts
- All active locks on all hospitals' inventory
- All inter-hospital requests, both inbound and outbound
- All ambulances and blood banks (these come from other modules, but the dispatcher sees them alongside hospital data)

The dispatcher can allocate or lock resources - assign a bed, reserve an ICU bed for an incoming patient, mark a resource as committed - but cannot change a hospital's profile or set its declared bed capacities. The dispatcher's access is operational, not administrative. The dispatcher is the person who needs the full picture to make coordination decisions, and the system gives it to them, but the dispatcher cannot rewrite hospital records.

### 4.6 Platform Administrator

The platform administrator can see everything and manage everything at the system level:

- Register new hospitals on the platform or deactivate hospitals that should no longer be on it
- View all dashboards for oversight
- Manage platform-level users and roles
- View the full audit log across all hospitals
- Configure system-wide settings

This is the highest level of access and is reserved for the people who run the platform itself.

### 4.7 Public / Citizen

The public sees a deliberately limited view. On the public map, a citizen can see:

- Hospital names, types, and locations
- The distance from their location
- A coarse availability indicator for the resource they are looking for - for example, "has ICU availability," "ICU capacity is low," or "ICU is full" - not exact numbers

The public does not see:

- Exact bed counts
- Internal contacts or phone numbers (though a hospital may choose to expose a general emergency number)
- Any administrative data
- Any data about other hospitals beyond the coarse availability indicator

This limitation is deliberate. The goal is to give a citizen enough information to go to the right place during an emergency, without exposing data that could be misused or that could cause hospitals to be overwhelmed by people checking real-time availability and then flooding them.

---

## 5. How Data Is Shared Between Hospitals and With the Rest of the System

### 5.1 What a Hospital Shares With the Platform

When a hospital joins the platform, it shares:

- Its identity - name, type, address, location, contact details
- Its declared bed capacities by category - how many beds of each type it has in total
- Its live bed and equipment inventory - how many are occupied and available right now, updated as things change
- Its contacts - who to call for what
- Its resource requests to and from other hospitals - when it asks for or offers resources

This data is stored in the platform's shared data store and is visible to the hospital's own staff and, in controlled form, to the dispatcher and (for identity and coarse availability) to the public.

### 5.2 What a Hospital Sees of Other Hospitals

A hospital does not browse other hospitals' raw data. If Hospital A needs ICU beds, it does not open a view of Hospital B's full inventory. Instead:

- Hospital A creates a resource request: "We need two ICU beds, within 30 kilometres, within the next hour."
- The system looks across all hospitals within range, finds those that have the required availability, ranks them by distance and suitability, and returns a list of recommendations.
- Hospital A sees: "Hospital B, 3.2 kilometres away, has 4 ICU beds available - recommended." It does not see Hospital B's full inventory, its contacts, or its other bed categories.
- If Hospital A accepts the recommendation, the system locks the relevant beds on Hospital B's side so no one else can take them, and the request moves into an active allocation.

This is the key boundary: hospitals see recommendations, not raw data about each other. Exact counts are shared only in the context of a formal request - and even then, only the count relevant to the request, not everything.

### 5.3 When a Hospital Needs Another Hospital's Exact Count

There are situations where a hospital genuinely needs to know another hospital's exact availability - for example, when a medical team is deciding whether to transfer a patient and needs to confirm that the receiving hospital can take them. In the current design, this happens through one of two paths:

- The receiving hospital's administrator calls the sending hospital's administrator directly, using the contact information in the system. This is the same phone-call step that happens today, but now the system has the right contact number ready instead of requiring the caller to find it.
- Or, after a formal request has been made, the system shows the relevant availability information to the requesting hospital in the context of that request - an explicit, consent-gated view, not a general browse of the other hospital's data.

For the purposes of this project, the important thing is to document and respect this boundary, not to pretend that every hospital can see every other hospital's raw data at any time.

### 5.4 What the Dispatcher Sees

The dispatcher is the one role that sees everything operationally relevant across all hospitals. This is intentional: the dispatcher's job is to coordinate across the whole system, and they cannot do that with partial information. The dispatcher sees every hospital's inventory, every ambulance, every blood bank, and every incident, all on one live view.

What the dispatcher does not see:

- Hospital administrative settings that are not operationally relevant - the dispatcher does not need to know which staff member has which internal role; they only need to know that someone at the hospital can update the bed count when needed
- Anything patient-related - patient data is not in scope at all
- User account management - that is for the platform administrator

The dispatcher's full access is auditable. Every allocation, every lock, every dispatch action is logged with the dispatcher's identity, the timestamp, and the details. So the trust the system places in the dispatcher is accountable - if anything is misused, there is a record.

### 5.5 What the Public Sees

The public sees the least, by design. The public map shows coarse availability indicators - "has ICU availability," "ICU capacity low," "ICU full" - not exact numbers. The public can also find the nearest blood bank with a requested blood group, again in coarse terms.

This is a privacy and safety decision, not an accidental limitation. The system deliberately does not give the public endpoint exact counts to return, so even if someone calls it, they get only what the system intends to share. The reason is straightforward: if every hospital's exact bed count were public, it could lead to people targeting hospitals that show as available, overwhelming them during a crisis, or misusing the information in other ways. Coarse availability gives citizens useful guidance without creating that risk.

### 5.6 What Is Never Shared

- Patient information of any kind - not in scope at all
- One hospital's exact inventory or internal contacts to another hospital without a formal request context
- A hospital's audit log or change history to anyone outside that hospital (except platform administrators, who see it for oversight)
- Exact availability data to the public - only coarse indicators

---

## 6. How Real-Time Updates Work

### 6.1 The Basic Idea

When someone updates a hospital's bed or equipment count - for example, a nurse marks an ICU bed as occupied after admitting a patient - the change is saved, logged, and immediately broadcast to anyone who is watching that hospital's data. The dispatcher sees the count drop on the map. The hospital's own dashboard refreshes. Other connected clients see the update without having to reload the page.

This is what makes the system feel live. A dispatcher watching the map sees beds become available or fill up in real time, rather than working from a snapshot that was accurate five minutes ago.

### 6.2 Who Gets Which Updates

- **Hospital staff and administrators** for a given hospital see every update to that hospital's inventory as it happens - they are the ones making the changes, so they need to see the result immediately.
- **The dispatcher** sees updates across all hospitals, because the dispatcher needs the full picture to make coordination decisions.
- **Other hospitals** do not get every raw update pushed to them - they only see the coarse picture that affects them, such as when a recommendation they previously received changes because availability shifted.
- **The public map** gets a throttled, debounced update - changes are batched for a few seconds and the coarse status is recomputed and pushed once, rather than pushing every single change. This prevents the public map from being flooded with updates and reinforces the boundary that the public does not see exact counts.

### 6.3 Handling Two People Updating at the Same Time

If two staff members try to update the same bed count at the same time - for example, two nurses admitting patients to the same ICU ward at once - the system does not let one overwrite the other. Each update carries a version number. The first update succeeds and increments the version. The second update, if it was based on the old version, is rejected and the user is told to refresh and try again.

This is a standard way to handle concurrent updates, and it matters here because hospital workflows can have genuine overlap - two admissions happening at the same time, two discharges, and so on. The system must not silently lose one of them.

---

## 7. Gaps in the Current System and How This Module Changes That

### 7.1 What Exists Today

Today, across most cities and regions, the way hospital capacity is tracked and shared is fragmented and manual:

- **Each hospital knows its own occupancy internally**, often in a ward register, a nurse's log, or a basic hospital information system. No other hospital, no ambulance crew, and no dispatch centre can see it without calling. A dispatcher or a relative calls multiple hospitals one by one asking whether they have an ICU bed - a process that takes 10 to 30 minutes and burns phone lines during a crisis.
- **Ambulance dispatch is local and manual.** A hospital's emergency desk or a police control room calls an ambulance operator they know. No one has a shared view of which ambulances are where, which are already assigned, or which is closest. The nearest available ambulance is found by whoever has the most phone numbers memorized.
- **Blood bank inventory is siloed.** Each blood bank knows its own stock. A hospital needing blood calls known blood banks one by one. There is no cross-blood-bank visibility - Bank A may be out of O-negative while Bank B two kilometres away has eight units, and no one knows.
- **Inter-hospital resource sharing is ad hoc and relationship-driven.** Hospital A calls a contact at Hospital B and asks for an ICU bed or blood. If the contact is available and responsive and the resource is free, the transfer happens. If not, the patient waits or is redirected. There is no formal request tracking, no status visibility, and no audit trail.
- **The public has no visibility.** A citizen in an emergency has no way to find the nearest hospital with an available ICU or the nearest blood bank with their blood group. They call 108 or 112 or rush to the nearest hospital and hope for the best.
- **There is no audit trail.** If a resource was double-booked or a dispatch was delayed, there is usually no system record to investigate. You get oral accounts: "I think the duty nurse updated it," "I think the dispatcher called and said it was available." No authoritative record.
- **Each hospital's data is trapped inside that hospital.** The region has no aggregate picture of capacity, demand, or bottlenecks.

This is not theoretical. It is the operational reality in most cities, including in India, even at reasonably well-equipped hospitals. Individual hospitals may have good internal systems, but the coordination layer between them is missing or manual.

### 7.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| Bed and ICU availability is invisible outside each hospital | A shared, live inventory view - every connected hospital, the dispatcher, and (coarsely) the public can see availability in real time, without calling |
| Ambulance dispatch is manual and local | An ambulance registry with live location and status, dispatchable from a unified command view, with the nearest-available logic automated (this is the ambulance module working with this module's data) |
| Blood bank inventory is siloed | A cross-blood-bank inventory view - any hospital can see which blood banks have the group they need, within a radius, without sequential phone calls (this is the blood bank module working with this module's coordination layer) |
| Inter-hospital sharing is ad hoc, relationship-dependent, untracked | A formal inter-hospital request workflow - request, accept or decline, track, audit. The transfer has a status from start to finish, visible to both hospitals and the platform |
| No single source of truth - everyone has a different picture | A single shared data store, logically isolated by organization and permission-controlled, that all stakeholders read from and write to, so everyone sees the same state |
| The public has no visibility into nearest available resources | A public, read-only emergency map showing nearest hospitals with the resource a citizen needs, and nearest blood banks with their blood group |
| No audit trail for allocations, dispatch, or transfers | An append-only audit log - every inventory change, every allocation, every dispatch, every inter-hospital request is logged with actor, timestamp, and before-and-after state, exportable for oversight |
| No aggregate regional picture | Analytics - occupancy trends, demand heatmaps, blood utilization, ambulance utilization, response-time metrics. The region's emergency capacity becomes visible and measurable for the first time |
| No resource locking - double-booking is a phone-call collision | Locking on inventory rows - when a dispatcher allocates a bed or an ambulance, the row is locked so no one else can allocate it, with a reason and an optional expiry. Double-booking becomes structurally impossible, not luck-dependent |
| No coarse public availability indicator | The public map deliberately shows coarse status - available, low, or full - rather than exact counts, a privacy and safety decision that still gives citizens actionable information |

### 7.3 How It Is Better - The Concrete Improvements

For the patient or citizen:

- **Faster resource discovery.** Instead of calling hospital after hospital, a dispatcher - or a citizen using the public map - sees available ICUs and blood banks in one view, ranked by distance. Discovery time drops from 10 to 30 minutes of phone calls to seconds.
- **Actionable public information during emergencies.** During a dengue outbreak, a road accident surge, or a mass-casualty event, the public map gives citizens a real-time picture of where capacity exists - something that does not exist today.

For the hospital:

- **No more surprise full-ICU situations from external demand.** When another hospital requests an ICU bed, the request flows through the system with a clear status. The receiving hospital sees it, can accept or decline with a reason, and the requesting hospital sees the response. No more "I called but nobody answered."
- **Inventory changes are visible to the right people automatically.** When a ward nurse marks an ICU bed occupied, the dispatcher and the map see it instantly. No phone call is needed to update the dispatcher's mental model. The hospital's own dashboard stays accurate without manual reconciliation.
- **Equipment tracking beyond beds.** Ventilators, oxygen, dialysis machines - tracked with the same locking and visibility as beds. A hospital that has ICU beds but no ventilators can signal that, and the matching engine can account for it.
- **Inter-hospital sharing becomes trackable, not anecdotal.** Every cross-hospital request is logged. Hospital administrators can see how often they lend versus borrow capacity, which strengthens both operational planning and government reporting.

For the ambulance crew:

- **Clear dispatch assignments with destination and context.** The assignment includes the incident, the destination hospital, and the reason. Status updates - en-route, on-scene, transporting - flow back to the command centre automatically. No more "go to X hospital, we'll tell you why on the way."
- **Live visibility of other ambulances.** The crew sees where other ambulances are on the map, useful for situational awareness during multi-vehicle accidents or distributed incidents.

For the dispatcher or command centre:

- **One view of everything.** Hospitals, beds, ambulances, blood banks, incidents - all on one map, all live. The dispatcher makes decisions from a complete picture, not from fragmented phone calls.
- **Automated matching recommendations.** The system computes the nearest hospital with the required resources, so the dispatcher decides from ranked options rather than from memory of who has what.
- **Resource locking prevents double-booking.** When an allocation is made, the resource is locked. The dispatcher does not need to remember "I already assigned that bed" - the system enforces it.
- **Escalation visibility.** If no resource is available within the threshold, the incident is flagged for escalation. The dispatcher sees it immediately rather than discovering it after several failed calls.

For the blood bank:

- **Demand visibility.** Blood banks see incoming requests from hospitals - what groups are in demand, from where, at what urgency. This helps with procurement planning and with prioritizing which low-stock alerts matter most.
- **Cross-bank fulfillment.** If Bank A is out of O-negative, the system can recommend Bank B. Today this happens only if someone knows to call Bank B.

For oversight, administration, and government:

- **Audit trail.** Every allocation, dispatch, and transfer is logged. If something goes wrong - a delay, a double-allocation, a resource that was promised but not delivered - there is a system record, not an "I don't remember" situation.
- **Aggregate analytics.** Occupancy trends, demand heatmaps, blood utilization by group, ambulance utilization, response-time metrics. For the first time, the region's emergency capacity is measurable, not anecdotal. This is the data that drives policy decisions - where to add ICU capacity, which blood groups to stock more of, where to station ambulances.
- **Exportable reports.** Hospital administrators and government oversight bodies can export reports on capacity, utilization, and response times - data that today is largely manual and fragmented.

---

## 8. How This Module Integrates With the Rest of the System

### 8.1 It Is the Data Foundation

The hospital module is the foundation that several other modules sit on. The ambulance module needs to know which hospitals are where and which have available beds. The blood bank module needs to know which hospitals are requesting blood. The dispatch module needs the full hospital inventory to make allocation decisions. The public map needs a coarse version of hospital availability to show citizens.

All of these modules read from the hospital module's data, and several of them also write to it - for example, the dispatch module locks a bed when it allocates one, and the ambulance module may mark an incident as resolved at a hospital. The hospital module is both a data provider and a data receiver, but it always owns the hospital data - no other module can rewrite a hospital's bed count without going through the hospital module's update path, which logs the change and broadcasts it.

### 8.2 It Depends on the Auth Module

The hospital module relies on the universal auth module for identity and access control. Every person who views or changes hospital data has an account in the auth module, with roles that define what they can do. The hospital module checks with the auth module before letting anyone see or change anything - "who is this?" and "are they allowed to do this?" - and then applies its own data-scope rules on top (for example, "this person is a hospital admin, but are they the admin of this particular hospital?").

This means the hospital module does not invent its own login system or its own roles. It uses the shared auth module, and the rules are consistent across the whole system.

### 8.3 It Feeds the Dispatch Module

The dispatch module is the primary consumer of the hospital module's live data. The dispatcher's view - the map with all hospitals, all inventory counts, all active locks - is built from the hospital module's data plus the ambulance and blood bank modules' data. When the dispatcher allocates a bed, the dispatch module writes through the hospital module (locking the inventory row and logging the change), so the hospital module remains the single source of truth for hospital data even when the action originates from dispatch.

### 8.4 It Feeds the Public Map

The public map reads a coarsened version of the hospital module's data - names, types, locations, and coarse availability indicators - through a public endpoint that deliberately does not have access to exact counts. The public map never sees the raw inventory; it sees only what the system chooses to expose. This is the same boundary described in the sharing section, enforced at the data level.

---

## 9. Mock Data Approach

### 9.1 Why We Use Mock Data

Real hospital APIs are not accessible. There is no live feed of bed availability from actual hospitals that the system can connect to. So the platform must be bootstrapped with realistic synthetic data that demonstrates every feature. This is not a workaround to hide - it is stated clearly in the report as a limitation and as the basis for a pilot demonstrator. The architecture does not depend on real data sources; it would accept real feeds through standardized adapters when hospitals adopt the system.

### 9.2 What the Mock Data Looks Like

The mock data should be:

- **Geographically plausible** - hospitals placed at real locations in a chosen city or region, with realistic distances between them. This makes the map and distance-based matching meaningful.
- **Inventory-plausible** - bed counts that match the type of hospital. A 300-bed multi-specialty hospital has a different ICU capacity than a 50-bed clinic. The mock data should reflect that, so the system's behaviour is realistic.
- **Dynamic, not static.** The demo should show beds changing over time, so the system can either simulate admissions and discharges on a timer or let the presenter toggle them manually. A static dataset that never changes does not demonstrate the live-updates feature.

### 9.3 How We Set It Up

- Define a realistic set of hospitals - 8 to 15 of varied types, in a chosen city or district. Use real hospital names if they are publicly known, or fictitious but realistic names.
- Assign realistic capacities to each - for example, a 300-bed multi-specialty hospital might have 200 general beds, 24 ICU beds, 8 CCU beds, 6 PICU beds, 8 operation theatres, and 6 isolation beds. A 50-bed clinic might have 40 general beds, 2 ICU beds, and 2 operation theatres.
- Set initial occupancy for each - some beds occupied to start, typically 60 to 85 percent depending on the type of ward and the time of day. General wards are higher occupancy; ICU is lower but more critical.
- Seed equipment - ventilators roughly proportional to ICU capacity, oxygen supply levels, dialysis machines, defibrillators.
- Seed contacts - a duty medical officer, an ICU in-charge, and an administrator for each hospital.
- Optionally, simulate activity - a background process that periodically adjusts occupied counts by a small amount in random hospitals, logs the change, and broadcasts it, so the demo feels alive even if the presenter is not manually changing things.

### 9.4 Why the Mock Data Matters

The mock data is a first-class asset, not an afterthought. It lives in a versioned seed script that anyone running the project can use to set up the same demo dataset. This makes the demo reproducible - you can reset to a known state before showing it - and it doubles as a documented example of how the system is used, which is a good place to catch bugs early.

---

## 10. Privacy and Safety Considerations

### 10.1 What the System Does Not Store

- Patient personal information - name, age, diagnosis, or anything like that. The system tracks resources, not patients. If the system were extended to track patients, that would open a separate set of privacy and compliance concerns that are not part of this project.
- Staff personal data beyond name, role, and contact details - acceptable for operational contact purposes, but not extended to home addresses or other personal identifiers.
- Financial data - not in scope.

### 10.2 What the System Does Store

- Hospital identities - name, address, type, and contact details. These are public information anyway; any hospital's name and address are visible on its own website or a map.
- Bed and equipment inventory counts - operational data, not patient data.
- Audit logs of inventory changes - operational, with no patient identifiers if the project stays in scope.

### 10.3 The Compliance Position

This system processes operational healthcare resource data - hospital bed availability, equipment counts, ambulance status, and blood bank inventory. It does not store patient personal health information in its current scope. In a production deployment that handled patient-level data, the system would need to comply with applicable regulations - India's data protection law, the Digital Health Act, and any state-level healthcare data norms - and would need encryption, role-based access auditing, and data minimization. For this academic project, the system operates on resource-level operational data with mock hospital identities and is clearly scoped as a pilot demonstrator.

### 10.4 Data Minimization in Practice

- The public endpoint deliberately shows coarse availability, not exact counts.
- Internal contacts are not exposed publicly.
- Audit logs are retained per a configurable policy - for the project, a simple version of this is fine, such as keeping recent logs readily available and treating older logs as archived.

### 10.5 Why Restraint Matters Here

There is a real risk in exposing too much data, even with good intentions:

- If every hospital's exact bed count were public, it could lead to people targeting hospitals that show available capacity, overwhelming them during a crisis.
- If internal contacts were public, they could be misused for purposes other than emergency coordination.
- If one hospital could see another's exact inventory at all times, hospitals might be reluctant to join the platform at all, because their capacity data would be visible to competitors or to anyone who knew how to look.

The system's approach is to share what is needed for coordination and nothing more. For the public, that means coarse availability. For hospitals, that means identity, coarse availability, and recommendations - not raw data about each other. For the dispatcher, that means full operational data because the job requires it, but with full audit logging so the trust is accountable.

---

## 11. What We Are Explicitly Not Doing

It is worth stating what this module does not attempt, so the scope is honest:

- **Patient-level medical records.** Not in scope. The module tracks resources, not patients. Adding patient records would open compliance, consent, and data minimization complexity that belongs to a different kind of system.
- **Replacing a hospital's internal systems.** This module is a coordination layer on top of existing hospital systems, not a replacement. Hospitals keep their internal systems; this platform aggregates the resource-level data they choose to expose. The production path is integration through standardized feeds, not replacement.
- **Real hospital API integration in this project.** No real hospital APIs are accessible, and building integrations to specific hospital vendors is out of scope for a student project. The system operates on self-reported and mock data, and is designed to accept real feeds when hospitals adopt it. This is a stated limitation, not an oversight.
- **Full healthcare data-standard compliance in this project.** The module's data model is compatible with the concepts in healthcare data standards, but it does not implement full standard schemas or a standards-compliant server. That is a production integration path, noted in the report.
- **Replacing emergency call infrastructure.** This module is a resource-coordination platform for hospitals, ambulances, and blood banks. It does not replace emergency call handling, phone triage, or the public emergency number system. It could integrate with such systems as a downstream resource allocator, but that is not part of this project.
- **Real-time patient monitoring or vitals.** Not in scope. The module tracks resource availability and dispatch status, not patient vitals. If a future version integrated with bedside monitors or ambulance telematics, that would be a separate data stream with its own requirements.

---

## 12. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Track every hospital's identity, bed inventory, equipment inventory, contacts, and change history in one place, and serve that data to the rest of the system in a controlled way |
| Data tracked | Hospital profiles, bed categories, live bed inventory by category with total/occupied/available/lock state, equipment inventory, contacts, change log, optional blocked time slots |
| Data architecture | One shared platform data store; each hospital's data kept separate by organization tag and access-controlled; no hospital can see another's raw data; acknowledged as a pilot approach with per-hospital ownership as the production path |
| Who gets access | Hospital admins see their own hospitals fully; bed managers update inventory for their own hospitals; read-only viewers see but do not change; dispatchers see all operational data across all hospitals and can allocate but not edit profiles; platform admins see everything and manage the system; the public sees coarse availability only |
| How data is shared | Within a hospital: full data for staff with the right role. Between hospitals: identity and coarse availability shared openly; exact counts shared only through formal request recommendations or in a request context, never browsed openly. With the dispatcher: full operational data, auditable. With the public: coarse availability only, deliberately limited. Never: patient data, one hospital's raw inventory to another without a request, internal contacts across organizations, exact counts to the public |
| Real-time | Updates broadcast immediately to hospital staff, the dispatcher, and relevant clients; public map gets throttled coarse updates; concurrent updates handled with version-based conflict prevention |
| Gaps addressed | No shared live view today, no cross-hospital visibility, no formal inter-hospital requests, no audit trail, no public visibility, no resource locking, no aggregate regional picture - all introduced by this module working with the rest of the system |
| What it does not do | Patient records, HIS replacement, real hospital API integration in this project, full standards compliance, emergency call replacement, patient vitals monitoring |
| Privacy stance | Share what is needed for coordination and nothing more; enforce the boundary structurally rather than by policy; be honest in the report about mock data and the production path |

---

