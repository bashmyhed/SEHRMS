# Blood Bank Module - Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft - subject to revision

---

## 1. Purpose and Scope

The Blood Bank Module is the part of the system that keeps track of blood stock across all the blood banks in the network - what groups are available, how many units of each, whether any are approaching expiry, and which blood banks have what. It exists to solve the problem that today, when a hospital needs blood, it has no way to know which blood bank nearby has the group it needs without calling each one individually.

Specifically, this module is responsible for:

- Registering blood banks on the platform - their identity, location, contact details, and the categories of blood and blood components they handle
- Tracking live inventory of each blood group and component - how many units are available now, how many have been issued, and how many are approaching expiry
- Recording every change to inventory - units received from donation camps or supplier deliveries, units issued to hospitals, units discarded for expiry or other reasons - as an auditable event
- Making inventory visible to hospitals (so they can find the blood they need), to the dispatcher (so the dispatch centre can factor blood availability into its decisions), to the blood banks themselves (so they can manage their own stock), and to the public in a limited, coarse form
- Supporting inter-blood-bank visibility and transfer, so that if one bank is out of a group, another bank nearby can be found and the stock transferred through a tracked process
- Supporting blood requests from hospitals - a hospital requests a specific group and quantity, and the system helps find where it is available and tracks the request from placement through fulfillment

This module connects to the hospital module (because hospitals are the ones requesting blood and receiving it), to the dispatch module (because blood availability can be a factor in emergency response decisions), and to the public map (because citizens may need to find the nearest blood bank with a specific group).

---

## 2. What Data We Track

### 2.1 Blood Bank Identity

For each blood bank registered on the platform, we store:

- A unique identifier and a name that people recognize - for example, the name of the blood bank as it is known locally
- The organization that operates it - a government blood transfusion service, a hospital's own blood bank, an independent voluntary blood bank, or a private facility
- Its address, city, state, and pincode
- Its location as latitude and longitude, so it can appear on the map and the system can compute distances from hospitals and from incident locations
- Its contact details - phone numbers, email, and the name of a contact person who can be reached for urgent issues
- Its operating hours - whether it operates 24 hours, and if not, when it is open
- Any certifications or accreditations it holds, and when they expire, where applicable
- The categories of blood and blood components it handles - whole blood, packed red blood cells, platelets, plasma, and any other components the bank wants to declare. Not every blood bank handles every component, so this is declared per bank rather than assumed.

We do not store patient information, donor personal information beyond what is operationally needed for contact, or financial data. The module tracks stock, not donors or recipients.

### 2.2 Blood Groups and Components

The system defines a set of blood groups and components that blood banks can track. The standard blood groups are A positive, A negative, B positive, B negative, O positive, O negative, AB positive, and AB negative. The standard components include whole blood, packed red blood cells, platelets, and fresh frozen plasma.

Each blood bank declares which of these it handles. A blood bank may handle all groups and components, or only some - for example, a smaller blood bank might handle only whole blood and packed red cells for the common groups, while a larger transfusion centre handles all groups and all components including platelets and plasma.

Having a defined set matters because it lets the system treat all blood banks consistently - "O negative" means the same thing everywhere - and lets the matching logic find the right stock for a hospital's request. A hospital requesting O negative platelets should be shown blood banks that actually have O negative platelets, not just any O negative stock.

### 2.3 Live Inventory

For each blood bank and each blood group and component it handles, we store a live snapshot:

- The total units currently in stock
- The number of units available - meaning in stock and not already committed to a request
- The number of units already issued or committed to requests
- A breakdown where useful - for example, units by expiry window, so the bank can see how much stock is fresh, nearing expiry, or expired
- Who last updated the inventory and when
- A version or update counter, so that two people updating the same count do not overwrite each other
- Whether any units are currently locked or reserved - for example, units that have been promised to a hospital but not yet collected, or units held for a scheduled surgery

The available count is what other parts of the system see - hospitals looking for blood, the dispatcher factoring blood into response decisions, and (in coarse form) the public. When a donation camp brings in units, the available count goes up. When a hospital collects blood, the available count goes down. When units expire and are discarded, they are removed from stock. These changes happen as they occur and are broadcast to anyone watching that blood bank's data.

