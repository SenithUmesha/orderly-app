<p align="center">
  <img src="https://www.orderlyapp.app/images/app-icon-rounded.png" width="112" alt="Orderly app icon" />
</p>

<h1 align="center">Orderly</h1>

<p align="center">
  <strong>Orders do not stop when the Wi-Fi does.</strong>
</p>

<p align="center">
  An offline-first order and customer manager for independent sellers and small businesses.
</p>

<p align="center">
  <a href="https://apps.apple.com/us/app/orderly-order-tracker/id6783097161">App Store</a>
  ·
  <a href="https://play.google.com/store/apps/details?id=lk.orderly.app">Google Play</a>
  ·
  <a href="https://www.orderlyapp.app/">Website</a>
  ·
  <a href="docs/engineering.md">Engineering notes</a>
</p>

<p align="center">
  <code>Flutter</code> · <code>Drift</code> · <code>SQLite</code> · <code>Firebase</code> · <code>Provider</code> · <code>offline-first</code>
</p>

<p align="center">
  <a href="https://github.com/SenithUmesha/orderly-app/actions/workflows/docs-check.yml"><img src="https://github.com/SenithUmesha/orderly-app/actions/workflows/docs-check.yml/badge.svg" alt="Docs integrity" /></a>
</p>

---

## why i built it

A lot of small businesses do not need a giant ERP.

They need to know:

- who ordered what
- how much is still unpaid
- what needs to be delivered next
- whether they already replied to the customer
- what the business actually made today

And they need all of that to keep working when the connection is bad.

Orderly started from that problem. The product is intentionally mobile-first: capture the order quickly, keep the local app responsive, and let synchronization happen around the user instead of making the user wait for the network.

```text
customer messages you
        ↓
log the order
        ↓
local database commits immediately
        ↓
customer + balance + dashboard update
        ↓
sync catches up when the network is ready
```

That local-first loop ended up becoming the most interesting part of the project.

## a quick look

<p align="center">
  <img src="https://www.orderlyapp.app/images/screenshot-dashboard.png" width="23%" alt="Orderly dashboard" />
  <img src="https://www.orderlyapp.app/images/screenshot-quick-add.png" width="23%" alt="Orderly quick add order" />
  <img src="https://www.orderlyapp.app/images/screenshot-orders.png" width="23%" alt="Orderly orders list" />
  <img src="https://www.orderlyapp.app/images/screenshot-customers.png" width="23%" alt="Orderly customer list" />
</p>

## what it does

### 📦 Fast order capture

Orderly is built around getting an order out of a chat and into a proper record without turning that into admin work.

An order can include multiple line items, customer details, payment state, delivery or pickup information, order source, notes, discounts, delivery fees, partial payments, settlement dates, courier details, and custom timestamps for past orders.

Known customers can be filled from phone history, and repeat information is reused wherever possible to keep the flow fast.

### 💸 Payments that are actually useful

Orders can be unpaid, partially paid, or paid. Balance due is calculated from the real charge and the advance already received.

That same state feeds order cards, customer history, dashboard metrics, receipts, and reporting so payment information is not duplicated across separate features.

### 🧾 Receipts without another tool

Completed orders can produce branded PDF receipts from inside the app.

Business details, logo, accent colour, payment information, line items, totals, and customer details flow into the generated document, which can be previewed and shared directly.

### 👥 Customer history grows from the orders

The customer side is derived from the work already being logged.

Profiles surface order history, lifetime value, unpaid balance, repeat-customer context, frequently ordered items, and quick contact actions. Phone numbers are normalized with the selected seller market in mind so matching stays useful across countries.

### 📊 A dashboard for the questions that come up every day

The dashboard focuses on operational information instead of vanity charts:

- revenue today / this week / across longer ranges
- orders created today
- pending work
- deliveries due
- outstanding balances
- customers still waiting for a reply
- best customers and popular items

Most dashboard cards deep-link back into filtered order views, so the analytics remain connected to an action.

### 🌍 Built beyond one hard-coded market

Seller-market configuration drives things like currency, date formatting, phone normalization, payment options, sample data, suggested cities, and courier hints.

