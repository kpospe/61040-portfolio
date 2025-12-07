# Problem Set 1: Reading and Writing Concepts

## Exercise 1: Reading a Concept

### Question 1: Invariants of the State

Two key invariants:

1. For any registry and any item, the total number of purchases for that item is ≤ the total quantity that has ever been requested for that item in that registry.
2. Every purchase must correspond to an existing request for the same item in the same registry. Aka, there are no “orphan” purchases that aren’t tied to a request.

The **most important invariant** is (2): that every purchase corresponds to an existing request.
It encodes the core purpose of a gift registry: people only buy gifts that the registry owner has actually requested.  

The **action most responsible** for preserving (2) is `purchase`.
It preserves the invariant by:

- requiring that the item is present as a request in the registry before a purchase can be made, and
- decrementing the request’s remaining count so that purchases can’t exceed what has been requested.

---

### Question 2: Fixing an Action

The action that can break the key invariant (2) is **`removeItem`**.

If `removeItem` is allowed to delete a request even when some purchases for that request already exist, then those purchases become “orphaned”.
We’ve recorded purchases whose corresponding request no longer exists, which violates the invariant.

A **fix** is to strengthen `removeItem` with a precondition:

```text
removeItem(registry, item)
  requires registry has a request for item,
           there are no purchases recorded for that request // ADDED
  effects  remove the request for item from the registry
```

This ensures we never delete a request that has purchases attached, so no orphan purchases can exist.

---

### Question 3: Inferring Behavior

From the spec, a registry can be opened and closed multiple times.
There is no restriction that `open` can be called only once or only after create, nor any limit on how many times `close` can be called.

A plausible reason for this design is to let the owner:

- temporarily *close* the registry (for example, while editing items or pausing purchases), and then
- re*open* it once changes are done or when they’re ready to accept gifts again.

---

### Question 4: Registry Deletion

Registry deletion isn't critical for the core behavior of the concept.
Functionally, closing the registry already hides it and prevents further purchases.

Keeping closed registries around can be useful for:

- resolving disputes (“Did someone actually buy this?”),
- analytics (“How many gifts were purchased for past events?”), or
- personal record-keeping.

Deletion may matter for long-term storage costs or data minimization, but it’s not essential to the main user experience.

---

### Question 5: Queries

Here are queries likely to be commonly executed by an owner and by a gift giver:

- By **owner**: “Which items were purchased, and by whom?”  
- By **giver**: “Which items are still available, and how many?”

---

### Question 6: Hiding Purchases

To support hiding purchases from the recipient while still recording them, I would add a registry-level **visibility flag**.

**(revised) state**

```text
a set of Registries with
  a visibilityFlag Boolean
  // true  -> recipient can see purchase details
  // false -> recipient can see limited or no purchase details
```

**(revised) actions**

```text
setVisibility(registry, flag: Boolean)
  requires registry exists
  effects  registry.visibilityFlag := flag
```

This lets the registry owner or system control whether the recipient can see purchase information before the event (often false) and then reveal it later (true), while givers and admins can still see actual purchases.

---

### Question 7: Generic Types

Using a generic `Item` type is preferable because names and descriptions are not reliable identifiers, while an `Item` ID is for the following reasons:

- **Uniqueness:**
  - Two different products can share the same name (“Black Toaster”), so names cannot uniquely identify requests.
  - An `Item` ID (SKU, product code, etc.) is guaranteed to be unique.
- **Mutability:**
  - Names, descriptions, and prices can change over time.
If identity were tied to these fields, edits would accidentally change which item past purchases refer to.
  - An `Item` ID remains stable even when metadata changes.
- **Cross-system stability:**
  - External catalogs may use different names or update metadata.
  - A stable `Item` ID lets the registry interoperate with other systems without breaking references.

---

---

## Exercise 2: Extending a Familiar Concept

### Question 1: Completing the Concept State Definition

**state**

```text
a set of Users with
  a username String
  a passwordHash String
  a confirmed Boolean
```

### Question 2: Requires/Effects of the Actions

**actions**

```text
register(username: String, password: String): (user: User)
  requires no user with this username exists
  effects create new user with:
          user.username = username
          user.passwordHash = hash(password)
          user.confirmed = true
          return that new user
```

```text
authenticate(username: String, password: String): (user: User)
  requires  a User exists with this username,
            user.confirmed = true,
            user.passwordHash = hash(password)
  effects   no state change
            return that user
```

### Question 3: Invariant

Each `username` is unique and associated with at most one `passwordHash`:

**Invariant:**
For any two Users `u1` and `u2`, if `u1.username = u2.username` then `u1 = u2`.

**Preservation:**
`register` enforces this by requiring that no user with the given username already exists before creating a new one.
No other action creates or deletes users, so uniqueness is maintained.

### Question 4: Extension - Email Confirmation

I will add an extra variable to the state per user

**revised state**

```text
a set of Users with
  a username String
  a passwordHash String
  a secretToken String   // NEW - addresses 4.1
  a confirmed Boolean
```

**revised actions**

```text
register(username: String, password: String): (user: User, token: String)
  requires no user with this username exists
  effects  create new user with:
           user.username = username
           user.passwordHash = hash(password)
           user.confirmed = false
           user.secretToken is set to a random, unique token
           return (user, user.secretToken)
```

NEW confirm action, addresses 4.2

```text
confirm(username: String, token: String)
  requires there exists a User with this username,
           user.secretToken = token,
           user.confirmed = false
  effects  set user.confirmed := true
```

