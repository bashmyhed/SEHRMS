# Incident & Request Management Module — Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Incident & Request Management Module is the entry point for all emergency events into the system. It tracks the lifecycle of an emergency incident — from the initial report, through triage and classification, resource requirement specification, active response coordination, to final resolution.

This module answers the question: what are we dealing with? What happened, where, when, how urgent is it, what resources does it need, who reported it, and what is the current status?

It does not handle the actual resource coordination — that is the dispatch module's job. It does not track resource availability — that is the hospital, ambulance, and blood bank modules' job. It is the "what is happening" layer that feeds into the "how do we respond" layer.

---

## 2. What Data We Track

### 2.1 The Incident Record

When an emergency enters the system, the module records:

- **A unique incident identifier** — both an internal identifier and a human-readable incident number (for example, a dated sequence number) so hospitals, ambulance crews, and reporters can refer to it easily over the phone or in follow-ups
- **Who reported it and how** — a citizen through the public interface, a dispatcher taking a call, a hospital calling in a referral, or an ambulance crew reporting a situation they encountered; plus contact information if the reporter provides it, for follow-up
- **Where it is happening** — a human-readable location description plus geographic coordinates, either captured automatically from the reporter's device or converted from an address
- **What type of emergency it is** — trauma or accident, cardiac, respiratory, neurological, obstetric, paediatric, burns, poisoning, drowning, violence or assault, psychiatric emergency, or other medical/trauma — with a more specific sub-type where known
- **A brief description** — what the reporter can say about the situation, in operational terms
- **Basic situational facts** — how many patients are involved if more than one, and age range or gender if known; never names, never medical history
- **When it was reported** — the timestamp that starts the clock on the whole response

We do not store patient names, medical records, or any personal health information. The incident is an event-level record — what is happening and where — not a patient record. Reporter contact is collected only if provided and only as much as is needed for follow-up. The public can report an emergency anonymously.

### 2.2 Triage and Priority

Once the emergency is reported, the module records the triage assessment:

- **A priority category** — from immediate and life-threatening, through urgent, down to minor cases that may not need an ambulance at all
- **The basis for the triage** — what signs or circumstances led to the classification, so another dispatcher can understand and, if needed, revise it
- **Who set or updated the triage and when** — triage can be revised as more information arrives, for example when the crew reaches the scene and finds the situation is worse than reported; every revision is recorded with the reason

Triage in this system is advisory, not diagnostic. The system can suggest a priority based on what is reported, but a dispatcher confirms it. It cannot and does not replace medical judgment, and that is stated plainly in the design.

### 2.3 Resource Requirements

From the incident type, description, and priority, the module records what the emergency needs:

- An ambulance, and if so what kind — a basic life support unit or an advanced life support unit, for example
- A hospital bed, and if so what category — an ICU bed for trauma, a cardiac care bed for a cardiac emergency, a maternity bed for an obstetric case
- Blood, and if so which group and how many units
- Any other resource the system is designed to handle, such as a ventilator or a specialty unit

Each requirement carries its own status — pending, fulfilled, partially fulfilled, or no longer needed — so the incident record shows at a glance whether everything the emergency needs has actually been arranged. Requirements can be suggested automatically from the incident type and then adjusted by the dispatcher; the suggestion is never binding.

### 2.4 Lifecycle and Status

The incident moves through a clear lifecycle, and the module records every step:

- Reported, then triaged, then resources being matched, then resources assigned, then active response with the ambulance en-route, then in transit, then arrived at hospital, then handover complete, then resolved
- With additional outcomes for cases that end differently — resolved on scene, cancelled, or the ambulance unable to reach the patient

Every transition is timestamped and logged along with who made it. Invalid jumps — for example, going straight from reported to resolved — are not allowed. If an incident sits in one status for too long, the system can flag it for the dispatcher's attention.

### 2.5 The Timeline

Every action on the incident is recorded in a chronological timeline: reported, triaged, requirements added, ambulance assigned, ambulance arrived at scene, departed scene, arrived at hospital, handover, resolved — each with a timestamp and the person or module that made it. The timeline links the incident to the dispatch record, the destination hospital, any blood requests, and the crew involved. This is what gives the system end-to-end traceability: for any incident, you can see the full story of what happened from report to resolution.

### 2.6 Outcome and Close

When the incident is resolved, the module records the outcome — the patient was admitted, transported to another hospital, the situation was resolved on scene, the ambulance could not reach the patient, or any other outcome — along with brief notes and the closing timestamp. The timestamps across the timeline are what make response-time measurement possible: time from report to dispatch, dispatch to arrival, arrival to handover, and total response time.

---

## 3. Who Gets Access to What

### 3.1 Dispatcher

The dispatcher has full access, because coordination is their job:

- Create incidents from any channel — a phone call, a public report, a hospital referral, a crew report
- See all active incidents in a list and on the map, filterable by type, priority, and area
- See full detail for any incident — the timeline, the requirements, the linked dispatch and resources
- Update triage, requirements, and status as the response progresses
- Search the full history of past incidents for follow-up and review

### 3.2 Hospital Staff

Hospitals see incidents that concern them:

- Incidents that are heading to their hospital — so they know what is coming, what it needs, and roughly when
- The ability to create referral incidents — for example, "we have a patient in our emergency department who needs an ICU transfer to another facility" — which enters the system as an incident reported by a hospital
- They update the incident when the patient arrives and is handed over

They do not see incidents going to other hospitals.

### 3.3 Ambulance Crew

The crew sees the incidents assigned to their unit through the dispatch module — where to go, what is known about the situation, and what the patient needs. They update the incident's progress from their side: arrived at scene, departed scene, arrived at hospital. Crews can also create incidents when they encounter a situation on the road — reporting an accident they pass becomes an incident in the system. They do not see incidents not assigned to them.

### 3.4 The Public

A citizen who reports an emergency can:

- Report without logging in — in an emergency, identity checks would only slow people down; a phone number is encouraged for follow-up but not required
- Receive an incident number and, with it, a basic status view of their own incident — "received," "ambulance dispatched," "on the way to hospital," "handover complete"
- See only their own incident, never anyone else's

### 3.5 Platform Administrator

The platform administrator sees the full incident history across the system for oversight, review, and analytics. They do not do the day-to-day incident management — that is the dispatcher's work.

---

## 4. How Data Is Shared

### 4.1 The Flow Into Dispatch

The incident module and the dispatch module work as a pair. The incident records what is happening and what is needed; the dispatch module takes that and coordinates the response. The flow is: incident created, triage done, resource requirements derived, the dispatch module matches and assigns resources, and the resulting dispatch record is linked back to the incident. From that point, as the ambulance and hospital update their statuses, those updates flow back into the incident's timeline. The incident and its dispatch are one connected story.

### 4.2 With the Resource Modules

- A requirement for an ambulance links to the ambulance module's assignment
- A requirement for a bed links to the hospital module's allocation
- A requirement for blood links to the blood bank module's request system

Each requirement's status — pending or fulfilled — is tracked on the incident, so the incident record always shows whether everything needed has actually been arranged.

### 4.3 With the Notification Service

Status changes trigger notifications to the right people: the reporter is told an ambulance has been dispatched, the hospital is told a patient is en-route, the crew is told their assignment. The incident module records what happened; the notification service delivers the messages.

### 4.4 With the Analytics Module

Incident records are one of the richest sources for analytics: the distribution of incident types, where emergencies happen, how urgency varies by time of day and season, and — combined with dispatch records — how long each stage of the response takes. Every incident contributes to that picture.

---

## 5. How Real-Time Updates Work

When an incident is created, it appears on the dispatch desk's map immediately, colour-coded by priority. When its status changes — resources assigned, ambulance en-route, arrived at hospital — the change flows through the timeline and the relevant parties are notified without anyone refreshing anything. If several reports come in for what appears to be the same event at the same location — a multi-vehicle accident reported by several callers, for example — the system flags the possible duplicates for the dispatcher to review rather than merging them automatically, and the map can show them clustered.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

- **Incidents are often not recorded in a structured way.** Emergency calls are logged in call-centre notes or not logged at all. There is no single incident record that follows the emergency from report to resolution.
- **Triage is inconsistent.** Prioritisation depends on the experience of whoever takes the call, with no structured guidance and no record of why a priority was chosen.
- **Resource needs are not formally specified.** "Send an ambulance" is the typical level of detail; what kind of ambulance, what kind of bed, whether blood is needed — these are figured out ad hoc, if at all.
- **No shared visibility.** The hospital does not know an incident is coming until the ambulance arrives or someone calls. The reporter hears nothing after making the call.
- **No historical incident database.** Because incidents are not recorded in a structured form, no one can analyse patterns — where accidents cluster, when cardiac cases peak, how long responses take. Improvement is based on memory.
- **No public self-reporting channel.** If someone cannot call — a hearing-impaired person, or someone in a situation where calling is unsafe — there is no other way to report.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| No structured incident record | A structured incident record with a full timeline from report to resolution |
| Inconsistent triage | Structured triage with guidance and a recorded basis, so prioritisation is consistent and reviewable |
| Resource needs not specified | Formal resource requirements on every incident, each with a tracked status |
| No shared visibility | Role-scoped visibility — dispatcher sees everything, hospital sees what is coming to it, crew sees its assignments, reporter sees their own incident's status |
| No historical database | A searchable incident history that feeds analytics — patterns, hotspots, response times |
| No public self-reporting | A public reporting channel that works alongside phone calls, with anonymous reporting allowed |

### 6.3 How It Is Better

