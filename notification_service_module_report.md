# Notification Service Module - Design Report

**Project:** Smart Emergency Healthcare Resource Management System
**Status:** Discussion draft

---

## 1. Purpose and Scope

The Notification Service Module is the part of the system that actually reaches people with the right message at the right time. Every other module - dispatch, ambulance, hospital, blood bank, auth - decides when a notification should be sent and to whom. The notification service is what sends it, through whatever channel is appropriate, and records that it was sent.

Specifically, this module is responsible for:

- Receiving notification requests from other modules - "send this message to this person or role, through this channel, at this priority, for this reason"
- Choosing the right channel for the message - in-app notification, push notification, email, SMS, or any other channel the system supports
- Sending the message through that channel, using the appropriate service or gateway
- Handling delivery failures - if an SMS fails to send, if an email bounces, if a push notification can't be delivered - and reporting those failures so the system can decide what to do
- Recording what was sent, to whom, when, through which channel, and whether it was delivered, so there is a record of the communication
- Supporting priority levels - so that an emergency alert gets through immediately while a routine update can wait or use a less intrusive channel
- Supporting templates - so that messages are consistent and can be updated in one place rather than rewritten in every module that sends them

This module does not decide what messages to send or to whom. That is the job of the modules that know the context - dispatch knows that an ambulance has been assigned and the crew needs to know, the hospital module knows that a patient is en-route and the hospital needs to prepare, the blood bank module knows that a request has been fulfilled and the hospital needs to collect the blood. The notification service is the delivery mechanism, not the decision-maker.

---

## 2. What Data We Track

### 2.1 Notification Request

When another module asks the notification service to send a message, the request records:

- Who the message is for - a specific user, a role, a group, or a channel. For example, "the crew member assigned to this ambulance," "all dispatchers currently on duty," "the hospital administrator of this hospital," "the reporter of this emergency."
- What the message is about - a short subject or label that identifies the kind of notification, so the recipient and the system both know what it is. For example, "ambulance assigned," "patient en-route to your hospital," "blood request fulfilled," "emergency report received."
- What the message says - the body of the message, in the appropriate form for the channel. For an in-app notification, this is a short message. For an email, this is the full email body. For an SMS, this is the text message. The content may be built from a template with specific details filled in.
- What channel to use - in-app, push, email, SMS, or any other channel the system supports. The requesting module specifies the channel, or the notification service chooses based on the priority and the recipient's preferences.
- What priority the message has - high, normal, or low. High-priority messages - like an ambulance assignment or an emergency alert - are sent immediately and may use multiple channels to make sure they get through. Low-priority messages - like a routine report availability notice - can wait and may use a single, less intrusive channel.
- What context the message is about - the emergency, the dispatch, the hospital, the blood bank, the ambulance, or whatever the message relates to. This is used to fill in the template with the right details and to link the notification to the thing it is about.
- When the message should be sent - immediately, in which case it is sent as soon as possible, or at a specific time, in which case it is scheduled. Most notifications in this system are immediate - they are about something that is happening now and needs to be known now.

We do not store the content of every message forever in full detail. We store what was sent, to whom, when, through which channel, and whether it was delivered. We store enough to have a record of the communication and to debug problems, but we do not treat the notification service as a long-term archive of message content. The dispatch module, the hospital module, and the other modules are where the substantive records live - the notification service records the delivery of the messages, not the full context that produced them.

### 2.2 Channel and Delivery Record

For each message that is sent, the module records:

- The notification request that produced it - so we can trace back why this message was sent
- The channel that was used - in-app, push, email, SMS
- The recipient - who the message was sent to, in terms the system can identify (user ID, role, external contact like a phone number or email address)
- When the message was sent
- Whether the message was delivered - for channels where delivery can be confirmed, like SMS delivery receipts or email bounce tracking, we record whether the message was delivered successfully or failed
- If the message failed, why - the error or reason, so the system can decide what to do (retry, escalate, report)
- If the message was retried, the retry history - how many times, when, and whether the retry succeeded