```text
  authenticate(username: String, password: String): (user: User)
    requires there exists a User with this username,
             user.confirmed = true,
             user.passwordHash = hash(password)
    effects  no state change
             returns the user
```

---

---

## Exercise 3: Comparing Concepts

**Concept Specification:**

**concept** PersonalAccessToken

**purpose** provide alternate authentication credentials that a user can use instead of their password

**principle**
  a user generates one or more tokens linked to their account;
  each token is an opaque string that can be presented instead of a password;
  tokens can be individually revoked without affecting the user’s password;

**state**

```text
a set of Users with
  a username String
  a tokens set of Token
```

**actions**

```text
createToken(user: User): (token: Token)
  effects generate new token,
          add token to user's token set,
          return token
```

```text
authenticate(username: String, token: Token): (user: User)
  requires a User exists with this username,
           token is in User’s token set
  effects  no state change,
           return the User;
```

```text
revokeToken(user: User, token: Token)
  requires token is in user’s token set
  effects remove token from user's token set
```

**Differences from PasswordAuthentication:**

- Users can have multiple tokens, but typically only one password.
- Tokens are individually removable.
Revoking one doesn’t affect others or the password.
- Tokens are generated, not chosen by users.
They’re meant for tools/automation rather than memorization.


**Improving GitHub’s documentation:**  

GitHub’s documentation could more clearly explain that Personal Access Tokens are:
- Removable, limited-scope credentials, separate from passwords.
- Designed to be created, changed, and deleted frequently for different tools or contexts.
- Safer than embedding a password because you can revoke a single token without changing your password or disrupting other uses.

This framing (“tokens are like keys for different tools you can revoke at any time”) is more informative than just “treat them like passwords.”

---

---

## Exercise 4: Defining Familiar Concepts

### Concept 1 - URL Shortener

**concept** URLShortener

**purpose** provide short, human-friendly identifiers that can be used to direct to longer URLs

**principle**
  a user submits a long URL
  system creates a short suffix (or uses user-provided suffix if unique),
  visiting the short URL redirects to the long one,
  creators can later delete a mapping, allowing the suffix to be reused

**state**

```text
a set of Mappings with
  a suffix String
  a longURL String
  a owner User
```

**actions**

```text
shorten(longURL: String): (suffix: String)
  effects generate unused suffix,
          create and store mapping,
          return suffix
```

```text
shortenWithSuffix(longURL: String, suffix: String): (suffix: String)
  requires no mapping uses this suffix
  effects  create mapping with this suffix
           return suffix
```

```text
resolve(suffix: String): (longURL: String)
  requires a mapping exists with this suffix
  effects  no state change,
           return long URL associated with this suffix
```

```text
deleteMapping(owner: User, suffix: String)
  requires a mapping exists for this suffix
           mapping.owner = owner
  effects  remove the mapping for this suffix
```

*Notes:*

- Using a generic URL type allows a separate concept to validate URLs and ensure they’re well-formed.
- deleteMapping releases the short suffix so it can eventually be reused, which is important for systems that use human-friendly words or a limited namespace.

---

### Concept 2 - Billable Hours Tracking

**concept** BillableHours[Employee, Project]

**purpose** record and organize time spent on client projects so it can be accurately aggregated for invoicing, payroll, and reporting

**principle**
  an employee starts a work session,
  when they finish, the system logs the session duration,
  forgotten sessions can be detected and corrected to reflect actual work

**state**

```text
a set of Sessions with
  a project Project
  a employee Employee
  a description String
  a startTime Timestamp
  a endTime Timestamp
```

**actions**

```text
startSession(employee: Employee, project: Project, desc: String): (session: Session)
  effects create session with the employee, project, and desc,
          set session.startTime = now
          set session.endTime = null
          return session
```

```text
endSession(session: Session)
  requires session exists,
           endTime = null
  effects  set endTime = now
```

```text
adjustSessionEnd(session: Session, newEnd: Timestamp)
  requires session exists
           session.startTime < newEnd
  effects  set session.endTime = newEnd
```

```text
autoClose(session: Session, time: Timestamp)
  requires session endTime = null,
           time > startTime + threshold
  effects  set endTime = time
```

*Note:*
- Instead of automatically extending sessions indefinitely for billing, a better fit for the purpose is to detect long-running sessions and prompt a human to adjust, keeping billed hours closer to reality.

---

### 3. Conference Room Booking

**concept** RoomBooking[RoomID, User]

**purpose** allocate shared rooms fairly without scheduling conflicts

**principle**
  a user selects a room, date, and time interval
  if interval doesn't overlap existing reservations for that room, the system reserves it; otherwise the request is denied
  users can cancel reservations and view schedules

**state**

```text
a set of Rooms with
  an id RoomID
  a reservations set of Reservations

a set of Reservations with
  a room RoomID
  a user User
  a startTime Timestamp
  an endTime Timestamp
```

**actions**

```text
book(user: User, room: RoomID, start: Timestamp, end: Timestamp): (res: Reservation)
  requires room and user exist,
           start < end,
           interval does not overlap existing reservations for the room
  effects  create reservation 'res'
           add res to room's reservations
           return res
```

```text
cancel(res: Reservation)
  requires res exists
  effects remove res from it's room's reservations
```

```text
listReservations(room: RoomID): (set of Reservations)
  requires room exists
  effects  no state change,
           return the room's reservations
```

*Notes:*

- The critical subtlety is the overlap check on book.
- Additional basic validity checks (start < end, room/user exist) make the concept safer and more complete.
