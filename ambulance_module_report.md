# Ambulance Module — Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Ambulance Module is the part of the system that knows about every ambulance in the network — what units exist, where they are, what state they are in, who is in them, and what they have been assigned to do. It is the module that turns ambulances from invisible, uncoordinated assets into tracked, dispatchable resources that the rest of the system can see and use.

Specifically, this module is responsible for:

- Registering ambulance units on the platform — each unit's identity, type, equipment, and which operator owns it
- Tracking the real-time status of each ambulance — idle, dispatched, en-route to the scene, on-scene, transporting a patient, at the hospital, returning, or out of service
- Tracking the live location of ambulances that are on active calls, so the dispatcher and the command centre can see where they are on the map
- Showing which ambulance is assigned to which emergency, and the status of that assignment from the moment of dispatch through to handover at the hospital
- Letting ambulance crews update their own status as they move through a call — without needing to call the dispatch centre to say "we are on our way" or "we have arrived"
- Recording the history of each ambulance's assignments, so there is a record of what each unit has done and how long its calls take
- Making ambulance data available to the dispatcher (full picture), to hospitals (relevant to incoming patients), to the ambulance crews themselves (their own assignments), and to the public (limited, coarse, if at all)

This module connects directly to the hospital module (because ambulances transport patients to hospitals), to the dispatch module (because the dispatcher assigns ambulances to emergencies), and to the incident module (because each ambulance assignment is tied to a specific incident). It also relates to the public interface, because citizens calling for help may want to know that an ambulance is on its way.

---

## 2. What Data We Track

### 2.1 Ambulance Unit Identity

For each ambulance unit registered on the platform, we store:

- A unique identifier for the unit — an internal ID the system uses, and a registration number or fleet number that people recognize
- The organization that owns or operates the unit — a government emergency service, a hospital's own ambulance fleet, a private operator, or some other service
- The type of unit — basic life support, advanced life support, cardiac-response vehicle, neonatal transport, or any other type the operator wants to declare. The type matters because not every emergency needs the same kind of ambulance, and the matching logic should account for that
- What equipment the unit carries — stretcher, oxygen, defibrillator, medications, ventilator, monitoring equipment, and any other equipment the operator wants to declare. This is about what the ambulance itself has on board, not what a hospital has
- The unit's home base or primary deployment area — where it is normally stationed, which helps with understanding coverage and with recommendations about which unit is closest to an emergency
- The unit's contact information — a phone number or radio identifier that the dispatch centre can use to reach the crew
- The name of the crew or the primary operator — for accountability and for the record, though this is operational contact information, not deeply personal data
- A status indicator — whether the unit is currently available, busy, or out of service, and if busy, what it is doing

We do not store patient information in this module — the ambulance transports patients, but the patient's identity and medical details are not part of what this module tracks. The emergency the ambulance is responding to is tracked in the incident module, and the handover at the hospital is tracked there too. This module knows that "ambulance X is transporting a patient to hospital Y for incident Z," not who the patient is.

### 2.2 Live Status and Location

For each ambulance, the module tracks a status that changes as the crew moves through a call. The status moves through a defined path:

- **Idle** — the unit is available and ready to be dispatched
- **Dispatched** — the unit has been assigned to an emergency and is heading out
- **En-route to the scene** — the unit is on its way to the emergency location
- **On-scene** — the unit has arrived at the emergency and is working
- **En-route to the hospital** — the unit has a patient on board and is heading to a hospital
- **At hospital** — the unit has arrived at the destination hospital, with or without a patient
- **Returning** — the unit is heading back after completing a call
- **Out of service** — the unit is unavailable for some reason — maintenance, crew change, breakdown, or operator-declared unavailability

Each status change is recorded with a timestamp and the crew member who made the change, so there is a timeline of what the ambulance did during a call.

For ambulances that are on active calls — dispatched, en-route to the scene, on-scene, or transporting — the module also tracks the live location as latitude and longitude, updated periodically by the crew's device. This is what lets the dispatcher see the ambulance moving on the map and know roughly where it is at any moment.

