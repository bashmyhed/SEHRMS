# Public Emergency Interface Module - Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Public Emergency Interface Module is the bridge between the public and the system. It is how a citizen reports an emergency when they are not calling an emergency number, and how they find the help they need - the nearest hospital with an available bed, the nearest blood bank with their blood group, and, if the system provides it, the status of the emergency they reported.

Specifically, this module is responsible for:

- Giving citizens a way to report an emergency through a simple web form - location, what is happening, how urgent it seems - and creating an incident in the incident module from that report
- Giving citizens a way to find the nearest hospital with the resource they need - a general bed, an ICU bed, a specific kind of care - through a public map or finder
- Giving citizens a way to find the nearest blood bank with the blood group they need, through the same public interface
- Giving citizens who have reported an emergency a way to see the status of their report - whether it was received, whether an ambulance has been dispatched, whether the ambulance is on its way, whether the patient has been handed over - if the system is designed to provide such updates
- Making sure that the public interface is clear about what it is and what it is not - that it is a supplementary tool, not a replacement for the emergency numbers, and that for immediate life-threatening emergencies, calling the emergency number is the right thing to do

This module does not replace the emergency call centre. It does not do triage. It does not coordinate the response. It is the public's entry point into the system, and the public's window into what the system can show them.

---

## 2. What Data We Track

### 2.1 Emergency Report

When a citizen reports an emergency through the public interface, the module records:

- Where the emergency is happening - ideally from the citizen's device location, if they are on a phone or a browser that shares location, or from an address they enter. The location is geocoded if it is an address, so the system can place it on the map and the incident module can use it.
- What is happening - a description of the emergency, in the citizen's own words. This is passed to the incident module as the description of the emergency, and it is used for triage and for the response.
- How urgent the citizen thinks it is - a simple indication from the citizen, like "this is life-threatening" or "this is urgent but not immediately life-threatening" or "this is not urgent." This is not a medical triage - it is the citizen's own sense of urgency - and it is used as information for the triage process, not as the triage itself.
- Who is reporting - optionally, a phone number or email, so the system can follow up or give updates. This is optional, because the system should let people report emergencies even if they do not want to give their contact information. But if they do give it, the system can use it to send them updates on their report.
- When the report was made - the timestamp that starts the incident.

We do not store the citizen's name, unless they choose to give it. We do not store their precise location after the report is made, beyond what is needed for the incident. We do not store their device information, their IP address, or anything beyond what is needed to handle the emergency report and, if they choose, to give them updates.

The emergency report is not a patient record. It is a report of an event, from a member of the public, and it is treated as such. The incident module receives it and handles it as an incident - it does not treat the reporter as a patient, and it does not store the reporter's personal health information.

### 2.2 Public Map Queries

When a citizen uses the public map to find a hospital or a blood bank, the module records:

- What they are looking for - a hospital with a general bed, a hospital with an ICU bed, a blood bank with a specific blood group
- Where they are looking from - their location, or a location they enter
- What results they see - the nearest hospitals or blood banks that match, with the coarse availability information that the system shows publicly

We do not store these queries as personal data. We may record them in aggregate, for analytics - for example, to know which blood groups are most often searched for, or which areas people are searching from - but we do not record each query as a record of a specific person looking for something. The public map is a tool for the public to use, not a tracking mechanism.

### 2.3 Status Updates for Reporters

When the system provides updates to a citizen who has reported an emergency, the module records:

- That the citizen chose to receive updates, and through which channel - SMS, email, or both, if they provided contact information
- What updates were sent - "your emergency has been received," "an ambulance has been dispatched," "the ambulance is on its way to the hospital," "the patient has been handed over" - and when
- Whether the updates were delivered - through the notification service's delivery records

This is not a message archive. It is a record of the updates that were sent to the reporter, linked to the incident they reported. The substantive content of the updates is in the notification service's records and in the incident's timeline. This module records that the updates were offered and sent, and that the reporter chose to receive them.

---

## 3. Who Gets Access to What

### 3.1 The Public

The public is the main user of this module. They can:

- Report an emergency through a simple web form
- Find the nearest hospital with a resource they need, through the public map
- Find the nearest blood bank with a blood group they need, through the public map
- If they have reported an emergency, see the status of their report, if the system is designed to provide updates