### 2.4 Expiry Tracking

Blood and blood components have shelf lives. Platelets, for example, expire much faster than red cells. The module tracks expiry so that:

- The blood bank can see which units are nearing expiry and prioritize using them
- The system can flag low-stock situations that are about to become critical - for example, if the only blood bank in a region with O negative is down to a few units and some of them are about to expire
- The system can track discards due to expiry, so there is a record of stock lost to expiry and the bank can plan better

Expiry is tracked per unit batch where the bank chooses to record it - the bank logs when a batch of units was collected or received, and the system computes the expiry window based on the component type. This is not required for every unit in the initial version, but it is the direction the module is designed for, because expiry management is one of the real operational challenges blood banks face.

### 2.5 Blood Requests

When a hospital needs blood, it creates a blood request through the system. Each request records:

- Which hospital made the request
- Which blood group and component is needed, and how many units
- How urgent the request is - routine, urgent, or emergency
- The reason for the request, in operational terms - for example, "trauma surgery, O negative needed for anticipated blood loss" - not patient identity
- When the request was made and by whom
- The status of the request - pending, fulfilled, partially fulfilled, declined, cancelled, or expired

The request is the mechanism by which hospitals ask for blood and the system helps them find it. It is also the mechanism by which blood banks see what is in demand and can prioritize their responses.

### 2.6 Inventory Change Log

Every change to a blood bank's inventory is recorded in a log that is never edited or deleted. Each entry records:

- Which blood bank's inventory changed
- Which blood group and component changed
- What kind of change it was - units received from donation or supplier, units issued to a hospital, units transferred to another blood bank, units discarded for expiry or other reasons, units committed or reserved
- The count before the change and the count after
- The reason for the change, in operational terms
- If the change is linked to a blood request, which request it is linked to
- Who made the change (which user)
- When the change was made
- Whether the change was made manually, through the system automatically, or came from an import or other source

This log is the audit trail for the blood bank module. It is what you look at to know how stock moved - what came in, what went out, what was transferred, what expired. It feeds the analytics - stock turnover, demand patterns, expiry rates - and it provides accountability for every unit that moves in or out of stock.

### 2.7 Inter-Blood-Bank Transfer Records

When one blood bank transfers stock to another - for example, Bank A is out of O negative and Bank B has some and agrees to send it - the transfer is recorded in a tracked way:

- Which bank sent and which bank received
- Which blood group and component, and how many units
- The status of the transfer - requested, accepted, in transit, received, or declined
- When the transfer was initiated and by whom
- Any notes about the transfer

This is what turns an ad hoc phone call between blood banks - "can you send me some O negative?" - into a tracked, auditable process. Both banks see the transfer status, and the system knows that the stock moved from one to the other, so inventory on both sides is updated correctly.

---

## 3. Who Gets Access to What

### 3.1 The Access Model

Access to blood bank data is controlled the same way as the rest of the system - through the universal auth module, with roles and permissions. The blood bank module asks "who is this?" and "are they allowed to do this?" and then applies its own data-scope rules on top.

The key roles that interact with blood bank data are:

- **Blood bank manager or staff** - the person managing the blood bank's inventory. They can see and update their own blood bank's inventory, create and manage requests they receive, and view their own change history.
- **Hospital staff** - the person at a hospital who needs blood. They can see blood bank inventory across the network (to find what they need), create blood requests, and track the status of requests they have made.
- **Dispatcher** - can see all blood bank inventory and factor blood availability into response decisions. Can see which blood banks have the blood needed for an incident and factor that into recommendations.
- **Platform administrator** - can see everything and manage blood banks on the platform.
- **Public** - in a limited, coarse form, can find the nearest blood bank with a requested blood group. Not exact counts.

### 3.2 Blood Bank Manager / Staff

A blood bank manager or staff member, logged into the system, can:

- View their own blood bank's full inventory - all groups and components, available and committed counts, expiry status where tracked
- Update their own inventory - add units received from donation or supplier, issue units to a hospital, transfer units to another bank, discard expired units, and so on
- View the requests they have received from hospitals - which hospital needs what, how urgent, and the status
- Accept or decline a request, or partially fulfill it if they cannot give the full amount
- View their own inventory change history
- See transfers they have initiated or received