The location is not tracked when the ambulance is idle and sitting at its base. That would be a privacy and battery-drain concern without operational benefit. Location tracking is for active calls, when knowing where the ambulance is matters for coordination and for the ambulance crew's own situational awareness.

### 2.3 Assignment and Emergency Link

Every ambulance assignment is linked to an emergency incident. The module records:

- Which ambulance is assigned to which emergency
- When the assignment was made and by whom — the dispatcher
- The status of the assignment — pending, en-route, on-scene, transporting, completed, or cancelled
- The destination hospital, if one has been chosen at the time of dispatch — sometimes the destination is decided on scene, so it may not be known at dispatch time
- Any notes the dispatcher or crew add about the assignment

This link is what makes the system coherent — it is not just that "ambulance A is busy," but that "ambulance A is busy because it was dispatched to emergency B at this time, is now transporting to hospital C, and is expected to arrive at this time." That coherence is what the dispatcher needs and what makes the system better than a phone-call-based process where each piece of information lives in a different person's head.

### 2.4 Crew and Capacity

For each ambulance unit, we track:

- How many crew members are normally on the unit
- Whether there is a trained paramedic or doctor on board — for advanced life support units, this matters
- Any special capabilities — for example, a unit that can handle a bariatric patient, or a paediatric team, or a unit with a specific language capability if that matters in your region

This is operational information, not deeply personal. It tells the dispatcher what the unit is capable of, so that if an emergency needs advanced life support, the system can recommend an advanced-life-support unit rather than a basic-life-support unit.

### 2.5 Assignment History

Every completed assignment is recorded in a history log:

- Which ambulance, which emergency, which dispatcher made the assignment
- The timeline — dispatched, arrived on scene, departed for hospital, arrived at hospital, completed
- The destination hospital, if any
- Any notes or outcomes the crew or dispatcher added

This history is the ambulance module's audit trail. It is what you look at to know what a unit has been doing, how many calls it has handled, and how long its calls tend to take. It also feeds the analytics — utilization, response times, and so on.

---

## 3. Who Gets Access to What

### 3.1 The Access Model

Access to ambulance data is controlled the same way as the rest of the system — through the universal auth module, with roles and permissions. The ambulance module asks "who is this?" and "are they allowed to do this?" and then applies its own data-scope rules on top.

The key roles that interact with ambulance data are:

- **Ambulance crew or operator** — the person in the ambulance or the operator managing the unit. They can see their own unit's status and assignment, update their own status, and view their own assignment history
- **Dispatcher** — can see all ambulances on the map, all statuses, all active assignments, and can assign ambulances to emergencies. This is the operational role that needs the full picture
- **Hospital staff** — may be able to see that an ambulance is on its way to their hospital, with a patient, and the expected arrival, but not the full fleet picture. They see what is relevant to them
- **Platform administrator** — can see everything and manage ambulance units on the platform
- **Public** — in most designs, the public does not see ambulance locations or statuses. Showing live ambulance locations publicly could create safety and privacy concerns, and could mislead people about response availability. The public map focuses on hospitals and blood banks, not on individual ambulance positions

### 3.2 Ambulance Crew or Operator

An ambulance crew member, logged into the system, can:

- See their own unit's current status and assignment
- Update their own status as they move through a call — mark themselves dispatched, en-route to the scene, on-scene, transporting, at hospital, and so on
- See the details of the emergency they have been assigned to — where to go, what is known about the situation (subject to what the incident module exposes, which is operational, not personal)
- See the destination hospital if one has been chosen
- View their own unit's assignment history

They cannot see other ambulances' statuses or locations, unless they also have a dispatcher role. They cannot reassign their unit or another unit. Their access is scoped to their own unit and their own calls.

This is deliberate — an ambulance crew member does not need to see the entire fleet's status to do their job. They need to know their own assignment and be able to update it. Giving them the full fleet picture would be noise, not help.

### 3.3 Dispatcher

