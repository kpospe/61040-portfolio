# Assignment 2: Functional Design

## Problem Statement

### Problem domain — Team coordination

Coordinating rehearsals, sectionals, and performances across busy calendars is a common challenge for student organizations, community groups, and professional teams alike.
Finding times to meet in subgroups can require a greater coordination effort from all parties involved.
As a president of my dance team, I’ve seen this friction many times firsthand.

---

### Problem —> Quorum-first scheduling for rehearsals

Finding times that work for the right people is the pain point: sometimes we need specific members (e.g., choreographer + lifts group) and other times we just need a minimum number (e.g., “any 8+ dancers for formations”). Existing tools optimize for “everyone” or “pick a time,” not “must include A/B OR at least N of this subgroup.” The result is endless back-and-forth, missed practice windows, and late-stage scrambles.

---

### Stakeholders

- **Team members (dancers)** — want minimal poll fatigue and clear commitments.  
- **Team leads (captains/choreographers)** — need fast quorum decisions that honor constraints and are auditable.  
- **Event/space schedulers (studios, campus rooms)** — benefit from earlier, reliable bookings and fewer changes.  
- **Tech officers / club ops** — need simple exports/integrations (calendar/rosters) to reduce manual work.

---

### Evidence & comparables

#### Evidence of the problem

**Meetings are very common in professional settings:**