They cannot see other blood banks' inventory in detail - they can see that another bank has stock of a group (through the matching results when they are looking for stock to transfer or when a hospital asks them to help find blood), but they do not browse another bank's full inventory. Their access is scoped to their own bank and the requests and transfers that involve them.

### 3.3 Hospital Staff

A hospital staff member who needs blood can:

- View blood bank inventory across the network - see which blood banks have the group and component they need, how many units are available, and how far away they are
- Create a blood request - specify the group, component, quantity, urgency, and reason
- Track the status of requests they have made - pending, being fulfilled, fulfilled, declined, or cancelled
- See which blood bank has fulfilled or committed to fulfilling their request
- Receive notification when their request is accepted, when blood is on its way, and when it has been fulfilled

The hospital staff member does not see every blood bank's full inventory in raw form - they see the matching results that help them find blood, with availability that is relevant to their request. If a hospital wants to know exactly how many units a specific blood bank has of a group, the proper path is to make a request and let the blood bank respond, or to view the bank's availability through the system's controlled visibility. The system does not give every hospital a raw browse of every blood bank's stock.

This is a deliberate boundary. If every hospital could see every blood bank's exact stock at all times, it could create problems - for example, hospitals might compete for limited stock based on real-time visibility, or blood banks might be reluctant to join the platform if their stock levels were fully visible to all hospitals all the time.

### 3.4 Dispatcher

The dispatcher can:

- See all blood bank inventory across the network - all groups and components, available counts, locations
- Factor blood availability into response decisions - when an incident involves a trauma or surgical situation where blood is likely needed, the dispatcher can see which blood banks have the relevant stock and factor that into the overall response picture
- See which hospitals have pending blood requests and which blood banks are fulfilling them
- Use blood availability as one of the factors in the overall dispatch and resource allocation picture, alongside hospital beds, ICU, and ambulances

The dispatcher does not manage blood requests directly - that is between the hospital and the blood bank. But the dispatcher sees blood availability as part of the full operational picture, because in an emergency, knowing that the nearest hospital with an ICU also has the bloodote you need is valuable information for the response.

### 3.5 Platform Administrator

The platform administrator can:

- Register new blood banks on the platform
- Update blood bank details - name, address, contact, operating hours, components handled
- Deactivate blood banks that should no longer be on the platform
- View all blood bank data and history for oversight
- See the full picture of blood stock across the region, demand patterns, and expiry trends

This is the management role for the blood bank network itself - adding banks, removing banks, correcting details, and overseeing the system-wide blood picture.

### 3.6 Public

The public, through the public map, can:

- Find the nearest blood bank that has a requested blood group in stock
- See coarse availability - for example, whether the blood bank has the group available, low, or out - not exact unit counts
- See the blood bank's location and contact details - the phone number to call, which is appropriate for public visibility because a person in an emergency needs to know where to go and who to call

The public does not see:

- Exact unit counts for any blood group
- Which specific components are available beyond the basic group-level coarse indicator
- Internal contacts or staff details beyond what the blood bank chooses to expose publicly
- Any information about requests, transfers, or inventory changes

This is a deliberate limitation, matching the pattern used for hospitals. The public gets enough to act - "the nearest blood bank with your group is here, call this number" - without exposing stock levels that could be misused or that could cause people to overwhelm a blood bank that shows available stock.

---

## 4. How Data Is Shared

### 4.1 Within the Blood Bank

The blood bank's own staff see their own inventory in full - all groups and components, available and committed counts, expiry status, requests they have received, transfers they are involved in, and their change history. This is the data they need to manage their stock.

When they update inventory - receive units, issue units, transfer units, discard expired units - the change is saved and broadcast to anyone watching that blood bank's data. The hospitals that have pending requests with that bank see the status change. The dispatcher, if watching, sees the updated availability picture.

### 4.2 With Hospitals

Hospitals see blood bank inventory in a controlled, request-oriented way:

