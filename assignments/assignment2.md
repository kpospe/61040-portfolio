# Assignment 2: Functional Design

## Problem statement

### Problem domain — Team coordination

Student clubs and performance teams (dance, a cappella, theater, sports) juggle rehearsals, sectionals, and shows around dense class schedules.
As a captain/treasurer on Mocha Moves, I’ve seen coordination overhead eclipse creative work; small frictions (polls, DM wrangling, reschedules) scale badly with team size.

### Problem — Quorum-first scheduling for rehearsals

Finding times that work for the right people is the pain point: sometimes we need specific members (e.g., choreographer + lifts group) and other times we just need a minimum number (e.g., “any 8+ dancers for formations”).
Existing tools optimize for “everyone” or “pick a time,” not “must include A/B OR at least N of this subgroup.”
The result: endless back-and-forth, missed practice windows, and late-stage scrambles.

### Stakeholders

Team members (dancers): want minimal poll fatigue; clear commitments.

Team leads (captains/choreographers): need fast quorum decisions that honor constraints; auditability.

Event/space schedulers (studios, campus rooms): benefit from earlier, reliable bookings and fewer changes.

Tech officers / club ops: need simple exports/integrations (calendar/rosters) to reduce manual work.

### Evidence & comparables

#### Evidence of the problem

