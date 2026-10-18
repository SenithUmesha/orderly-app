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

## release status

Orderly is available on the **App Store and Google Play**. The private mobile source is at **1.14.0+94**, with the 1.14.0 release candidate validated on Android and iOS. At the last confirmed checkpoint, Android build **94** was signed and ready for manual upload, while iOS build **95** was available in internal TestFlight. Public availability of 1.14.0 has not yet been confirmed.

**Preparing for 1.14.0:**

- Dedicated **Account & Security** settings, including provider-aware sign-in information and a separate contact email
- Custom **app accent colors**, independent of receipt branding
- **Full `.orderly` backup and restore** for supported business data and preferences
- **System default**, English, Sinhala, Tamil, Spanish, and Brazilian Portuguese language choices
- Simplified, fixed **Orderly Pro Lifetime** pricing at **US$49.99** as a store reference; checkout always shows the price provided by Apple or Google

The established **1.13.x** improvements remain part of the app: Dashboard **Needs Attention**, editable customer messages, **Order Again**, recent/favourite items, and persistent order-detail actions. This showcase describes the product and engineering; it does not distribute the private production code or certify a store rollout.

---

## why i built it

A lot of small businesses do not need a giant ERP.

They need to know:

- who ordered what
- how much is still unpaid
- what needs to be delivered next
- whether they already replied to the customer
- what needs attention today
- what the business actually made

And they need all of that to keep working when the connection is bad.

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

That local-first loop is the core of the product.

---

## a quick look

<p align="center">
  <img src="https://www.orderlyapp.app/images/screenshot-dashboard.png" width="23%" alt="Orderly dashboard" />
  <img src="https://www.orderlyapp.app/images/screenshot-quick-add.png" width="23%" alt="Orderly quick add order" />
  <img src="https://www.orderlyapp.app/images/screenshot-orders.png" width="23%" alt="Orderly orders list" />
  <img src="https://www.orderlyapp.app/images/screenshot-customers.png" width="23%" alt="Orderly customer list" />
</p>

---

## what it does

### fast order capture

Orders can include multiple line items, customer details, payment state, delivery or pickup information, order source, notes, discounts, delivery fees, partial payments, settlement dates, courier details, and custom timestamps for past orders.

Known customers can be filled from phone history. Recent and favourite items speed up repeat entry, while **Order Again** lets a seller start from an existing order and review it before saving a new one.

### follow-up that stays actionable

The 1.13.x line added a dedicated **Needs Attention** surface to Dashboard. It brings together overdue work, orders due today, payment follow-ups, and unanswered customers with direct links back into matching order views.

Daily reminders can surface the same operational context and open Dashboard when tapped.

### customer communication

Order detail keeps **Chat, Call, and More** visible while scrolling, so contact actions do not disappear on long orders.

WhatsApp can open as a blank chat or use reusable message templates for common moments. Receipt and reply actions are fully labelled instead of hidden behind ambiguous icons.

### payments that are actually useful

Orders can be unpaid, partially paid, or paid. Balance due is calculated from the real order charge and advance already received.

That state feeds order cards, customer history, dashboard metrics, receipts, and reporting.

### receipts without another tool

Completed orders can produce branded PDF receipts inside the app.

Business details, logo, accent colour, payment information, line items, totals, and customer details flow into the generated document, which can be previewed and shared directly.

### customer history grows from the orders

Customer profiles surface order history, lifetime value, unpaid balance, repeat-customer context, frequently ordered items, and quick contact actions.

Phone numbers are normalized using the selected seller market so matching remains useful across countries.

### dashboard built around decisions

The dashboard focuses on operational information:

- revenue today / this week / longer ranges
- orders created today
- pending work
- deliveries due
- outstanding balances
- customers waiting for a reply
- Needs Attention follow-ups
- best customers and popular items

Most cards deep-link back into filtered order views so analytics stay connected to an action.

### local formats and language choices

Seller-market settings drive local currency display, date formatting, phone normalization, payment options, sample data, and courier hints without changing the lifetime purchase price.

The current public release includes **English, Sinhala, and Tamil**; **Spanish, Brazilian Portuguese, and System default** selection are part of the 1.14.0 release candidate.

### your data is not trapped

Orderly supports CSV import/export, PDF export, printing, and receipt sharing. The 1.14.0 release candidate also adds full `.orderly` backup/restore for supported orders, customers, business settings, and preferences.

---

## the fun part: sync

Orderly treats the local SQLite database as the working copy of the app.

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

When connectivity returns, the sync engine drains pending mutations and refreshes remote state.

Remote data cannot simply replace the local database because there may be local edits that have not reached the server yet. The merge path preserves dirty rows, removes only clean rows that disappeared remotely, and applies remote rows only when they do not overwrite pending local work.

The production app also handles reconnect-triggered sync, app-resume sync, overlapping sync triggers, first-device hydration, remote account deletion, pending offline changes, and server-side entitlement changes.

More detail is in **[docs/engineering.md](docs/engineering.md)**.

---

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

| Area | Choice |
| --- | --- |
| Mobile | Flutter / Dart |
| State | Provider |
| Local database | Drift + SQLite |
| Cloud | Firebase Authentication + Cloud Firestore |
| Sync | Local outbox + dirty rows + merge-based refresh |
| Analytics | Firebase Analytics |
| Diagnostics | Crashlytics + Performance + structured app logging |
| Documents | CSV + PDF + print/share pipelines |
| Purchases | App Store / Google Play in-app purchase flows |
| Localization | Flutter gen-l10n + ARB |
| Automation | GitHub Actions |

---

## commercial model

Every new account starts with a full-feature **20-order trial**.

**Orderly Pro Lifetime** is a one-time purchase that removes the order cap permanently. Cloud backup and sync are included during the trial and with Pro. Existing data stays accessible after the trial ends.

The current reference price is **US$49.99**. Apple and Google provide the actual checkout price in the customer's store currency. There is no recurring subscription.

---

## shipping it

Orderly is live on **iOS/iPadOS** and **Android**.

The private production repository has checks for formatting, static analysis, tests, purchase/restore, social-auth configuration, pricing-model consistency, and release validation. Its build-time checks help prevent a release artifact from silently shipping with the Pro purchase path disabled. The 1.14.0 release candidate passed 300 Flutter tests and 16 Firestore-rules tests.

This public repository is the **product + engineering showcase**. The production source remains private because it contains commercial app code, backend integration, signing configuration, and release infrastructure.

---

## one more thing

The goal was never to build the most complicated order-management system possible.

It was to make the common path reliably boring:

> a customer sends an order → you log it → it stays there → you know what happens next.

Making that statement stay true offline, across devices, through app restarts, and through conflicting edits is where most of the engineering went.

---

Built by [Senith Umesha](https://github.com/SenithUmesha).