- When a hospital needs blood, it searches the network for the group and component it needs. The system returns a list of blood banks that have the stock, with availability and distance - a matching result, not a raw browse of every bank's full inventory.
- The hospital creates a request. The blood bank receives the request and can accept, decline, or partially fulfill it.
- When the blood bank accepts the request, the relevant units are marked as committed to that request, and the hospital sees the status change.
- When the blood is collected or delivered, the request is marked fulfilled, and the blood bank's inventory is updated to reflect the issue.

This is the controlled sharing path. The hospital does not get a live feed of every blood bank's stock changes. It gets the information it needs to find and request blood, and it gets updates on its own requests. The blood bank, in turn, sees the requests it has received and can respond to them.

This is better than the current system, where a hospital calls blood banks one by one, often reaching banks that do not have the group, and with no tracking of the request once it is made.

### 4.3 With Other Blood Banks

Blood banks do not browse each other's full inventory. But they do have a controlled way to share stock when needed:

- If Bank A is out of a group, Bank A can look for banks that have the group - the same matching mechanism that hospitals use, applied within the blood bank network. Bank A sees which banks have the stock and can initiate a transfer request.
- Bank B, the receiving bank in this case, sees the transfer request and can accept or decline it.
- When the transfer is accepted, the stock is marked as in transit, and when it is received, both banks' inventories are updated - Bank A's stock goes up, Bank B's stock goes down.
- The transfer is recorded in the transfer log, so there is a record of the movement.

This turns an ad hoc phone call - "can you send me some O negative?" - into a tracked, auditable process. Both banks know the status of the transfer, and the system knows the stock moved, so inventory is correct on both sides.

Blood banks do not see each other's full inventory in raw form. They see matching results when they need to find stock, and they see the transfers and requests that involve them. The boundary is the same as for hospitals - enough to coordinate, not a raw browse of every bank's stock.

### 4.4 With the Dispatcher

The dispatcher sees all blood bank inventory across the network as part of the full operational picture. This is because the dispatcher's job is to coordinate the overall response, and blood availability is one of the factors that matters in an emergency - if an incident is likely to need blood, the dispatcher benefits from knowing which blood banks have the relevant stock and where they are.

The dispatcher does not manage individual blood requests - that is between hospitals and blood banks - but the dispatcher sees the availability picture and can factor it into response decisions. For example, if two hospitals are equally suitable for an incident but one is near a blood bank with the needed blood and the other is not, that may influence the dispatch decision.

### 4.5 With the Public

The public, through the public map, can find the nearest blood bank with a requested blood group, in coarse terms:

- The public enters their location and the blood group they need
- The system returns the nearest blood banks that have that group available, in coarse availability terms - available, low, or out
- The public sees the blood bank's location and the phone number to call

The public does not see exact unit counts, and does not see detailed component availability beyond the basic group-level indicator. This is deliberate - the public's need during an emergency is to find a blood bank with the right group and know where it is and how to reach it. Exact stock levels are not something the public needs, and exposing them could create problems - people overwhelming a bank that shows as available, or blood banks being reluctant to join the platform if their stock is fully public.

---

## 5. How Real-Time Updates Work

### 5.1 Inventory Updates

When a blood bank's inventory changes - units received, units issued, units transferred, units discarded - the change is recorded and broadcast to relevant watchers:

- Hospitals with pending requests involving that blood bank see the status change - if the bank accepted a request, the hospital sees it. If the bank fulfilled a request, the hospital sees it.
- The dispatcher, if watching the overall blood picture, sees the updated availability.
- The blood bank's own dashboard refreshes to show the new counts.

This keeps the system coherent - when stock moves, the people who need to know about it see the update without having to ask.

### 5.2 Request Status Updates

When a blood request changes status - from pending to accepted, from accepted to fulfilled, from fulfilled to collected - the change is recorded and the hospital that made the request sees the update. The hospital does not have to poll or call to find out what happened - the system tells it.

This is one of the concrete improvements over the current system, where a hospital makes a request by phone and then often has to call again to find out whether the blood is ready, whether it has been sent, whether it has been received.

### 5.3 Transfer Status Updates