This is the delivery record. It is what the system uses to know that a message got through, and what it uses to debug when a message did not get through. It is also what feeds the accountability story - if a critical notification was not delivered, the record shows that, and the system can act on it.

### 2.3 Templates

The module stores message templates - reusable message formats that other modules can use instead of writing the same message over and over. Each template records:

- A name or identifier for the template - so modules can request it by name
- The template content - the message text, with placeholders for the details that change from one notification to the next. For example, "Ambulance {ambulance_number} has been dispatched to {location}. ETA {eta}."
- The channel or channels the template is for - some templates are for SMS, some for email, some for in-app notifications, some for all of them
- The priority the template implies - so the system knows how urgently to send messages that use this template
- The language or languages the template is in - if the system supports multiple languages, each template can have versions in different languages

Templates are the way the system keeps messages consistent. If the wording of an ambulance assignment message needs to change, it changes in one template, and every module that sends that message uses the updated version. This is better than each module writing its own version of the same message and having them drift apart over time.

### 2.4 Recipient Preferences

If the system supports recipient preferences - and it should, at least in a basic form - the module stores:

- Which channels a user prefers for which kinds of notification - for example, a dispatcher might prefer in-app notifications for routine updates but SMS for critical alerts; an ambulance crew member might prefer push notifications when they are on duty
- Which channels a user has available - whether they have the app installed, whether they have a valid phone number for SMS, whether they have an email address on file
- Any opt-outs or quiet hours - if a user has asked not to receive certain kinds of notifications at certain times, the system respects that, except for high-priority alerts that must get through

Recipient preferences are a way of respecting the people who use the system. They get the notifications they need, through the channels they prefer, without being overwhelmed by notifications they do not want or through channels that do not work for them.

---

## 3. Channels

### 3.1 In-App Notification

An in-app notification is a message that appears inside the application when the user is using it. It shows up as a notification in the application's notification centre, or as a banner or alert in the relevant view.

In-app notifications are the most integrated channel - they appear in the system the user is already using, they can link directly to the relevant part of the system, and they do not depend on external services. They are the right channel for messages that the user will see while they are using the system - for example, a dispatcher seeing a new emergency appear in their queue, or a hospital staff member seeing a notice that an ambulance is en-route.

The limitation of in-app notifications is that they only work when the user is actively using the application. If the user is not logged in, or has the application open in another tab, or is not looking at the right view, they may not see the notification. So in-app notifications are useful, but they are not sufficient on their own for time-critical messages.

### 3.2 Push Notification

A push notification is a message that is sent to the user's device, even when the application is not open. It appears as a notification on the device - on the lock screen, in the notification centre, or as a banner - and the user can tap it to open the application to the relevant part.

Push notifications are the right channel for time-critical messages to users who have the application installed on their mobile device - for example, an ambulance crew member who needs to know immediately that they have been dispatched, or a hospital administrator who needs to know that an ambulance is en-route. Push notifications reach the user even if they are not actively using the application, and they can be attention-getting in a way that in-app notifications are not.

The limitations of push notifications are that they require the user to have the application installed and to have opted in to push notifications, and that delivery depends on the push service being used - Firebase Cloud Messaging for Android and web, Apple Push Notification service for iOS. The system uses a push service, so it is relying on that service to deliver the message, and it can only confirm delivery as well as that service allows.

### 3.3 Email

Email is a message sent to the user's email address. It is appropriate for messages that are not time-critical, or that benefit from a written record, or that contain more detail than a short notification can hold. For example, a report that a hospital's bed capacity has changed, or a summary of the week's dispatch activity, or a confirmation of a blood request that the hospital can keep for its records.

Email is not the right channel for time-critical messages, because email is not guaranteed to be read quickly, and because it depends on the user checking their email. Email is also not the right channel for messages that need to be acted on immediately - an ambulance assignment should not go by email, because the crew may not see it for hours.