Scheduling waste is real and large.
Doodle’s report (6,500 pros; 19M meetings analyzed) quantified substantial time lost to coordination, with hours per week burned on logistics—classic group-scheduling friction our use case inherits. 
[Doodle](https://assets.ctfassets.net/p24lh3qexxeo/axrPjsBSD1bLp2HYEqoij/d2f08c2aaf5a6ed80ee53b5ad7631494/Meeting_Report_2019.pdf?)
[Inc](https://www.inc.com/peter-economy/a-new-study-of-19000000-meetings-reveals-that-meetings-waste-more-time-than-ever-but-there-is-a-solution.html)

Hyper-scheduling among college students.
Reporting shows students plan nearly everything in Google Calendar—highlighting both the ubiquity of scheduling and the growing coordination load in student contexts. 

[Wall Street Journal](https://www.wsj.com/lifestyle/college-students-schedule-google-calendar-e2a3d796?utm_source=chatgpt.com)
[New York Times](https://www.nytimes.com/2025/08/16/business/leisure-crafting.html)
[Forbes](https://www.forbes.com/councils/forbesbusinesscouncil/2023/04/14/how-hyper-scheduling-moves-the-needle-in-business-and-in-life/)

#### Comparables (and limitations for this problem)

3) When2Meet — fast, zero-login availability heatmaps; great for “everyone” but no native minimum-quorum logic or “must-include” constraints; multi-TZ and subgroup views are limited. 
[When2Meet](https://www.when2meet.com/)

4) Rallly — modern group-polling (open source, no login), but still poll-centric; lacks constraint solving like “N of cohort X” or required individuals baked into the result. 
[Rallly](https://rallly.co/)
GitHub
+2

5) Google Calendar ‘Find a time’ / Appointment Schedules — good for small, shared-calendar groups and 1:1 booking; not designed for large polls or quorum constraints across many participants without shared access. 
[Google Help](https://support.google.com/a/answer/1626902?hl=en)

6) Clockwise/links schedulers — streamline 1:1/round-robin booking; still not “we need __ Role + __ Role + any _# others this weekend.” 
[Clockwise](https://wf.getclockwise.com/overview)

## Application Pitch
Name

Quorum

Motivation (one sentence)

When teams only need the right subset—not everyone—Quorum finds rehearsal times that satisfy must-include people and/or minimum headcount in one pass, cutting poll fatigue and reschedules.

Key features (three)

Quorum Polls (N-of-K + Must-Include)
What: Create an availability poll that encodes constraints like “Captain + Choreo + any 6 of Lifts Group.”
Why it helps: Replaces “everyone or nothing” with constraint-aware matches, so you lock rehearsals sooner without waiting on stragglers.
Impact: Leads can progress blocking choreo; members see fewer redundant pings; schedulers get earlier, firmer room bookings.

Subgroup Targeting & Saved Cohorts
What: Define cohorts (e.g., “Front Row,” “Lifts,” “New Members”), then target polls at those groups or mix them (AND/OR) per session.
Why it helps: Mirrors how teams actually work (sections), eliminating manual filtering.
Impact: Members only get relevant polls; leads run parallel sectionals without chaos.

Auto-Publish with Attendance Hooks
What: Set a decision deadline; when constraints are met, Quorum auto-books the top slot, pushes calendar invites/ICS, and opens an attendance check-in.
Why it helps: Collapses “decide → announce → add to calendars → track who showed” into one flow.
Impact: Leads save time; members’ calendars update instantly; ops gets clean records for accountability and space use.

## Concept specifications

concept Cohorts[User]
purpose define and maintain named subsets of users for targeted scheduling
principle team leads create named cohorts (e.g., “Lifts”, “Front Row”) and manage membership;
          other concepts can reference cohorts as opaque sets without assuming anything else about users
state
  a set of Cohort with
    name String
    members Set<User>
actions
  createCohort(name: String): (c: Cohort)
    requires no Cohort with name exists
    effects create c with name and members = ∅; return c

  renameCohort(c: Cohort, newName: String)
    requires c exists and no Cohort with name = newName
    effects set c.name := newName

  addMember(c: Cohort, u: User)
    requires c exists
    effects add u to c.members

  removeMember(c: Cohort, u: User)
    requires c exists and u ∈ c.members
    effects remove u from c.members

  deleteCohort(c: Cohort)
    requires c exists
    effects remove c

concept Availability[User, TimeWindow]
purpose capture up-to-date availability windows per user for later composition
principle users (or an importer) post availability windows; availability can be replaced with a fresh snapshot
state
  a set of Snapshots with
    owner User
    windows Set<TimeWindow>  // each window has inclusive start < exclusive end
    version Number
invariants
  for any Snapshot, windows are pairwise non-overlapping; each window.start < window.end
actions
  upsertSnapshot(owner: User, windows: Set<TimeWindow>, version: Number): (s: Snapshot)
    requires for all w ∈ windows, w.start < w.end
    effects if snapshot for owner exists and version ≥ current.version,
              replace its windows and version; else create new snapshot; return snapshot

  clear(owner: User)
    requires snapshot for owner exists
    effects remove the snapshot

  get(owner: User): (windows: Set<TimeWindow>)
    requires snapshot for owner exists
    effects return the snapshot.windows (no mutation)

concept QuorumPoll[User, Cohort, TimeWindow]
purpose find times that satisfy “must-include” people and/or minimum headcount constraints
principle a lead opens a poll with candidate slots and invited users; invitees mark available slots;
          constraints (must-include, min-total, min-per-cohort) define feasibility; lead locks or a sync can publish
state
  a set of Polls with
    title String
    invited Set<User>
    candidates Set<TimeWindow>
    responses Map<User, Set<TimeWindow>>   // subset of candidates
    mustInclude Set<User>                  // must ⊆ invited
    minTotal Number?                       // optional overall headcount threshold
    minPerCohort Map<Cohort, Number>       // optional thresholds per cohort
    decisionDeadline Timestamp
    status Enum{Draft, Open, Locked, Expired}

actions
  createPoll(title: String, invited: Set<User>, candidates: Set<TimeWindow>, decisionDeadline: Timestamp): (p: Poll)
    requires candidates ≠ ∅
    effects create p with fields set, responses = ∅, mustInclude = ∅,
            minTotal = null, minPerCohort = {}, status = Open; return p

  setConstraints(p: Poll, must: Set<User>, minTotal: Number?, minPerC: Map<Cohort, Number>)
    requires p.status = Open and must ⊆ p.invited
    effects set p.mustInclude := must; p.minTotal := minTotal; p.minPerCohort := minPerC

  respond(p: Poll, u: User, available: Set<TimeWindow>)
    requires p.status = Open and u ∈ p.invited and available ⊆ p.candidates
    effects set p.responses[u] := available

  withdrawResponse(p: Poll, u: User)
    requires p.status = Open and u ∈ p.invited and u ∈ dom(p.responses)
    effects remove u from p.responses

  lock(p: Poll)
    requires p.status = Open
    effects set p.status := Locked

  expire(p: Poll)
    requires p.status = Open
    effects set p.status := Expired

  feasibleSlots(p: Poll): (slots: Set<TimeWindow>, witness: Map<TimeWindow, Set<User>>)
    requires p.status ∈ {Open, Locked}
    effects compute and return the set of candidate slots for which:
            (1) p.mustInclude ⊆ responders for that slot
            (2) if p.minTotal set, |responders| ≥ p.minTotal
            (3) for each (c, n) in p.minPerCohort, |responders ∩ c.members| ≥ n
            also return one witness set per slot (no mutation)

concept SessionPublishing[User, TimeWindow, Location]
purpose publish a finalized rehearsal/session to participants and manage its lifecycle
principle a lead (or a sync) creates a session with a chosen slot and participants, then publishes/cancels/updates it;
          exports calendar artifacts without assuming a particular calendar provider
state
  a set of Sessions with
    title String
    slot TimeWindow
    participants Set<User>
    location Location?
    status Enum{Scheduled, Cancelled, Completed}
    icsUID String?   // optional external identifier
actions
  publish(title: String, slot: TimeWindow, participants: Set<User>, ?location: Location): (s: Session)
    requires participants ≠ ∅
    effects create s with status = Scheduled; return s

  updateTime(s: Session, newSlot: TimeWindow)
    requires s.status = Scheduled
    effects set s.slot := newSlot

  updateParticipants(s: Session, newParticipants: Set<User>)
    requires s.status = Scheduled and newParticipants ≠ ∅
    effects set s.participants := newParticipants

  cancel(s: Session)
    requires s.status = Scheduled
    effects set s.status := Cancelled

  exportICS(s: Session): (icsData: Blob)
    requires s.status = Scheduled
    effects return an iCalendar blob for s (no mutation)

concept Attendance[User, Session, Code]
purpose track RSVP and day-of attendance for scheduled sessions
principle once a session is published, participants can RSVP; near start time, check-in opens with a code; check-ins are recorded
state
  a set of Trackers with
    session Session
    rsvp Map<User, Enum{NoResponse, Accepted, Declined, Tentative}>
    checkInOpen Flag
    code Code?
    windowStart Timestamp?
    windowEnd Timestamp?
    checkIns Map<User, Timestamp>   // check-in time
actions
  init(session: Session): (t: Tracker)
    requires no Tracker exists for session
    effects create tracker with rsvp initialized to NoResponse for each session.participant;
            checkInOpen = false; return t

  setRSVP(t: Tracker, u: User, status: Enum{Accepted, Declined, Tentative})
    requires u ∈ dom(t.rsvp)
    effects set t.rsvp[u] := status

  openCheckIn(t: Tracker, code: Code, windowStart: Timestamp, windowEnd: Timestamp)
    requires t.checkInOpen = false and windowStart < windowEnd
    effects set t.code := code; t.windowStart := windowStart; t.windowEnd := windowEnd; t.checkInOpen := true

  checkIn(t: Tracker, u: User, code: Code, now: Timestamp)
    requires t.checkInOpen = true and u ∈ dom(t.rsvp) and code = t.code and t.windowStart ≤ now ≤ t.windowEnd
    effects record t.checkIns[u] := now

  closeCheckIn(t: Tracker)
    requires t.checkInOpen = true
    effects set t.checkInOpen := false

## Essential synchronizations (representative)

sync cohort_change_updates_feasibility
when Cohorts.addMember(c, u) or Cohorts.removeMember(c, u)
then for each open poll p in QuorumPoll where c ∈ keys(p.minPerCohort):
       recompute QuorumPoll.feasibleSlots(p)

sync availability_prefills_poll_response
when Availability.upsertSnapshot(owner=u, windows=W, version=_)
where exists open poll p with u ∈ p.invited and p has AutoFill flag (stored outside QuorumPoll or as a poll option)
then QuorumPoll.respond(p, u, W ∩ p.candidates)

sync auto_publish_on_quorum
when QuorumPoll.feasibleSlots(p) returns nonempty S
and current time ≤ p.decisionDeadline
and p.status = Open
and p has AutoPublish flag (stored outside QuorumPoll or as a poll option)
then choose t ∈ S by rule “earliest slot with maximum satisfied participants”
     let P := witness[t]
     SessionPublishing.publish(title=p.title, slot=t, participants=P, ?location=null)
     QuorumPoll.lock(p)

sync deadline_publish_or_expire
when current time reaches p.decisionDeadline and p.status = Open
then if QuorumPoll.feasibleSlots(p) ≠ ∅:
       choose best t and P by same rule; SessionPublishing.publish(p.title, t, P, ?location=null); QuorumPoll.lock(p)
     else
       QuorumPoll.expire(p)

sync attendance_bootstrap
when SessionPublishing.publish returns s
then Attendance.init(s)

**Brief note on roles & generic instantiation:**

Cohorts defines opaque groups of User. Other concepts never assume any structure of User beyond identity.

Availability stores TimeWindow snapshots per User. A TimeWindow is generic so you can later bind it to your time library’s type.

QuorumPoll references User, Cohort, and TimeWindow generically. It does not depend on how cohorts are implemented—only that a cohort denotes a set of users (provided at sync-time). It stores local responses so it still works without Availability; the availability sync simply pre-fills.

SessionPublishing is independent of polls; it can schedule anything given a slot and participants. Poll-driven publishing is achieved via syncs.

Attendance is bound to Session (from SessionPublishing) but treats it as an opaque identifier; it doesn’t inspect the session beyond identity.

Invariants to keep in mind (not restated in every spec):

TimeWindow validity (start < end) everywhere.

In QuorumPoll, mustInclude ⊆ invited.

In Attendance, dom(rsvp) = participants at init, and check-ins can only occur in the open window.

## UI Interface

## User journey (captain scheduling a constrained rehearsal)

Brianna, Mocha Moves captain, needs a Saturday rehearsal that must include the choreographer and at least 6 dancers from the Lifts cohort.

Kickoff. She lands on the dashboard and hits New Quorum Poll (Sketch 1 → A). On the create screen, she titles the poll, invites the full team plus the Lifts cohort, adds the choreographer to must-include, sets Lifts ≥ 6 and min total = 8, picks a few candidate slots for Sat/Sun, and sets a decision deadline for Friday 5pm (Sketch 2: A–H). She enables AutoFill and AutoPublish so the result locks itself once constraints are met before the deadline (Sketch 2 → G).

Responses roll in. As dancers submit availability—or have it prefilled—Brianna watches the heatmap fill (Sketch 3: A–E). Toggling Feasible only instantly hides slots that don’t satisfy the must-include or cohort minimums (Sketch 3 → F). She can peek at responders for promising slots to ensure the lifts balance looks right.

Decision. By Thursday night, a slot bubbles to the top. On the feasibility view, the list is ranked by the rule “earliest with maximum satisfied,” and the witness pane shows the choreographer plus 7 Lifts are available (Sketch 4: A–B). Because AutoPublish is on, the system publishes that slot immediately and locks the poll (Sketch 4: C–D).

Publish & distribute. The session page appears with participants already populated and a one-click Add to calendar button (Sketch 5: A–C). No extra copy-pasting, no DM chains.

Day-of attendance. Ten minutes before rehearsal, Brianna taps Open check-in, generating a code and a short window (Sketch 6: B–C). Dancers show up and check in on their phones; the live list confirms who’s present (Sketch 6: D). After warm-up, she closes check-in (Sketch 6: E).

Aftermath. Back on the dashboard, the session shows as Scheduled (later Completed) and the poll stays Locked for recordkeeping (Sketch 1: B/D). If the Lifts roster changes next week, she updates it once in Cohorts, and future polls inherit the new membership (Sketch 7: A–C).

Outcome. The rehearsal was scheduled a day earlier than usual, with fewer messages, and the attendance log is ready for accountability. The key is that the system solved for the right subset, not “everyone,” with explainable feasibility and zero extra admin once quorum hit.
