# Workflows & Automation Spec — ASP Bookings

> The exact rules for every automated email/notification ASP Bookings should send once a real
> backend + email sending exists (see `INTEGRATIONS.md` §1 for the send mechanism). Written up
> front so backend work can implement these directly rather than re-deriving them. Where a rule is
> already represented in today's mockup (as a simulated activity-log entry / toast, per the app's
> "mocked send, real data" convention), that's noted — those are the reference implementation for
> content/wording; only the actual delivery is fake today.

---

## 1. Daily Digest — to Moshe, every morning

**Trigger:** scheduled job, once daily (suggested 7:00am local).
**Recipient:** Moshe (owner/developer — not one of the in-app `ADMIN_USERS`, so this needs his own
email on file once real backend work starts, separate from the office logins).
**Content:**
- **Open Leads Needing Follow-Up** — every event with status `lead`, `negotiating`, or
  `contract_sent` (i.e. not yet booked), each with artist, client, date, and days since the lead
  was created.
- **A pastable WhatsApp-style message for Ilan** — the same open-leads list condensed into plain
  text Moshe can copy straight into a WhatsApp message to Ilan (ASP's CEO) as a nudge to follow up
  that day. Not an automated WhatsApp send — Moshe pastes it himself.
- **Awaiting Down Payment** — events with status `contract_sent` (contract + 15% invoice already
  sent, booking fee not yet received).
- **Awaiting Final Payment** — events with status `booked` and `balanceReceived: false` (15%
  received, balance still outstanding).

**In-app today:** built as an on-demand "Daily Digest" page (real computed data, reachable from
the management dashboard) with a "Copy WhatsApp Message" button — see the app itself. Becomes a
scheduled job emailing the same content once backend exists; the in-app page stays as a live
preview/on-demand fallback.

## 2. New lead → first client email: contract + invoice

**Trigger:** admin sends the contract (today: the "Send Contract + Invoice" button on an event,
moves status `lead`/`negotiating` → `contract_sent`).
**Recipient:** client.
**Content:** the ASP contract (see the in-app Contract doc template) + a QuickBooks-generated
invoice link for the 15% booking fee.
**In-app today:** fully simulated — `doSendContract()` logs `"Contract + QuickBooks invoice (15%
booking fee) emailed to {client}."` and toasts. This is the reference wording; no changes needed
here, just wire to a real QuickBooks invoice + email send later.

## 3. 15% received → booking confirmed

**Trigger:** admin marks the booking fee received (today: "Mark Deposit Received", moves status →
`booked`).
**Recipients & content (three separate notifications, same trigger):**
1. **Client** — confirmation email with full gig details (date/time/location) **plus the
   remaining balance amount and a QR code** (Zelle-linked) to pay it. The QR-code mechanism
   already exists in-app for balance *reminder* emails (`renderReminderPreview()`, generates a
   real QR image via `api.qrserver.com` encoding a Zelle reference string) — the booking
   confirmation email should use the same QR block, just triggered a step earlier (at booking
   confirmation, not only at reminder time).
2. **Artist** — schedule notification (gig is now locked on their calendar).
3. **Ilan + Moshe** — notified that the booking is confirmed (office-wide visibility on newly
   locked gigs). This is new — today only the artist gets notified at this step, not Ilan/Moshe.

## 4. Balance reminder cadence

**Rule:** every 2 weeks by default, tightening to **every week once the gig is within 30 days**,
and stopping entirely once the balance is marked received.
**In-app today:** `event.reminderIntervalDays` exists but is a flat manual value (default was a
hardcoded `5`, adjustable via a 3/5/7/10/14-day dropdown on the event) — it doesn't implement this
two-phase rule, and nothing re-evaluates it as the gig date approaches (there's no real cron in a
static frontend). **Fix applied:** the *default* value at event creation now follows the formula
(14 days if the gig is >30 days out, 7 days if closer) via a shared `defaultReminderCadence(date)`
helper, still manually overridable per event. **For the real backend:** the scheduled job should
compute the interval live from `daysUntil(gig date)` rather than trusting the stored field, so it
actually tightens as the date approaches — the stored `reminderIntervalDays` should become a
manual-override-only field (null = follow the automatic rule).

## 5. Flight needed → booking secretary

**Trigger:** admin marks "flight needed" on an event.
**Recipient:** the booking secretary (`ASP Office — Bookings` account).
**Content:** gig details + a flag that a flight needs to be booked. She books it externally and
enters the confirmed details back into the app (existing flow, unchanged).
**In-app today:** logs `"Flight needed — bookkeeping notified."` as a `system`-type entry (not
`email`) — cosmetic-only distinction in the mock activity log, but should read as `email` once
this is real, since that's what actually happens.

## 6. Flight/ground-transport details added → Moshe + artist (not Ilan)

**Trigger:** flight or ground-transport details are saved (booked) on an event.
**Recipients:** Moshe and the artist. **Explicitly not Ilan** — he doesn't need travel logistics
noise, only the booking-confirmed notification from §3.
**In-app today:** logs artist-only (`"Flight booked & added to itinerary — {artist} notified."`),
doesn't mention Moshe. Same for ground transport. Needs Moshe added to both.

## 7. Artist weekly gig digest (opt-in)

**Trigger:** scheduled job, weekly.
**Recipient:** any artist who has opted in.
**Content:** that artist's gigs for the upcoming week.
**In-app today:** not built as a live schedule (no cron), but the **opt-in preference itself**
lives in Settings → Notifications as a new toggle (`weeklyGigDigest`), artist-only, same
in-app/email split pattern as the other notification items — ready for the real job to read.

## 8. Artist day-of reminder (opt-in)

**Trigger:** morning of a gig day.
**Recipient:** any artist who has opted in, for gigs happening that day.
**Content:** a same-day reminder of the gig's time/location.
**In-app today:** same as §7 — opt-in toggle only (`dayOfReminder`) in Settings, no live schedule
yet.

## 9. Financial tracking: gig breakdowns, artist payouts, expenses

Scope: per-gig financial breakdown (price, ASP commission, artist payout, any charges),
who's-been-paid-out tracking, and general expense tracking — **manual entry**, living entirely
inside the existing **Financials** page (not a separate section/nav item).

**Bank account linking — explicitly deferred, not built.** Moshe's call (2026-08-06): document
the requirement now, build nothing yet. When it's picked up: this is real financial-account
access, so it falls under `CLAUDE.md`'s "hardware/financial-control APIs are crown jewels" rule —
server-only, encrypted, admin-gated, rate-limited, audit-logged, minimum scopes, never logged. The
proven integration pattern for this is **Plaid** (bank-linking-as-a-service: hosted Link flow for
the OAuth-like bank connection, Transactions API for pulling activity) — add it to
`INTEGRATIONS.md` as a new "Banking / Bank Feeds" section when this gets picked up, alongside the
existing Payments (Stripe) section, since they're different capabilities (moving money vs. reading
transaction history).

---

## Open items / not yet specified
- Real email address for Moshe's own notifications (Daily Digest, §3, §6) — office accounts today
  are `bookings@`/`bookkeeping@`/`ilan@aspmanagement.com`; Moshe isn't one of them.
- Whether the Daily Digest's WhatsApp message should also cover artist follow-ups (e.g. an artist
  who hasn't confirmed something) or stay scoped to client leads only, as asked.