The public does not need to log in to use any of these features. The public interface is open, because emergencies do not wait for people to create accounts. If a citizen wants updates on their report, they may need to provide contact information, but they do not need an account to report an emergency or to find a hospital or blood bank.

### 3.2 What the Public Sees

The public sees:

- A simple emergency report form, with a clear message that for immediate life-threatening emergencies, calling the emergency number is the right thing to do
- A map that shows hospitals and blood banks, with coarse availability information
- The ability to search for a hospital or blood bank by location and resource
- If they have reported an emergency, the status of their report, if the system provides updates

The public does not see:

- The internal dispatch records
- Other people's emergency reports
- The exact bed counts or blood unit counts - only the coarse availability indicators
- Any information about the system's internal workings

This is the right scope. The public gets enough to act - to report an emergency, to find a hospital or blood bank, to track their own report - without seeing the internal workings of the system or the data of other people.

### 3.3 The Platform Administrator

The platform administrator can see:

- The aggregate usage of the public interface - how many emergencies were reported through it, how many map queries were made, how many people used the public interface to find hospitals or blood banks
- Any problems with the public interface - if reports are not being created properly, if the map is not showing the right data, if the updates are not being sent

The platform administrator does not see the content of individual emergency reports as a stream of personal data. They can see that reports are coming in and that the system is handling them, but they do not browse individual reports unless there is a specific need to investigate something.

### 3.4 What the Public Does Not Access

The public does not access the dispatch module, the hospital module's internal data, the ambulance module, the blood bank module's internal data, or the auth module. They access only the public interface, which shows them a limited, carefully-designed view of what the system can show the public.

This is a deliberate design. The public interface is a thin layer on top of the system, showing only what is appropriate for the public to see. Everything else is behind the access controls of the other modules.

---

## 4. How Data Is Shared

### 4.1 Emergency Report to Incident Module

When a citizen reports an emergency, the public interface creates an incident in the incident module. The incident module receives the report - location, description, urgency, reporter contact - and creates an incident record. The incident module then handles the incident through its normal workflow - triage, resource matching, dispatch, response, resolution.

The public interface does not handle the incident itself. It creates the incident and passes it to the incident module, which is the module that manages incidents. The public interface's job is to make it easy for citizens to report emergencies, and to create the incident record that the rest of the system can act on.

### 4.2 Status Updates to the Reporter

If the system is designed to provide updates to the reporter, the public interface is the channel through which the reporter can see those updates. The reporter can look at the public interface - by returning to the page where they reported the emergency, or by using an incident ID and their contact information to retrieve their report's status - and see the current status of their emergency.

The updates come from the notification service and from the incident module's timeline. The public interface shows them to the reporter in a simple, easy-to-understand form - "your emergency has been received," "an ambulance has been dispatched," "the ambulance is on its way," "the patient has been handed over at the hospital." The public interface does not show the internal dispatch record or the detailed timeline - it shows only the updates that are appropriate for the reporter to see.

### 4.3 Public Map to Hospital and Blood Bank Modules

The public map shows data from the hospital module and the blood bank module - but only the data that those modules are designed to show publicly. The hospital module provides the names, types, locations, and coarse availability of hospitals. The blood bank module provides the names, locations, and coarse availability of blood banks by blood group.

The public interface does not get the full data from these modules. It gets the public-facing view that the modules are designed to expose. The hospital module and the blood bank module control what the public sees, through the access controls and the coarse-data design that are part of those modules.

This means the public interface is not a backdoor into the system. It shows only what the system is designed to show the public, and nothing more.

### 4.4 Aggregate Analytics to Platform Administrator

The public interface shares aggregate usage data with the platform administrator - how many emergencies were reported, how many map queries were made, how people are using the public interface. This is useful for understanding how the public is using the system, and for improving the public interface.

The aggregate data is not personal. It does not identify individual reporters or individual map users. It is the kind of data that helps the platform administrator understand the system's public usage, without exposing personal information.

---

## 5. How Real-Time Updates Work

### 5.1 Emergency Report Submission

When a citizen submits an emergency report, the public interface immediately creates the incident and shows the citizen a confirmation - the incident number, a summary of what was reported, and what happens next. The confirmation is immediate, so the citizen knows that the report was received and is being acted on.

