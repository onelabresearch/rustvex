# Rustvex Authentication and Authorization

**Status:** proposed application-layer pattern for v1. Rustvex does not provide
a built-in authentication service.

This document explains how authentication, sessions, authorization, and
application tenancy fit on top of the database defined in
[ARCHITECTURE.md](./ARCHITECTURE.md).

## 1. Boundary

Rustvex provides the database mechanics needed to build auth safely:

- `Bytes` fields for stored hashes
- unique indexes for identities and session-token hashes
- snapshot-consistent read contexts
- atomic mutations and read-your-writes
- typed document IDs
- change-driven invalidation

The application owns:

- password hashing and verification
- token generation and hashing
- cookies and request authentication
- OAuth or other identity-provider communication
- authorization policy
- MFA, account recovery, rate limiting, and abuse prevention
- email/SMS delivery and external side effects

Rustvex never stores plaintext passwords or plaintext session tokens. Use a
maintained security library for cryptography rather than implementing it inside
Rustvex.

## 2. Suggested logical schema

One simple starting model is:

```rust
table! {
    users {
        email: String,
        password_hash: Bytes,
        active: Bool,
        auth_version: Integer,

        unique index by_email(email),
    }

    sessions {
        user: Id<users>,
        token_hash: Bytes,
        issued_auth_version: Integer,
        expires_at: Timestamp,
        revoked: Bool,

        unique index by_token(token_hash),
        index by_user(user),
        index by_expiry(expires_at),
    }
}
```

Email normalization is application policy. Normalize before querying or writing
`by_email` and keep that policy stable. The raw password and session token never
enter these documents.

`auth_version` is incremented when a password changes or all sessions should be
invalidated. A session is valid only when its `issued_auth_version` matches the
current user.

Typed `Id<users>` prevents mixing table identities at compile time. It does not
prove that a user exists or that the current actor may access it.

## 3. Authentication flows

### 3.1 Signup

```text
normalize email
-> hash password outside a Rustvex transaction
-> begin mutation and acquire writer gate
-> insert user with unique by_email entry
-> commit
```

The Rustvex unique index performs the final atomic conflict check. Do not rely
on a separate read-before-insert as the uniqueness guarantee.

### 3.2 Login

```text
read user by normalized email
-> verify password outside the writer gate
-> generate a random session token outside the writer gate
-> hash the token
-> begin mutation
-> reread user and confirm:
       active is true
       password_hash and auth_version still match the observed login state
-> insert session with unique token_hash
-> commit
-> return plaintext token only after commit
```

Password verification can be intentionally expensive, so it must not hold the
global writer gate. The mutation reread prevents a login from creating a session
after the user was disabled or its credentials changed.

If the unique token hash conflicts, return a structured error and generate a new
token outside the failed mutation. Never log the plaintext token.

### 3.3 Session validation

```text
hash presented token
-> open one read context
-> get session by unique by_token index
-> reject missing, revoked, or expired session
-> get referenced user in the same snapshot
-> reject inactive user or auth_version mismatch
-> construct application actor
```

The session and user reads share one PostgreSQL snapshot. Session validation is
a read and does not acquire the writer gate.

### 3.4 Logout, revocation, and password changes

Logout either deletes the session or marks it revoked in a mutation. Deletion is
the simpler default when no retained session record is required.

To invalidate every session for a user, increment the user's `auth_version`.
Existing sessions then fail validation without requiring an immediate scan.
Old session rows can be deleted later in bounded maintenance work.

A password-change mutation updates `password_hash` and increments
`auth_version` atomically.

### 3.5 External identity providers

OAuth exchanges and provider API calls happen outside Rustvex transactions.
After the provider response is verified, use a short mutation to create or
update the local identity and session.

Provider identities can use a separate logical table with a unique compound
index over provider name and provider subject. Provider tokens need their own
storage, encryption, rotation, and disclosure policy; Rustvex does not define
that policy.

## 4. Authorization

