# Dispatch Module - Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Dispatch Module is the central orchestration layer of the entire system. It is where emergencies come in, where the system decides what resources are needed and where to find them, where ambulances are assigned, where hospitals are matched to patients, and where the overall response is coordinated from start to finish.

Specifically, this module is responsible for:

- Receiving emergency requests from various sources - the public interface, hospitals calling in, ambulances reporting situations they have encountered, and any other channel the system supports
- Understanding what each emergency needs - what type of response, what resources, what urgency, what location
- Looking across all available resources - ambulances, hospitals with beds and ICU, blood banks with the needed blood - and finding the best matches
- Assigning resources to the emergency - dispatching an ambulance, recommending a hospital, arranging blood if needed
- Tracking the response from dispatch through to handover at the hospital, with a clear timeline of what happened and when
- Notifying the relevant parties at each stage - the ambulance crew, the destination hospital, the blood bank if blood is needed, and anyone else who needs to know
- Keeping a complete record of the dispatch - what was requested, what was assigned, what happened, what the outcome was - so there is a full audit trail and the data needed for analytics

This module does not own the resources themselves. It does not own ambulances (that is the ambulance module), hospitals (the hospital module), or blood banks (the blood bank module). It sits above them and coordinates them. It is the brain of the system, and the other modules are the parts of the body it directs.

---

## 2. What Data We Track

### 2.1 Emergency Request

When an emergency comes in, the module records:

- Where the emergency is happening - as precisely as possible, ideally with latitude and longitude from the caller's device or from an address that has been geocoded
- What type of emergency it is - trauma, cardiac, respiratory, neurological, obstetric, paediatric, fire-related, drowning, poisoning, or any other type the system is designed to handle. The type matters because different emergencies need different resources
- What is known about the situation - a brief description, in operational terms, not patient identity. For example, "road accident with entrapment," "chest pain with sweating," "difficulty breathing," "unconscious adult male"
- How urgent it appears to be - based on what is reported and, if there is a triage process, on a triage assessment. Urgency drives the priority of the response
- Who reported the emergency - a citizen through the public interface, a hospital, an ambulance crew, or another source
- When the emergency was reported - the timestamp that starts the clock on the response
- Any contact information for the reporter or the patient, if available and appropriate - enough to reach the scene or to follow up, but not more than is needed

We do not store the patient's name, age, address, medical history, or any personal health information in this module. The emergency is described in operational terms - what is happening and where - not as a patient record. Patient-level data is not in scope for this project.

### 2.2 Resource Requirements

From the emergency type, description, and urgency, the module derives or is told what resources are needed:

- An ambulance - and if so, what type. A cardiac emergency may need an advanced-life-support ambulance; a simple transport may only need a basic-life-support unit
- A hospital bed - and if so, what kind. A trauma emergency may need an ICU bed; a cardiac emergency may need a CCU or a cardiac care bed; a minor injury may only need a general bed
- Blood - and if so, which group and how many units. A trauma surgery may need O-negative packed cells; a medical emergency may not need blood at all
- Any other resource the system is designed to handle - for example, a specific specialty unit or a particular piece of equipment

The resource requirements can be derived automatically from the emergency type and description, or they can be specified by the person reporting the emergency or by a triage process. The module records what is needed so that matching can be done against it.

### 2.3 Matching and Recommendations

When the module looks for resources, it records:

- Which resources were considered - which ambulances were in range, which hospitals had the beds needed, which blood banks had the blood needed
- Which were recommended - the best match or matches based on distance, availability, type suitability, and any other criteria the system uses
- The reasoning behind the recommendation - why this hospital, why this ambulance, why this blood bank. This is useful for the dispatcher to understand and trust the recommendation, and for audit purposes later
- Which recommendations were accepted, declined, or overridden - if the dispatcher chose a different option than the one recommended, that is recorded with the reason

This is not just a decision log - it is a record of the system's reasoning. It tells us not only what was done, but why, and what alternatives were available. That is valuable for audit, for improvement, and for understanding how the system is used in practice.

### 2.4 Dispatch and Assignment

When resources are assigned, the module records:

- Which ambulance was assigned to which emergency, by whom, and when
- Which hospital was chosen as the destination, by whom, and when - whether it was the recommended hospital or a different one chosen by the dispatcher
- Which blood request was made, to which blood bank, by whom, and when - if blood was needed and a request was placed
- The status of each assignment - pending, in progress, completed, cancelled, or failed
- The timeline of the assignment - dispatched, arrived at scene, departed for hospital, arrived at hospital, completed - pulled from the ambulance module and the hospital module as the response progresses
- Any changes to the assignment - if an ambulance was reassigned, if the destination hospital was changed, if a blood request was modified - with the reason and the timestamp

This is the core record of the dispatch. It ties together the emergency, the ambulance, the hospital, and any blood request into one coherent dispatch record with a full timeline.

### 2.5 Notification and Status

The module records what notifications were sent and to whom:

- When the ambulance crew was notified of the assignment - and whether they accepted it
- When the destination hospital was notified that an ambulance was en-route - and with what information
- When the blood bank was notified of a request - and the status of that request
- When the reporter or caller was updated - if the system provides updates to the person who reported the emergency, such as "an ambulance has been dispatched" or "the ambulance is on its way to the hospital"

The module does not send the notifications itself - that is the job of the notification service, which is part of the platform infrastructure. But it records what was sent, to whom, and when, so there is a record of the communication that happened as part of the dispatch.

### 2.6 Outcome and Close

When the dispatch is complete, the module records:

- The outcome - the patient was admitted, the patient was transported to another hospital, the emergency was resolved on scene, the ambulance was unable to reach the scene, or any other outcome
- The timestamps - when the emergency was reported, when the ambulance was dispatched, when the patient was handed over, when the dispatch was closed
- Any notes from the dispatcher, the crew, or the hospital about what happened
- The response time metrics - time from report to dispatch, time from dispatch to scene, time from scene to hospital, total response time. These are computed from the timestamps in the record

This is the close of the dispatch record. It is what feeds the analytics - response times, outcomes, resource utilization - and it is what provides a complete account of what happened for audit and review.

---

## 3. Who Gets Access to What

### 3.1 The Access Model

Access to dispatch data is controlled the same way as the rest of the system - through the universal auth module, with roles and permissions. The dispatch module asks "who is this?" and "are they allowed to do this?" and then applies its own rules on top.

The key roles that interact with dispatch data are:

- **Dispatcher** - the person doing the coordination. They can see all emergencies, all resources, all assignments, and all dispatch records. They can create new dispatches, assign resources, change assignments, and close dispatches. This is the central operational role.
- **Ambulance crew** - can see their own assignments and the emergencies they are responding to, but not the full dispatch picture. They see what is relevant to them - where to go, what is known about the situation, what hospital they are headed to.
- **Hospital staff** - can see emergencies that are coming to their hospital, with the relevant details, but not the full dispatch picture. They see what is coming to them and can prepare.
- **Platform administrator** - can see everything across all dispatches, for oversight and review. Can view the full audit trail and the analytics that come from dispatch data.
- **Public** - does not see dispatch data. The public sees the emergency interface for reporting emergencies and, in some designs, the status of their own reported emergency - "your emergency has been received," "an ambulance has been dispatched" - but not the internal dispatch record or any other emergency's data.

### 3.2 Dispatcher

The dispatcher, logged into the system, can:

- See all active and recent emergencies, with their status, location, urgency, and what resources have been assigned
- See all available resources - ambulances on the map, hospitals with bed availability, blood banks with stock - alongside the emergencies, so they can make matches
- Create a new dispatch for an incoming emergency - record the emergency, derive or specify the resource requirements, and start the matching process
- See the system's recommendations - which ambulance, which hospital, which blood bank - and the reasoning behind them
- Accept a recommendation, override it with a different choice, or decline it and wait for better options
- Assign an ambulance to an emergency, choose a destination hospital, and place a blood request if needed
- Change an assignment if circumstances change - reassign an ambulance, change the destination hospital, modify a blood request
- Close a dispatch when the response is complete, with the outcome and any notes
- View the full history of all dispatches, for review and analytics

The dispatcher is the role that does the work of coordination, and the module gives them the full picture and the full set of actions. Every action the dispatcher takes is logged with their identity, so it is auditable.

### 3.3 Ambulance Crew

An ambulance crew member can:

- See the assignments that involve their unit - the emergency, the location, the destination hospital, the status, and any notes
- Update the status of their assignment as they move through the call - dispatched, en-route, on-scene, transporting, arrived - which feeds back into the dispatch record and updates the dispatcher and the hospital
- See the emergency details relevant to them - where to go, what is known about the situation, what hospital they are headed to