The incident is created in the incident module at the same time, so the rest of the system can act on it right away. The dispatcher sees the new incident, the triage process starts, and the response coordination begins. The public interface's confirmation to the citizen is the first step in the response, and it happens immediately.

### 5.2 Status Updates for Reporters

If the system provides updates, the reporter can see the current status of their emergency whenever they check the public interface. The status is not a live feed in the sense of pushing updates to the reporter automatically - though the system may also send updates through SMS or email, through the notification service - but it is a current status that the reporter can check at any time.

The status shown is simple and clear. It does not show the internal details of the dispatch. It shows the stages that matter to the reporter - received, dispatched, on the way, handed over - in plain language, so the reporter can understand what is happening without needing to know the internal workings of the system.

### 5.3 Public Map Availability

The public map shows the current coarse availability of hospitals and blood banks. This availability is updated as the hospital module and the blood bank module update their data, and the public map reflects those updates. The map is not real-time in the sense of showing every change the moment it happens - it is updated on a reasonable schedule, so that the public sees current but not erratic information.

This is the right balance. The public map shows current availability, so that citizens can make informed decisions, but it does not show every change the moment it happens, which could be confusing or could expose too much information. The map is current, not live in the most granular sense.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

Today, the way the public interacts with emergency services is limited and often unclear:

- **The main way to report an emergency is to call an emergency number.** This is the right way for immediate, life-threatening emergencies, and it should remain the main way. But not everyone can call - some people have hearing or speech disabilities, some people are in situations where calling is dangerous or not possible, some people are not aware of the emergency number, and some people prefer to use a digital interface. For these people, there is often no alternative way to report an emergency.
- **Finding a hospital or blood bank is difficult.** A citizen who needs to find a hospital with an available ICU bed, or a blood bank with a specific blood group, often has no good way to do this. They may call hospitals one by one, or search online and find outdated or unreliable information, or go to the nearest hospital and hope it has what they need. There is no public, live, reliable source of this information.
- **There is no way to track an emergency report.** If a citizen reports an emergency through a non-emergency channel - for example, a municipal app or a helpline that is not the emergency number - they often have no way to know what happened with their report. They report it, and then they wait, with no confirmation, no status, no idea whether anyone is acting on it.
- **Public information about hospitals and blood banks is often outdated or unreliable.** Websites and apps may show hospital locations and contact information, but they rarely show current availability, and they almost never show blood bank availability by blood group. The information that exists is often static, outdated, or not trustworthy.
- **There is no simple, clear public interface for emergency help.** A citizen in an emergency needs simple, clear, actionable information - where to go, what to do, who to call. Most public interfaces do not provide this in a focused, emergency-appropriate way.

This is the baseline. It is not theoretical - it is how the public interacts with emergency services in most places today.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| No alternative way to report an emergency beyond calling | A simple web form for reporting emergencies, for people who cannot or prefer not to call, with location capture and immediate confirmation |
| No easy way to find a hospital with the needed resource | A public map that shows the nearest hospitals with the resource a citizen needs - general bed, ICU bed, specific care - with coarse availability and distance |
| No easy way to find a blood bank with the needed blood group | A public map that shows the nearest blood banks with the requested blood group, with coarse availability and contact information |
| No way to track an emergency report | For reporters who choose to receive updates, a way to see the status of their emergency report - received, dispatched, on the way, handed over - in plain language |
| Public information about hospitals and blood banks is outdated or unreliable | A live, system-sourced view of hospital and blood bank availability, coarse but current, that the public can rely on |
| No simple, clear public interface for emergency help | A focused, emergency-appropriate public interface that gives citizens simple, actionable information - report an emergency, find a hospital, find a blood bank, track a report |

### 6.3 How It Is Better - The Concrete Improvements

For the citizen reporting an emergency:

- **An alternative way to report.** If they cannot call, or prefer not to call, or are in a situation where calling is not possible, they can report the emergency through the public interface. The report is received immediately, and the incident module starts the response.
- **Immediate confirmation.** The citizen knows right away that the report was received. This reduces anxiety and uncertainty, and it tells the citizen that the system is acting on their report.
- **Updates on their report, if they choose.** The citizen can see the status of their emergency - received, dispatched, on the way, handed over - which gives them some visibility into the response and reduces the feeling of being left in the dark.