When an inter-blood-bank transfer moves through its stages - requested, accepted, in transit, received - the change is recorded and both banks see the update. The sending bank knows when the stock has been received by the other bank (and can update its own inventory down). The receiving bank knows when the stock is on the way and when it has arrived (and can update its inventory up).

This keeps both banks informed about the transfer without them having to call each other at each stage. The system tracks it and tells them.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

Today, across most cities and regions, blood bank inventory is siloed and coordination between banks and hospitals is manual and fragmented:

- **Each blood bank knows only its own stock.** A blood bank knows what it has, but there is no system that aggregates stock across all blood banks in a region. If a hospital needs a specific group, it has no way to know which banks have it without calling each one.
- **Hospitals call blood banks one by one.** A hospital needing O negative calls the banks it knows, one after another, asking whether they have it. Often the answer is no, and the hospital has to keep calling. This takes time, and in an emergency, time matters.
- **There is no cross-blood-bank visibility.** Bank A may be out of O negative while Bank B two kilometres away has eight units and no one knows. The information is trapped in each bank.
- **There is no formal request tracking.** A hospital asks for blood by phone, and the arrangement is informal - the bank says "yes, come collect it" or "we will send it," and the hospital and bank remember the arrangement. There is no system record of the request, its status, or whether it was fulfilled.
- **There is no transfer tracking between banks.** If one bank is out of a group and another has it, the transfer happens through a phone call and informal arrangement. There is no tracked record of the transfer, no status visibility, and no systematic update of inventory on both sides.
- **Expiry is managed locally and manually.** Each blood bank manages its own expiry tracking, often with manual logs. There is no system help for prioritizing near-expiry stock or for seeing the regional expiry picture.
- **The public has no way to find blood.** A citizen who needs to find blood - for themselves or for a family member - has no systematic way to find the nearest blood bank with the right group. They ask around, call known banks, or go to the nearest hospital and hope.
- **There is no demand visibility for blood banks.** A blood bank does not see, in a systematic way, what groups are in demand across hospitals, where the demand is coming from, or how urgent it is. This makes procurement and stock planning harder than it could be.

This is the baseline. It is not theoretical - it is how blood coordination works in most places today, even where individual blood banks have good internal systems, because the coordination layer between banks, hospitals, and the public is missing or manual.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| Each blood bank knows only its own stock | A network-wide view of blood inventory across all registered blood banks, with groups, components, availability, and location - visible in controlled form to hospitals, the dispatcher, and (coarsely) the public |
| Hospitals call blood banks one by one | A matching mechanism that lets a hospital find which blood banks have the group and component it needs, ranked by distance and availability, without sequential phone calls |
| No cross-blood-bank visibility | Blood banks can see which other banks have stock of a group they need, through the same matching mechanism, so transfers can be initiated systematically |
| No formal request tracking | A blood request workflow - a hospital creates a request, the blood bank responds, the status is tracked from pending through fulfillment, with notifications at each stage |
| No transfer tracking between banks | A tracked inter-blood-bank transfer process - request, accept or decline, in transit, received - with inventory updated on both sides and a record in the transfer log |
| Expiry managed manually and locally | Expiry tracking per unit batch, with the system flagging near-expiry stock and supporting better prioritization and planning |
| Public has no way to find blood | A public, read-only blood bank finder - the nearest blood bank with a requested group, with location and contact, in coarse availability terms |
| No demand visibility for blood banks | Blood banks see the requests they receive - what groups are in demand, from where, at what urgency - helping with procurement and stock planning |
| No audit trail for blood movements | An inventory change log - every unit received, issued, transferred, or discarded is recorded with reason, actor, and timestamp, auditable and exportable |

### 6.3 How It Is Better - The Concrete Improvements

For the hospital:

- **Faster blood discovery.** Instead of calling bank after bank, the hospital searches the network and sees which banks have the blood it needs, ranked by distance and availability. Discovery time drops from a series of phone calls to a few seconds of looking at the results.
- **Tracked requests.** Once a request is made, the hospital can see its status - pending, accepted, fulfilled - without calling to find out. The system tells the hospital at each stage.
- **Advance notice of blood availability.** When a blood bank accepts a request, the hospital knows. When the blood is ready for collection or on its way, the hospital knows. This lets the hospital plan around the blood arrival.

