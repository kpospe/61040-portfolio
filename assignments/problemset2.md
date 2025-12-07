# Assignment 2: Composing Concepts

## Concept Questions

### 1. Contexts

Contexts are used to ensure nonces are unique relative to some scope, rather than globally.

In the URL shortening app, the context is the `shortUrlBase` (the domain or domain + namespace for the shortening).

For each base, we maintain a separate set of used nonces so that nonces are unique per base, and the same suffix can be reused safely under different bases.

---

### 2. Storing Used strings

For each context, `NonceGeneration` must store, the set of all strings that have already been generated so that `generate` can guarantee it never returns a string already in that set.

In an implementation with a context counter, we could deterministically encode the counter value as a string to get the nonce.

This relates the spec for the set of used strings to the counter.

In this case, the abstraction is:

For each context, the spec’s set of used strings is exactly the set of strings produced by encoding all counter values that have ever been generated so far for the context.

---

### 3. Words as Nonces

**Advantage (from user perspective):**

Word-based nonces are more memorable and human-readable. They’re easier to type, recall, and share verbally.

**Disadvantage (from user perspective):**

The space of words is small and enumerable. That makes short URLs easier to guess or brute-force which is bad for privacy and security.
Also, if we never recycle words, we can also run out.

**Modifying `NonceGeneration`:**

We can add a dictionary mode and an explicit release action for recycling nonces.

**concept** NonceGeneration[Context]

**purpose** generate unique strings within a context

**principle**
    each generate returns a string not currently used in that context;
    when a resource expires, its nonce can be released back to the pool

**state**

```text
a set of Contexts with
    a used set of Strings
    a availableWords set of Strings
    a mode of DICTIONARY or COUNTER or HYBRID
    a counter Number
```

**actions**

```text
generate(context: Context):(nonce: String)
    effect
        if mode = DICTIONARY: 
            pick a word from availableWords,
            update availableWords and used
            return word
        else if mode = COUNTER:
            use counter-based encoding to get nonce,
            update used
            return nonce
        else if mode = HYBRID:
            pick a word from availableWords,
            use counter-based encoding to get encoding,
            add nonce = word + encoding to used
            return nonce
```

```text
release (context: Context, nonce: String)
    requires nonce is in used
    effect remove nonce from used,
           if mode is DICTIONARY or HYBRID
               add nonce to availableWords
```

*Note:*
A sync with `ExpiringResource` can call `release` when a shortened URL expires, so dictionary words can be reused instead of the pool being exhausted.

---

---

## Synchronization Questions

### 1. Partial Matching

In the `generate` sync, we only need `shortUrlBase`:

- It tells `NonceGeneration` which context to use.
- `targetUrl` is irrelevant at this stage; we’re just getting a fresh nonce for that base.

In the `register` sync, we need both `shortUrlBase` and `targetUrl`:

- We take the generated nonce and base and create the actual short URL mapping to the specific `targetUrl`.

`generate` prepares a suffix for a base.
`register` commits suffix + base + target.

---

### 2. Omitting Names

We don’t always omit argument names because:

- The shorthand only works safely when the argument name and variable name are identical, and the same value is passed through unchanged.
- Sometimes the same underlying value is associated with different argument names in different actions (e.g., one action calls it shortUrlBase, another calls it context or base).

In those cases, omitting names would make it unclear which variable is bound where and could incorrectly bind values between actions.

---

### 3. Inclusion of Request

The first two syncs describe the system’s response to a user request.
A user calls `Request.shortenUrl`, and the syncs say “when that request completes, call `generate` and then `register”`.

The third sync (`setExpiry`) instead responds to internal system behavior.
It is triggered by `UrlShortening.register` completing (once a short URL actually exists), and then calls `ExpiringResource.setExpiry`.

By that point we no longer need the original request action; we just care that a shortening was created.

---

### 4. Fixed Domain

If the domain is fixed, then `shortUrlBase` is constant and doesn't need to be a parameter.
Synchronizations would drop the `shortUrlBase` binding and hardcode the base.

---

### 5. Adding a Sync

Add a sync that responds to the `expireResource` completion and deletes the shortening:

```text
sync expire_delete
when ExpiringResource.expireResource (): (resource: String)
then UrlShortening.delete (shortUrl: resource)
```

This sync ensures that when the system expiration returns a resource (a short URL), the corresponding shortening is deleted from UrlShortening.

If we are using dictionary words, we could add another sync that also calls `NonceGeneration.release` for the associated context.

---

---

## Extending the Design

### 1. Additional Concepts

#### Concept 1: Ownership

**concept** Ownership[Owner, Resource]

**purpose** record which owner is associated with which resource

**principle**
  when a resource is created for an owner, we store that relationship;
  other parts of the system can later ask who owns a resource

**state**

```text
a set of Records with
    a resource Resource
    an owner Owner
```

**actions**

```text
assign (resource: Resource, owner: Owner)
    requires no Record exists with this resource
    effect   create a new Record with this resource and owner

ownerOf (resource: Resource) : (owner: Owner)
    requires a Record exists with this resource
    effect   no state change;
             return owner
```