Up to 25% of professionals have more than 15 meetings every week, which is why it is so beneficial to streamline the scheduling process. [Doodle](https://assets.ctfassets.net/p24lh3qexxeo/axrPjsBSD1bLp2HYEqoij/d2f08c2aaf5a6ed80ee53b5ad7631494/Meeting_Report_2019.pdf?)

**Proper attendees matter to meeting success:**

In a study of 19M meetings, 31% of respondents cited the result of a poorly-organized meeting was dealing with "irrelevant attendees slowing [their] progress." [Inc](https://www.inc.com/peter-economy/a-new-study-of-19000000-meetings-reveals-that-meetings-waste-more-time-than-ever-but-there-is-a-solution.html)

**Scheduling is becoming a popular way to coordinate with others:**

Reporting shows students are increasingly planning social outings through scheduling apps, highlighting both the ubiquity of scheduling needs and the growing coordination load in student contexts. [Wall Street Journal](https://www.wsj.com/lifestyle/college-students-schedule-google-calendar-e2a3d796?)

**Hyper-Scheduling can be more helpful than Spontaneity:**

A Forbes Council Member recounts how he felt most flexible and free after hyper-scheduling all every hour of every day on Google Calendar [Forbes](https://www.forbes.com/councils/forbesbusinesscouncil/2023/04/14/how-hyper-scheduling-moves-the-needle-in-business-and-in-life/)

---

#### Comparables

**[When2Meet](https://www.when2meet.com/)**

- fast, zero-login availability heatmaps
- great for “everyone” but no minimum-quorum logic or “must-include” constraints
- subgroup views are difficult to use.

**[Rallly](https://rallly.co/)**

- modern group-polling (open source, no login), but still poll-centric
- lacks constraint solving like “N of cohort X” or required individuals baked into the result

**[Google Calendar](https://calendar.google.com/)**

- Find a time’ / Appointment Schedules — good for small, shared-calendar groups and 1:1 booking
- not designed for large polls or quorum constraints across many participants 
[Google Help](https://support.google.com/a/answer/1626902?hl=en)

**[Clockwise](https://wf.getclockwise.com/overview)**

- streamline 1:1/round-robin booking
- lacks constraint solving like “N of cohort X” or required individuals

---

---

## Application Pitch

**Name:** Quorum

**Motivation:**
Quorum finds meeting times for groups, accounting for subteam-specific roles, minimum or maximum headcount requirements, or must-include individuals.

**Key features:**

1. **Quorum Polls (N-of-K + Must-Include)**

- **What:**
Create an availability poll that encodes constraints like “Manager + Cashier + any 6 of Line Cooks.”  
- **Why it helps:**
Replaces “everyone or nothing” with constraint-aware matches, so you lock rehearsals sooner without waiting on stragglers.  
- **Impact:**
Leads can progress blocking choreo;
members see fewer redundant pings;
schedulers get earlier,
firmer room bookings.

2. **Subgroup Targeting & Saved Cohorts**

- **What:**
Define cohorts (e.g., “Cashiers,” “C-Suite,” “Employees on Probation”), then target polls at those groups or mix them (AND/OR) per session.  
- **Why it helps:**
Mirrors how teams actually work (departments), eliminating manual filtering.  
- **Impact:**
Members only get relevant polls; leads run parallel sectionals without chaos.

3. **Auto-Publish with Attendance Hooks**

- **What:**
Set a decision deadline;
when constraints are met,
Quorum auto-books the top slot, pushes calendar invites/ICS, and opens an attendance check-in.  
- **Why it helps:**
Collapses “decide -> announce -> add to calendars -> track who showed” into one flow.  
- **Impact:**
Leads save time;
members’ calendars update instantly;
ops gets clean records for accountability and space use.

---

---

## Concept Design

### Type Definitions (generic types used below)

- **User** — An abstract identifier for a person (e.g., team member).  
- **TimeWindow** — A half-open interval with fields `start` and `end` where `start < end`.  
- **Location** — A simple descriptor for where a session happens (e.g., room name, address).  
- **Code** — A short authentication token (e.g., 4–6 digit string) used for check-in.  

These are generic types so the implementation can later bind them to specific representations (e.g., a particular time library or ID type).

---

### Concepts Specs

**Concept 1:** Cohorts[User]

**Purpose** Define and maintain named subsets of users for targeted scheduling.

**Principle**

- Team leads create named cohorts (e.g., “Lifts”, “Front Row”) and manage membership.
- Other concepts can reference cohorts as opaque sets without assuming anything else about users.

**State**

```text
a set of Cohort with:
  a name String
  a members set of User
```

**Actions**

```text
createCohort(name: String): (cohort: Cohort)
  requires no existing Cohort has name
  effects  create cohort,
           set cohort's name to name,
           set cohort's members to a null set of User,
           return cohort
```

```text
renameCohort(cohort: Cohort, newName: String)
  requires cohort exists,
           no Cohort has newName
  effects  set cohort.name = newName
```

```text
addMember(cohort: Cohort, user: User)
  requires cohort exists
  effects add user to cohort.members
```

```text
removeMember(cohort: Cohort, user: User)`
  requires cohort exists,
           user is in cohort's members
  effects  remove user from cohort's members
```

```text
deleteCohort(cohort: Cohort)
  requires cohort exists
  effects  remove cohort from the set of Cohort
```

---

**Concept 2:** Availability[User, TimeWindow]

**Purpose**  
Capture up-to-date availability windows per user for later composition.

**Principle**  
Users (or an importer) post availability windows,
Each user has a current “snapshot” that can be replaced wholesale.
Individual historical edits are not tracked.

**State**

```text
a set of Snapshot with
  an owner User
  a windows set of TimeWindow
  a version Number
```

**Invariants**

For any snapshot:

- Windows are pairwise non-overlapping.  
- For every `window in windows`, `window.start < window.end`.

**Actions**

```text
updateSnapshot (owner: User, windows: Set<TimeWindow>, version: Number): (snapshot: Snapshot)
  requires for all window in windows, window.start < window.end
  effects
           if a Snapshot exists for owner and version >= current.version then
             update its windows and version
           else
             create a new Snapshot for owner with these windows and version
           return that Snapshot
```

```text
clear (owner: User)
  requires a snapshot for owner exists
  effects  remove the snapshot for owner
```

```text
get (owner: User): (windows: Set<TimeWindow>)
  requires a snapshot for owner exists
  effects  return windows for that snapshot (no mutation)
```

---

**Concept:** QuorumPoll[User, Cohort, TimeWindow]

**Purpose**  
Find times that satisfy “must-include” people and/or minimum headcount constraints.

**Principle**  
A lead opens a poll with candidate slots and invited users. Invitees mark availability on those slots.
Constraints (must-include, minimum total, minimum per cohort) define feasibility;
a lead or sync later publishes a session.

**State**

```text
a set of Poll with
  a title String
  an invited set of User
  a candidates set of TimeWindow
  a responses Map from User to a set of TimeWindow // subset of candidates
  a mustInclude set of User // mustInclude is a subset invited
  a minTotal Number? // optional overall threshold
  a minPerCohort Map from Cohort to Number // optional thresholds per cohort
  a decisionDeadline Timestamp
  a status of DRAFT or OPEN or LOCKED or EXPIRED // Enum
```

**Actions**

```text
createPoll (title: String, invited: Set<User>, candidates: Set<TimeWindow>, decisionDeadline: Timestamp): (poll: Poll)
  requires candidates is not a null set
  effects  create a poll with:
           - fields set as given  
           - responses is not an empty map 
           - mustInclude is not an empty set 
           - minTotal is null 
           - minPerCohort is an empty map // Mapping  
           - set status to Open 
           - return poll
```

```text
setConstraints (poll: Poll, neededAttendees: Set<User>, minTotal: Number?, minPerCohort: Map<Cohort, Number>)
  requires poll.status is Open,
           neededAttendees is a subset poll.invited
  effects  poll.mustInclude = neededAttendees  
                              poll.minTotal is set to minTotal  
                              poll.minPerCohort is set to minPerCohort
```

```text
respond (poll: Poll, user: User, available: Set<TimeWindow>)
  requires poll.status is Open,
           user is in poll.invited,
           available is a subset of poll.candidates
  effects  set poll.responses[user] to available
```

```text
withdrawResponse (poll: Poll, user: User)
  requires poll.status is Open,
           user is in poll.invited,
           user is in domain(poll.responses) // poll has a response for this user
  effects  remove user from poll.responses
```

```text
lock (poll: Poll)
  requires poll.status is Open
  effects  set poll.status to Locked
```

```text
expire (poll: Poll)
  requires poll.status is Open
  effects  set poll.status to Expired
```

```text
feasibleSlots (poll: Poll): (slots: Set<TimeWindow>, witness: Map<TimeWindow, Set<User>>)
  requires poll.status is Open or Locked
  effects  no state change;
           let slots be the set of candidate TimeWindow time such that:
             - poll.mustInclude is a subset of responders for time, and
             - if poll.minTotal is set, then number of responders for time ≥ poll.minTotal, and
             - for each (cohort, minNumber) in poll.minPerCohort,
                 number of responders who are in cohort.members ≥ minNumber
           for each such time in slots,
             witness[time] = some set of responders for time satisfying the constraints
```

---

**Concept:** SessionPublishing[User, TimeWindow, Location]

**Purpose**  
Publish a finalized meeting to participants and manage its lifecycle.

**Principle**  
A lead (or a sync) creates a session with a chosen slot and participants, then publishes/cancels/updates it.
The concept exports calendar artifacts without assuming a particular calendar provider.

**State**

```text
a set of Session with
  a title String
  a slot TimeWindow
  a participants set of User
  a location Location
  a status of SCHEDULED or CANCELLED or COMPLETED // Enum
  an icsUID String? // optional external identifier
```

**Actions**

```text
publish (title: String, slot: TimeWindow, participants: Set<User>, location: Location?): (session: Session)
  requires participants is not an empty set
  effects  create session with status Scheduled;
           return session
```

```text
updateTime (session: Session, newSlot: TimeWindow)
  requires session.status = Scheduled
  effects  set session.slot = newSlot
```

```text
updateParticipants (session: Session, newParticipants: Set<User>)
  requires session.status = Scheduled,
           newParticipants is not a null set
  effects  set session.participants = newParticipants
```

```text
cancel (session: Session)
  requires session.status = Scheduled
  effects  set session.status = Cancelled
```

```text
exportICS (session: Session): (icsData: Blob)
  requires session.status = Scheduled
  effects  no state mutation;
           return an iCalendar blob for session
```

---

### Concept: Attendance[User, Session, Code]

**Purpose**  
Track RSVP and day-of attendance for scheduled sessions.

**Principle**  
Once a session is published, participants can RSVP.
Near start time, check-in opens with a code;
check-ins are recorded within a time window.

**State**

```text
a set of Tracker with
  a session Session
  an rsvp Map from User to {NoResponse or Accepted or Declined or Tentative}
  a checkInOpen Boolean
  a code Code?
  a windowStart Timestamp?
  a windowEnd Timestamp?
  a checkIns Map from User to Timestamp
  ```

**Actions**

```text
init (session: Session): (tracker: Tracker)
  requires no tracker exists for session
  effects  create tracker with
                 - rsvp[user] = NoResponse for each user in session.participants
                 - set checkInOpen = false  
                 - return tracker
```

```text
setRSVP (tracker: Tracker, user: User, status: Enum{Accepted, Declined, Tentative})
  requires user is in the domain(tracker.rsvp) // tracker has RSVP for this user
  effects  set tracker.rsvp[user] = status
```

```text
openCheckIn(tracker: Tracker, code: Code, windowStart: Timestamp, windowEnd: Timestamp)
  requires tracker.checkInOpen = false,
           windowStart < windowEnd
  effects  set these parameters to the associated variables in tracker,
           set tracker.checkInOpen = true
```

```text
checkIn (tracker: Tracker, user: User, code: Code, now: Timestamp)
  requires tracker.checkInOpen is true 
           user is in the domain(tracker.rsvp), // tracker has RSVP for this user,
           code = tracker.code,
           tracker.windowStart ≤ now ≤ tracker.windowEnd
  effects  record tracker.checkIns[user] = now
```

```text
closeCheckIn (tracker: Tracker)
  requires tracker.checkInOpen = true
  effects  set tracker.checkInOpen = false
```

---

---

## Essential Synchronizations

**Sync 1: Cohort Change Updates Feasibility**

```text
sync cohort_change_updates_feasibility
when Cohorts.addMember(cohort, user) or Cohorts.removeMember(cohort, user)
where an open poll exists such that cohort is in the keys of poll.minPerCohort
then for each open poll with cohort in the keys of poll.minPerCohort, call QuorumPoll.feasibleSlots(poll)

```

**Sync 2: Availability Autofill**

```text
sync availability_prefills_poll_response
when Availability.updateSnapshot(owner, windows, version)
where there exists an open poll with owner in poll.invited
      and poll has an AutoFill flag
then for each such poll,
       QuorumPoll.respond(poll, owner, windows ∩ poll.candidates)
```

**Sync 3: Autopublish Quorum**

```text
sync auto_publish_on_quorum
when QuorumPoll.feasibleSlots(poll) returns nonempty Slots
where poll.status = Open,
      current time <= poll.decisionDeadline,
      poll has AutoPublish flag
then choose slot in Slots by rule “earliest with maximum satisfied participants”
     let participants = witness[slot]
     SessionPublishing.publish(poll.title, slot, participants, location = null)
     QuorumPoll.lock(poll)

  ```

**Sync 4: Poll hits Decision Deadline**

```text
sync deadline_publish_or_expire
when current time reaches poll.decisionDeadline
where poll.status = Open
then if QuorumPoll.feasibleSlots(poll) is not empty, then
       choose slot and participants by the same rule as auto_publish_on_quorum
       SessionPublishing.publish(title, slot, participants, location = null)
       QuorumPoll.lock(poll)
     else
       QuorumPoll.expire(poll)

```

**Sync 5: Track Published Syncs**

```text
sync attendance_bootstrap
when SessionPublishing.publish(...) returns session
then Attendance.init(session)
```

**Brief note on roles & generic instantiation**

- `Cohorts` defines opaque groups of `User`.  
- `Availability` stores `TimeWindow` snapshots per `User`.  
- `QuorumPoll` references `User`, `Cohort`, and `TimeWindow` generically and does not depend on how cohorts or time are implemented.  
- `SessionPublishing` is independent of polls; polls and sessions are connected via syncs.  
- `Attendance` is bound to `Session` but treats it as an opaque identifier.

---

---

## UI Interface

Here are some high-level sketches for the UI I plan to implement.

![Dashboard](/assets/Dashboard.pdf)

![Creating New Poll](/assets/New_Poll.pdf)

![Availability Heatmap](/assets/Availability_Heatmap.pdf)

![Scheduling Feasibility](/assets/Scheduling_Feasibility_Summary.pdf)

![Session Attendance Tracking](/assets/Session_Attendance_Tracking.pdf)

---

---

## User Journey — Bre Schedules a Role-Dependent Basketball Practice

Bre is the captain of her intramural basketball team, and she’s trying to schedule a weekend practice before playoffs
This session needs **both team captains, at least four defensive players, and ideally anyone from the new recruits** group so they can integrate into drills.
Normally, coordinating all of this requires dozens of messages and manual interpretation of teammates’ texts like “I’m free after church” or “can do mornings but not afternoon.”
Today, Bre opens Quorum to avoid that chaos.

1. Triggering the need (Dashboard – Sketch 1)

Bre lands on the dashboard, where she sees her team’s upcoming commitments and past polls.
With playoffs approaching, she taps “Create New Poll” to start scheduling a constrained practice session.

2. Creating a new poll with role-based requirements (New Poll – Sketch 2)

On the poll creation screen, she fills in the title (“Playoff Practice”), invites the whole team, and selects predefined cohorts—Captains, Defense, Newbies.
She marks the two captains as must-include, sets Defense ≥ 4, and a minimum total of 7 players.
She adds candidate windows across Saturday and Sunday.

Her teammates tend to send natural-language availability instead of structured times, so Bre enables the AI availability parser.
Quorum will automatically interpret messages like “free after 3pm Saturday” or “I can do early Sunday but not late” and convert them into usable blocks.
She sets a decision deadline for Friday at 8PM and submits the poll.

3. Watching responses and AI-parsed availability flow in (Availability Heatmap – Sketch 3)

As players respond (some by clicking slots, others by letting the AI interpret their text), Bre sees the heatmap start to fill.
Darker bands show strong overlap; lighter ones show weak availability.
She toggles “Show Feasible Only”, instantly hiding windows that fail the captain requirement or the defense-minimum threshold.
When she taps a slot, she sees which players match the requirements and which ones the AI interpreted availability for—making it easy to confirm accuracy.

4. Reviewing constraint-aligned options (Feasibility Summary – Sketch 4)

By Thursday night, the Feasible Slots Summary ranks all candidate windows automatically.
One slot—Sunday 10AM–12PM—shows both captains available, five defensive players present, and two newbies able to attend.
She expands the slot card to see how each constraint is satisfied.
The explanation confirms everything: Quorum integrated the AI-parsed messages correctly, and the practice meets all thresholds.

Bre taps “Schedule This Time,” and Quorum publishes the session.
The session page shows the final time, roster, and an “Add to Calendar” button her teammates can use immediately.
No group chat spam, no uncertainty.

6. Tracking attendance on the day of (Attendance Check-In Flow – Sketch 5)

Ten minutes before practice begins, Bre opens the Attendance panel.
She taps “Open Check-In Window,” and Quorum generates a passcode that players enter on their phones.
She watches check-ins appear in real time—green checks beside players’ names.
After warm-ups, she closes the check-in window to finalize the roster.

Outcome

Bre scheduled practice an entire day earlier than usual with zero misinterpretation of teammates' free-form texts.
The system respected her role-specific constraints, highlighted the strongest overlap, and handled the administrative load—leaving Bre free to focus on coaching instead of coordinating.
The journey demonstrates how AI-augmented availability parsing and constraint-aware scheduling combine to solve a problem that affects every student team trying to find time together.