- **For the dispatcher:** every emergency is a structured record with what it needs already specified, so coordination starts from information, not from a vague "send help."
- **For the hospital:** advance notice of what is coming and what it needs, so preparation starts before the ambulance arrives.
- **For the reporter:** confirmation that the report was received and a way to track what is happening — which reduces anxiety and builds trust.
- **For planning and oversight:** a real incident database for the first time, which is what makes hotspot identification, response-time measurement, and data-driven improvement possible.

---

## 7. How This Module Integrates With the Rest of the System

- **Dispatch module** — the tightest integration: the incident provides location, type, triage, and requirements; the dispatch module coordinates resources and links the dispatch record back; status updates flow in both directions so the incident timeline reflects the live response.
- **Hospital module** — hospitals create referral incidents, see incidents heading to them, and update the incident on handover.
- **Ambulance module** — crews create incidents they encounter and update the incident's progress through their status changes.
- **Blood bank module** — blood requirements on an incident link to the blood request system, with fulfilment status tracked back on the incident.
- **Notification service** — status changes trigger notifications to the reporter, hospital, and crew.
- **Public interface** — the citizen reporting form is one of the intake channels for incidents.
- **Analytics module** — incident records feed demand patterns, hotspots, and response-time measurement.
- **Auth module** — as everywhere, identity and role-based access come from the universal auth module; the incident module applies its own scoping on top.

---

## 8. Privacy and Safety Considerations

### 8.1 Event-Level Only, No Patient Data

The incident is an event record, not a patient record. No names, no medical history, no personal health information. This keeps the module useful for coordination and analytics while staying clear of patient-data compliance concerns. If patient-level tracking were ever needed, it would be a separate module with its own compliance story.

### 8.2 Anonymous Public Reporting

The public can report without an account. A contact number is encouraged for follow-up but never required. This matters for accessibility — people who cannot call still have a way to report.

### 8.3 The Public Form Is Not a Replacement for Emergency Numbers

This is stated prominently on the public interface: for immediate life-threatening emergencies, the caller should call the emergency number first. The public reporting form is a supplementary channel, not a replacement for emergency call infrastructure. This is a responsibility decision, not just a design note.

### 8.4 Triage Is Advisory

The system suggests a priority; a person confirms it. Nothing in this module makes a medical decision, and the design is explicit about that boundary.

### 8.5 Who Sees What

Each role sees only its own slice: the dispatcher sees everything operationally relevant, hospitals see only their incoming incidents, crews only their assignments, reporters only their own incident. No role sees more than it needs.

---

## 9. What We Are Explicitly Not Doing

- **Patient medical records or personal health information** — the incident is event-level only.
- **Detailed clinical documentation** — not in scope; this is coordination, not clinical care.
- **Law enforcement or crime scene management** — this system is healthcare-focused.
- **A structured mass-casualty mode** — the incident record supports a patient count for simple multi-patient cases, but a full mass-casualty workflow with triage tags and resource scaling is a future extension, not part of this project.
- **Predictive incident forecasting** — predicting where and when emergencies will happen is an analytics extension for later, built on the incident history this module creates.
- **Integration with police or fire dispatch systems** — separate systems, out of scope.
- **Automatic duplicate merging** — likely duplicates are flagged for a dispatcher to review; the system never silently merges two reports.

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Track emergency incidents from report through triage, resource specification, active response, and resolution — structured incident records that feed dispatch and analytics |
| Data tracked | Incident identity and number, reporter information and channel, location with coordinates, type and sub-type, description, patient count, triage with basis and revision history, resource requirements each with status, lifecycle status with history, linked dispatch/hospital/blood records, outcome, timestamps |
| Who gets access | Dispatcher: full creation and management. Hospital: incoming incidents and referral creation. Ambulance crew: assigned incidents, crew-reported incidents, progress updates. Public: anonymous reporting and status view of their own incident. Platform admin: full history for oversight |
| How data is shared | Incident feeds requirements to dispatch, dispatch links back, status updates flow through the ambulance and hospital modules into the incident timeline, notifications go out at each stage, analytics consumes the history |
| Gaps addressed | No structured incident records today, inconsistent triage, unspecified resource needs, no shared visibility, no historical database, no public self-reporting channel — each introduced by this module |
| What it does not do | Patient records, clinical documentation, law enforcement management, mass-casualty mode, predictive forecasting, police/fire integration, automatic duplicate merging |
| Privacy stance | Event-level only, no patient data; anonymous public reporting allowed; public form explicitly supplementary to emergency numbers; triage is advisory, confirmed by a person; role-scoped visibility throughout |
| Integration | Works with dispatch (coordination), hospital (destination and referrals), ambulance (crew reports and progress), blood bank (blood requirements), notification service (status updates), public interface (citizen intake), analytics (incident history), and auth (identity and roles) |

---

*This report keeps the same text-focused, minimal-technical style as the other module reports.*