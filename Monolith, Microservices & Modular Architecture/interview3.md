# Prevent Silent Overwrites Without Locking the Database

## 1. Interview Question

**How do you stop silent overwrites when two users edit the same record
at the same time, without locking the database?**

## 2. Problem: Lost Update

Imagine two support agents open the same ticket.

1.  Both agents read ticket version `v4`.
2.  Agent A changes the status to `Approved` and saves.
3.  The database now contains version `v5`.
4.  Agent B, still working with the old `v4` data, changes the notes and
    saves the entire stale record.
5.  Agent B may accidentally overwrite Agent A's latest change.

This is called a **Lost Update**. A user's update silently replaces
another user's update.

## 3. Solution: Optimistic Concurrency Control

Use **Optimistic Concurrency Control (OCC)**, also called **Optimistic
Locking**.

The idea is simple:

-   Do not lock the record while users are viewing or editing it.
-   Return a version identifier with the record.
-   When saving, ask the server to update the record only if that
    version is still current.
-   If another user has already changed it, reject the stale update.

## 4. How ETag and If-Match Work

### Step 1: Client reads the record

``` http
GET /tickets/101
```

Response:

``` http
HTTP/1.1 200 OK
ETag: "v4"
```

The client stores the ETag (`"v4"`) along with the ticket data.

### Step 2: Client sends the update

``` http
PUT /tickets/101
If-Match: "v4"
Content-Type: application/json
```

The `If-Match` header means:

**Update this ticket only if its current version still matches `v4`.**

### Step 3: Server performs an atomic database update

``` sql
UPDATE tickets
SET status = 'Approved',
    version = version + 1
WHERE id = 101
  AND version = 4;
```

The version check and update happen in one atomic database statement.

-   If one row is updated, the save succeeded. The version becomes `v5`.
-   If zero rows are updated, someone else changed the record first. The
    client's version is stale.

### Step 4: Server rejects the stale update

For a stale `If-Match` value, return:

``` http
HTTP/1.1 412 Precondition Failed
```

The client should reload the latest record and show the user that
someone else changed it. The user can then review and reapply their
changes.

## 5. Important HTTP Status Codes

  -----------------------------------------------------------------------------
  Status                        Meaning                 When to use
  ----------------------------- ----------------------- -----------------------
  `200 OK`                      Update succeeded        Version matched and
                                                        update completed

  `412 Precondition Failed`     The condition supplied  `If-Match` does not
                                by the client failed    match the current ETag

  `428 Precondition Required`   A required precondition Application requires
                                is missing              `If-Match`, but client
                                                        did not send it

  `409 Conflict`                General                 Business rule conflict
                                application-level       not specifically caused
                                conflict                by a failed HTTP
                                                        precondition
  -----------------------------------------------------------------------------

`428` is defined by RFC 6585. Requiring clients to send `If-Match` is an
application/API policy.

## 6. Important ETag Detail

Use a **strong ETag** for `If-Match`, for example:

``` http
ETag: "v4"
```

Avoid a weak ETag such as:

``` http
ETag: W/"v4"
```

`If-Match` uses strong comparison, so a weak ETag will not satisfy the
condition.

## 7. Why This Does Not Lock the Database for a Long Time

We do not keep a database lock while a person is reading a screen or
thinking about an edit.

The database only performs a short, atomic update when the save request
arrives. The version condition ensures that only one update based on the
same old version can succeed.

------------------------------------------------------------------------

# Important Follow-up Questions

## Q1. What if the first PUT succeeds, but the response is lost and the client retries?

The server may have completed the first update even though the client
did not receive the response.

If the client retries with the old ETag, it may receive `412`, because
the record is now at a newer version.

A robust solution is to use an **Idempotency-Key** for the operation.
The server stores the result associated with that key and can return the
original result for a retry, rather than applying the operation again.

Example:

``` http
PUT /tickets/101
If-Match: "v4"
Idempotency-Key: 8f2a-operation-123
```

An idempotency key is especially useful when the client cannot tell
whether the first request completed.

## Q2. How do you test and monitor optimistic concurrency?

**Test:** 1. Open the same record in two clients. 2. Both clients
receive the same ETag, such as `"v4"`. 3. Send two updates using
`If-Match: "v4"`. 4. Verify that one update succeeds and the other
receives `412`. 5. Verify that the version increments only once.

**Monitor:** - Count `412 Precondition Failed` responses. - Track
conflicts by API route. - Watch for unusual spikes that may indicate
stale clients or heavy concurrent editing.

A `412` is not necessarily a server failure; it can be an expected
concurrency conflict.

## Q3. Why not use SELECT FOR UPDATE when the user opens the screen?

`SELECT ... FOR UPDATE` holds a database row lock until the transaction
ends.

Keeping that lock while a user reads or edits a screen is dangerous
because the user may take seconds or minutes to save. Long-held locks
can block other requests and consume database connections.

Optimistic concurrency avoids holding a lock during the user's
think-time.

## Q4. Why use a version column instead of a hash of the record?

A version column is usually simpler and cheaper:

-   Increment it whenever the record changes.
-   Compare the version during update.
-   It is easy to understand and index.

A hash can also detect changes, but computing and comparing hashes may
add complexity. Hashing can also depend on how the record is serialized.
For most record-update APIs, a version column is a practical choice.

## Q5. What if two users update different fields?

With one version number for the whole record, a change to any field
makes the old version stale.

For example: - Agent A changes `status`. - Agent B changes `notes`. -
Agent B may still receive `412`, even though the fields do not overlap.

This is a **coarse-grained conflict**. The application can ask the user
to reload and retry, safely merge non-overlapping changes, or use more
granular concurrency controls where justified.

Do not automatically retry a stale full-record update, because that can
recreate the lost-update problem.

## Q6. What is the difference between 409 and 412?

-   **412 Precondition Failed:** The request's HTTP precondition failed.
    Example: `If-Match` ETag is stale.
-   **409 Conflict:** The request conflicts with the current application
    or resource state for another reason. Example: trying to perform an
    operation that violates a business rule.

When `If-Match` fails, `412` is the precise status code.

------------------------------------------------------------------------

# 30-Second Interview Answer

I would use **Optimistic Concurrency Control** with ETag and If-Match.

When a client fetches a record, the server returns an ETag representing
its current version. When the client updates the record, it sends that
ETag in the `If-Match` header.

The server performs an atomic database update using a version condition.
If the version matches, the update succeeds and the version is
incremented. If another user has already updated the record, the
condition fails and the server returns `412 Precondition Failed`.

The client then reloads the latest data and asks the user to resolve the
conflict. This prevents lost updates without holding database locks
while users are editing.

## Quick Revision

-   **Problem:** Lost Update --- a stale save overwrites a newer change.
-   **Solution:** Optimistic Concurrency Control / Optimistic Locking.
-   **Read:** Server returns an ETag.
-   **Write:** Client sends the ETag using `If-Match`.
-   **Database:** Atomic `UPDATE ... WHERE version = expected_version`.
-   **Success:** Update one row and increment version.
-   **Conflict:** Update zero rows; return `412 Precondition Failed`.
-   **Missing precondition:** Optionally return
    `428 Precondition Required`.
-   **Retry safety:** Use an Idempotency-Key where appropriate.
-   **Key benefit:** No long-running lock while the user is editing.