They cannot see other assignments, the full emergency list, or the dispatcher's view of all resources. They see only their own work.

### 3.4 Hospital Staff

Hospital staff can:

- See emergencies that are coming to their hospital - the ambulance en-route, the expected arrival, the type of emergency and what is known about it, any blood request that is part of the dispatch
- Prepare for the incoming patient based on this information
- Update the hospital's side of the dispatch when the patient arrives - marking the handover, which feeds back into the dispatch record

They cannot see the full dispatch picture or other hospitals' incoming emergencies. They see only what is coming to them.

### 3.5 Platform Administrator

The platform administrator can:

- See all dispatches, all emergencies, all assignments, and all outcomes across the system
- View the full audit trail of dispatch actions
- Access the analytics that come from dispatch data - response times, outcomes, resource utilization, demand patterns
- Review and investigate any dispatch record in detail

This is the oversight role. The platform administrator does not do the day-to-day coordination - that is the dispatcher's job - but can review everything that has happened and use the data for improvement and accountability.

### 3.6 Public

The public does not see dispatch data. Through the public interface, a citizen can:

- Report an emergency and receive a confirmation that it has been received
- Receive updates on their own reported emergency - "an ambulance has been dispatched," "the ambulance is on its way," "the ambulance has arrived at the hospital" - if the system is designed to provide such updates
- Find the nearest hospital or blood bank, as described in those modules

The public does not see other emergencies, the dispatch record, the resources assigned, or any internal coordination data. The public's view is limited to their own reported emergency and to the resources they can find for themselves.

---

## 4. How Data Is Shared

### 4.1 Within the Dispatch Desk

Everyone at the dispatch desk - all dispatchers working at the same time - sees the same picture. All active emergencies, all available resources, all assignments in progress. This is the shared operational picture that lets any dispatcher pick up any emergency and continue the coordination. It also makes the system resilient - if one dispatcher is busy with a complex case, another can see it and help, or can handle a new emergency without starting from zero.

The dispatch desk sees everything operationally relevant. They do not see patient-level data - that is not in scope - and they do not see administrative data that is not relevant to coordination, such as user account management or system configuration.

### 4.2 With the Ambulance Module

When the dispatcher assigns an ambulance, the dispatch module tells the ambulance module which ambulance is assigned to which emergency, and the ambulance module records the assignment and notifies the crew. As the ambulance's status changes, the ambulance module feeds those changes back into the dispatch record, so the dispatcher sees the ambulance moving through the call - dispatched, en-route, on-scene, transporting, arrived - in real time.

The dispatch module does not directly change the ambulance's status. It asks the ambulance module to assign the ambulance and record the assignment. The ambulance module owns the ambulance's status; the dispatch module sees that status as it changes. This is the right division: the dispatch module coordinates, the ambulance module tracks the ambulance.

### 4.3 With the Hospital Module

When the dispatcher chooses a destination hospital, the dispatch module records the choice and tells the hospital module that an ambulance is coming to that hospital. The hospital module then notifies the hospital's staff. When the ambulance's status changes to arrived, the hospital can record the handover, and the dispatch module sees the full timeline across both modules.

The dispatch module also uses the hospital module's data - bed and ICU availability - when matching emergencies to hospitals. The hospital module owns the bed data; the dispatch module uses it for matching and records the choice it makes.

### 4.4 With the Blood Bank Module

When an emergency requires blood, the dispatch module places a request with the blood bank module - group, quantity, urgency, and reason. The blood bank module records the request and makes it visible to the relevant blood banks, which can accept or decline it. The status of the request flows back into the dispatch record so the dispatcher sees where the blood request stands.

### 4.5 With the Incident Module

The emergency that comes into the dispatch module is also an incident in the incident module. The dispatch module records the operational coordination - what was assigned, what happened, what the outcome was - while the incident module records the event itself. The two are linked: for any incident you can see what response was coordinated and what the outcome was.

### 4.6 With the Public Interface

The public interface is one of the ways emergencies come into the system. A citizen reports an emergency through it, and the dispatch module receives it as an incoming request. The dispatch module can send updates back to the public interface - "your emergency has been received," "an ambulance has been dispatched" - but the public interface sees only the reporter's own emergency and nothing else.

