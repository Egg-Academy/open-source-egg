# Calendar Booking System

A design for several named bookable services on one calendar. A guest picks a time with no account. Staff set the times up, see who booked, and receive email when a booking is made, cancelled, or moved.

A coding assistant can implement this on its own. It does not create accounts, take payment, or depend on a particular login or email vendor.

Configure these for the implementing organisation:

| Constant | Example | Used for |
| --- | --- | --- |
| `ORGANISATION_NAME` | `Example Trust` | Page titles and emails |
| `SITE_ORIGIN` | `https://app.example.org` | Public booking links and the private manage link |
| `PUBLIC_DAYS` | `14` | How many days the public page shows first |
| `PUBLIC_DAYS_MORE` | `28` | How far **Show more times** extends |
| `MATERIALIZE_DAYS` | `365` | How far ahead a weekly rule creates concrete times |
| `MAX_HOSTS` | `10` | Hosts stored on one rule or one time |
| `MAX_EXTRA_FIELDS` | `20` | Extra questions returned by the host app for one service |

Replace the example name and origin. Do not point these at a real organisation or a live site from someone else's deployment.

---

## What it is

An operator defines **services**. Each service is one public page: a name, a short description the organisation writes, a length, a time zone, and a set of times. Examples an organisation might create for itself: an intro conversation, a site visit, a group information session. Those names are theirs. This design does not ship any of them.

A visitor opens the page, picks a time that still has room, and enters their name, email, an optional phone, an optional note, and any extra questions the host app adds. They get a private link for that booking. From that link they can see the booking, cancel it, or move it to another time on the same service.

Staff see every service, add weekly repeating times and one-off times, choose hosts from the host app's staff list, and read who is booked. Hosts on that time get email. The visitor gets email.

The pages below are a **format**. Each one has a title, the facts of the booking, and the actions. The organisation writes the one-paragraph description on the public page. This design does not include what happens at the appointment itself.

## Out of scope

Do not build any of the following inside this module:

- Signup, login, or password reset for the person booking. Staff sign-in already exists in the host app and is reached through `requireOperator()`.
- Payments, invoices, deposits, or promo codes.
- Reminder emails, calendar-file attachments, and SMS.
- In-app notification feeds. Hosts hear about bookings by email.
- Video rooms, lobby pages, maps, or any step that runs during the appointment.
- File uploads on the booking form.
- A marketing site, a progress stepper, or a video above the calendar.
- A specific email provider, auth library, or database. Use the seams below.

A later product may read a booking's status and time. That read is outside this module.

---

## Plug-in interfaces

Implement these seams. Everything else in this document is the module.

### Operator session

Admin screens call one function the host app already has:

```ts
requireOperator(): { operatorId: string }
```

`operatorId` is stored on `createdByOperatorId` when a service, rule, or one-off time is created, and on `cancelledByOperatorId` when staff cancel a booking. The person booking never logs in.

### Staff list

Hosts are people who already exist in the host app. This module does not create staff accounts.

```ts
listStaff(): Promise<
  Array<{
    staffId: string;
    displayName: string;
    email: string;
  }>
>
```

`staffId` is stable. `email` is where host messages go. Resolve it at send time, so a later change of address is used. If a stored `staffId` is no longer in the list, skip that person and send to the remaining hosts.

### Extra questions

The booking form always has name, email, optional phone, and optional note. The host app may add questions per service. This module renders them and stores the strings the host app accepts. It does not invent questions of its own.

```ts
type ExtraField = {
  key: string; // [a-z0-9_]{1,40}, unique within the service
  label: string;
  required: boolean;
  kind: "short_text" | "long_text" | "number" | "single_choice";
  choices?: string[]; // required when kind is single_choice
};

extraFields(serviceId: string): Promise<ExtraField[]>

acceptExtraAnswers(args: {
  serviceId: string;
  answers: Record<string, unknown>;
}): Promise<Record<string, string>>
```

`extraFields` drives the form. Cap the list at `MAX_EXTRA_FIELDS`. `acceptExtraAnswers` validates and returns the map to store. Omit keys the visitor left blank when the field was optional. On failure, return field errors and do not create the booking. Unknown keys are dropped. Values longer than 2000 characters are rejected.