The advantage of email is that it is universal - everyone has an email address, and email does not require the user to have the application installed. It is also a good channel for messages that the user may want to keep/refer to later, because emails are easy to store and search.

### 3.4 SMS

SMS is a text message sent to the user's phone number. It is the most reliable channel for reaching someone quickly, because almost everyone has a phone number, SMS does not require an application or an internet connection, and SMS is typically read quickly. SMS is the right channel for critical, time-sensitive messages when the stakes are high and the system needs to be confident the message gets through - for example, an ambulance assignment to a crew member who may not have the app open, or an emergency alert to a dispatcher.

The limitations of SMS are that it costs money per message, it is a limited channel in terms of content (short text only), and it does not provide a rich link back into the system the way a push notification or in-app notification can. SMS is also less private than some other channels, because it goes through the phone network and the content is visible on the device without authentication.

For this project, SMS is a channel that can be used for critical alerts when the system needs to be confident of delivery, but it is not the default channel for every notification. The system should use SMS selectively, for messages where the cost is justified by the importance of the message getting through.

### 3.5 Other Channels (Future)

There are other channels the system could support in the future, but they are not part of this project:

- **Voice calls** - for alerts that need to be heard immediately, or for recipients who are not literate or do not read notifications. This would require a voice gateway and synthetic speech or pre-recorded messages, and it is not part of this project.
- **Radio or pager** - for recipients who use radio or pager systems, such as some emergency services. This would require integration with those systems, and it is not part of this project.
- **Webhook or API callback** - for recipients who are systems rather than people, such as a hospital's own system that wants to be notified when an ambulance is en-route. This is a way of integrating with other systems, and it could be part of a future integration layer, but it is not part of this project.

For this project, the channels are in-app, push, email, and SMS. These are the channels that cover the needs of the system's users - the people who use the application and the people who need to be reached when they are not using it - and they are channels the system can support with reasonable effort.

---

## 4. Who Gets Access to What

### 4.1 The Access Model

The notification service is not something that users interact with directly - there is no "notification service screen" that a dispatcher or hospital administrator logs into. Instead, the notification service is a backend service that other modules use. The access question is not "who can log into the notification service?" but "who can ask the notification service to send a message, and what can they ask it to do?"

The answer is: the modules that know when a notification should be sent. The dispatch module, the ambulance module, the hospital module, the blood bank module, and any other module that needs to notify someone can ask the notification service to send a message. They do so through a defined interface, and the notification service sends the message through the appropriate channel.

The notification service does not let just anyone send messages to anyone. It is used by the system's modules, not by end users directly. If an end user could ask the notification service to send a message to anyone, that would be a way to spam or harass other users. So the notification service is a backend service, used by the system's modules under the rules that those modules follow.

### 4.2 What Modules Can Ask For

Each module can ask the notification service to send messages that are appropriate to its function:

- **The dispatch module** can ask the notification service to send an ambulance assignment to the crew, to notify a hospital that an ambulance is en-route, to notify a blood bank of a request, to notify the reporter of an emergency that the dispatch is progressing, and to send any other notifications that are part of the dispatch workflow.
- **The ambulance module** can ask the notification service to send status change notifications - for example, to notify the dispatcher when the ambulance's status changes, or to notify the hospital when the ambulance is arriving.
- **The hospital module** can ask the notification service to send notifications to hospital staff - for example, that an ambulance is coming, that a bed count has changed, that a blood request has been fulfilled.
- **The blood bank module** can ask the notification service to send notifications to blood bank staff - for example, that a request has been received, that stock is low, that a transfer has been initiated.
- **The auth module** can ask the notification service to send notifications related to account management - for example, a password reset link, a welcome message when an account is created, a notification that a role has been assigned.

Each module asks for the notifications that are part of its function, and the notification service sends them. The notification service does not decide which notifications to send - the modules do - but it is the mechanism through which they are sent.

### 4.3 What the Platform Administrator Can See