For the blood bank:

- **Demand visibility.** The bank sees the requests it receives - what groups are needed, from which hospitals, at what urgency. This helps the bank understand demand patterns and plan procurement and stock accordingly.
- **Tracked transfers.** When the bank transfers stock to another bank, the transfer is tracked - both banks know the status, and the system updates inventory on both sides. No more informal phone arrangements with no record.
- **Expiry management support.** The system flags near-expiry stock and tracks discards, helping the bank prioritize and reduce waste.
- **Less time spent on phone calls.** Instead of spending time calling hospitals back to tell them the status of their requests, the bank updates the request status in the system and the hospital sees it. The phone calls drop away for routine status updates.

For the dispatcher:

- **Blood factored into the response picture.** When an incident is likely to need blood, the dispatcher can see which blood banks have the relevant stock and where they are. This is one more piece of the operational picture that helps the dispatcher make better decisions.

For the public:

- **A way to find blood in an emergency.** A citizen can find the nearest blood bank with the group they need, with location and contact, without having to ask around or call banks one by one. This is useful in emergencies and in planned situations alike.

For oversight and administration:

- **An auditable record of every blood movement.** Every unit received, issued, transferred, or discarded is logged with reason, actor, and timestamp. This is what you look at to understand stock movements, demand patterns, expiry rates, and how the blood network is functioning.
- **A regional blood picture.** For the first time, the region has an aggregate view of blood stock across all banks - what groups are well-supplied, which are short, where the shortages are, and how stock moves between banks. This is the data that drives policy decisions about blood bank placement, procurement, and emergency preparedness.

---

## 7. How This Module Integrates With the Rest of the System

### 7.1 With the Hospital Module

This is the primary integration. Hospitals are the ones who request blood and receive it, so the blood bank module and the hospital module work together closely:

- A hospital creates a blood request through the blood bank module, but the request is associated with the hospital's identity from the hospital module.
- When a blood bank fulfills a request, the blood bank module records the issue and the hospital module reflects that the hospital has received the blood (in terms of the request status, not patient-level details).
- The hospital module's data - where hospitals are, what they need - informs the matching logic in the blood bank module, so that when a hospital searches for blood, the system can rank results by distance from the hospital and by relevance to the request.

The two modules are not independent - blood requests are hospital requests for blood, and the blood bank module handles the supply side while the hospital module handles the demand side. But each module owns its own data: the blood bank module owns blood stock and blood bank data, and the hospital module owns hospital data. The blood request is the link between them.

### 7.2 With the Dispatch Module

The dispatch module can use blood bank data as one of the factors in the overall response picture. When an incident comes in, the dispatch module looks at the available resources - hospitals with beds and ICU, ambulances, and blood availability nearby - and can factor blood into its recommendations.

The dispatch module does not manage blood requests directly - that is between hospitals and blood banks - but it sees blood availability as part of the full operational picture. For example, if an incident involves trauma and the nearest hospital with ICU capacity is also near a blood bank with the needed blood group, that strengthens the case for dispatching to that hospital.

The blood bank module feeds the dispatch module with blood availability data, and the dispatch module uses it alongside hospital and ambulance data. The blood bank module does not depend on the dispatch module - it would work even if dispatch were not part of the system - but the combination is more powerful than either alone.

### 7.3 With the Incident Module

In some cases, a blood request may be linked to an incident - for example, a trauma incident where blood is needed urgently. The blood bank module can reference the incident to understand the context of a request, and the incident module can reflect that blood has been requested and fulfilled for that incident.

This link is optional in the initial version - not every blood request is tied to an incident; some are routine surgical requests. But the capability to link them is there, because it makes the system more coherent: you can see, for a given incident, what resources were mobilized - which hospital, which ambulance, which blood.

### 7.4 With the Auth Module

The blood bank module uses the universal auth module for identity and access control, just like every other module. Every blood bank manager, hospital staff member, dispatcher, and platform administrator who interacts with blood bank data has an account in the auth module, with roles that define what they can do. The blood bank module checks with the auth module before letting anyone see or change anything - "who is this?" and "are they allowed to do this?" - and then applies its own data-scope rules on top.