In the shortener, we instantiate this as `Ownership[User, ShortUrl]`.

#### Concept 2: Analytics

**concept** Analytics[Key]

**purpose** count how many times each key is used

**principle**
    each time a key (e.g. a short URL) is used, we increment a counter;
    other concepts decide who may see the counts

**state**

```text
a set of Counters with
    a key Key
    a count Number
```

**actions**

```text
createCounter (key: Key)
    requires no Counter exists for this key
    effect   create a Counter with this key and count = 0
```

```text
increment (key: Key)
    requires a counter exists with this key
    effect increments the counter
```

```text
getCount (key: Key): (count: Number)
    requires a Counter exists for this key
    effect   no state change;
             return count for this key
```

*Note:*

- Analytics knows nothing about users or ownership
  - permission checks happen in syncs.

---

### 2. Essential Synchronizations

Assume:

- `ShortUrl` is the type of short URLs.
- `User` is the type of users.
- We instantiate `Ownership[User, ShortUrl]` and `Analytics[ShortUrl]`.

#### Sync 1: When shortenings are created — create counter and assign ownership

This sync binds the requester (user) who made the shortening and the produced `shortUrl`:

```text
sync analytics_on_register
when
    Request.shortenUrl (targetUrl, shortUrlBase, requester: User)
    NonceGeneration.generate (): (nonce: String)
    UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase, targetUrl): (shortUrl: ShortUrl)
then
    Ownership.assign (resource: shortUrl, owner: requester)
    Analytics.createCounter (key: shortUrl)
```

#### Sync 2: When shortenings are looked-up — increment counter

```text
sync analytics_on_lookup
when UrlShortening.lookup (shortUrl: ShortUrl): (targetUrl: String)
then Analytics.increment (key: shortUrl)
```

This increments the counter upon each successful lookup.

#### Sync 3: When a user requests analytics — check ownership and respond

```text
sync respond_analytics_request
when Request.getAnalytics (shortUrl: ShortUrl, requester: User)
where Ownership.ownerOf (resource: shortUrl) = requester
then
    Analytics.getCount (key: shortUrl): (count: Number)
    Request.respondAnalytics (count: count)
```

*Note:*

- The `where` clause enforces that only the owner sees analytics.
- `Analytics` remains modular.
It just stores counts and doesn’t know about owners.

---

### 3. Evaluation of Additional Feature Requests

#### Feature 1: Allowing users to choose their own short URLs

**How to realize:**

- Extend `Request.shortenUrl` to optionally include a `desiredSuffix`.
- Modify `UrlShortening.register` to:
  - Use `desiredSuffix` if it is valid and not already used for that base.
  - Otherwise fall back to `NonceGeneration.generate`.

``Ownership`` and `Analytics` syncs don’t need to change.
They work with whatever `shortUrl` is created.

**Security caution:** Allowing custom suffixes requires validation and rate limiting to prevent abuse (e.g., squatting or creating offensive URLs).

#### Feature 2: Using words-as-nonce (dictionary strategy)

**How to realize:**

- Use the DICTIONARY / HYBRID modes in NonceGeneration from Part 1.
- Add a sync:

```text
sync release_word_on_expire
when ExpiringResource.expireResource (): (resource: ShortUrl)
then NonceGeneration.release (context: shortUrlBase for resource,
                              nonce: suffix of resource)
```

This sync assumes there are helpers to map a short URL back to base + suffix.

This strategy improves memorability but requires recycling and some security tradeoffs.

#### Feature 3: Including the target URL in analytics

**How to realize:**

We want to group by target.
There are two options:

1. Extend Analytics with an additional concept:

**concept** TargetIndex[Target, ShortUrl]

**state**

```text
a set of Entries with
    a target Target
    a shortUrl ShortUrl
```

Then, in the register sync, we would record (`targetUrl`, `shortUrl`). To get per-target counts, one could look up all `shortUrl`s for a target and sum their `Analytics` counts.

2. Store a `targetHash` in each `Analytics` counter and provide queries that aggregate by `targetHash`.

**Privacy implication:** Storing `targetUrl` in `Analytics` may expose link targets to parties who shouldn’t see them, so any analytics that include targets must enforce the same visibility/ownership rules as the rest of the system.

#### Feature 4: Generate short URLs that aren't easily guessed

**How to realize:**

- Have NonceGeneration use long random nonces that are cryptographically secure RNG, large alphabet, and sufficient length.
- Avoid dictionary and predictable patterns.
- Optionally rate-limit lookup attempts and avoid exposing a browsable list of existing short URLs.

This reduces the risk that attackers can guess valid short URLs by enumeration.

#### Feature 5: Reporting analytics to creators who haven't registered as users

This feature is undesirable and should not be included.
It could introduce privacy and authentication issues.
Without some identity and consent mechanism, automatically reporting analytics to unregistered creators risks sending sensitive usage data to the wrong party.

A safer approach:
- Treat “creator identity” as a more abstract Owner (e.g., an email address or signed token) and store it via Ownership.
- Require an opt-in (and verified contact) when creating a shortening if they want periodic analytics reports.
- Use a batch process outside the core concepts to send summaries only to verified contacts.