The dispatcher can:

- See all ambulances on the map, with their live location and status, during active calls
- See all ambulances' current availability — which units are free, which are busy, which are out of service
- Assign an ambulance to an emergency — choose a unit, confirm the assignment, and the system records it and notifies the crew
- See the full timeline of each active assignment — where the ambulance is, what stage it is at, where it is headed
- Reassign an ambulance if needed — for example, if a higher-priority emergency comes in and the closest available unit needs to be redirected
- Cancel a dispatch if the emergency is resolved before the ambulance arrives

The dispatcher is the role that needs the full picture, and the module gives it to them. Every assignment the dispatcher makes is logged with their identity, so it is auditable.

### 3.4 Hospital Staff

Hospital staff, when an ambulance is on its way to their hospital, can see:

- That an ambulance is en-route to their hospital
- Which emergency it is for — operational details, not patient identity
- The expected arrival, based on the ambulance's current location and status

They do not see the full fleet picture. They see what is relevant to them — that a patient is coming and from where — so they can prepare. This is useful because it lets the hospital know to expect a patient without the ambulance crew having to call ahead separately; the system does that notification automatically.

### 3.5 Platform Administrator

The platform administrator can:

- Register new ambulance units on the platform
- Update ambulance unit details — type, equipment, home base
- Deactivate units that should no longer be on the platform
- View all ambulance data and history for oversight

This is the management role for the ambulance fleet itself — adding units, removing units, correcting details.

### 3.6 Public

The public does not see ambulance locations or statuses through this system. There are several reasons for this:

- Showing live ambulance positions publicly could create privacy and safety concerns for the crew
- It could mislead people into thinking an ambulance is available or responding to their situation when it is not
- The public's need during an emergency is to find a hospital or a blood bank, not to track individual ambulances — the public map is designed around that need
- In some systems, public visibility of emergency response units has caused problems — people have interfered with responding units, or have made decisions based on incomplete information about where units are and what they are doing

This is a deliberate scope decision. If a public-facing feature for ambulance visibility is ever wanted — for example, a coarse "an ambulance has been dispatched to your area" notification — it would be a separate, carefully-designed feature, not a live map of all units. That is not part of this project.

---

## 4. How Data Is Shared

### 4.1 Within the Ambulance Crew

The crew sees their own unit's data — their status, their assignment, their destination, their history. This is the data they need to do their job and to keep the system informed about what they are doing. When they update their status, the change is saved and broadcast to whoever is watching — the dispatcher, and the hospital that is the destination, if one has been chosen.

### 4.2 With the Dispatcher

The dispatcher sees everything operationally relevant about all ambulances — all units, all statuses, all live locations during active calls, all active assignments. This is the full picture the dispatcher needs to make dispatch decisions. The data flows both ways: the dispatcher assigns ambulances and sees the result, and the ambulances report their status back to the dispatcher as they move through the call.

This is the core sharing relationship in the ambulance module — the dispatcher is the central node that sees everything and instructs everything, and the ambulances report back to the dispatcher.

### 4.3 With the Hospital

When an ambulance is assigned to an emergency and a destination hospital has been chosen, the hospital sees that an ambulance is en-route to them, with the operational details of the emergency. This is useful because it lets the hospital prepare — know that a patient is coming, from where, and with what urgency — without the ambulance crew having to make a separate phone call. The system does that notification automatically, based on the assignment.

The hospital does not see the full fleet picture. It sees only what is coming to it. This is the right scope — the hospital needs to know about its own incoming patients, not about every ambulance in the city.

### 4.4 With Other Ambulances

In most designs, ambulances do not see each other's real-time locations or statuses. This is a deliberate choice:

- Ambulance crews do not need to know where every other unit is to do their job
- Sharing live locations between ambulances could create confusion or unnecessary distraction
- In some cases, it could raise safety or coordination concerns — for example, if two ambulances are responding to the same emergency and one crew can see the other's exact location, that might be useful in some situations but could also be more information than needed