The platform administrator can see the notification delivery records - what was sent, to whom, when, through which channel, and whether it was delivered. This is useful for oversight - to know that notifications are being sent and delivered as expected, and to debug problems when they are not.

The platform administrator does not see the full content of every message in a way that is browsable as a stream of messages - the notification service is not a messaging centre for the platform administrator to read. It is a delivery service, and the platform administrator can see its delivery records for oversight and debugging, but the substantive content of the messages is in the modules that produced them.

### 4.4 What the Recipient Sees

The recipient sees the notification through the channel it was sent through - as an in-app notification, a push notification on their device, an email in their inbox, or an SMS on their phone. The recipient does not see the notification service itself - they see the message that the notification service delivered.

The recipient can act on the notification - tap a push notification to open the application to the relevant part, click a link in an email, reply to an SMS if the system supports that - and that action takes them back into the system, where the relevant module handles it.

---

## 5. How Real-Time Updates Work

### 5.1 Immediate Sending for High-Priority Messages

When a module asks the notification service to send a high-priority message - an ambulance assignment, an emergency alert, a notification that a patient is en-route - the notification service sends it immediately through the appropriate channel or channels. There is no delay, no batching, no waiting. The message goes out as soon as the request is received, because the whole point of a high-priority message is that it needs to be known now.

For messages that need to get through urgently, the notification service may use multiple channels - for example, send a push notification and an SMS for an ambulance assignment, so that if the crew does not see the push notification, the SMS provides a backup. This is a decision the requesting module makes - it asks for the channels it wants - and the notification service carries it out.

### 5.2 Delivery Confirmation and Retries

For channels where delivery can be confirmed - SMS delivery receipts, email bounce tracking, push notification delivery reports - the notification service records whether the message was delivered or not. If a message fails to deliver, the notification service records the failure and can retry, if the requesting module asks for retries, or can report the failure back to the requesting module so it can decide what to do.

This is important for reliability. If an ambulance assignment SMS fails to send, the system should know that, and it should either retry or have the dispatch module send the assignment through another channel. The notification service is not a fire-and-forget mechanism - it is a delivery service that reports on whether the delivery succeeded.

### 5.3 Not Real-Time in the Sense of Live Data

The notification service is not itself a source of live data. It does not stream the ambulance's location or the hospital's bed count. It sends messages triggered by events in the other modules - an assignment was made, a status changed, a bed count changed - and those messages reach the recipients through the channels they are sent through.

The real-time aspect of the notification service is that it sends those messages immediately when the event happens, not that it provides a live feed of data. The live data comes from the modules that track the data - the ambulance module, the hospital module, and so on - and the notification service is what turns changes in that data into messages that reach the right people.

---

## 6. Gaps in the Current System and How This Module Changes That

### 6.1 What Exists Today

Today, in most cities and regions, the way people are notified about emergency responses is manual and fragmented:

- **Notifications are made by phone.** When an ambulance is dispatched, the dispatcher calls the crew on the phone or by radio. When an ambulance is en-route to a hospital, the crew calls the hospital, or the dispatcher calls the hospital. When a blood request is fulfilled, the blood bank calls the hospital. Every notification is a phone call, made by a person, to another person.
- **There is no record of the notification.** The phone call happens, and the information is conveyed, but there is no record that the notification was sent, to whom, when, or whether it was received. If there is a question later - "did the hospital know the ambulance was coming?" - the answer is "I think the crew called," or "I think the dispatcher called," with no definitive record.
- **Notifications depend on people being available.** If the crew does not answer the phone, the dispatcher tries again or finds another way to reach them. If the hospital staff member who needs to know is not available, the call goes to voicemail or another person. The notification is only as reliable as the phone call.
- **There is no consistency in what is said.** One dispatcher may tell the crew "you are dispatched to X, ETA Y," while another may say "go to X" without the ETA. One hospital may be told "an ambulance is coming" without the details, while another is told the full situation. The information conveyed depends on who is making the call and what they remember to say.
- **There is no visibility into whether the notification was received.** The dispatcher who makes the call does not know for sure whether the recipient got the message, understood it, and will act on it. They may get a confirmation - "yes, we received your call" - but that confirmation is verbal and not recorded.
- **Notifications are not scalable.** As the number of emergencies, ambulances, hospitals, and blood banks grows, the number of phone calls needed to notify everyone grows with it. A single dispatcher can only make so many calls, and the calls take time away from other work.