A blood bank manager's role, for example, lets them manage their own bank's inventory and respond to requests they receive, but not see other banks' full inventory. A hospital staff member's role lets them search for blood and create requests, but not manage any blood bank's inventory. The auth module provides the role information; the blood bank module applies the scope.

### 7.5 With the Public Map

The blood bank module feeds the public map with a coarsened view of blood bank data - names, locations, contact phone numbers, and coarse availability by blood group. The public map does not get exact unit counts, and it gets only the groups the public is searching for, in coarse terms. This is the same boundary used for hospitals - the public sees enough to act, not the raw stock data.

Blood banks may choose to expose their phone number publicly (it is a public contact number for people seeking blood, which is appropriate), but internal contacts and staff details are not exposed.

---

## 8. Privacy and Safety Considerations

### 8.1 What We Do Not Track

- **Patient information.** The module tracks blood stock and blood requests, not patients. A blood request is associated with a hospital and a reason in operational terms, not with a patient's identity or medical details. Patient data is not in scope for this project.
- **Donor personal information.** The module does not track individual donors or their personal details. It tracks stock - units received, issued, transferred, discarded - not who donated them. Donor management is a separate concern for the blood bank's own systems.
- **Financial data.** The module does not track the financial side of blood - costs, pricing, billing. It tracks stock and requests, not money.

### 8.2 What We Do Track

- **Blood bank identities and contact details.** Name, address, location, phone, and the components handled. These are operational details needed for coordination.
- **Blood stock by group and component.** How many units of each group and component are available, committed, and (where tracked) near expiry.
- **Blood requests.** Which hospital requested what, when, at what urgency, and the status of the request.
- **Inventory changes.** Every unit received, issued, transferred, or discarded, with reason, actor, and timestamp.
- **Transfers between banks.** Which bank sent what to which bank, with status and record.

All of this is operational data about blood stock and blood movement. It is not personal data about patients or donors, and it is not financial data.

### 8.3 Controlled Visibility of Stock

One of the key privacy and safety decisions in this module is how much stock information to expose, and to whom:

- **Blood banks see only their own stock in full.** They can see matching results when they need to find stock at another bank, but they do not browse other banks' raw inventory.
- **Hospitals see matching results when they search for blood.** They see which banks have the group they need, with availability and distance, but not a raw feed of every bank's stock changes.
- **The dispatcher sees all stock as part of the operational picture.** This is appropriate because the dispatcher's job is coordination, and blood availability is one of the factors that matters.
- **The public sees coarse availability only.** Nearest blood bank with the group, available or low or out, with location and contact - not exact counts.

This controlled visibility matters because if every hospital could see every blood bank's exact stock at all times, several things could go wrong:

- Hospitals might compete for limited stock based on real-time visibility, rushing to collect blood from a bank that shows available units, even if that bank is already dealing with multiple requests.
- Blood banks might be reluctant to join the platform if their stock levels were fully visible to all hospitals all the time - they might worry about being seen as well-supplied or poorly-supplied, or about hospitals bypassing them based on real-time stock data.
- The public, if shown exact counts, might make decisions based on incomplete information - assuming a bank has enough blood because it shows available units, without understanding that those units may already be committed to other requests.

The system's approach is to share what is needed for coordination and nothing more. For hospitals, that means matching results and request tracking. For blood banks, that means their own stock and the requests and transfers involving them. For the dispatcher, that means the full operational picture, because the job requires it. For the public, that means coarse availability and contact. The boundary is enforced structurally - the public endpoint does not have exact counts to return, and hospitals do not get a raw feed of every bank's stock.

### 8.4 Blood Request Information

A blood request is associated with a hospital and a reason in operational terms - "trauma surgery, O negative needed" - not with a patient's identity. The blood bank that receives the request sees the hospital, the group and component needed, the quantity, the urgency, and the reason. It does not see the patient's name, age, or diagnosis. This is the right scope: the blood bank needs to know what blood is needed, how urgently, and for what kind of situation, so it can prioritize appropriately. It does not need the patient's identity, and the system does not expose it.

