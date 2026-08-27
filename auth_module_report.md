# Universal Authentication & Access Control Module — Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft — subject to revision

---

## 1. Purpose and Scope

This module is the single place in the system where identity, login, roles, and permissions are handled. Every other part of the system — hospitals, ambulances, blood banks, dispatch, public map — checks with this module before letting someone see or change anything.

What this module is responsible for:

- Who a person is (their account, their name, their email, their phone)
- Proving they are who they say they are (login, token issuance, token validation)
- What they are allowed to do (roles and permissions)
- Which organization they belong to (so the system knows whether a hospital admin should see only their own hospital or everything)
- Recording what happened (every login, every role change, every failed login attempt is logged)

What this module is NOT responsible for:

- The actual business of updating bed counts, dispatching ambulances, or moving blood between banks — that belongs to the modules that do those things
- Patient information — patient data is not in scope for this project at all
- Replacing a hospital's internal staff records — this system only knows about people who need access to the platform itself

In short: this module answers two questions for every other module — "who is this?" and "are they allowed to do what they're asking?" Everything else is decided by the module being asked.

---

## 2. What Data We Track

### 2.1 Account Information

For each person who uses the system, we store:

- A unique identifier (internal, not shown to other users)
- Email address — used as the login name
- A password hash — never the actual password; only a one-way mathematical representation that cannot be reversed
- Full name
- Phone number
- Whether the account is active or deactivated
- Whether the email has been verified (optional — for this project, verification may be skipped if users are created by an administrator rather than signing up themselves)
- When they last logged in
- When the account was created and last updated