For the citizen looking for a hospital or blood bank:

- **A reliable, live source of information.** The citizen can find the nearest hospital with the resource they need, or the nearest blood bank with the blood group they need, through the public map. The information comes from the system, not from outdated websites or unreliable sources.
- **Actionable information.** The map shows distance, location, and coarse availability, so the citizen can make a decision - go to this hospital, call this blood bank - based on current information.

For the system:

- **More ways for emergencies to be reported.** The public interface adds a channel for emergency reports, which means more emergencies can be captured by the system, and the system's data and response are more complete.
- **Public visibility of the system's value.** The public map shows the public what the system offers - live hospital and blood bank availability - which builds awareness and trust in the system.
- **Data on public usage.** The system can see how the public uses the interface - how many emergencies are reported, how many map queries are made, what people are searching for - which helps the system understand its public users and improve the interface.

---

## 7. How This Module Integrates With the Rest of the System

### 7.1 It Creates Incidents in the Incident Module

The public interface is one of the ways emergencies enter the system. When a citizen reports an emergency, the public interface creates an incident in the incident module. The incident module then handles the incident through its normal workflow - triage, resource matching, dispatch, response, resolution.

The public interface does not handle the incident itself. It creates the incident and passes it to the incident module. The incident module is the module that manages incidents, and the public interface is the channel through which some incidents are reported.

### 7.2 It Shows Data from the Hospital and Blood Bank Modules

The public map shows data from the hospital module and the blood bank module - but only the public-facing data that those modules are designed to expose. The hospital module provides the names, types, locations, and coarse availability of hospitals. The blood bank module provides the names, locations, and coarse availability of blood banks by blood group.

The public interface displays this data on the map, with the search and filtering that the public needs - find the nearest hospital with an ICU bed, find the nearest blood bank with O-negative, and so on. The public interface is the presentation layer for this data, and the hospital and blood bank modules are the data sources.

### 7.3 It Links to the Notification Service for Reporter Updates

When a citizen reports an emergency and chooses to receive updates, the public interface records that choice and links it to the incident. The notification service then sends updates to the citizen through the channels they chose - SMS, email, or both - as the incident progresses. The public interface also lets the citizen check the status of their incident directly, by returning to the page where they reported it or by using an incident ID and their contact information.

The public interface is the channel through which the citizen sees updates, and the notification service is the mechanism that sends them. The incident module provides the status information that the updates are based on.

### 7.4 It Depends on the Auth Module Only for Optional Account Features

The public interface does not require users to log in. Reporting an emergency, finding a hospital or blood bank, and checking the status of a report can all be done without an account. This is important - emergencies do not wait for people to create accounts, and the public interface should not require account creation for its core functions.

If the system offers optional account features for the public - for example, a way to save preferred hospitals or blood banks, or a way to manage notification preferences - those features would use the auth module. But the core public interface functions do not require the auth module, because they do not require login.

### 7.5 It Feeds Aggregate Analytics

The public interface shares aggregate usage data with the analytics module - how many emergencies were reported through it, how many map queries were made, what people are searching for, how the public is using the system. This data helps the platform understand its public users and improve the public interface.

The aggregate data is not personal. It does not identify individual reporters or individual map users. It is the kind of data that helps the system improve, without exposing personal information.

---

## 8. Privacy and Safety Considerations

### 8.1 What We Do Not Store

- **The citizen's name, unless they choose to give it.** The emergency report does not require a name. If the citizen chooses to give their name, it is stored only as much as is needed to handle the report and, if they choose, to give them updates.
- **The citizen's precise location after the report is made.** The location is used for the incident and for the response, and it is stored as part of the incident record - the incident module stores it, not the public interface as a separate location record. The public interface does not keep a persistent record of the citizen's location beyond what is needed for the incident.
- **The citizen's device information, IP address, or other technical data.** The public interface does not store technical data about the citizen's device or connection, beyond what is needed to serve the interface and, if necessary, to debug problems.
- **The citizen's contact information if they do not choose to give it.** The emergency report is optional about contact information. If the citizen does not give a phone number or email, the system does not have one, and it does not try to obtain one.

### 8.2 What We Do Store