For this project, ambulances see only their own data. If inter-ambulance coordination features are ever wanted — for example, two ambulances responding to the same mass-casualty incident being able to see each other — that would be a specific feature with a specific purpose, not a general live-feed of all units to all units.

### 4.5 With the Public

As described in the access section, the public does not see ambulance locations or statuses through this system. The public map focuses on hospitals and blood banks. This is a deliberate scope and privacy decision.

---

## 5. How Real-Time Updates Work

### 5.1 Status Updates from the Crew

When an ambulance crew updates their status — from idle to dispatched, from dispatched to en-route to the scene, from en-route to the scene to on-scene, from on-scene to transporting, and so on — the change is recorded with a timestamp and broadcast to the dispatcher and to the destination hospital if one has been chosen. The dispatcher sees the status change on the map and in the assignment timeline. The hospital sees that the ambulance is now en-route, or now transporting, or has arrived.

This is what makes the system useful for the dispatcher — they do not have to call the ambulance to ask "where are you?" The ambulance reports its status as it moves, and the dispatcher sees it.

### 5.2 Location Updates During Active Calls

When an ambulance is on an active call — dispatched, en-route to the scene, on-scene, or transporting — its location is updated periodically from the crew's device. The dispatcher sees the ambulance moving on the map, which helps with:

- Knowing roughly when the ambulance will arrive at the scene or at the hospital
- Making decisions about whether to dispatch another unit if the first is taking too long
- Situational awareness during multi-vehicle accidents or distributed incidents, where the dispatcher wants to see where all responding units are

The location is not tracked when the ambulance is idle at its base. That would consume battery and raise privacy concerns without any operational benefit. Location tracking is for when it matters — during active calls.

### 5.3 Notification to the Hospital

When an ambulance is assigned to an emergency with a destination hospital, the hospital is notified automatically. The notification includes the operational details of the emergency and the ambulance's current status. As the ambulance's status changes, the hospital sees the updated status — en-route, on-scene, transporting, arrived. This keeps the hospital informed without requiring the ambulance crew to call ahead separately.

This is one of the concrete improvements over the current system, where the ambulance crew often has to call the hospital on arrival or en-route, and the hospital has no way to know an ambulance is coming until it arrives or is told by the crew.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

Today, in most cities and regions, ambulances operate as largely invisible and uncoordinated resources:

- **There is no shared registry of ambulances.** Each ambulance operator — a hospital's own fleet, a private service, a government emergency vehicle — knows its own units, but there is no single system that knows all the ambulances in a city and where they are. The dispatcher who needs an ambulance calls the operators they know and asks, one by one.
- **Ambulance location is not visible to anyone but the crew.** The crew knows where they are. The dispatch centre, if it has a way to track units at all, may have a separate radio-based or proprietary system. Other hospitals, other ambulances, and the public have no way to know where an ambulance is.
- **Status updates are manual and phone-based.** When an ambulance is dispatched, the dispatcher often has to call or radio the crew to find out whether they have arrived, whether they are transporting, whether they have handed over at the hospital. The dispatcher's picture of where each ambulance is in its call is only as good as the last phone call.
- **Ambulance assignments are not linked to emergencies in a shared system.** The dispatcher knows "I sent that ambulance to that call," but the hospital does not know an ambulance is coming until it arrives or is told, and the system has no coherent record linking the ambulance, the emergency, and the hospital.
- **There is no formal record of what each ambulance has done.** Utilization, response times, and call history are often tracked informally or not at all. There is no auditable log of which ambulance went to which emergency, when, and with what outcome.
- **There is no matching logic.** The nearest available ambulance is found by whoever has the most phone numbers or the best radio coverage, not by a system that knows where all units are and can compute the closest available one.