This is the baseline. It is not theoretical - it is how notifications work in most emergency response systems today, even in places with emergency numbers, because the notification process is still manual and phone-based.

### 6.2 What This Module Introduces

| Current gap | What this module introduces |
|-------------|----------------------------|
| Notifications are made by phone, manually | Automated notifications through multiple channels - in-app, push, email, SMS - triggered by events in the system, so the right people are notified automatically when something happens |
| No record of the notification | A delivery record for every notification - what was sent, to whom, when, through which channel, and whether it was delivered - so there is a definitive record of the communication |
| Notifications depend on people being available by phone | Notifications reach people through their devices, even when they are not actively using the system - push notifications to their phone, SMS to their number, email to their inbox - so the notification is not limited by whether they answer a phone call |
| No consistency in what is said | Templates for each kind of notification, so that the same message is sent every time, with the right details filled in, and the wording is consistent across all notifications of that type |
| No visibility into whether the notification was received | Delivery confirmation for channels that support it, and retry or escalation for messages that fail, so the system knows whether the notification got through and can act if it did not |
| Notifications are not scalable | Automated notifications scale with the system - one event triggers one notification, regardless of how many emergencies are happening, without requiring a dispatcher to make a phone call for each one |

### 6.3 How It Is Better - The Concrete Improvements

For the dispatcher:

- **Notifications happen automatically.** The dispatcher does not have to make phone calls to notify the crew, the hospital, and the blood bank. The system does it, through the notification service, as part of the dispatch workflow. The dispatcher can focus on the coordination, not on the communication.
- **The dispatcher knows the notification was sent.** The dispatch record shows that the notification was sent, to whom, when, and through which channel. The dispatcher does not have to remember whether they called the hospital or not - the record shows it.
- **The dispatcher is notified of delivery failures.** If a notification fails to deliver, the dispatcher - or the system - is notified, so the dispatcher can act on it. The dispatcher does not have to wonder whether the crew got the assignment.

For the ambulance crew:

- **Notifications arrive on their device, immediately.** The crew gets the assignment as a push notification or SMS, without waiting for a phone call. They can see it even if they are not at a phone. The notification includes the details they need - where to go, what is known, what hospital they are headed to.
- **The notification is consistent.** Every ambulance assignment notification says the same thing, with the same structure, so the crew knows what to expect and what to look for. The template ensures that the details are always included.

For the hospital:

- **The hospital is notified automatically when an ambulance is en-route.** The hospital does not have to wait for the crew to call, or for the dispatcher to call. The notification service sends the notification as part of the dispatch workflow, and the hospital sees it in their application or receives it by email or SMS.
- **The notification includes the relevant details.** The hospital knows what emergency the patient is coming from, what is known about the situation, and when the ambulance is expected to arrive. The template ensures that the right details are included every time.

For the blood bank:

- **The blood bank is notified automatically when a request is placed.** The blood bank does not have to wait for a phone call from the hospital. The notification service sends the notification as part of the blood request workflow, and the blood bank sees it in their application or receives it by email or SMS.
- **The notification includes the request details.** The blood bank knows what group and quantity are needed, how urgent it is, and which hospital needs it. The template ensures that the right details are included.

For the public:

- **The reporter of an emergency can receive updates.** If the system is designed to provide updates to the reporter, the notification service sends them - "your emergency has been received," "an ambulance has been dispatched," "the ambulance is on its way." The reporter is not left in the dark.

For oversight and administration:

- **A delivery record for every notification.** The platform administrator can see what notifications were sent, to whom, when, through which channel, and whether they were delivered. This is what can be reviewed if there is a question about whether a notification was sent or received.
- **Consistency across the system.** Templates ensure that notifications are consistent, so the system speaks with one voice, not with every module sending its own version of the same message.

---

## 7. How This Module Integrates With the Rest of the System

### 7.1 It Is a Backend Service Used by Other Modules

The notification service is not a module that stands alone and is used directly by end users. It is a backend service that the other modules use. The dispatch module, the ambulance module, the hospital module, the blood bank module, and the auth module all ask the notification service to send messages as part of their work.

The integration is simple in concept: a module asks the notification service to send a message, and the notification service sends it through the appropriate channel. The module that asks specifies what message to send, to whom, through which channel, and at what priority. The notification service does the sending and records the delivery.

### 7.2 It Does Not Decide What to Send

The notification service does not decide which notifications to send or to whom. That is decided by the modules that know the context. The dispatch module decides that an ambulance has been assigned and the crew needs to know. The hospital module decides that a patient is en-route and the hospital needs to prepare. The blood bank module decides that a request has been fulfilled and the hospital needs to collect the blood.

The notification service is the delivery mechanism, not the decision-maker. It sends what it is asked to send, through the channels it is asked to use, at the priority it is asked to use. This keeps the notification service focused on its job - delivery - and keeps the decision-making in the modules that have the context to make those decisions.

### 7.3 It Records Delivery, Not Substance

The notification service records that a message was sent, to whom, when, through which channel, and whether it was delivered. It does not record the full context that produced the message - that is in the modules that produced it. The notification service records the delivery of the message, not the decision that led to it.

This is the right separation. The dispatch module records the dispatch - the assignment, the timeline, the outcome. The notification service records that the dispatch module asked for a notification to be sent to the crew, and that it was delivered. The two records are linked - the notification record refers to the dispatch - but they are separate, and each is owned by the module that is responsible for it.

### 7.4 It Depends on External Services for Some Channels

The notification service depends on external services for some of its channels:

- **For push notifications**, it depends on a push service - Firebase Cloud Messaging for Android and web, Apple Push Notification service for iOS. The notification service sends the message to the push service, and the push service delivers it to the device.
- **For SMS**, it depends on an SMS gateway - a service that sends the SMS through the phone network. The notification service sends the message to the gateway, and the gateway sends it as an SMS.
- **For email**, it depends on an email service - either an email server the platform operates, or a service like SendGrid or another email delivery service.

The notification service is not self-contained - it relies on these external services to deliver messages through the channels that need them. This is normal for a notification service - it is the nature of the channels - but it is worth acknowledging that the notification service's reliability is partly dependent on the reliability of these external services.

### 7.5 It Feeds the Accountability Story

The notification service's delivery records feed the accountability story for the system. If a critical notification was not delivered, the delivery record shows that, and the system can act on it. If a question comes up about whether a hospital was notified that an ambulance was coming, the delivery record shows whether the notification was sent and whether it was delivered.

This is valuable for audit, for debugging, and for trust. It means the system's notifications are not a black box - they are a record that can be looked at, understood, and acted on if something goes wrong.

---

## 8. Privacy and Safety Considerations

### 8.1 What We Do Not Store

- **The full content of every message as a long-term archive.** The notification service stores what was sent, to whom, when, and whether it was delivered - enough to have a record of the communication and to debug problems. It does not store the full content of every message forever as a message archive. The substantive content is in the modules that produced the messages.
- **Personal information beyond what is needed to send the message.** The notification service needs to know who to send the message to - a user ID, a phone number, an email address - and it stores that recipient information as much as is needed to send the message. It does not store additional personal information about the recipient beyond what is needed for the notification.

### 8.2 What We Do Store