- **The emergency report, as an incident.** The location, description, urgency, reporter contact (if given), and timestamp. This is stored as part of the incident record in the incident module, and the public interface creates the incident and passes it to the incident module.
- **The reporter's choice to receive updates, and their contact information if they gave it.** This is stored so that the system can send updates to the reporter, and so that the reporter can check the status of their incident. It is stored only as much as is needed for that purpose.
- **Aggregate usage data.** How many emergencies were reported, how many map queries were made, what people are searching for. This is stored for analytics, in aggregate, without identifying individual users.

### 8.3 The Emergency Reporting Form - Design for Safety

The emergency reporting form is the most sensitive part of the public interface, because it is used by people in emergencies, and because it could be misused if it gives the wrong impression. The form must be designed with care:

- **It must be clear that it is not a replacement for the emergency number.** For immediate, life-threatening emergencies, calling 108 or 112 or the local emergency number is the right thing to do. The public interface should say this clearly, prominently, and at the point where the citizen is deciding whether to report. It should not give the impression that the public interface is a substitute for the emergency number.
- **It must be simple and fast.** A person in an emergency does not have time to fill out a long form. The form should ask for the minimum needed - where, what is happening, how urgent - and should make it easy to submit. It should auto-capture location if possible, so the citizen does not have to enter it manually.
- **It must be reliable.** The form must work when the citizen needs it to work. It should be available, fast, and clear. It should not fail at the moment it is needed.
- **It must not create a false sense of security.** Submitting the form does not guarantee an immediate response. The system will act on the report, but the response depends on the resources available, the triage, and the dispatch. The form should not promise more than it can deliver.

This is a safety and responsibility consideration, not just a design consideration. The public interface is a tool for the public, and it must be designed to serve the public safely and honestly.

### 8.4 The Public Map - Design for Accuracy and Honesty

The public map shows coarse availability - not exact counts, not real-time in the most granular sense, but current, reliable, coarse information. This is the right design for the public, because:

- **Exact counts could be misused.** If the public could see exact bed counts or blood unit counts, they might make decisions based on incomplete information - assuming a hospital has enough beds because it shows available, without understanding that those beds may be committed to other patients, or that the count may have changed by the time they arrive.
- **Coarse information is still useful.** A citizen who needs an ICU bed can see which hospitals have ICU availability, in coarse terms, and can go to the nearest one. That is useful, actionable information, without the risks of exact counts.
- **The map is honest about what it shows.** The map shows coarse availability, and it is clear about that. It does not pretend to show exact, guaranteed, real-time information. It shows what it shows, and it shows it honestly.

This is the right balance. The public map gives the public useful, actionable information, without exposing data that could be misused or that could create false expectations.

### 8.5 Updates for Reporters - Design for Clarity and Honesty

If the system provides updates to reporters, those updates must be clear, honest, and appropriate:

- **They must be in plain language.** The updates should be understandable to a citizen who is not familiar with the system - "your emergency has been received," "an ambulance has been dispatched," "the ambulance is on its way to the hospital," "the patient has been handed over." Not internal jargon.
- **They must be honest about what they mean.** "An ambulance has been dispatched" means an ambulance has been assigned and is on its way - it does not mean the patient is safe, or that the emergency is resolved. The updates should not create a false sense of security.
- **They must not expose other people's information.** The updates should not include information about other patients, other emergencies, or the internal workings of the system. They should be about the reporter's own emergency, and only the information that is appropriate for the reporter to know.

This is the right design. The updates are useful and reassuring, but they are clear, honest, and appropriate.

---

## 9. What We Are Explicitly Not Doing

It is worth stating what this module does not attempt, so the scope is honest:

- **Replacing the emergency call number.** The public interface is a supplementary tool, not a replacement for 108, 112, or the local emergency number. It must be clear about this, and it must not give the impression that it is a substitute. The emergency number remains the right way to report immediate, life-threatening emergencies.
- **Performing triage.** The public interface collects the citizen's report - location, description, urgency - and passes it to the incident module. It does not perform triage. Triage is done by the incident module, or by a trained dispatcher, or by a medical professional, depending on the system's design. The public interface does not make medical judgments.
- **Providing medical advice.** The public interface does not provide medical advice. It does not tell the citizen what is wrong, what to do medically, or what to expect. It reports the emergency and helps the citizen find resources. Medical advice is not its function.
- **Tracking the patient after handover.** The public interface shows the status of the emergency report up to handover at the hospital. It does not track the patient after that - the patient's treatment, admission, outcome - because that is patient-level information and is not in scope.
- **Showing exact resource counts to the public.** The public map shows coarse availability - available, low, full - not exact counts. This is deliberate, as described in the privacy and safety section.
- **Allowing the public to see other people's emergency reports.** The public interface shows only the reporter's own emergency, if they have reported one and chosen to receive updates. It does not show other people's reports, and it does not show a live feed of all emergencies in the area. This would be a privacy violation and could cause panic or misuse.
- **Providing real-time location tracking of ambulances to the public.** The public map shows hospitals and blood banks, not ambulances. As described in the ambulance module report, showing ambulance locations publicly is not part of this project, for safety and privacy reasons.
- **Handling non-emergency requests.** The public interface is for emergencies - reporting an emergency, finding emergency resources. It is not for non-emergency hospital appointments, blood donation appointments, or other non-urgent requests. Those are different functions, and they are not part of this module.

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Give the public a way to report emergencies, to find the nearest hospital or blood bank with the resource they need, and, if they report an emergency, to see the status of their report - all through a simple, clear, safe, public interface that supplements but does not replace the emergency call number |
| Data tracked | Emergency reports - location, description, urgency, optional reporter contact, timestamp - stored as incidents in the incident module; public map queries in aggregate, not as personal data; reporter's choice to receive updates and their contact information if given; aggregate public usage data for analytics |
| Who gets access | The public: report emergencies, find hospitals and blood banks on the map, check the status of their own reported emergency - no login required for core functions. Platform administrator: aggregate usage data and problem visibility. No one else accesses the public interface directly - it is the public's channel, showing only what is appropriate for the public |
| How data is shared | Emergency report → incident module (creates incident, passes it for triage and response). Public map ← hospital module (public-facing hospital data) and blood bank module (public-facing blood bank data). Reporter updates ← notification service (sends updates) and incident module (provides status) - shown to reporter through public interface. Aggregate usage → analytics module. No access to dispatch, ambulance, internal hospital/blood bank data, or auth module |
| Real-time | Emergency report submission creates incident immediately and shows confirmation immediately. Reporter can check current status of their incident at any time through the public interface. Public map shows current coarse availability, updated on a reasonable schedule. Updates may also be pushed via SMS/email through notification service. Not a live feed of all emergencies or exact resource changes |
| Gaps addressed | No alternative emergency reporting beyond calling (especially for those who can't or won't call), no easy way to find hospital/blood bank with needed resource, no way to track an emergency report, public info on hospitals/blood banks often outdated/unreliable, no simple clear public interface for emergency help - all introduced by this module, with the public map drawing on hospital and blood bank modules and the reporting flow feeding the incident and notification modules |
| What it does not do | Replace emergency call number, perform triage, provide medical advice, track patient after handover, show exact resource counts to public, show other people's emergency reports, show live ambulance locations, handle non-emergency requests |
| Privacy stance | No name or personal data required to report; location used for incident and stored as part of incident record, not as separate persistent location tracking; contact info stored only if given and only for updates; no device/IP tracking; no other people's reports visible; coarse availability only on public map; updates in plain language, honest about what they mean, only about reporter's own emergency; aggregate usage without personal identification |
| Integration | Creates incidents in incident module (primary integration). Displays data from hospital and blood bank modules (public-facing views). Links to notification service for reporter updates. Optional auth module for account features (not required for core functions). Feeds aggregate analytics. Is the public-facing entry point and window, with all internal modules behind it |

---

*This completes the Public Emergency Interface Module report in the same text-focused, minimal-technical style. We have now covered:*

- *Universal Auth Module*
- *Hospital Module*
- *Ambulance Module*
- *Blood Bank Module*
- *Dispatch Module*
- *Notification Service Module*
- *Incident & Request Management Module*
- *Public Emergency Interface Module*

*The remaining piece to round out the system is the Analytics Module - what the system learns from all of this data: response times, utilization, demand patterns, incident types and locations, blood usage, outcomes, and the dashboards and reports for the platform administrator and for the hospitals and operators to improve their operations.*

*Would you like me to write the Analytics Module report next?*