This is the baseline. It is not theoretical — it is how most ambulance coordination works today, even in places with emergency numbers, because the coordination layer between ambulances, hospitals, and dispatch is missing or manual.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| No shared registry of ambulances across operators | A single registry of all ambulance units on the platform, with their type, equipment, home base, and capability, so the dispatcher can see the whole fleet |
| Ambulance location invisible to everyone but the crew | Live location tracking during active calls, visible to the dispatcher and to the destination hospital, so the dispatcher can see where each unit is on the map |
| Status updates are manual and phone-based | Crew self-service status updates — the crew marks their own progress through the call, and the dispatcher and hospital see it immediately, without a phone call |
| Assignments not linked to emergencies in a shared system | Every ambulance assignment is linked to an emergency, with a full timeline from dispatch through to handover, visible to the dispatcher and the destination hospital |
| No formal record of what each ambulance has done | An assignment history log — which ambulance went to which emergency, when, with what timeline and outcome — auditable and exportable |
| No matching logic for nearest available ambulance | The dispatcher can see all units on the map and their availability, and the dispatch module can use that to recommend the nearest available unit for an emergency |
| Hospital has no way to know an ambulance is coming | Automatic notification to the destination hospital when an ambulance is assigned, with status updates as the ambulance moves — the hospital knows to expect a patient without a separate phone call |
| No distinction between unit types in dispatch decisions | Ambulance types — basic life support, advanced life support, and so on — are registered and visible, so the dispatcher can match the right kind of unit to the emergency |

### 6.3 How It Is Better — The Concrete Improvements

For the ambulance crew:

- **Clear assignments with context.** The crew sees the emergency details, the location, and the destination hospital if chosen — all in one place, rather than getting a phone call with partial information
- **Self-service status updates.** The crew updates their own status as they move through the call — no need to call the dispatcher to say "we are on scene" or "we are transporting." The system records it and tells the dispatcher and hospital
- **Live visibility of their own assignment.** The crew can see where they are on the map relative to the emergency and the hospital, which helps with navigation and situational awareness

For the dispatcher:

- **The full fleet picture on one map.** Every ambulance, every status, every live location during active calls — all in one view. The dispatcher makes decisions from a complete picture, not from phone calls to individual operators
- **Automated nearest-available awareness.** The dispatcher can see which unit is closest to an emergency and available, rather than relying on memory or phone queries
- **A coherent timeline for every active call.** The dispatcher sees, for each active assignment, where the ambulance is, what stage it is at, and where it is headed — no more "I think that ambulance is on its way, but I'm not sure"

For the hospital:

- **Advance notice of incoming patients.** When an ambulance is on its way to the hospital, the hospital knows — automatically, without the crew having to call ahead. The hospital can prepare
- **Status updates as the ambulance moves.** The hospital sees the ambulance's progress — en-route, on-scene, transporting, arrived — so it knows what stage the response is at

For oversight and administration:

- **An auditable assignment history.** Every assignment is logged — which ambulance, which emergency, which dispatcher, what timeline, what outcome. This is what you look at to understand utilization, response times, and how the fleet is being used
- **A clear fleet registry.** The platform knows what units exist, what type they are, what equipment they carry, and where they are based — information that today is scattered across operators and often not aggregated at all

---

## 7. How This Module Integrates With the Rest of the System

### 7.1 With the Hospital Module

This is the most direct integration. When an ambulance is dispatched to an emergency and a destination hospital is chosen, the ambulance module records the link and tells the hospital module that an ambulance is en-route to that hospital. The hospital module, in turn, shows the hospital's staff that a patient is coming — what emergency it is for, what stage the ambulance is at, and when it is expected to arrive.

When the ambulance arrives at the hospital and the crew marks their status as "at hospital," the hospital module sees that update and the hospital knows the patient has arrived. This closes the loop — the hospital knows when a patient is coming and when they arrive, without the ambulance crew having to make separate calls at each stage.

The hospital module also informs the ambulance module, in the other direction, about bed availability — because when the dispatcher is choosing a destination hospital for an ambulance, they need to know which hospitals have the capacity the patient needs. That is the hospital module's data, used by the dispatch module and visible to the dispatcher alongside the ambulance data.

### 7.2 With the Dispatch Module

