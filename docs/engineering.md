# Orderly engineering notes

Orderly looks like a fairly simple order tracker from the outside. The interesting engineering problem is that the app cannot treat the network as a prerequisite for doing work.

Small sellers may be taking orders from WhatsApp, Instagram, Facebook, walk-ins, or direct messages while moving between Wi-Fi and mobile data. The app therefore needs to remain useful when Firestore is slow, temporarily unreachable, or unavailable altogether.

This document describes the architecture of the private production app without publishing the commercial source code.

---

## 1. Design rule: local work should feel local

The basic rule is:

> the user should not need to wait for the cloud to record an order.

The local Drift/SQLite database is the working state used by the app. Firestore provides cloud persistence and cross-device synchronization, but it is not placed in the critical path of normal order entry.

A simplified write looks like this:

```text
user edits an order
      ↓
repository validates / normalizes data
      ↓
SQLite transaction
      ├── persist new local state
      ├── dirty = true
      └── upsert outbox mutation
      ↓
UI observes committed local data
      ↓
SyncService is kicked in the background
```

This gives the UI deterministic local behavior regardless of current connectivity.

---

## 2. Local database responsibilities

Drift sits above SQLite and stores more than the visible business entities.

Conceptually the local store contains:

```text
orders
customers
user profile
sync metadata
outbox mutations
soft-deletion metadata
```

Business rows carry synchronization metadata alongside the normal domain fields. Important concepts include:

- a stable entity ID
- `updatedAt` information
- the device that last modified the row
- whether the local row is still `dirty`
- optional deletion timestamps

That metadata is what lets the database distinguish between:

1. clean data that mirrors the server,
2. local data that has not reached the server yet, and
3. entities intentionally deleted but still inside the tombstone/grace period.

---

## 3. The outbox

Offline writes are represented by an outbox rather than by hoping a network request eventually succeeds.

Each pending mutation records enough information to identify:

```text
user
entity type
entity id
action: upsert | delete
```

Orderly currently synchronizes orders, customers, and user-profile state through this mechanism.

The outbox gives the sync engine a durable queue. If the app is killed after a local write but before Firestore accepts it, the intent to synchronize still exists after restart.

That matters much more than retrying an in-memory future.

---

## 4. Synchronization lifecycle

`SyncService` owns the high-level synchronization session.

It reacts to several events:

- an authenticated session becoming ready
- connectivity changing from offline to online
- the app returning to the foreground
- a repository scheduling new work
- a server-side force-refresh signal
- changes to cloud-access / entitlement state

A sync pass is broadly:

```text
1. verify the authenticated user still matches the active session
2. verify cloud sync is allowed
3. verify connectivity
4. publish syncing state
5. load pending outbox mutations
6. push each mutation
7. refresh the remote snapshot
8. merge it into SQLite without destroying dirty local work
9. prune sufficiently old tombstones
10. publish the new steady state
```

### Coalescing repeated triggers

Mobile apps produce duplicate triggers all the time: connectivity may return at the same moment the app resumes while a repository also requests sync.

Orderly does not start multiple competing sync passes. If one is already in flight, a `syncNeeded` flag is set. The running pass finishes, then another pass starts immediately if necessary.

This avoids silently dropping a trigger without creating concurrent sync races.

---

## 5. First-device hydration

Offline-first does not mean a completely new device should invent an empty account while offline.

The app stores whether the account has completed its initial remote hydration on that device.

If a returning local database has already been hydrated, the app can open from local state and synchronize opportunistically.

If this is the first session on the device and no remote snapshot has ever been downloaded, Orderly requires one successful online hydration before presenting the account as ready for offline use.

That avoids a nasty class of problems where a real cloud account appears empty simply because the first launch happened without connectivity.

---

## 6. Push before pull

During a sync pass, pending local mutations are processed before the full remote refresh.

That ordering protects fresh local edits from immediately being replaced by an older server snapshot.

For an order mutation, the engine can compare the local row with the server copy before deciding which side should win.

The comparison uses modification time together with the last-modifying device as deterministic conflict metadata rather than relying only on Firestore arrival order.

The exact production conflict code remains private, but the useful principle is:

> synchronization needs a deterministic winner that is independent of network timing.

---

## 7. Merge, do not replace

One of the most important implementation changes in the project was moving away from wholesale remote-snapshot replacement.

A naive refresh is tempting:

```text
delete local rows
insert everything from Firestore
```

That is unsafe in an offline-first system. A local row may be missing from Firestore precisely because it is waiting in the outbox.

The production merge instead follows rules like:

```text
for each local row:
    if dirty:
        preserve it
    else if no longer present remotely:
        remove it

for each remote row:
    if corresponding local row is dirty:
        preserve local pending work
    else:
        apply remote state
```

The real database method performs the merge transactionally so observers do not see a half-replaced dataset.

---

## 8. Deletes are data too

Immediate hard deletion makes conflict handling unnecessarily fragile.

Orderly uses soft-deletion metadata for synchronized entities. A delete can therefore propagate as state before the physical row is removed.

This helps with cases such as:

- another device still holding an older copy,
- a delete being queued offline,
- a sync interruption midway through the operation,
- accidental deletion that may need support-side investigation.

After the deletion has existed long enough and no pending sync work depends on the row, old tombstones are pruned.

The current production flow keeps synchronized deleted order data through a grace period rather than immediately destroying it locally.

---

## 9. Session safety

Another failure mode is allowing stale local state to keep synchronizing after the Firebase user has signed out or changed.