---

## 5. How Real-Time Updates Work

### 5.1 The Live Dispatch Picture

The dispatch desk sees a live picture of everything happening - all active emergencies, all available resources, all assignments in progress. When a new emergency comes in, it appears immediately. When an ambulance's status changes, the desk sees it immediately. When a hospital's bed availability changes in a way that matters to an open case, the desk sees it. The dispatcher works from what is happening now, not from a snapshot that is minutes old.

### 5.2 Status Changes Flow Through the Dispatch Record

When an ambulance's status changes - dispatched, en-route, on-scene, transporting, arrived at hospital - that change is recorded in the timeline of the dispatch. The dispatch record is not just a record of what was assigned; it is a live timeline of what happened as the response progressed.

### 5.3 Notifications at Each Stage

At each stage, the relevant parties are notified through the notification service: the ambulance crew when assigned, the destination hospital when the ambulance is en-route, the blood bank when a request is placed, the hospital when the patient arrives, and the reporter if the system provides updates. The dispatch module records what was sent and when.

### 5.4 Changes and Overrides

If circumstances change - a higher-priority emergency comes in, the destination hospital becomes full, a blood request needs modification - the dispatcher makes the change, and the dispatch module records it with reason and timestamp. The change flows to the affected modules and parties are notified. The dispatch is a living coordination that adapts as the situation evolves, not a rigid plan set at the start.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

Today, in most cities and regions, emergency dispatch is fragmented and manual:

- **No unified dispatch centre that sees everything.** Each operator - the government emergency number, each hospital, each private ambulance service - dispatches its own resources from its own limited view. One operator's dispatcher does not see another operator's ambulances or partner hospitals.
- **Requests come through separate channels.** A citizen calls one number, a hospital calls another, an ambulance crew reports by radio or phone. These requests are handled by different people in different places, with no shared picture.
- **Resource matching is manual and local.** A dispatcher finds an ambulance, a hospital, or blood by calling people they know. Each is a separate phone call, limited to the dispatcher's own network.
- **No record of the dispatch.** What was requested, assigned, and the outcome is often not recorded at all, or only in informal notes.
- **No coordination across resources.** The ambulance dispatch, hospital selection, and blood arrangement happen as separate actions, often by different people, with no shared record tying them together.
- **No notification flow.** The hospital does not know an ambulance is coming until told; the blood bank does not know its blood is needed until called; the reporter hears nothing after reporting.
- **No analytics.** With no coherent dispatch record, there is no data on response times, outcomes, or utilization. Improvement is based on anecdote and memory.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| No unified dispatch centre | A single dispatch module that sees all emergencies, resources, and assignments across all operators, hospitals, and blood banks in one live picture |
| Separate intake channels | A single intake for emergencies - from the public interface, hospitals, ambulances, or other sources - all entering one system |
| Manual, local matching | A matching process that looks across all available resources and recommends the best matches by distance, availability, type suitability, and urgency |
| No dispatch record | A full dispatch record - what was requested, assigned, what happened, the outcome - with a complete timeline pulled from the modules tracking each part of the response |
| No coordination across resources | A coordinated dispatch that brings ambulance, hospital, and blood together as one response to one emergency, with a single record tying them together |
| No notification flow | Automated notifications at each stage - crew, hospital, blood bank, and reporter where appropriate |
| No analytics from dispatch | A dispatch record that feeds analytics - response times, outcomes, utilization, demand patterns - so improvement is based on data, not anecdote |

### 6.3 How It Is Better

- **For the dispatcher:** a complete picture instead of a fragmented one; recommendations with reasoning instead of starting from zero; a coherent record instead of scattered notes.
- **For the ambulance crew:** clear assignments with context - emergency details, location, destination - in one place, and automatic notification of changes.
- **For the hospital:** advance notice of incoming patients with emergency details and expected arrival, so they can prepare, rather than finding out when the ambulance arrives.
- **For the blood bank:** clear requests with group, quantity, urgency, and reason, responded to through the system with tracked status.
- **For the public:** confirmation their report was received, and updates on the response if the system provides them - reducing anxiety and building trust.
- **For oversight:** a complete, reviewable record of every dispatch, and analytics based on real data.

---

## 7. How This Module Integrates With the Rest of the System