This is important for privacy and for the scope of the project. Keeping patient data out of the blood request means the module stays in its lane - blood stock and blood requests - without opening patient-level privacy and compliance concerns.

---

## 9. What We Are Explicitly Not Doing

It is worth stating what this module does not attempt, so the scope is honest:

- **Donor management.** The module does not track individual donors, donation appointments, donor eligibility, or donor records. That is a separate system concern for blood banks, and it is not part of this project. The module tracks stock, not donors.
- **Blood testing or quality control.** The module does not track blood testing results, cross-matching, or quality control processes. It tracks inventory counts and movements, not the testing pipeline. Blood banks manage testing within their own systems.
- **Patient cross-matching or transfusion records.** The module does not track which patient received which unit of blood, or any transfusion records. That is patient-level data and is not in scope. The module tracks that a hospital requested and received blood, not which patient it went to.
- **Financial transactions.** The module does not handle pricing, billing, or the financial side of blood - costs, compensation for donation, or payment for blood. It tracks stock and requests, not money.
- **National or regional blood authority integration.** In this project, blood banks are registered on the platform and manage their stock through it. In a production rollout, a national or regional blood authority might want to integrate its own systems with the platform, or the platform might need to report to such an authority. That is a future integration path, not part of this project.
- **Emergency blood delivery logistics.** The module tracks that blood has been issued to a hospital or transferred to another bank, but it does not manage the physical delivery logistics - who picks up the blood, how it is transported, cold-chain management during transport. Those are operational details that the blood bank and the hospital handle, and the module records the result (the transfer or issue) without managing the logistics.
- **Public availability of exact stock levels.** As described throughout, the public sees coarse availability only. Exact stock levels are not exposed publicly, by design.
- **Real-time notification to donors.** The module does not notify donors when blood is needed or when their specific donation is used. Donor notification is a separate concern for blood banks, not part of this project.

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Track blood stock across all registered blood banks - groups, components, availability, expiry - and support blood requests from hospitals, inter-bank transfers, and public blood bank finding, all with controlled visibility and full audit |
| Data tracked | Blood bank identity and contact, blood groups and components handled, live inventory by group and component (available, committed, near-expiry where tracked), expiry tracking per batch, blood requests (hospital, group, component, quantity, urgency, reason, status), inventory change log, inter-bank transfer records |
| Who gets access | Blood bank managers see and update their own bank's inventory and respond to requests they receive; hospital staff search for blood, create and track requests; dispatcher sees all blood inventory as part of the operational picture; platform admin manages all banks and sees the regional picture; public finds nearest blood bank with a group, in coarse terms only |
| How data is shared | Within a blood bank: full data for its own staff. With hospitals: matching results when searching, request tracking when a request is made - not a raw feed of every bank's stock. With other blood banks: matching results when looking for stock to transfer, tracked transfers - not raw inventory browsing. With the dispatcher: full blood availability picture as part of the operational view. With the public: nearest bank with the group, coarse availability, location, and contact - not exact counts |
| Real-time | Inventory changes broadcast to relevant watchers; request status updates pushed to the requesting hospital; transfer status updates pushed to both banks - all without the parties having to poll or call |
| Gaps addressed | No network-wide blood visibility today, hospitals call banks one by one, no cross-bank visibility, no formal request tracking, no transfer tracking, expiry managed manually, public has no way to find blood, no demand visibility for banks, no audit trail for blood movements - all introduced by this module working with the hospital, dispatch, and public map modules |
| What it does not do | Donor management, blood testing or quality control, patient cross-matching or transfusion records, financial transactions, national blood authority integration, emergency delivery logistics, public exact stock levels, donor notifications |
| Privacy stance | Patient and donor data not in scope; controlled visibility of stock - each role sees only what it needs; public sees coarse availability only; blood requests carry operational reason, not patient identity; structural enforcement of the boundary, not just policy |
| Integration | Depends on auth module for identity and access; feeds hospital module with blood availability for matching and receives requests associated with hospitals; feeds dispatch module with blood availability as a response factor; optionally links to incidents; feeds public map with coarsened blood bank data; owns blood stock and request data, links to hospitals through requests |

---