Before remote sync work, the service checks that the active Orderly session still matches the authenticated Firebase user.

If the server indicates that the account was deleted or access was revoked, the sync layer can terminate the local session rather than continuing to behave as though the remote account still exists.

This is especially important in an app that intentionally retains useful local data for offline operation.

---

## 10. Sync is visible state

The app exposes synchronization as state rather than hiding every network condition behind a spinner.

Examples include concepts such as:

```text
idle
syncing
synced
offline with pending changes
error
initial hydration required
```

The state includes useful metadata such as pending-change count and last successful synchronization time.

That gives settings and other UI surfaces enough information to communicate what is actually happening without blocking the order workflow itself.

---

## 11. Repository boundary

Feature code does not directly coordinate SQLite and Firestore calls.

Repositories sit between feature state and infrastructure. The general shape is:

```text
presentation
    ↓
repository
    ↓
local database
    ↓
sync scheduling
```

Remote reconciliation lives in the sync layer, which keeps screens from accumulating networking and conflict-resolution rules.

This also makes it practical to test business behavior with local/fake dependencies instead of requiring live Firebase services for every feature test.

---

## 12. Order model complexity

What began as a simple customer + item + price record grew into a more useful small-business domain.

An order can now represent information such as:

- customer snapshot and normalized phone
- multiple line items
- quantities and units
- discounts and delivery fees
- optional cost information
- payment state and advance payments
- delivery / pickup / courier logistics
- delivery address and city
- courier/tracking information
- order source
- notes
- settlement date
- reply state
- operational status
- creation/update timestamps

The same core model needs to work consistently across quick-add, edit, detail, customer history, dashboard metrics, CSV/PDF flows, receipts, and synchronization.

This is why keeping domain behavior below the widgets became increasingly important as the app grew.

---

## 13. Customer identity and seller markets

A phone number is not a globally stable string.

Orderly uses seller-market context for normalization and formatting decisions. The selected market also influences:

- currency display
- date preferences
- phone handling
- available payment methods
- suggested cities
- courier hints
- onboarding/demo values

Customer matching therefore works with normalized representations rather than blindly comparing whatever formatting the user typed.

This lets repeat-customer history remain useful while supporting businesses beyond a single country configuration.

---

## 14. Localization

The primary UI ships in:

- English
- Sinhala
- Tamil

Flutter's generated localization flow uses ARB resources, with locale state persisted for the user and applied at the `MaterialApp` level.

Localization is not limited to menu labels. Product flows including onboarding, orders, customers, filtering, trial/upgrade UX, notifications, and operational messaging have been progressively moved into the same localization system.

---

## 15. Receipts and exports

The document pipeline is built into the mobile product rather than delegated to a web dashboard.

Orderly can produce:

- branded single-order PDF receipts
- PDF business reports
- CSV exports
- printable/shareable output

Receipt generation uses the business profile and the order snapshot to build a document that can be previewed before sharing.

The important design detail is that generated documents use the same domain values and market-aware formatting rules as the rest of the app, reducing the chance that an on-screen total and a customer receipt disagree.

---

## 16. Purchases and entitlements

The mobile client supports native store purchase flows on Android and Apple platforms.

Purchase verification is intentionally separated from the core order domain. The wider Orderly system includes server-side relay/verification endpoints and Firestore-backed entitlement state so the app is not simply trusting a local purchase callback as permanent account truth.

The current public product offers a 20-order trial and a one-time Pro unlock. Store-returned localized prices are used in customer-facing purchase UI rather than hard-coding a currency into the client.

Entitlement changes can trigger a force refresh so an already-running mobile session picks up server-side state changes without requiring a full reinstall or manual cache reset.

---

## 17. The web/backend side

The private web project is a Next.js application deployed through Cloudflare infrastructure.

It provides several responsibilities outside the mobile client:

```text
public marketing + legal pages
purchase verification / relay endpoints
pricing configuration
admin tooling
store notification handling
account-level operations
```

Keeping these responsibilities outside the mobile package means the order application does not need privileged store credentials or administrative Firebase access.

---

## 18. Observability

Offline-first bugs are difficult to diagnose because the interesting failure may happen long after the original user action.

Orderly combines:

- structured application logging
- Firebase Crashlytics
- Firebase Performance
- analytics around important product flows
- explicit sync-state/error reporting

Sync failures preserve enough state for a later retry rather than turning a transient backend error into lost user work.

---

## 19. Testing and CI

The production repository runs GitHub Actions on pushes and pull requests.

The current verification job performs:

```text
dart formatting check
flutter analyze
pricing-model consistency check
flutter test --coverage
```

The codebase also contains fake infrastructure for feature/widget tests so screens and repositories can be tested without making live cloud calls.

Separate beta/release automation handles the more involved Android and Apple delivery pipeline.

---

## 20. What I would keep from this project

If I were starting another offline-capable app tomorrow, the pieces I would carry over are not the exact classes. They are the constraints:

1. **Commit locally before doing remote work.**
2. **Persist sync intent in an outbox.**
3. **Never replace a remote snapshot over unsynced local state.**
4. **Make conflict resolution deterministic.**
5. **Represent deletions long enough for them to synchronize safely.**
6. **Do not claim offline readiness on a new device until initial hydration has happened.**
7. **Treat account/session changes as part of the sync problem.**
8. **Expose sync state instead of pretending connectivity is binary.**

The UI is the visible product. Most of the reliability comes from these less visible decisions underneath it.

---

This repository is a public engineering showcase. The production source, store configuration, backend configuration, and operational tooling remain private.