- **It coordinates the other modules, it does not replace them.** The ambulance module owns ambulances, the hospital module owns hospitals, the blood bank module owns blood banks. The dispatch module sits above them and directs them as part of one coordinated response.
- **It depends on the auth module** for identity and access, like every other module. The dispatcher role allows full coordination; the platform administrator role allows oversight without day-to-day coordination.
- **It receives emergencies from the public interface and the incident module** - those are the entry points through which emergencies reach the dispatch module.
- **It feeds back into the modules it coordinates** - assignment to the ambulance module, destination notice to the hospital module, requests to the blood bank module - and those modules feed status updates back into the dispatch record.
- **It feeds analytics.** The dispatch record, with all its timestamps, is what makes response-time and outcome analytics possible.

---

## 8. Privacy and Safety Considerations

### 8.1 What We Do Not Track

- **Patient identity or personal health information.** The emergency is tracked in operational terms - what is happening, where, what resources are needed - not as a patient record.
- **Reporter personal information beyond what is needed to respond.** Contact details are recorded only as much as is operationally necessary to reach the scene or follow up.
- **Information about bystanders or others not part of the response.**

### 8.2 What We Do Track

The emergency in operational terms; the resource requirements; the matching and recommendations with reasoning; the assignments and their timeline; the notifications sent; the outcome and response-time measures. All of it operational data about the emergency response.

### 8.3 Who Sees the Dispatch Record

Dispatchers see all active and recent dispatches; the platform administrator sees everything for oversight; ambulance crews see only their own assignments; hospital staff see only emergencies coming to them; the public sees only their own reported emergency. Each role sees what it needs to do its job, and nothing more.

### 8.4 The Dispatch Record as an Audit Trail

The dispatch record is an audit trail for the emergency response - what was done, when, by whom, with what outcome. If something goes wrong, there is a system record to review, not a "I don't remember" situation.

### 8.5 Why We Do Not Store Patient Data

Storing patient data would open privacy, consent, and health-data-regulation concerns appropriate for an EHR system, not for a resource-coordination demonstrator. Keeping patient data out of scope is a deliberate choice that keeps the project focused. If patient tracking is ever needed, it would be a separate module with its own compliance story.

---

## 9. What We Are Explicitly Not Doing

- **Replacing emergency call centres or emergency numbers.** This module coordinates resources after an emergency is reported; it does not replace call handling, phone triage, or pre-arrival instructions.
- **Triage or medical assessment.** The module may receive a triage assessment from a trained person or tool and use it for prioritization, but it does not perform clinical triage itself.
- **Medical decision-making.** The system supports decisions with information and recommendations; the final decision about what a patient needs rests with medical professionals.
- **Direct communication with the patient.** The module coordinates resources; it does not provide medical advice or pre-arrival instructions.
- **Real-time video or audio from the scene.** Not in scope; that is a separate communication function.
- **Predictive or AI-based dispatch in this project.** Matching is rule-based on current state. Predictive dispatch is a future extension, not part of the core project.
- **A separate mass-casualty mode.** The module handles multiple concurrent emergencies with the same process; a dedicated mass-casualty mode with special logic is not part of this project.

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Coordinate emergency responses across all resources - receive emergency requests, match resources, assign them, track the response, and record the complete dispatch from start to finish |
| Data tracked | Emergency request (location, type, description, urgency, reporter, timestamp); resource requirements; matching and recommendations with reasoning; assignments with full timeline; notifications sent; outcome and close with response-time measures |
| Who gets access | Dispatchers see and coordinate everything; ambulance crews see their own assignments; hospital staff see emergencies coming to them; platform administrator sees everything for oversight; the public sees only their own reported emergency |
| How data is shared | The dispatch desk sees one live picture; the module coordinates with the ambulance, hospital, and blood bank modules for assignments and status, receives emergencies from the public interface and incident module, and records everything as one coherent dispatch |
| Gaps addressed | No unified dispatch centre, separate intake channels, manual local matching, no dispatch record, no cross-resource coordination, no notification flow, no analytics - each introduced by this module |
| What it does not do | Replace call centres, perform triage, make medical decisions, communicate with patients directly, handle live video/audio, predictive dispatch, or mass-casualty mode in this project |
| Privacy stance | No patient-level data; emergencies tracked in operational terms; each role sees only what it needs; the dispatch record is an auditable account of the coordination |

---

