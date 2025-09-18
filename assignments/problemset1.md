# Problem Set 1: Reading and Writing Concepts

## Exercise 1: GiftRegistration

**Question 1: Invariants of the State**
1. For any request, the sum of purchased counts ≤ the request’s original count.  
2. Every purchase must correspond to an existing request for the same item in the same registry.  

The *most important is* invariant 1, because it ensures gifts cannot be oversold.  
The *most affected action* is `purchase`. It preserves it by decrementing the request count and disallowing purchases exceeding availability.

**Question 2: Fixing an Action**  
`addItem` can break invariant 1. If new items are added after some purchases, the request count could be inconsistent with total purchased.  
A *fix* could be to prohibit `addItem` after registry is opened, or require that new requests have no prior purchases.

**Inferring Behavior**  
A registry can be opened and closed repeatedly (no restriction).  
A *reason* is that opening and closing allows the owner to temporarily close it to possibly update items.

**Registry Deletion**  
It's not critical in practice. Closed registries can just remain hidden. Deletion is only useful for storage cleanup, not for functionality.

**Queries**
- By owner: “Which items were purchased, and by whom?”  
- By giver: “Which items are still available, and how many?”

**Hiding Purchases**  
I'd add state `visibilityFlag` on a registry.  
- Action: `setVisibility(registry, flag: Boolean)`  
- Requires: registry exists.  
- Effects: controls whether recipient can see purchase details.

**Generic Types**  
Using `Item` as a generic lets applications plug in SKU codes or other structured identifiers. Names/descriptions are mutable and non-unique, so they’re unsuitable as primary identity.

---

## Exercise 2: PasswordAuthentication

**State**
  A set of Users with  
    a username String 
    a passwordHash String 
    a confirmed Boolean  

**Actions**
register(username: String, password: String): (user: User)
  requires username not already registered
  effects create new user with username, passwordHash, confirmed = true

authenticate(username: String, password: String): (user: User)
  requires a user with this username exists, confirmed = true,
  and stored passwordHash matches password
  effects returns the user; no state change


**Invariant**  
Each username is unique and associated with at most one passwordHash.  
Preserved by `register`, which checks nonexistence before creating.

**Extension: Email Confirmation**

- Additional state:  
  - secretToken: String (per user, unguessable)  

- Revised actions:  
register(username: String, password: String): (user: User, token: String)
  requires username not already registered
  effects create new user with username, passwordHash, confirmed = false
  create secret token and return it

confirm(username: String, token: String)
  requires user exists with this username and matching token
  effects set confirmed = true

authenticate(username: String, password: String): (user: User)
  requires user exists, confirmed = true, and password matches
  effects returns the user

---

## Exercise 3: PersonalAccessToken

**Concept Specification**

concept: PersonalAccessToken

purpose: provide alternative authentication credentials without requiring passwords

principle
  a user generates one or more tokens linked to their account;
  each token is an opaque string that can be presented instead of a password;
  tokens can be individually revoked without affecting the user’s password

state
  a set of Users with
    a *username* String
    a *tokens* set of Token

actions
createToken(user: User): (token: Token)
  effects generate new token, add to user’s token set, return token

authenticate(username: String, token: Token): (user: User)
  requires token ∈ user’s token set
  effects return the user; no state change

revokeToken(user: User, token: Token)
  requires token ∈ user’s token set
  effects remove token

**Differences from PasswordAuthentication**
- Users can have multiple tokens, but only one password.  
- Tokens can be revoked individually.  
- Tokens are auto-generated opaque strings, not chosen by the user.  

**Improving GitHub’s documentation**  
The page could explicitly frame tokens as revocable, limited-scope credentials that are distinct from passwords. Instead of just saying 'treat them like passwords.' Clarifying their advantages (including multiple per user) would reduce confusion.

---

## Exercise 4: Three Concepts

### 1. URL Shortener

concept: URLShortener
purpose: provide short, persistent identifiers for longer URLs
principle:
  a user submits a long URL
  system creates a short suffix (or uses user-provided suffix if unique)
  resolving the short URL redirects to the long one

state
  a set of Mappings with
  a *suffix* String
  a *longURL* String

actions
  shorten(longURL: String): (suffix: String)
    effects generate unused suffix, store mapping, return suffix

shortenWithSuffix(longURL: String, suffix: String): (suffix: String)
  requires suffix not already used
  effects create mapping with this suffix

resolve(suffix: String): (longURL: String)
  requires suffix exists
  effects return mapped long URL

*Note:* Must handle collisions; persistence is essential.

---

### 2. Billable Hours Tracking

concept: BillableHours
purpose: record time spent on client projects
principle:
  an employee starts a session with a project and description, ends the session when done, and the system logs duration, forgotten sessions can be auto-closed

state
a set of Sessions with
  a *project* Project
  a *employee* Employee
  a *description* String
  a *startTime* Timestamp
  a *endTime* Timestamp

actions
startSession(employee: Employee, project: Project, desc: String): (session: Session)
  effects create session with startTime = now, endTime = null

endSession(session: Session)
  requires session exists and endTime = null
  effects set endTime = now

autoClose(session: Session, time: Timestamp)
  requires session endTime = null and time > startTime + threshold
  effects set endTime = time

*Note:* Auto-close prevents indefinite open sessions.

---

### 3. Conference Room Booking

concept: RoomBooking
purpose: allocate shared rooms fairly without conflicts
principle:
  a user selects a room, date, and time interval
  if available, the system reserves it; otherwise the request is denied

state
  a set of Rooms with
    an *id* RoomID
    a *reservations**** set of Reservations

  a set of Reservations with
  a *room* RoomID
  a *user* User
  a *startTime* Timestamp
  an *endTime* Timestamp

actions
book(user: User, room: RoomID, start: Timestamp, end: Timestamp): (res: Reservation)
  requires interval does not overlap existing reservations for the room
  effects create reservation

cancel(res: Reservation)
  requires res exists
  effects remove reservation

listReservations(room: RoomID): (set of Reservations)
  requires room exists
  effects return reservations

*Notes:* Overlap-checking is the subtlety here.
