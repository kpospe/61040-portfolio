# Assignment 2: Modular Design

## Part 1 — Concept Questions

### 1. Contexts

Contexts were used to create nonces that are unique *relative to some scope*.
In the URL shortening app, the context is the `shortUrlBase` (the domain or domain + namespace under which short strings must be unique).

### 2. Storing used strings

The specification requires a set of previously used strings per context so the generator can ensure it never returns a previously used nonce.
**Abstraction function(ish):**
The counter in the implementation represents the set of used strings.
This allows for us to convert the counter integer into a string in a deterministic manner.

### 3. Words as nonces

**Advantage (from user perspective):** Word-based nonces are more memorable and human-readable (easier to type and recall).  

**Disadvantage (from user perspective):** They are easier to guess, which increases the possibility of accidental or malicious discovery.
Also, dictionary space is limited so collisions are more likely or to avoid collisions, these strings must be longer.

**Modifying NonceGeneration:**
add a `Vocabulary` and a `strategy` mode admitting `WORDS` (and possibly `WORDS+NUMBER` or `COMPOSITE`).
For example:

```
concept NonceGeneration [Context]
purpose generate unique strings within a context (optionally from a vocabulary)
principle each generate returns a string not returned before for that context
state
    a set of Contexts with
        a used set of Strings
        a Vocabulary set of Strings
        a mode of DICTIONARY or COUNTER or HYBRID
actions
    generate (context: Context) : (nonce: String)
        effect if mode = DICTIONARY: pick an unused word from Vocabulary, mark used, return it
        else if mode = COUNTER: use counter-based encoding, update used set
        else if mode = HYBRID: combine word+small-rand to increase space
    addVocabularyWords (words: set of String)
        effect adds words to Vocabulary
```

Note: HYBRID can append small numeric suffixes or combine words to increase entropy while keeping some memorability.

---


## Part 2 — Synchronization Questions

### 1. Partial matching

The `generate` sync only needs the base/namespace to decide which context to use when producing a nonce.
The `register` sync must include `targetUrl` because the register action has the mapping `(shortUrl -> targetUrl)`.
In other words, `generate` is preparing a nonce for a base, while `register` is finalizing a shortening by pairing nonce with the target URL.

### 2. Omitting names

This isn’t used universally because:
- explicit names improve readability when a sync binds multiple variables from different places.
- avoiding omission can prevent accidental variable shadowing or confusion when several bindings would otherwise look identical.

### 3. Inclusion of request
The first two syncs are triggered by an explicit user request `Request.shortenUrl`.
The third sync (`setExpiry`) is triggered by the completion of the internal `UrlShortening.register` action.
Since it's an internal response to the successful creation of a shortening, it doesn't have a request.

### 4. Fixed domain
If the domain is fixed, then `shortUrlBase` is constant and doesn't need to be a parameter.
Synchronizations would drop the `shortUrlBase` binding and use a known constant (or omit the context arg).

### 5. Adding a sync
Add a sync that responds to the `expireResource` completion and deletes the shortening:

```
sync expire_delete
when ExpiringResource.expireResource (): resource
then UrlShortening.delete (shortUrl: resource)
```

This sync ensures that when the system expiration returns a resource (a short URL), the corresponding shortening is deleted from UrlShortening.

---


## Part 3 — Extending the Design

### 1. Additional concepts

#### Concept 1: ShorteningOwnership

```
concept ShorteningOwnership
purpose associate a shortened URL to the user who created it (owner)
principle only the owner can view private analytics for a short URL
state
    a set of Ownerships with
        shortUrl String
        owner UserId
actions
    assign (shortUrl: String, owner: UserId)
        requires no ownership exists for shortUrl
        effect save mapping shortUrl -> owner
    ownerOf (shortUrl: String) : (owner: UserId)
        requires ownership exists
        effect returns owner
```

#### Concept 2: Analytics

```
concept Analytics
purpose count lookups for short URLs and provide counts only to owners
principle increment on each lookup, counts visible only to the registered owner
state
    a set of Counters with
        shortUrl String
        count Number
actions
    createCounter (shortUrl: String)
        requires no counter exists for shortUrl
        effect creates a counter with count = 0
    increment (shortUrl: String)
        requires a counter exists for shortUrl
        effect increments the counter
    getCount (shortUrl: String, requester: UserId) : (count: Number)
        requires a counter exists for shortUrl
        requires ShorteningOwnership.ownerOf(shortUrl) = requester
        effect returns count
```