- **The delivery record.** What was sent, to whom, when, through which channel, and whether it was delivered. This is the record of the communication, and it is what the system uses to know that notifications are getting through.
- **The notification request, in summary.** That a module asked for a notification to be sent, for what context, at what priority, through which channel. The request is linked to the context - the dispatch, the emergency, the hospital - so the record is traceable, but the full context is in the module that made the request.
- **Templates.** The message templates that modules use, so that messages are consistent and can be updated in one place.
- **Recipient preferences.** The channels a user prefers, the channels they have available, any opt-outs or quiet hours, so that the system respects their preferences.

### 8.3 Why We Use Templates

Templates are a privacy and safety feature as well as a consistency feature. By using templates, the system ensures that notifications say only what they are supposed to say - the template defines the content, and the template is reviewed and approved. This reduces the risk of a module accidentally including information it should not include, or of a notification being inconsistent in a way that confuses recipients.

Templates also make it easier to update notifications. If a template needs to change - because the wording is unclear, or because the details it includes need to change - it changes in one place, and every module that uses that template gets the updated version. This is better than each module writing its own messages and having to update them all separately.

### 8.4 Channel Selection and Privacy

The system chooses the channel for each notification based on the priority, the recipient's preferences, and the context. This is a privacy and safety consideration:

- **High-priority notifications** may use multiple channels to make sure they get through - push and SMS, for example - because the importance of the message getting through justifies the use of multiple channels.
- **Routine notifications** may use a single channel, the one the recipient prefers, to avoid overwhelming them with multiple notifications for the same thing.
- **Sensitive notifications** - for example, notifications that relate to a specific patient's emergency - should be sent through channels that are reasonably private. SMS is visible on the device without authentication, so it may not be the best channel for sensitive content. In-app and push notifications require authentication to see the full context, so they are more private for sensitive messages.

The system should choose channels with these considerations in mind, so that notifications are not only effective but also appropriate in terms of privacy and sensitivity.

### 8.5 Delivery Reliability and Safety

The notification service's reliability matters for safety, because notifications are how the system reaches people with time-critical information. If a notification fails to deliver and the system does not know about it, the recipient may not get the information they need, and the response may be affected.

The notification service addresses this by recording delivery and retrying or reporting failures. For channels where delivery can be confirmed, the service confirms delivery and retries if it fails. For channels where delivery cannot be confirmed - some push services, for example, do not provide reliable delivery confirmation - the service sends the message and records that it was sent, but cannot confirm delivery. In those cases, the system may use multiple channels for critical messages, so that if one channel fails, another may get through.

This is a safety consideration: the system should not assume that every notification gets through. It should design for the possibility of failure, by using multiple channels for critical messages and by recording what was sent and whether it was delivered, so that problems can be detected and acted on.

---

## 9. What We Are Explicitly Not Doing

It is worth stating what this module does not attempt, so the scope is honest:

- **A general-purpose messaging platform.** The notification service is for system-triggered notifications - messages that are sent because something happened in the system that the recipient needs to know about. It is not a messaging platform for arbitrary communication between users. Users do not use the notification service to send messages to each other - they use it only to receive notifications from the system.
- **Interactive messaging or conversation.** The notification service sends messages, not conversations. It does not support back-and-forth messaging between users. If a recipient needs to respond to a notification, they do so through the application - by acting on the notification, by opening the relevant part of the application, by using the application's own communication features if it has them. The notification service does not handle the response as a message thread.
- **Voice or video notifications.** The notification service uses text-based channels - in-app, push, email, SMS. It does not send voice calls or video messages. Voice and video would require different infrastructure and different considerations, and they are not part of this project.
- **Scheduled marketing or broadcast messages.** The notification service sends notifications that are triggered by events in the system - an assignment was made, a status changed, a bed count changed. It does not send scheduled marketing messages, promotional messages, or broadcast messages to all users at once, except for system announcements that are part of the platform's operation (for example, a maintenance notification to all users before a scheduled downtime).
- **Message content moderation or filtering.** The notification service sends the messages it is asked to send by the system's modules. It does not moderate or filter the content of those messages - that is the responsibility of the modules that produce them, which are part of the system and are expected to produce appropriate messages. The notification service is a delivery mechanism, not a content reviewer.
- **Reliable delivery confirmation for every channel.** For some channels - some push services, for example - delivery confirmation is not reliable or not available. The notification service records what it can confirm and does not pretend to confirm delivery for channels where it cannot. It may use multiple channels for critical messages to compensate, but it does not guarantee delivery confirmation for every channel.
- **Recipient preferences beyond a basic set.** The notification service supports recipient preferences - which channels they prefer, which channels they have available, opt-outs, quiet hours - but in a basic form. It does not support a full-featured preference centre with granular controls over every kind of notification. The basic form is enough for the project, and a more elaborate preference system would be a future enhancement.