Authorization is Rust application logic. A request must establish an actor, then
apply its access policy to every read and mutation.

Prefer indexes that begin with an ownership boundary:

```rust
table! {
    notes {
        tenant: Id<tenants>,
        owner: Id<users>,
        title: String,
        updated_at: Timestamp,

        index by_tenant_updated(tenant, updated_at),
        index by_owner_updated(owner, updated_at),
    }
}
```

Then an authorized tenant query includes:

```text
tenant = actor.tenant_id
```

Unauthorized rows are excluded by the index range. Do not fetch a broad range
and filter it afterward; Rustvex v1 intentionally has no post-index filters.

When access depends on a membership or permission document, perform that lookup
and the resource query through one read context so both observe the same
snapshot. Individual indexed queries still target one table.

Mutation authorization runs inside the mutation transaction after the writer
gate is acquired. Recheck the relevant user, membership, ownership, and version
state before writing.

Typed IDs are not permissions. Document knowledge is not permission. Query
cursors are not permissions.

## 5. Cursors and subscriptions

Every paginated request reconstructs its authorized query from the current
actor. The cursor is accepted only when it matches that query and lies within
its bounds. A cursor must never widen the actor's access.

V1 subscriptions depend on one logical table. The application establishes
authorization before registering and rechecks it when processing a request or
recreating a subscription.

Permission or session changes may require the application to close and recreate
affected subscriptions. Rustvex does not provide an atomic authorization update
across several independent subscriptions.

## 6. Application tenancy

Rustvex has no built-in tenant-isolation primitive.

An application may store a tenant ID on every protected document and place it
first in every authorization-sensitive index. Every mutation must verify the
actor's tenant before it reads or writes protected documents.

This is application-enforced isolation. Deployments requiring a hard database
isolation boundary should use separate Rustvex databases and credentials.

## 7. PostgreSQL roles

PostgreSQL cannot apply row-level policies to fields hidden inside Rustvex
payload bytes.

Use:

```text
owner role:
    physical storage migrations

runtime role:
    DML used only by the Rustvex adapter

reporting or unrelated application roles:
    no direct Rustvex table writes
```

Code holding runtime credentials is trusted to honor Rustvex semantics. Direct
writes can bypass validation, uniqueness, indexes, revisions, subscriptions,
and authorization, so the adapter must be the only normal write path.

## 8. Concurrency and performance

Auth reads remain concurrent. Signup, session creation, logout, revocation, and
password changes are mutations and therefore share the global writer gate with
other Rustvex writes.

Keep expensive hashing, remote calls, and random-token generation outside that
gate. Keep the final state check and persistence inside one short mutation.

Serialized auth writes are appropriate for the initial low-to-moderate write
target. Measure writer-gate wait time under signup and login bursts before
claiming support for high-volume identity workloads.

## 9. Required auth acceptance tests

| Test | Required outcome |
|---|---|
| Concurrent signup | Two signups for one normalized email yield one user and one unique conflict. |
| Login disable race | A user disabled after password verification cannot receive a new session. |
| Password-change race | Login cannot commit using an obsolete password/auth version. |
| Session token conflict | Duplicate token hash is rejected without partial session state. |
| Session snapshot | Session and user validation observe one read snapshot. |
| Revocation | Deleted or revoked session stops authenticating. |
| Global session invalidation | Incremented auth version invalidates all older sessions. |
| Tenant index bounds | A tenant-scoped query cannot return another tenant's documents. |
| Mutation authorization | Permission changes are rechecked inside the gated mutation. |
| Cursor bounds | A modified cursor cannot expand the authorized query range. |
| Subscription recreation | Session/permission loss causes affected subscriptions to be closed or recreated. |
| Direct-write protection | Unrelated database roles cannot write Rustvex physical tables. |
| Secret handling | Plaintext passwords and session tokens never appear in stored documents, logs, or errors. |

These tests verify integration behavior. They do not certify the cryptographic
strength of the application's chosen hashing, token, cookie, OAuth, or MFA
implementation.