We do not store:
- Passwords in any readable form
- Patient information of any kind
- Financial information
- Addresses beyond what is needed for contact (a phone number and email are enough for platform access; a person's home address is not relevant)

### 2.2 Roles

A role is a label that groups together a set of permissions. Examples of roles in this system:

**Platform-wide roles** (apply to the whole system, not tied to any one hospital):

- Platform administrator — can see everything, manage all users, oversee the entire system
- Dispatcher — can see all hospitals, all ambulances, all incidents; can allocate resources but cannot change hospital profiles or manage users
- System administrator — the highest level, can create organizations, manage any user, configure the system

**Organization-scoped roles** (apply only within one hospital or blood bank organization):

- Organization administrator — can manage their own hospital's facilities, set bed capacities, manage their own staff, accept or decline resource requests from other hospitals, export their own reports
- Bed and inventory manager — can update bed counts and equipment counts for their own hospital, view their own inventory dashboard and change history
- Contact person / duty officer — can view their own hospital's dashboard, receive alerts; read-only or limited-write, useful for someone who needs visibility but should not change capacity numbers
- Read-only viewer — can see their own hospital's data but cannot change anything; useful for a medical superintendent or auditor

A person can have more than one role at the same time — for example, someone could be an inventory manager in their own hospital and also have dispatcher access for night shifts. The system looks at all of a person's active roles when deciding what they are allowed to do.

Roles are predefined — the system does not let administrators invent new roles on the fly. This keeps the permission model simple and predictable. Adding a new role means thinking about it deliberately, not clicking a button and accidentally granting too much.

### 2.3 Permissions

A permission is a single, specific capability. Examples:

- View bed inventory (for one's own hospital, or for all hospitals if the person is a dispatcher)
- Update bed inventory (for one's own hospital only, unless the person is a dispatcher acting through the dispatch workflow)
- View ambulance locations and status
- Dispatch or allocate a resource (assign a bed, assign an ambulance to an incident)
- Lock or unlock a resource (reserve a bed for an incoming patient so no one else can take it)
- Manage contacts for a hospital
- Create or deactivate user accounts (only platform administrators and organization administrators, and only within their scope)
- Assign or remove roles from users (same scope rule)
- View the audit log of system activity (platform-level only)
- View the public map (no login required — this is the unauthenticated public role)

Permissions are attached to roles, not directly to people. A person gets permissions by having roles. This makes it easier to reason about: if you know someone is a "bed manager," you know they can update inventory for their own hospital. You do not need to check a long list of individual permissions for each person.

### 2.4 Organization Membership

For roles that are tied to a hospital or blood bank, the person is linked to that organization. This link is what lets the system answer "should this hospital admin be able to see Hospital B's data?" — the answer is no, unless the person also has a platform-wide role like dispatcher.

The organization link is stored alongside the role assignment, along with who assigned it and when. If a role is temporary (for example, a locum doctor or a temporary dispatch operator covering a shift), the role can have an expiry time after which it no longer applies.

### 2.5 Authentication Audit Log

Every authentication-related event is recorded in a log that is append-only — entries are never edited or deleted. Events include:

- Successful login
- Failed login attempt (wrong password, or email not found — the system does not reveal which, to prevent someone from learning which emails are registered)
- Logout
- Token refresh
- Password change
- Password reset request
- Role assigned to a user
- Role removed from a user
- User account created
- User account deactivated

Each log entry records what happened, who did it (if applicable), when it happened, and from where (IP address and browser/device information, for investigation purposes). This log is the system's memory of who accessed what and when — it is what you look at if you need to investigate a security question.

---

## 3. Who Gets Access to What

### 3.1 The Access Model in Plain Terms

The system divides access along two lines:

**First line: platform-wide vs. organization-scoped.**

- Platform-wide roles see everything relevant to their function. A dispatcher sees every hospital's bed availability, every ambulance, every incident. A platform administrator sees everything including user accounts and the audit log.
- Organization-scoped roles see only their own organization's data. A hospital administrator for Hospital A sees Hospital A's beds, Hospital A's contacts, Hospital A's inbound and outbound resource requests — but not Hospital B's.

**Second line: what the role is allowed to do.**

Within their scope, each role has a specific set of permissions. A bed manager can update inventory but cannot delete user accounts. A dispatcher can allocate resources but cannot change a hospital's profile or set its bed capacity. An organization administrator can manage their own staff but cannot see another hospital's inventory.

### 3.2 Access by Stakeholder Type

**Hospital administrator (organization-scoped):**
- Sees their own hospital's full profile, bed inventory, equipment inventory, contacts, and change history
- Can update bed and equipment counts for their own hospital
- Can manage staff accounts within their organization (create, deactivate, assign roles)
- Can see and respond to resource requests from other hospitals (accept or decline)
- Sees their own outbound resource requests (requests they sent to other hospitals)
- Cannot see other hospitals' inventory or profiles (except coarsely, through the matching recommendations when they request a resource)
- Cannot see the platform audit log or manage system-wide users

**Bed and inventory manager (organization-scoped):**
- Sees their own hospital's bed and equipment inventory
- Can update occupied counts and equipment availability
- Can view the change history for their own hospital
- Sees active locks on their inventory (so they know a bed is reserved for an incoming patient)
- Cannot manage users or change hospital profile
- Cannot see other hospitals

**Dispatcher (platform-wide):**
- Sees all hospitals, all bed/equipment inventory, all ambulances, all blood banks, all active incidents on a single live view
- Can allocate or lock resources — assign a bed, assign an ambulance, mark a blood request as fulfilled — using the dispatch workflow
- Cannot edit hospital profiles or change a hospital's declared bed capacity
- Cannot manage user accounts
- Is the operational nerve centre — sees everything operationally relevant, but cannot change administrative data

**Platform administrator (platform-wide, highest level):**
- Sees everything
- Can create or deactivate organizations
- Can create or deactivate any user
- Can assign any role to any user
- Can view the full audit log
- Can configure system-wide settings
- Is the last resort for oversight and troubleshooting

**Contact person / duty officer (organization-scoped, read-heavy):**
- Sees their own hospital's dashboard
- May have limited write access (e.g. acknowledge an incoming request) or be read-only
- Receives alerts relevant to their hospital
- Useful for someone present in the hospital who needs awareness but is not the person managing inventory or staff

**Read-only viewer (organization-scoped):**
- Sees their own hospital's data, cannot change anything
- Useful for auditors, medical superintendents, or anyone who needs visibility without edit access

**Public (no login):**
- Sees a coarse view of hospital availability on a map — for example, "Hospital X has ICU availability" or "ICU capacity is low" — without exact bed counts
- Sees nearest blood banks with a requested blood group in stock (coarse — available/low/out, not exact unit counts)
- Cannot see internal contacts, exact counts, or any administrative data
- This is deliberately limited to prevent misuse while still giving citizens useful information during an emergency

### 3.3 How the System Decides Whether to Allow Something

Every time someone tries to do something protected (view someone else's data, change a bed count, dispatch an ambulance), the system checks three things in order:

1. **Are you who you say you are?** — Is your login token valid? Have you logged in? If not, the request is rejected immediately.

2. **Do you have the permission for this kind of action?** — Does your role set include the permission needed? For example, updating bed inventory requires the "update inventory" permission. If your roles do not include it, the request is rejected.

3. **Are you allowed to do this to this specific thing?** — For organization-scoped permissions, does the resource belong to your organization? A hospital admin for Hospital A trying to update Hospital B's bed count is rejected at this step, even if they have the "update inventory" permission — because the permission is scoped to their own organization. Platform-wide roles like dispatcher skip this scope check for actions that are meant to operate across all hospitals.

This three-step check happens for every protected action, everywhere in the system. It is the same logic in the hospital module, the ambulance module, the blood bank module, and the dispatch module — only the specific permission and scope differ.

---

## 4. How Data Is Shared

### 4.1 Within a Hospital Organization

Data that belongs to a hospital is visible in full to people with organization-scoped roles in that hospital. This includes:

- The hospital's profile (name, type, address, contact information, capacity details)
- Bed inventory (exact counts by category)
- Equipment inventory (ventilators, oxygen, dialysis machines, etc.)
- Contacts (who to call for what)
- Change history (who changed what and when)
- Inbound and outbound resource requests with other hospitals

This data is not visible to other hospitals' staff, except through specific sharing mechanisms described below.

### 4.2 Between Hospitals (Cross-Organization Sharing)

This is the area where the system is most careful, because sharing the wrong data could cause confusion, competition, or misuse.

**What is shared:**

- Hospital identity basics — name, type, location (these are public information anyway; any hospital's name and address are visible on its own website or a map)
- Coarse availability for the public map — whether a hospital currently has ICU availability, whether capacity is low or full, without exact numbers
- Matching recommendations when a hospital requests a resource — if Hospital A requests ICU beds, the system can tell Hospital A that Hospital B has ICU availability and is 3 kilometres away, without showing Hospital A Hospital B's full inventory

**What is NOT shared openly between hospitals:**

- Exact bed counts for another hospital — Hospital A does not browse Hospital B's inventory directly. Hospital A sees a recommendation, not a data dump. If Hospital A wants to know Hospital B's exact ICU count, the proper path is to send a resource request and let Hospital B respond — at that point, the relevant count may be shared in the context of the request, but not before.
- Internal contacts of another hospital — phone numbers and duty officer names are not exposed across organizations. If Hospital A needs to call Hospital B, the system can show the request workflow, but the contact details are Hospital B's own to share or not.
- Another hospital's audit log, change history, or internal administrative data — never shared.

**Why this matters:**

If every hospital could see every other hospital's exact bed counts at all times, several things could go wrong:

- A hospital might be reluctant to join the platform at all, because its capacity data would be visible to competitors or to patients who might flood it with calls
- During a crisis, visible bed counts could lead to callers overwhelming a hospital that shows as "available," even if the hospital is already dealing with a surge it has not yet updated in the system
- The data could be misused — for example, someone could use real-time availability information for purposes other than emergency coordination

The system's approach is to share enough for coordination to work (identity, coarse availability, recommendations) without exposing everything. Exact counts are shared only in the context of a formal request that the receiving hospital has agreed to, or by the dispatcher who has a platform-wide operational role and is trusted to use the information for coordination, not for anything else.

### 4.3 With the Dispatcher / Command Centre

The dispatcher is the one role that sees everything operationally relevant — all hospitals' inventory, all ambulances, all blood banks, all incidents. This is intentional: the dispatcher's job is to coordinate across the whole system, and they cannot do that without a complete picture.

The dispatcher does NOT see:
- User account management data (that is for platform administrators)
- Hospital administrative settings that are not operationally relevant (e.g. which staff member has which internal role — the dispatcher sees that a person can update inventory for a hospital, but not necessarily the full role breakdown)
- Anything patient-related (not in scope at all)

The dispatcher is trusted with the full operational picture because their function requires it. This trust is auditable — every allocation, every lock, every dispatch action the dispatcher takes is logged, so if anything is misused, there is a record.

### 4.4 With the Public

The public gets the least data, deliberately. The public map shows:

- Hospital names, types, and locations (public information)
- A coarse availability indicator for the resource the user is looking for — for example, "ICU available," "ICU capacity low," or "ICU full" — not exact numbers
- Distance from the user's location
- Nearest blood banks with the requested blood group, again in coarse terms

The public does NOT see:
- Exact bed counts
- Internal contacts or phone numbers (the public map can show a general emergency number if the hospital chooses to expose one, but not the duty officer's personal number)
- Which specific resources are where
- Any administrative data

This is a deliberate privacy and safety decision. The goal is to give a citizen in an emergency enough information to go to the right place, without exposing data that could be misused or that could cause a hospital to be overwhelmed by people checking its availability status and then flooding it.

### 4.5 With Platform Administrators and Auditors

Platform administrators and authorized auditors see everything the system tracks, including the full audit log. This is the oversight layer — the people who need to investigate problems, verify that the system is being used correctly, and generate reports.

This access is itself auditable: every time a platform administrator views the audit log or exports data, that action is logged. So there is a record of who looked at what, when.

---

## 5. Gaps in the Current System and How This Module Addresses Them

### 5.1 The Current Situation

Today, in most cities and regions, there is no centralized way to control who can see or do what across emergency healthcare resources. The gaps are not just about data — they are about identity and access too.

- **Every hospital has its own informal access model.** Staff access is managed within the hospital, often informally — a nurse has access to the ward register because she works there, a doctor has access because he is on the team. There is no unified identity across hospitals, no shared login, no concept of a dispatcher who can see multiple hospitals.
- **There is no dispatcher role in most settings.** In places that have emergency dispatch at all, the dispatcher works from phone numbers and personal knowledge — they know which hospital to call, which ambulance operator to contact, but they have no system-managed identity, no login, no access control. Their access is their memory and their phone list.
- **Ambulance operators and blood banks have no shared identity with hospitals.** An ambulance crew is a separate organization with no login relationship to the hospitals they serve. A blood bank is separate from the hospitals it supplies. There is no single place where all these actors have accounts with defined roles.
- **There is no audit trail of who accessed what.** If a hospital's bed count was changed incorrectly, or if a resource was allocated by someone who should not have allocated it, there is usually no log to check. You get oral accounts: "I think the duty nurse updated it," "I think the dispatcher called and said it was available." No authoritative record.
- **Access is often all-or-nothing within a hospital.** Once someone is "inside" a hospital's system or knows its phone numbers, they often have broad access. There is rarely a distinction between "can view inventory" and "can change inventory" and "can manage staff" — it is often just "works here, so can see and do most things."
- **There is no way to give someone temporary access.** A locum doctor covering a week, a temporary dispatch operator during a crisis, a volunteer coordinator — in the current system, giving them access means giving them a phone number or a login that often stays active after they leave.

### 5.2 How This Module Addresses Each Gap

| Current gap | What this module introduces |
|-------------|----------------------------|
| No shared identity across hospitals, ambulances, blood banks, dispatch | One user account system that all these actors use. A hospital admin, an ambulance crew member, a blood bank manager, and a dispatcher all have accounts in the same system, with roles that define what they can do. |
| No dispatcher role as a system-managed identity | The dispatcher is a first-class role in this module — a defined, auditable identity with a specific permission set, not just a person with a phone list. |
| No distinction between viewing and changing access | Permissions are granular. "View inventory" and "update inventory" are different permissions, assigned through different roles. A duty officer can be given view-only access while the bed manager has update access. |
| No audit trail of access and changes | Every login, logout, role assignment, role removal, and failed login is logged in an append-only audit log. Every action taken by a logged-in user in any module is attributable to that user. If something goes wrong, you can trace who did what and when. |
| No way to scope access to one's own organization | Organization-scoped roles mean a hospital admin for Hospital A is structurally prevented from seeing Hospital B's inventory — not by a policy that people can ignore, but by the access control system enforcing it. The system does not show Hospital B's data to Hospital A's admin at all. |
| No temporary or expiring access | Role assignments can have an expiry time. A temporary dispatcher covering a night shift can be given the dispatcher role with an expiry, and the access automatically stops when the shift ends. A locum doctor can have view access for the duration of their cover. |
| No oversight of who looked at what | The platform administrator audit log records who accessed the audit log, who exported data, and when. Oversight is itself overseen. |
| Fear of misuse or data leakage | The module enforces the coarse-public, fine-private distinction structurally. The public sees only coarse availability because the public endpoint is built to return only coarse data — it is not a matter of asking people not to share exact counts, it is that the system does not give the public endpoint exact counts to begin with. |

### 5.3 What This Module Does Not Solve (and Why That Is Acceptable)

This module handles identity, login, roles, permissions, and access auditing. It does not solve:

- **Physical security of the hospital.** A person with legitimate platform access who walks into a hospital and sees a bed that the system says is available may still find it occupied — the system's data is only as current as the last update. This module does not solve that; the hospital module's real-time update discipline does.
- **Trust in the person.** The system can control what a person is allowed to do in the software, but it cannot control what they do outside it. If a dispatcher misuses their knowledge, the audit log records it but cannot prevent it in real time. This is true of any software system and is not a flaw unique to this one.
- **Consent of hospitals to join.** The system can define roles and permissions, but it cannot force a hospital to create an account and start sharing data. That is an organizational adoption problem, not an access-control problem. The module is ready to enforce access control once hospitals are on board; getting them on board is a separate step.
- **Patient identity and consent.** Patient data is not in scope. If the system were extended to track patients, a separate consent and privacy layer would be needed. That is not this module's concern and not this project's scope.

---

## 6. How This Module Integrates With the Rest of the System

### 6.1 The Integration Model

This module is the foundation that every other module sits on. The integration is simple in concept:

- Every person who uses any part of the system has an account in this module.
- When they log in, they get a token that says who they are, what roles they have, and (for organization-scoped roles) which organization they belong to.
- Every other module — hospital, ambulance, blood bank, dispatch, public map — checks that token before doing anything protected.
- The check is always the same: is the token valid, does the person have the permission for this action, and (for organization-scoped actions) is the resource within their organization?

This means no module invents its own login system. There is one login, one set of roles, one set of permissions, one audit log. If you want to change how access works, you change it in this module, and every other module benefits from the change automatically.

### 6.2 How a New Actor Joins the System

The flow for bringing a new person into the system is always the same, regardless of whether they are a hospital administrator, an ambulance crew member, a blood bank manager, or a dispatcher:

1. **An existing authorized person creates the account.** This is either a platform administrator (for platform-wide roles like dispatcher) or an organization administrator (for staff within a hospital or blood bank). The account is created with the person's name, email, and phone.

2. **Roles are assigned.** The person who created the account also assigns roles. For example, creating a new bed manager in Hospital A means assigning the "bed manager" role scoped to Hospital A's organization. Creating a new dispatcher means assigning the "dispatcher" role with no organization scope.

3. **The person logs in.** They use their email and password. On success, they receive a token that carries their identity, roles, and organization.

4. **The person uses the system.** Every action they take is checked against their roles and permissions. The system knows what they are allowed to do and enforces it.

5. **If the person leaves or changes role, access is adjusted.** The role is removed or changed by an administrator, or allowed to expire if it was temporary. The person's access changes immediately (or within the token lifetime, depending on the token strategy chosen).

This is the same flow for every actor type. The difference between a hospital admin and a dispatcher is only in what roles they are given, not in how they join the system.

### 6.3 How the Modules Rely on This Module

Each module asks this module two things:

- **"Who is this?"** — The module extracts the user's identity and roles from the token. It does not re-verify the password or re-check the login; that was done once at login. The token is the proof.

- **"Are they allowed to do this?"** — The module checks the permission. For organization-scoped actions, it also checks that the resource belongs to the user's organization (or that the user has a platform-wide role that bypasses the scope check).

The modules do not need to know how roles are stored or how permissions are mapped. They just ask "does this user have permission X for resource Y?" and get a yes or no. This keeps each module simple and keeps the access-control logic in one place where it can be reviewed and tested.

### 6.4 How the Public Fits In

The public map and public finder endpoints do not require a login. They are available to anyone. However:

- They are rate-limited, so someone cannot flood them with requests
- They return only coarse data, by design — the public endpoints do not have access to exact counts, so even if someone calls them, they get only what the system intends to share
- If a lightweight token is needed in the future for abuse prevention (for example, to tie requests to an identity without requiring full login), the public user can be given a minimal "public" role with no permissions beyond accessing the public endpoints — but for now, no login is required for public access

This keeps the public-facing part of the system open and useful in an emergency, while keeping the data it shares deliberately limited.

---

## 7. Privacy and Safety Considerations

### 7.1 What This Module Does Not Expose

- Patient information of any kind — not in scope
- Staff personal information beyond what is needed for platform access (name, email, phone) — not home addresses, not personal identifiers beyond contact details
- A hospital's internal administrative data to other hospitals or to the public — exact bed counts, internal contacts, audit logs are never shared outside the hospital except to platform administrators and the dispatcher, and only for operational purposes
- Exact availability data to the public — only coarse indicators

### 7.2 Why We Deliberately Do Not Show Everything

There is a real risk in exposing too much data, even with good intentions:

- **If every hospital's exact bed count were public,** it could lead to patients or callers targeting hospitals that show available capacity, overwhelming them during a crisis. It could also create competition or mistrust between hospitals.
- **If internal contacts were public,** they could be misused for purposes other than emergency coordination.
- **If audit logs were visible to hospitals,** they could expose the activities of other hospitals or of platform staff in ways that create friction or legal exposure.

The system's principle is: share what is needed for coordination, nothing more. For the public, that means coarse availability. For hospitals, that means identity, coarse availability, and recommendations — not raw data about other hospitals. For the dispatcher, that means full operational data, because the dispatcher's job requires it, but with full audit logging so the trust is accountable.

### 7.3 The Trust Model

The system's trust model is:

- **Platform administrators** are trusted with full access, because they run the system. Their actions are fully audited.
- **Dispatchers** are trusted with full operational access, because their job requires it. Their actions are fully audited.
- **Hospital administrators and staff** are trusted with their own hospital's data, because they need it to manage their hospital. They are structurally prevented from seeing other hospitals' data.
- **The public** is given the minimum useful information, because there is no trusted relationship and the cost of over-sharing is real.

This trust model is enforced by the access control system, not by asking people to behave well. The public endpoint literally cannot return exact counts because it does not have them. A hospital admin literally cannot see another hospital's inventory because the system does not show it to them. The enforcement is built into the system, not into a policy document.

---

## 8. What Makes This Better Than the Current Approach

| Current approach | What this module does instead | Why it matters |
|------------------|-------------------------------|----------------|
| Each hospital manages its own staff access informally, with no shared identity | One account system, one login, one set of roles for everyone across hospitals, ambulances, blood banks, and dispatch | A dispatcher can be a real system identity with a defined role, not just a person with a phone list. Access is consistent across the whole system. |
| No distinction between viewing and changing access in many settings | Separate permissions for viewing, updating, managing staff, managing settings | A duty officer can view without being able to accidentally change a bed count. A bed manager can update inventory without being able to delete user accounts. |
| No audit trail of who accessed what | Append-only audit log of every login, logout, role change, and failed attempt | If something goes wrong, there is a record. Accountability is built in, not added after the fact. |
| No organization scoping — once you are "in," you often see everything | Organization-scoped roles prevent hospital A's staff from seeing hospital B's data | Hospitals are more likely to join if their data is not visible to every other hospital. The system enforces the boundary, not just a policy. |
| No temporary access | Roles can expire | Temporary staff, locums, and shift-based dispatchers get access only for as long as they need it, without manual cleanup later. |
| Public has no useful information or has too much | Public gets coarse, useful information without exact counts | Citizens get help during emergencies without creating a risk of overwhelming hospitals or exposing data that could be misused. |
| Access changes are manual and often delayed | Role assignment and removal are system actions, immediate (or within token lifetime), and logged | When someone leaves or changes role, their access changes with them. No stale access lingering because someone forgot to revoke it. |

---

## 9. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | One centralized module for identity, login, roles, permissions, and access auditing. Every other module depends on it. |
| Data tracked | Account info (email, name, phone, password hash, active status, last login), roles (platform-wide and organization-scoped, predefined), permissions (granular, attached to roles), organization membership for scoped roles, and an append-only audit log of all auth events |
| Who gets access | Platform administrators (everything), dispatchers (all operational data), organization administrators (their own org), bed managers (their own org's inventory, update rights), contact persons (their own org, view-heavy), read-only viewers (their own org, view only), public (coarse availability only, no login) |
| How data is shared | Within a hospital: full data for staff with scoped roles. Between hospitals: identity and coarse availability shared; exact counts shared only in the context of a formal request or by the dispatcher with operational need, never browsed openly. With the public: coarse availability only, deliberately limited. With platform administrators: everything, fully audited. |
| Privacy stance | Do not expose exact counts publicly; do not expose other hospitals' raw data to each other; do not expose internal contacts across organizations; do not store or expose patient data. Share what is needed for coordination, nothing more. Enforcement is structural — the public endpoint does not have exact counts to return, and hospital admins do not see other hospitals' inventory at all. |
| Gaps addressed | No shared identity today; no dispatcher as a system role; no granular view/change distinction; no audit trail; no cross-hospital scoping; no temporary access; no oversight of oversight. This module introduces all of these. |
| What it does not solve | Physical security of hospital data accuracy, trust in people outside the software, hospital adoption consent, patient identity and consent — these are out of scope or belong to other modules. |
| Integration | Every module checks the token and asks "does this user have permission for this action on this resource?" The auth module answers; the modules apply their own data-scope rules on top. Same flow for every actor type — hospital admin, ambulance crew, blood bank manager, dispatcher — only the roles differ. |

---