---

## 10. Summary

| Aspect | What we are doing |
|--------|------------------|
| Purpose | Deliver system-triggered notifications to the right people at the right time, through the right channel, with delivery recorded and failures handled, so that the coordination done by the other modules actually reaches the people who need to know |
| Data tracked | Notification requests from other modules - who the message is for, what it is about, what it says, which channel, at what priority, for what context; delivery records - what was sent, to whom, when, through which channel, whether delivered, retry history; templates - reusable message formats with placeholders; recipient preferences - channel preferences, available channels, opt-outs, quiet hours |
| Who gets access | The notification service is a backend service used by the system's modules - dispatch, ambulance, hospital, blood bank, auth - which ask it to send messages as part of their work; the platform administrator can see delivery records for oversight and debugging; recipients see the messages through the channels they are sent through - in-app, push, email, SMS - and act on them through the application |
| How data is shared | Modules ask the notification service to send messages as part of their workflows - the dispatch module for assignments and status changes, the hospital module for en-route notifications and bed changes, the blood bank module for request notifications, the ambulance module for status changes, the auth module for account notifications; the notification service sends them and records delivery; it does not decide what to send or to whom - the modules do - and it does not record the full context, only the delivery |
| Real-time | High-priority messages sent immediately when the event happens; delivery confirmed and retried for channels that support it; multiple channels used for critical messages to increase the chance of delivery; the notification service is not a source of live data but sends messages triggered by live events in the other modules |
| Gaps addressed | Notifications today are manual phone calls with no record, depend on people being available, are inconsistent in what is said, have no visibility into receipt, and do not scale - this module replaces that with automated, multi-channel, templated, recorded, reliable notifications that scale with the system |
| What it does not do | General-purpose messaging, interactive conversations, voice or video notifications, marketing or broadcast messages, content moderation, reliable delivery confirmation for every channel, elaborate preference management |
| Privacy stance | Does not store full message content as a long-term archive; stores only what is needed for delivery and debugging; uses templates to ensure appropriate content; chooses channels with privacy in mind - sensitive messages through more private channels; records delivery for accountability without overexposing message content |
| Integration | Backend service used by all other modules; depends on external push, SMS, and email services for some channels; records delivery and feeds accountability; does not decide what to send - the modules do; does not record full context - the modules do; records only what is needed for delivery and debugging |

---

*This completes the Notification Service Module report in the same text-focused, minimal-technical style. We have now covered Auth, Hospital, Ambulance, Blood Bank, Dispatch, and Notification Service - the core modules of the system.*

*The remaining pieces, if you want to go deeper, are the Public Emergency Interface (how emergencies enter the system and how citizens interact with it, including the public map and the emergency reporting flow) and the Analytics Module (what the system learns from the dispatch records, the response times, the utilization data, and the demand patterns, and how that is presented to the platform administrator and to the hospitals and operators for improvement).*

*Which of these, if any, would you like to detail next? Or would you prefer to move on to a different kind of document - for example, a consolidated system-level view, a data flow diagram in words, or a document that ties all the modules together into a single narrative of how an emergency flows through the system from report to resolution?*