# Fin Tracker — System Documentation

A shared net-worth, income, loan, and goal tracker for two people, backed by
Firebase Firestore, hosted as a single static HTML file on GitHub Pages.

This file exists so the system doesn't depend on anyone remembering a chat
history. Keep it in the same repo as `fin-tracker.html`, and update it
whenever the data model or setup changes.

---

## 1. Architecture

- **`fin-tracker.html`** — the entire app: HTML, CSS, and JS in one file. No
  build step, no server code. Hosted as a static file (e.g. GitHub Pages).
- **Firebase Firestore** — the database. All data lives in one collection,
  `fintracker`, with one document per data type (see schema below).
- **`fintracker-reminders.gs`** — a separate Google Apps Script, running on
  its own schedule, independent of the web app. Sends EMI-due and year-end
  emails. Lives in your Google account at script.google.com, not in the repo
  (though you can keep a copy here for reference).

Nothing else is required to run this — no npm, no backend server, no paid
hosting.

## 2. Data model

Collection `fintracker`, documents (each holds `{ items: [...] }`):

| Document        | Contents                                                        |
|------------------|-------------------------------------------------------------------|
| `income`         | `{id, month, person, type, amount, notes}`                       |
| `assets`         | `{id, name, productType, platform, assetClass, owner, principal, currentValue, purchaseDate, status, goalId}` |
| `liabilities`    | `{id, loanName, borrower, lender, originalAmount, outstandingBalance, emi, interestRate, startDate, endDate, status, emiDay, autoTrack, lastAccrualDate}` |
| `goals`          | `{id, name, priority, type, targetDate, status, currentAmount, targetAmount, notes}` |
| `history`        | `{id, month, assets, liabilities}` — manual month-end snapshots for the trend chart |
| `loanLedger`     | `{id, loanId, date, type, amount, principalComponent, interestComponent, note}` — every auto-EMI, prepayment, and adjustment |
| `assetClasses`   | plain array of strings — the user-extensible asset class list     |

**When adding a new field to any record:** always read it with a fallback
(`item.newField || defaultValue`), never assume it exists — older records
won't have it. This is how the app already handles schema growth; keep doing
it rather than migrating old records.

## 3. Setting up a new device / browser

The Firebase config is stored in that browser's `localStorage`, not in the
app file itself. A new browser (new phone, cleared cache, different person)
will show the "Connect Firebase" screen on first load.

To reconnect:
1. Go to Firebase Console → your project → Project settings → General →
   scroll to "Your apps" → copy the `firebaseConfig` object.
2. Open the app, click the connection status pill in the header, paste the
   config, and connect.

**Keep a copy of your `firebaseConfig` somewhere durable** (a password
manager note, or a private doc) — it's not secret (it's a client-side web
key, not a credential), but losing track of it means digging through
Firebase Console again.

## 4. Backups

- **Manual**: the ⬇️ button in the header exports everything to an `.xlsx`
  file (Summary, Income, Assets, Liabilities, Goals, NW History, Loan
  Ledger). Do this periodically and keep the file somewhere outside
  Firestore — Google Drive, email to yourself, etc.
- **Automatic**: not yet set up. The cleanest option would be extending
  `fintracker-reminders.gs` with a scheduled job that exports and emails an
  `.xlsx` (or writes to a Google Sheet) monthly, using the same Firestore
  REST pattern already in that script. Worth adding before relying on this
  for years, not just months.

## 5. Security

Currently there's no login — Firestore rules alone gate access. For a
system meant to last, the recommended next step is **Firebase
Authentication** (email link or Google sign-in, restricted to your two
email addresses), with Firestore rules requiring `request.auth != null`.
Ask for this to be added when ready — it changes the connect flow (you'll
sign in once per device instead of just pasting config) but is the
standard way to actually lock this down long-term.

## 6. The EMI auto-tracking logic

Loans with "auto-track" on use the reducing-balance method: each month,
`interest = outstanding × (rate/12)`, `principal = EMI − interest`,
`new outstanding = old outstanding − principal`. The EMI itself never
changes — only how much of it goes to interest vs. principal, which shifts
as the balance shrinks. This runs client-side, once per page load, catching
up any months missed since the last time either of you opened the app — it
does not require the app to be open on the actual due date.

## 7. Change log

Keep a short dated entry here whenever the schema or core logic changes, so
future changes don't silently break old data:

- **2026-09**: Initial system — income, assets, liabilities, goals, net
  worth history, Firebase storage, EMI auto-tracking, goal-asset linking,
  custom asset classes, edit-in-place, Excel export.