The core UI is localized in **English, Sinhala, and Tamil**.

### 📤 Your data is not trapped

Orderly supports CSV import/export, PDF export, printing, and receipt sharing. The idea is simple: a business tool should make it easy to take your own records with you.

---

## the fun part: sync

Orderly treats the local SQLite database as the working copy of the app.

A normal write does not wait for Firestore:

```text
UI action
   ↓
repository
   ↓
Drift / SQLite transaction
   ├── write the entity
   ├── mark the row dirty
   └── enqueue an outbox mutation
            ↓
        UI is already done
            ↓
      SyncService runs later
            ↓
         Firestore
```

When connectivity returns, the sync engine drains pending mutations and then refreshes remote state.

The important bit is what happens during that refresh: remote data cannot simply replace the local database because there may be local edits that have not reached the server yet. The merge path preserves dirty rows, only removes clean rows that disappeared remotely, and only applies remote rows when they do not overwrite pending local work.

Conflict checks use update timestamps together with the device that last modified the row. Deletions are represented as tombstones first instead of immediately destroying the record, and stale deleted rows are pruned later.

There is also explicit handling for:

- reconnect-triggered sync
- app-resume sync
- multiple sync triggers arriving while one is already running
- first-device hydration before offline use is allowed
- remote account deletion / revoked access
- pending changes while offline
- force-refresh events from server-side account or entitlement changes

More of that is in [the engineering notes](docs/engineering.md).

## architecture

```text
Flutter UI
   ↓
Provider / feature state
   ↓
Repositories
   ↓
┌───────────────────────────┐
│ Drift / SQLite            │  ← working local state
│ + outbox + sync metadata  │
└─────────────┬─────────────┘
              ↓
         SyncService
              ↓
      Cloud Firestore
```

The app is split by product feature, with shared local storage, repositories, services, diagnostics, localization, reporting, billing, and sync infrastructure underneath.

```text
lib/
├── core/
│   ├── local/
│   ├── models/
│   ├── repositories/
│   ├── services/
│   ├── diagnostics/
│   └── utils/
├── features/
│   ├── auth/
│   ├── onboarding/
│   ├── dashboard/
│   ├── orders/
│   ├── customers/
│   ├── settings/
│   └── upgrade/
├── shared/
└── l10n/
```

## under the hood

| Area | Choice |
| --- | --- |
| Mobile | Flutter / Dart |
| State | Provider |
| Local database | Drift + SQLite |
| Cloud | Firebase Authentication + Cloud Firestore |
| Sync | Local outbox + dirty rows + merge-based refresh |
| Connectivity | connectivity-aware sync triggers |
| Analytics | Firebase Analytics |
| Diagnostics | Crashlytics + Performance + structured app logging |
| Documents | CSV + PDF + print/share pipelines |
| Purchases | App Store / Google Play in-app purchase flows |
| Localization | Flutter gen-l10n + ARB |
| Automation | GitHub Actions |

## the wider product

The mobile app is only one part of the system.

Orderly also has a web side that handles the public product site, legal/support pages, store purchase verification, pricing configuration, and internal admin tooling. The mobile client can stay focused on the actual order workflow while server-side verification handles purchase state and account-level operations.

That split also keeps store-specific logic out of the core order domain.

## shipping it

Orderly is live on both **iOS/iPadOS** and **Android**.

The private production repository has CI that checks formatting, static analysis, pricing-model consistency, and the test suite on pushes and pull requests. Separate release automation handles mobile beta/release workflows.

The public repo you are looking at is the product + engineering showcase. The production source stays private because it contains the commercial app, release configuration, and backend integration code.

## one more thing

The goal was never to build the most complicated order-management system possible.

It was to make the common path boringly reliable:

> a customer sends an order → you log it → it stays there → you know what happens next.

Making that statement stay true offline, across devices, across app restarts, and through conflicting edits is where most of the engineering went.

---

Built by [Senith Umesha](https://github.com/SenithUmesha) as one of those _“this should probably be an app”_ ideas that kept growing.