The dispatch module is the primary consumer of the ambulance module's data. When an emergency comes in, the dispatch module looks at the available ambulances — their location, their type, their status — and recommends the most suitable unit. When the dispatcher confirms the assignment, the dispatch module creates the assignment and the ambulance module records it and notifies the crew.

During the call, the dispatch module shows the dispatcher the ambulance's status and location, pulled from the ambulance module. When the call is complete, the dispatch module records the outcome and the ambulance module updates the assignment history.

The dispatch module and the ambulance module are tightly coupled in operation, but the ambulance module owns the ambulance data — the dispatch module does not rewrite ambulance status directly; it asks the ambulance module to update it, and the ambulance module logs the change and broadcasts it.

### 7.3 With the Incident Module

Every ambulance assignment is linked to an emergency incident. The ambulance module does not track incidents themselves — that is the incident module's job — but it references incidents to know what each ambulance has been assigned to. The incident module tells the ambulance module "ambulance X is assigned to incident Y," and the ambulance module tells the incident module "ambulance X is now transporting to hospital Z for incident Y."

This link is what makes the system coherent. Without it, you have ambulances doing things and emergencies happening, but no connection between them in the system. With it, you have a clear record of which ambulance responded to which emergency, when, and with what outcome.

### 7.4 With the Auth Module

The ambulance module uses the universal auth module for identity and access control, just like every other module. Every ambulance crew member, dispatcher, hospital staff member, and platform administrator who interacts with ambulance data has an account in the auth module, with roles that define what they can do. The ambulance module checks with the auth module before letting anyone see or change anything — "who is this?" and "are they allowed to do this?" — and then applies its own data-scope rules on top.

The ambulance crew member's role, for example, lets them see and update their own unit's status but not see the full fleet. The dispatcher's role lets them see the full fleet and make assignments. The auth module provides the role information; the ambulance module applies the scope.

### 7.5 With the Public Map

The ambulance module's data does not feed the public map in this project. The public map shows hospitals and blood banks, in coarse form. Ambulance locations and statuses are not shown publicly — this is a deliberate scope and privacy decision, as described in the access section. If a future version wanted to show the public that "an ambulance has been dispatched to your area," that would be a separate, carefully-designed feature, not a live map of all units.

---

## 8. Privacy and Safety Considerations

### 8.1 What We Do Not Track

- **Patient identity or medical details.** The ambulance module knows about the ambulance and the emergency it is responding to, not about the patient. Patient information is not in scope for this project at all
- **Crew personal information beyond operational contact details.** Name and role for accountability and contact, not home addresses, personal phone numbers, or other deeply personal data
- **Continuous location tracking of idle ambulances.** Location is tracked only during active calls, when it has operational value

### 8.2 Why We Do Not Show Ambulance Locations Publicly

- **Safety.** Showing the live location of emergency response units publicly could create safety concerns for the crew — it could make them visible to people who might interfere with their work, or could expose their movements in ways that are not appropriate for public view
- **Misleading availability signals.** A member of the public seeing an ambulance on the map might assume it is available or responding to their situation, when in fact it is already committed to another call. This could create false expectations and dangerous decisions
- **Scope.** The public's need during an emergency, in this system, is to find a hospital or a blood bank — that is what the public map is designed to help with. Tracking individual ambulances is not the public's need, and adding it would create the above risks without clear benefit

This is a deliberate scope decision, not an oversight. The system chooses to give the public useful but limited information — hospitals and blood banks — rather than a live tracker of all emergency vehicles, which would carry risks and limited benefit.

### 8.3 What the Crew Sees and Why It Is Appropriate

The crew sees their own unit's data — their status, their assignment, their destination, their history. This is the data they need to do their job. They do not see the full fleet's locations or statuses, because that is not their job and could be distracting or confusing.

The dispatcher sees the full fleet because that is the dispatcher's job — coordination requires the full picture. The asymmetry is intentional: each role sees what it needs to do its job, and nothing more.