If `extraFields` returns an empty list, the form is the four standard fields only.

### Mailer

```ts
sendMail(args: {
  to: string;
  subject: string;
  html: string;
  disableLinkTracking: true;
}): Promise<void>
```

Turn link tracking off. The manage URL is a secret. Copy is in [Emails](#emails).

### Admin link

Host emails need a link back to the calendar. The host app owns that path:

```ts
adminCalendarUrl(): string
```

Example result: `https://app.example.org/admin/calendar`.

### Filling the calendar ahead

The host app's own scheduler calls this module function at least once a day:

```ts
fillUpcomingTimes(now: Date): Promise<{ created: number; repaired: number }>
```

What it does is in [Weekly rules](#weekly-rules). The scheduler product (cron, queue, or a button) stays in the host app. Also call it when a weekly rule is created or reactivated, so times exist before the next scheduled run.

---

## Actors and screens

| Actor | How they get in | What they do |
| --- | --- | --- |
| **Operator** | `requireOperator()` | Create services, weekly rules, and one-off times. Read bookings. Cancel, move, or resend email |
| **Visitor** | Public page `/book/{slug}` | Pick a time and submit the form. No account |
| **Booker** | One emailed link `/booking/{token}` | View, cancel, or move that booking. No account |

Routes below are the module's paths. Mount them wherever the host app serves pages. Keep the token in the path of the manage page only.

### 1. Service list

Route: operator only. Title: **Calendar**.

- Button: **New service**.
- Table columns: Name, Public page (the `/book/{slug}` path), Length, Time zone, Active, action to open the service.
- A service with `active` false stays in the list and its public page shows "This calendar is not open."

### 2. New service and service settings

Fields:

| Field | Rule |
| --- | --- |
| Name | Required. 1–80 characters. Shown as the page title |
| Slug | Required. Lowercase letters, digits, and hyphens. 2–64 characters. Unique. The public path is `/book/{slug}` |
| Description | Required. 1–500 characters. One paragraph above the calendar. The organisation writes this |
| Length | Integer minutes, 5–480. End time of every new slot is start plus this length |
| Time zone | An IANA name, for example `Europe/London`. Wall-clock times are entered in this zone |
| Default capacity | Integer, 0–500. `1` means one booking fills the time. `0` means no seat limit. New rules and one-off times start from this number and can override it |
| Active | When false, the public page does not offer times |

Creating a service does not create times. The operator adds a weekly rule or a one-off time next.

Changing the length updates the rule's stored length for times created after the change. `fillUpcomingTimes` repairs future slots that have no confirmed booking. It does not move a slot that already has a confirmed booking.

### 3. Service page — times

Operator only. Show the settings from the previous section, then two ways to add times, then the list of concrete times.

**Weekly rule**

| Field | Rule |
| --- | --- |
| Weekday | 0 Sunday through 6 Saturday, in the service time zone |
| Start time | `HH:mm`, 24-hour, in the service time zone |
| Capacity | Defaults to the service. `0`–500 |
| Hosts | One to `MAX_HOSTS` staff ids from `listStaff()` |
| End date | Optional. Last calendar day the rule may create, in the service time zone. Empty means the rule runs until it is turned off |
| Active | An inactive rule creates no further times |

Saving a rule calls `fillUpcomingTimes` for that rule immediately.

**One-off time**

| Field | Rule |
| --- | --- |
| Date | A calendar day in the service time zone, today or later |
| Start time | `HH:mm` in the service time zone |
| Capacity | Defaults to the service |
| Hosts | One to `MAX_HOSTS` staff ids |

The end is start plus the service length. A one-off time has no weekly rule.

**Upcoming times**

List concrete times from the start of today in the service time zone, soonest first. A **Show past** control includes earlier times.

Each row shows:

- Date and start–end in the service time zone
- Seats: `{confirmed} / {capacity}`, or `{confirmed}` when capacity is `0`
- Host display names
- Booker name and email for each confirmed booking
- Whether the time came from a weekly rule or is a one-off

Actions on a row:

| Action | Effect |
| --- | --- |
| **Close** / **Open** | Sets `staffClosed`. A closed time is not offered, even when seats remain. Opening clears `staffClosed`. Fullness does not clear a staff close |
| **Change hosts** | Replaces the host list on that time only |
| **Remove** | See [Removing times](#removing-times) |

Open a row to see the booking records: name, email, phone, note, extra answers labelled with `extraFields`, status, **Move**, **Cancel booking**, **Resend booker email**, **Email hosts again**.

The public page is the same calendar a visitor uses. Link to `/book/{slug}` from this screen. Do not build a second preview with different content.

### 4. Public booking page

Route: `GET /book/{slug}`. No login.

Unknown slug: a short "not found" message.

Inactive service: the name, and "This calendar is not open." No times.

Active service, top to bottom:

1. Organisation name, then the service name as the title.
2. The description paragraph. Nothing else above the calendar: no video, no stepper, no account prompt.
3. Times grouped by calendar day in the service time zone. The first load covers today through `PUBLIC_DAYS` days, including today. **Show more times** extends the end to `PUBLIC_DAYS_MORE` days from today. The server rejects a range longer than that, and a range that starts before the start of today in the service time zone.
4. A day heading is the weekday and date. Under it, one button per time that has not ended.
5. Button label: start and end in the service time zone. When the visitor's zone differs, a second line shows the same instant in their zone. When capacity is greater than 1, add "{n} places left". Capacity `0` or `1` does not show a count.
6. A time that is full, staff-closed, or has no host still appears, disabled, labelled **Unavailable**. Do not explain which. Hide a time only when its end is already past.
7. When the window has no times at all: "No times in this window."

**Book** opens a panel on the same page. The panel shows the chosen date and time, then:

| Field | Rule |
| --- | --- |
| Name | Required. 1–120 characters |
| Email | Required. Trimmed and stored lowercased. One address |
| Phone | Optional. Up to 40 characters |
| Note | Optional. Up to 2000 characters |
| Extra fields | From `extraFields(serviceId)`, in the order returned |

**Confirm booking** submits. On success the panel is replaced by a confirmation: the service name, the date and time, and the sentence "We emailed you a private link to view, cancel, or move this booking." Show the same link on the confirmation as a fallback, because the visitor has no account if the email fails. Do not show it again on later visits except via that link.

### 5. Private booking page

Route: `GET /booking/{token}`. No login. Possession of the token is the credential.

The raw token is 32 random bytes, encoded so it is safe in a path. Store only the SHA-256 hash. Look up by hash. A miss shows: "This link does not match a booking. Ask the organisation to resend it." Do not say whether a similar token existed.

A match shows:

- Organisation name and service name
- Status: **Booked** or **Cancelled**
- Date and time in the service time zone, plus the visitor's local line when the zones differ
- Host display names. Do not show host emails
- The booker's own name and email

Actions, only when status is `confirmed` and the start time is still in the future:

- **Cancel booking**. Ask for a confirm. Then run [Cancel](#transitions).
- **Move**. Show the same day-and-time format as the public page, for this service only, excluding the current time. On confirm, run [Move](#transitions).

When the start time has passed, the page is read-only.

When status is `cancelled`, show the time it was, and a link to `/book/{slug}` to make a new booking. Cancelling does not revive the old row. A new booking is a new record and a new token.

---

## End-to-end flow

```text
Operator creates a service (name, slug, description, length, time zone, capacity)
  -> Operator adds a weekly rule or a one-off time, with at least one host
  -> fillUpcomingTimes creates concrete times out to the horizon
  -> Visitor opens /book/{slug} and submits the form
  -> Booking is stored, manage token is emailed, hosts are emailed
  -> Booker opens /booking/{token}
  -> Booker cancels, or moves onto another open time on the same service
  -> Booker and hosts are emailed again
```

Staff can cancel or move from the service page. Those actions send the same emails.

---

## Records

Plain fields. Names are suggestions for the implementer. Do not copy a particular database.

### Service

| Field | Notes |
| --- | --- |
| `id` | Stable id |
| `name`, `slug`, `description` | Public |
| `durationMinutes` | Length |
| `timeZone` | IANA |
| `defaultCapacity` | `0`–500, default `1` |
| `active` | Public page gate |
| `createdByOperatorId` | From `requireOperator()` |
| `createdAt`, `updatedAt` | |

### Weekly rule

| Field | Notes |
| --- | --- |
| `id` | |
| `serviceId` | |
| `weekday` | 0–6 in `timeZone` |
| `startTime` | `HH:mm` |
| `capacity` | Copied onto new slots |
| `hostStaffIds` | Ordered list, 1–`MAX_HOSTS` |
| `endDate` | Optional last day |
| `active` | |
| `skipDates` | Calendar days (`YYYY-MM-DD` in the service zone) that must not be created again. See [Removing times](#removing-times) |
| `createdByOperatorId` | |

### Time slot

A concrete window. Created from a rule or as a one-off.

| Field | Notes |
| --- | --- |
| `id` | |
| `serviceId` | |
| `ruleId` | Null for a one-off |
| `startAt`, `endAt` | Instants. `endAt` is `startAt` plus the length used when the slot was created |
| `capacity` | `0` means unlimited |
| `staffClosed` | Operator close. Separate from "full" |
| `hostStaffIds` | Ordered, 1–`MAX_HOSTS` |
| `createdByOperatorId` | |

Offered on the public page when all of these are true:

- The service is active
- `staffClosed` is false
- `endAt` is in the future
- `hostStaffIds` is not empty
- Capacity is `0`, or confirmed bookings on this slot are fewer than `capacity`

"Full" is derived. Do not store a second status that drifts from the booking rows.

### Booking

| Field | Notes |
| --- | --- |
| `id` | |
| `serviceId`, `slotId` | Move updates `slotId` and keeps `id` |
| `status` | `confirmed` or `cancelled` |
| `manageTokenHash` | SHA-256 of the path token. Unique. Never store the raw token |
| `bookerName` | |
| `bookerEmail` | Lowercased |
| `bookerPhone` | Null if blank |
| `note` | Null if blank |
| `extraAnswers` | Map of field key to string, from `acceptExtraAnswers` |
| `rescheduledAt` | Set when a move commits. Null until then |
| `cancelledAt` | |
| `cancelledByOperatorId` | Set when staff cancel. Null when the booker cancels |
| `createdAt`, `updatedAt` | |

One confirmed booking per `bookerEmail` per slot. The same email may hold confirmed bookings on other slots, including other services.

---

## Weekly rules

`fillUpcomingTimes(now)`:

1. Load every weekly rule with `active` true whose service is active.
2. If `hostStaffIds` is empty, skip the rule.
3. Horizon end is `now + MATERIALIZE_DAYS`, cut earlier to the end of `endDate` in the service time zone when `endDate` is set.
4. Expected starts are each `weekday` at `startTime` in the service time zone, from `now` through the horizon end, excluding any day listed in `skipDates`.
5. For each expected start, look for an existing slot with this `ruleId` on that calendar day.
6. If none exists, create a slot: `endAt = start + durationMinutes`, capacity and hosts copied from the rule, `staffClosed` false.
7. If one exists, it is still in the future, and it has no confirmed booking, and its start or end no longer matches the rule, update `startAt` and `endAt`.
8. If it has a confirmed booking, leave it unchanged.
9. Do not create a second slot for the same rule on the same calendar day.

Count `created` and `repaired` for the return value.

Weekday and start time are wall-clock values. Convert them in the service time zone, including across daylight-saving changes, so 09:00 stays 09:00 local.

---

## Removing times

### One occurrence

If the slot has no confirmed booking, delete the slot. If `ruleId` is set, add that calendar day to the rule's `skipDates` so the next fill does not recreate it.

If it has confirmed bookings, refuse until the operator confirms a second step: cancel each confirmed booking (status, `cancelledAt`, emails) and then delete the slot. Add the `skipDates` entry when it came from a rule.

### This and following

Only for a slot that has a `ruleId`. Set the rule's `endDate` to the day before the selected slot, in the service time zone. Delete later slots on that rule that have no confirmed booking. Keep slots that have a confirmed booking, and tell the operator how many were kept. Those kept slots stay bookable until they are full, closed, or removed on their own.

### All upcoming on a rule

Set the rule `active` false and set `endDate` to yesterday in the service time zone. Delete future slots on that rule with no confirmed booking. Keep slots that have a confirmed booking, and report the count.

### Whole rule from the rule list

Same as **All upcoming on a rule**.

Past slots are left as history either way.

---

## Transitions

Do the booking write and the seat check in one transaction. Send email after the transaction commits. If email fails, keep the booking and surface the error so the operator can resend. Do not roll the booking back.

| Action | From | To | Guards |
| --- | --- | --- | --- |
| Create booking | Slot offered, form valid | `confirmed` | Service active. Slot offered (see [Time slot](#time-slot)). `acceptExtraAnswers` succeeds. No other `confirmed` row for this email on this slot. Capacity re-checked inside the transaction (`0` skips the count). Issue a new token, store the hash only |
| Cancel by booker | `confirmed`, start still in the future | `cancelled` | Token hash matches. Set `cancelledAt`. Leave `cancelledByOperatorId` null |
| Cancel by operator | `confirmed` | `cancelled` | `requireOperator()`. Allowed even after the start time. Set `cancelledAt` and `cancelledByOperatorId` |
| Move | `confirmed`, start still in the future | `confirmed` on the new slot | New slot is a different slot on the same service, and it is offered. Re-check capacity in the transaction, ignoring this booking's old seat. Set `rescheduledAt`. Keep `id` and the same token hash. Free the old seat in the same transaction |
| Resend booker email | any | unchanged | Token in the email is a new raw token only if the old one was never stored, which it was not. Generate a new raw token, replace `manageTokenHash`, and email that. The previous link stops working |
| Email hosts again | `confirmed` | unchanged | Sends the host-booked or host-moved message that matches whether `rescheduledAt` is set. Does not rotate the booker's token |

A cancel or a move that frees a seat makes the old time offered again, unless `staffClosed` is set or the end has passed.

Moving emails the booker once, and emails each distinct host address on the old time and the new time.

---

## Checks

A step finishes only when these are true.

**Publish a time**

- Service is active, length and time zone are set, and at least one host id resolves in `listStaff()`.
- Start is a real wall-clock time in that zone. End is after start.
- The day is not in `skipDates` when the slot comes from a rule.

**Accept a booking**

- The public range is within today through `PUBLIC_DAYS_MORE` in the service time zone.
- The slot is offered at submit time, not only when the page was rendered.
- Name and email pass the field rules. Email is stored lowercased.
- Extra answers passed `acceptExtraAnswers`.
- The email does not already have a confirmed booking on that slot.
- After insert, confirmed count is still within capacity, or capacity is `0`.
- The raw token appears in the booker email and on the success panel, and nowhere else. Logs, analytics, and host emails do not contain it.

**Cancel or move from the private page**

- Token hash matches a booking.
- Status is `confirmed`.
- Start is still in the future.
- The new slot, for a move, is offered and belongs to the same service.

**Rate limits**

Apply before the write. Suggested bounds, counted by the host app however it already limits public posts:

| Action | Bound |
| --- | --- |
| Create booking | 10 per email address per hour |
| Token lookup, cancel, or move | 30 per client address per 5 minutes |

Over the limit, refuse with a short "try again shortly" message and do not change rows.

---

## Emails

HTML. Escape every value that came from a person or from staff-written text. No tracking pixels. `disableLinkTracking` is true. The manage URL is included only in messages to the booker.

Times use the service time zone, written as a date line and a start–end line, with the zone's plain name (the IANA name, underscores replaced with spaces).

`{organisation}` is `ORGANISATION_NAME`. `{service}` is the service name.

### Booker booked

- To: booker email
- Subject: `You're booked — {service}`
- Body: greeting with their name. One sentence that `{service}` with `{organisation}` is confirmed. Date, time, and host display names. A button **Open your booking** to `/booking/{token}`, then the raw URL on its own line. One sentence that the link is private and is how they view, cancel, or move the booking.

### Booker cancelled

- To: booker email
- Subject: `Booking cancelled — {service}`
- Body: greeting, the date and time that was cancelled, and a link to `/book/{slug}` to choose a new time. The old manage link may stay in the message so they can see the cancelled record. State that the cancelled booking is not put back by opening it.

### Booker moved

- To: booker email
- Subject: `New time confirmed — {service}`
- Body: greeting, the new date and time, host display names on the new time, and the same **Open your booking** button. The token is unchanged, so the existing link still works.

### Host booked

- To: each host email on that time
- Subject: `New booking — {date line}`
- Body: `{service}` has a new booking. Date and time. Booker name, email, phone (or an em dash), note if present, and each extra answer as its field label plus the value. Link: `adminCalendarUrl()`. Do not include the manage token.

### Host cancelled

- To: each host email on that time
- Subject: `Booking cancelled — {date line}`
- Body: a booking was cancelled. Date and time it was. Booker name and email. Link: `adminCalendarUrl()`.

### Host moved

- To: each distinct host email on the previous time or the new time
- Subject: `Booking moved — {new date line}`
- Body: previous date and time, new date and time, booker name, email, phone, and note. Link: `adminCalendarUrl()`.

Send booker mail and host mail after the database commit. A failure of one recipient does not undo the booking and does not skip the other recipients. Show the operator which addresses failed when they are the one who took the action. When the visitor took the action, the success or confirm panel still tells them the outcome of the booking, and the on-screen manage link is the fallback if their own email failed.

---

## Operations

Operator operations require `requireOperator()`. Visitor and booker operations require no session. Booker operations require a token hash match.

| Operation | Who | Effect |
| --- | --- | --- |
| `listServices` | operator | Service list |
| `createService` / `updateService` | operator | Settings. Slug unique |
| `listRules(serviceId)` | operator | Weekly rules, including inactive |
| `createRule` / `updateRule` | operator | Then `fillUpcomingTimes` |
| `removeRuleUpcoming` | operator | [All upcoming on a rule](#all-upcoming-on-a-rule) |
| `listSlots(serviceId, includePast)` | operator | Concrete times with confirmed bookings and host names |
| `createOneOff` | operator | A slot with `ruleId` null |
| `setStaffClosed` | operator | |
| `setSlotHosts` | operator | That slot only |
| `removeSlot(slotId, scope)` | operator | `one`, `following`, or `upcoming`. `following` and `upcoming` require a `ruleId` |
| `listBookings(slotId)` | operator | Full booking fields, including phone, note, and extra answers |
| `cancelBooking` | operator | [Cancel by operator](#transitions) |
| `moveBooking` | operator | Same guards as a booker move, except the start-in-the-future guard is waived so staff can correct a mistake after the start |
| `resendBookerEmail` | operator | Rotates the token hash |
| `emailHostsAgain` | operator | |
| `listAvailability(slug, from, to)` | visitor | Offered and unavailable times in range. No booker names, emails, phones, notes, or tokens |
| `createBooking` | visitor | [Create booking](#transitions) |
| `getBooking(token)` | booker | The fields in [Private booking page](#5-private-booking-page). No other bookings, no host emails |
| `cancelBookingByToken` | booker | |
| `moveBookingByToken` | booker | |

`listAvailability` returns start, end, whether the button is enabled, places left when capacity is greater than 1, and host display names only when you need them for the booker email after submit. The public button itself does not list hosts.

---

## One way this was built

A community organisation ran a public page of short intro times. Operators added a weekly weekday-and-time rule, and a daily job expanded it into concrete times about a year ahead. Guests booked without an account. A private link on the confirmation email opened a page that could cancel or move that one booking. Hosts were existing staff, and they were emailed. This document is that shape, widened so one organisation can run several named services, and so a time can hold more than one booking. It does not depend on the stack that first ran it.

---

## Suggested module layout

Names are a guide. Use the host app's folders.

```text
calendar-booking/
  constants.ts           # the table at the top
  records.ts             # field limits and parsers
  offer.ts               # is this slot offered, places left
  fillUpcomingTimes.ts   # weekly expansion, skip dates, repair
  fillUpcomingTimes.test.ts
  transitions.ts         # create, cancel, move, token hash
  transitions.test.ts
  mailCopy.ts            # the six messages
  screens/
    ServiceList.tsx
    ServicePage.tsx      # settings, rules, one-off, upcoming times
    PublicCalendar.tsx   # /book/:slug
    BookingPage.tsx      # /booking/:token
  jobs/fillUpcomingTimes.ts
```

Keep `offer`, `fillUpcomingTimes`, and `transitions` free of UI imports so they can be unit tested.

---

## Build order

Implement in this order. Each step is usable before the next.

1. **Records and parsers.** Service, rule, slot, booking. Slug, email, capacity, time zone, token hash.
2. **Service list and settings.** No public page yet.
3. **One-off times and the operator list.** Close, change hosts, remove.
4. **Weekly rules and `fillUpcomingTimes`.** Skip dates. Repair of unbooked future slots. A daily call from the host scheduler.
5. **Public page and create booking.** Range cap, seat transaction, success panel, booker and host email.
6. **Private page.** View, cancel, move, same-service rule.
7. **Operator cancel, move, resend, and email hosts again.** Removal scopes that keep booked slots and report the count.

Do not start the public page before a one-off time can be stored with a host. Do not start the private page before create booking issues a token hash.

---

## Tests to write with the feature

- A weekly rule at 09:00 creates 09:00 local on both sides of a daylight-saving change.
- `fillUpcomingTimes` does not create a second slot on a day that already has one, and does not recreate a day in `skipDates`.
- Repair updates a future slot with no booking. It does not change a slot that has a confirmed booking.
- Capacity `1`: a second booking is refused. Capacity `3`: the third succeeds and the fourth is refused. Capacity `0`: a booking still succeeds when others exist.
- Two creates racing for the last seat: one commits, one is refused.
- The same email cannot hold two confirmed bookings on one slot, and can hold them on two slots.
- `acceptExtraAnswers` failure creates no booking.
- Public availability includes no emails, phones, notes, or tokens.
- Token lookup misses on the raw value if only the hash was stored. A wrong token and a rotated token both miss.
- Cancel from the private page frees the seat. A second cancel is refused. Cancel after the start time is refused for the booker and allowed for the operator.
- Move refuses a slot on another service, a full slot, and the same slot. It keeps the booking id and the token hash.
- Removing one occurrence of a rule adds `skipDates`. Removing upcoming keeps a slot that has a confirmed booking and deletes one that does not.
- Booker emails contain the manage path. Host emails do not.
- A failed `sendMail` leaves the booking `confirmed`.

---

## Acceptance pass

Use mailboxes you control. Use the name `Example Trust` until you switch the constants. Do not use a real member's details.

1. Create a service, 30 minutes, capacity 2, your time zone, with a one-sentence description.
2. Add a weekly rule for a weekday a few days out, with your own staff id as host. Confirm concrete times exist before the next day.
3. Open `/book/{slug}` in a private window. Book the time with a test name and mailbox. Confirm the success panel shows the private link, the booker email arrives, and the host email arrives without that link.
4. Book the same time again with a second test email. Confirm "1 place left" becomes unavailable after the second. A third address is refused.
5. From the first private link, move to another time. Confirm the first time has a free seat again, both parties got the moved email, and the original link still opens the booking.
6. Cancel from the private link. Confirm the page shows cancelled and links to the public page, and a new booking from that public page is a different record.
7. Remove the weekly rule's upcoming times while one future slot still has a confirmed booking. Confirm that slot remains and the empty ones are gone.
8. Close a time from the service page and confirm the public button says **Unavailable**.

---

## Their duties

The organisation that runs this tool decides why it collects names, emails, phones, notes, and extra answers, how long it keeps them, and who on staff can open the service page. This design does not set a retention period and does not provide a legal basis.

Extra questions are free text the host app defines. Do not use them to collect identity documents, payment card numbers, or passwords. There is no upload slot for that, on purpose.

The manage URL is a secret. Anyone who has it can view, cancel, or move that booking until the start time. Hash it at rest, keep it out of logs and analytics, and rotate it when staff resend the booker email.

Staff email addresses from `listStaff()` are operational contact details. Send booking messages only to hosts chosen on that time.