(We keep authorization logic simple by referencing ownership in getCount precondition.)

---


### 3. Essential synchronizations

#### 3.1 When shortenings are created — create counter and assign ownership
This sync binds the requester (user) who made the shortening and the produced `shortUrl`:

```
sync analytics_on_register
when
Request.shortenUrl (targetUrl, shortUrlBase, requester: UserId)
NonceGeneration.generate (): (nonce)
UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase, targetUrl): (shortUrl)
then
ShorteningOwnership.assign (shortUrl, requester)
Analytics.createCounter (shortUrl)
```


#### 3.2 When shortenings are translated — increment counter

```
sync analytics_on_lookup
when UrlShortening.lookup (shortUrl): (targetUrl)
then Analytics.increment (shortUrl)
```

This increments the counter upon each successful lookup.

#### 3.3 When a user examines analytics — return only to owner

```
sync respond_analytics_request
when Request.getAnalytics (shortUrl, requester: UserId)
then Analytics.getCount (shortUrl, requester): (count)
Request.respondAnalytics (count)
```

`Analytics.getCount` has the precondition that `requester` equals the `ShorteningOwnership.ownerOf(shortUrl)`, ensuring privacy.

---


### Part 4 — Evaluate additional feature requests

#### 4.1 Allowing users to choose their own short URLs
**How to realize:** Add an option on `Request.shortenUrl` to include `desiredSuffix`.
Modify `UrlShortening.register` precondition to allow specifying `shortUrlSuffix` if and only if it doesn't already exist and it meets validation rules (e.g. charset, length, reserved words).
No change required to analytics concept itself (ownership assigned as above).  
**Security caution:** need rate limits and validation to avoid abuse.

#### 4.2 Using words-as-nonce (dictionary strategy)
**How to realize:**
Modify `NonceGeneration` (or add Vocabulary and `mode = DICTIONARY`) as shown in Part 1.
Optionally choose HYBRID to combine words + small numeric suffix to increase space while preserving memorability.

#### 4.3 Including the target URL in analytics (grouping by target)
**How to realize:**
Add a `targetHash` field to `Analytics` counters or create a `TargetIndex` concept that maps `targetUrl` (or its canonicalized hash) to a set of shortUrls.
On UrlShortening.register, pass the `targetUrl` to the analytics sync so `Analytics` can either:
- maintain counters keyed by `shortUrl` and also maintain a mapping `targetHash -> aggregatedCount` (update both on increment), or
- provide an API to query all shortUrls for a target and sum their counters.

**Privacy implication:** storing `targetUrl` in analytics may expose link targets to parties that shouldn't see them. This ensures visibility rules if analytics are shared.

#### 4.4 Generate short URLs that aren't easily guessed
**How to realize:**
Use a secure random generator in `NonceGeneration` with long enough nonces.
Avoid dictionary tokens and rate limit enumerations and don't expose lists of existing short URLs.
Optionally use per-owner salt so the same nonce space can't be enumerated across owners.

#### 4.5 Reporting analytics to creators who haven't registered as users
**How to realize:**
This introduces privacy and authentication issues.
A safe approach is to require owners to provide a contact and consent to receive analytics during registration.
We store a `contact` and `consent` in `ShorteningOwnership`.
Then, we implement a periodic reporting process that contacts consented contacts.
Without consent or a valid contact this is undesirable (privacy & spam concerns).
So supporting *reported* analytics to non-registered creators is possible but must be opt-in and requires contact verification.

## Notes / Design Rationale
- *Separation of concerns:* Analytics and Ownership are separate concepts so analytics logic doesn't clutter UrlShortening. This keeps our tasks modular.
- *Privacy:* `getCount` enforces owner-only access.
Syncs bind the requester's identity to ensure only owners can view counts.
- *Extensibility:* word-based generation and user-chosen short URLs are supported by small changes to `NonceGeneration` and `UrlShortening.register` preconditions.