### 8.4 Emergency Information Shown to the Crew

When an ambulance is assigned to an emergency, the crew sees the operational details of the emergency — where to go, what is known about the situation, and the destination hospital if chosen. This is information the crew needs to respond appropriately. It is operational, not personal — it does not include the patient's name, age, or medical history, which are not in scope for this system.

This is the right balance: the crew gets enough information to do their job safely and effectively, without the system exposing personal data that is not needed for the response.

---

## 9. What We Are Explicitly Not Doing

It is worth stating what this module does not attempt, so the scope is honest:

- **Automatic location tracking without crew input.** The system does not silently track ambulance locations all the time. Location is updated during active calls, based on the crew's device. Continuous background tracking would raise privacy and battery concerns without operational benefit for idle units
- **Autonomous or driverless ambulances.** This is a system for coordinating existing ambulance units with human crews, not a vehicle automation project
- **Telemedicine or patient monitoring in the ambulance.** The module tracks the ambulance's status and location, not the patient's vitals during transport. If a future version integrated with monitoring equipment in the ambulance, that would be a separate data stream with its own privacy and technical requirements
- **Integration with every ambulance operator's existing dispatch system.** In this project, ambulances are registered on the platform and their status is managed through the platform. In a production rollout, some operators might want to integrate their existing dispatch systems with the platform — that is a future integration path, not part of this project
- **Public visibility of ambulance positions.** As described, the public does not see ambulance locations or statuses. This is deliberate
- **Navigation or routing inside the ambulance.** The module tracks where the ambulance is and where it is going, but it does not provide turn-by-turn navigation to the crew. The crew uses their own navigation tools. The system can show the location and the destination on the map, but it does not replace the crew's navigation
- **Priority or traffic-signal preemption.** The module does not interface with traffic management systems to give ambulances priority at signals. That is a separate infrastructure integration, not part of this project

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Register all ambulance units on the platform, track their real-time status and location during active calls, link each ambulance to the emergency it is responding to, and make the right data visible to the crew, the dispatcher, the destination hospital, and the rest of the system |
| Data tracked | Ambulance unit identity — registration, type, equipment, home base, contact — live status through a defined path from idle to dispatched to en-route to on-scene to transporting to at hospital to returning to out of service, live location during active calls only, assignment and emergency link with full timeline, crew and capacity info, assignment history log |
| Who gets access | Ambulance crew sees their own unit's status, assignment, destination, and history; dispatcher sees all ambulances, all statuses, all live locations during active calls, all active assignments, and can assign and reassign; hospital sees only what is coming to it; platform admin sees and manages all units; public sees no ambulance data |
| How data is shared | Crew to dispatcher and destination hospital: status and location during active calls. Dispatcher to crew: assignments and emergency details. Ambulance to hospital: automatic notification of en-route status and progress. No sharing of live locations between ambulances, no public visibility of ambulance positions |
| Real-time | Crew status updates broadcast to dispatcher and hospital immediately; location updates during active calls visible to dispatcher on the map; hospital notified automatically when ambulance is assigned and as it progresses |
| Gaps addressed | No shared ambulance registry today, location invisible, status updates phone-based, assignments not linked to emergencies in a shared system, no formal assignment history, no nearest-available matching, hospital has no advance notice of incoming ambulance — all introduced by this module working with the dispatch, hospital, and emergency modules |
| What it does not do | Patient tracking, continuous idle location tracking, autonomous vehicles, telemedicine or patient monitoring, integration with all existing operator dispatch systems, public ambulance location visibility, in-ambulance navigation, traffic-signal preemption |
| Privacy stance | Location tracked only during active calls, not idle; patient data not in scope; crew data limited to operational contact; no public ambulance tracking; each role sees only what it needs, asymmetrically — crew sees own unit, dispatcher sees all |
| Integration | Depends on auth module for identity and access; feeds dispatch module with fleet data and receives assignments; links to emergencies in the emergency module; notifies hospitals through the hospital module; does not feed the public map |

---

