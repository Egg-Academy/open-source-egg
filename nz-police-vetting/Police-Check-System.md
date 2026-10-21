# Police check system — replication spec

Standalone design for an NZ Police Vetting Service (NZPVS) police-check workflow. A coding assistant can implement this module on its own. It does not create accounts, assign product permissions, or depend on a particular login or email vendor.

The official artefact is the current **NZ Police Vetting Service Request & Consent Form** (AcroForm PDF), plus a manual submission on the NZ Police vetting portal. This app prepares that form, keeps the evidence trail, and records the result. It does not call a police API.

Configure these for the implementing organisation:

| Constant | Example | Used for |
| --- | --- | --- |
| `AGENCY_LEGAL_NAME` | `Example Trust` | PDF question 1.1 and emails |
| `AGENCY_PORTAL_LABEL` | `Example Trust (E00xxx)` | Reviewer instructions for the portal |
| `SITE_ORIGIN` | `https://app.example.org` | Person form links |
| `INVITE_TTL_DAYS` | `14` | Token lifetime |
| `CONSENT_VALID_MONTHS` | `3` | Signed consent expiry (NZ Police rule) |
| `IDENTITY_MAX_BYTES` | `8 * 1024 * 1024` | Upload cap |
| `IDENTITY_RETENTION_DAYS` | `14` | Purge unverified ID files |
| `IDENTITY_VIEW_URL_SECONDS` | `120` | Signed view links |
| `NZPVS_PORTAL_URL` | `https://services.police.govt.nz/vetting/` | Reviewer submission guide |

Replace the PDF template when NZ Police publish a new form. Keep the semantic field names in this document. Re-map them to whatever AcroForm names the new file uses.

This folder is the share bundle. The form PDF and the portal screenshots sit next to this file. See [Files included with this spec](#files-included-with-this-spec).

---

## Out of scope

Do not build any of the following inside this module:

- User signup, login, password reset, or linking a police-check person to a product account.
- Permission flags, role grants, or “block product access until cleared”.
- Marketing, CRM, or onboarding flows.
- A specific email provider, auth library, or object store. Use the three interfaces below.

A later product may read `PoliceCheck.status === CLEARED` and `vetExpiresAt`. That read is outside this module.

---

## Plug-in interfaces

Implement these three seams. Everything else in this document is the module.

### Operator session

Reviewer screens call one function your app already has:

```ts
requireOperator(): { operatorId: string }
```

`operatorId` is stored on audit fields (`createdByOperatorId`, `reviewedByOperatorId`, `verifiedByOperatorId`). The person being checked never logs in.

### Mailer

```ts
sendMail(args: {
  to: string;
  subject: string;
  html: string;
  disableLinkTracking: true;
}): Promise<void>
```

Turn link tracking off on the invite. The URL is a secret. Two messages only: **invite** and **cleared**. Copy is in [Emails](#emails).

### Private files

```ts
interface PrivateFiles {
  put(args: { key: string; bytes: Buffer; contentType: string }): Promise<void>;
  signedGet(key: string, ttlSeconds: number): Promise<string>;
  delete(key: string): Promise<void>;
}
```

Store consent PDFs, returned police PDFs, and temporary identity uploads here. Never serve them from a public URL. Identity view links expire in `IDENTITY_VIEW_URL_SECONDS`.

---

## Actors and screens

| Actor | How they get in | What they do |
| --- | --- | --- |
| **Reviewer** | `requireOperator()` | Directory, create person, run the case, sight ID, sign Section 1, download PDF, submit on the police portal, record the result |
| **Person** | One emailed link `/vetting/{token}` | Section 2, identity uploads, electronic signature. No account |

### 1. People directory

Route: reviewer only. Title: **Police checks**.

- Search by name or email.
- Button: **Add person**.
- Table columns: Name, Email, Role (job title), Police check status, Renewal, Checks (count), action to open the person.

Status cell rules, using cases ordered newest first:

1. If any case is in progress (status is not `CLEARED`, `WITHDRAWN`, or `EXPIRED`), show that case’s [list phase](#list-phase-derived-not-stored) and its display label.
2. Else if a `CLEARED` case exists, show **Completed** and the renewal date from `vetExpiresAt`.
3. Else if older terminal cases exist, show “No active check”.
4. Else show an em dash.

Display label for a case: admin `label` if set, else job title from Section 1, else `Police check · {created date}`.

Renewal badge from `vetExpiresAt` (or blank):

| Days until date | Label | Colour hint |
| --- | --- | --- |
| past | Overdue | red |
| 0–30 | Due soon | orange |
| 31–90 | Upcoming | yellow |
| later | Current | teal |

### 2. Add person

Fields: preferred first name, family name, email, optional legal given names (first and middle names as on ID), optional date of birth, optional internal notes. Email stored lowercased and trimmed.

This creates a **person** row only. It does not create a check and it does not create a login.

### 3. Person page — manage checks

Shows the person’s name and email, and every check newest first. Each row shows the list phase and the display label.

**Start new police check** opens setup. If a case is already in progress, warn and offer **Continue open check**. **Start another anyway** is allowed; parallel checks are deliberate, not the default.

### 4. Start-check setup

Required before the invite can be sent:

| Field | Maps to |
| --- | --- |
| Paid worker or volunteer | Section 1.6 `engagementType` |
| Job title | Section 1.11 |
| Role description | Section 1.3 — what they will actually do |
| Role location | Portal only, not printed on the PDF. Example: `Online remote` |
| Core or non-core children’s worker | Section 1.8 |

Core-worker guidance to show beside the control: tick **Core** when the person is the main responsible adult, has regular unsupervised contact (including online messaging), could be alone with children, or has overnight responsibility. Tick **Non-core** only when they always work under another adult in a supervised group and will not have private contact with children.

On create:

- Allocate the next integer `agencyReference` (`max(agencyReference) + 1`, unique).
- Seed `agencyData` and `subjectData` (see defaults below).
- Status `NOT_STARTED`.
- Reviewer may send the invite immediately if setup is complete.

### 5. Case workspace

One check, reviewer only. Show a progress list, then the work for the current step.

Progress milestones, in order. The first incomplete one is **next**; later ones are **todo**; finished ones are **done**.

1. Person submitted Section 2 (`consentSignedAt` set).
2. Identity confirmed and uploads removed (`identityVerifiedAt` set).
3. Agency representative sign-off saved (`agencyRepElectronicSignature`).
4. Consent PDF complete (see [completeness](#when-the-consent-pdf-is-complete)).
5. Submitted to NZ Police (status is `SUBMITTED_TO_POLICE`, `RESULT_RECEIVED`, or `CLEARED`).
6. All clear recorded (status `CLEARED`).

If status is `WITHDRAWN` or `EXPIRED`, show only that single done step.

Also show a review badge while `consentSignedAt` is set and `adminAcknowledgedFacilitatorAt` is null or older than `consentSignedAt`. Opening the person’s submission sets `adminAcknowledgedFacilitatorAt` to now. That badge is attention, not a status change.

### 6. Person form

Route: `GET /vetting/{token}` with no login.

Invalid, expired, or already-submitted tokens show a short message asking them to contact the agency. A valid token shows:

1. Agency name, job title, and role description (read-only, from Section 1).
2. Identity: given names, family name, gender, date of birth, country of birth, place of birth, NZ driver licence (optional).
3. Address: street (required), suburb, post code, town/city (required).
4. Up to three previous names (family, first, middle). Include this; the PDF has three rows.
5. Identity document uploads (primary, secondary, optional name-change evidence).
6. Consent: typed full name, checkbox “this constitutes my electronic signature”, then submit.

If the person is 14 or 15, also collect parent/guardian full name, signature text, date, and electronic-signature checkbox. Those PDF fields exist for that age band.

Prefill given names from `legalGivenNames` when set, otherwise preferred first name. Prefill family name and date of birth from the person row. Country of birth defaults to `New Zealand`.

Dates on the form and in the PDF are `dd/mm/yyyy`. HTML date inputs convert to that format before save.

Mobile: inputs at 16px so iOS does not zoom. On a coarse pointer with no hover, offer a camera capture control as well as a file picker.

Submit is one-shot. The server sets `subjectAuthDate` to today. The client cannot choose that date.

### 7. Submit-to-police guide

A reviewer modal, not an API integration. Steps:

1. Open `NZPVS_PORTAL_URL`, sign in with RealMe, open `AGENCY_PORTAL_LABEL`, choose **Submit Request**. Image: `guide/Step 1.png`.
2. Role category: usually **Teaching** or **Caregiving**. If unsure, **Teaching**. Image: `guide/Step 2.png`.
3. Role in the home of the person being vetted: usually **No**. Image: `guide/Step 3.png`.
4. Paid or volunteer: follow `engagementType`. Show `guide/Step 4 - Paid.png` when `PAID_WORKER`, otherwise `guide/Step 4 - Volunteer.png`.
5. Copy each portal field (below) into the matching box. Download the consent PDF and upload it on the portal. Leave **batch number** blank. Image: `guide/Step 5.png`.

Then the reviewer confirms **I’ve submitted this to police**. Tell them NZ Police email a result later (often one to three months) and they must come back to record it.

### 8. Result

Reviewer uploads the PDF NZ Police returned, then records **all clear** with the renewal date printed on that result (`vetExpiresAt`). Send the cleared email.

A new check is a new case. Do not edit a cleared case into a renewal.

---

## End-to-end flow

```text
Add person
  → Start check (role, title, location, paid/volunteer, core/non-core)
  → Email 14-day link
  → Person completes Section 2, uploads ID, e-signs
  → Reviewer sights ID and deletes the files
  → Reviewer completes Section 1 and e-signs
  → System generates the consent PDF and marks the case ready
  → Reviewer copies fields into the NZ Police portal and uploads the PDF
  → Reviewer marks submitted
  → NZ Police return a result PDF (email, outside this system)
  → Reviewer uploads that PDF and records all-clear plus expiry date
  → Person receives a cleared email
```

Consent on a signed form is valid for three months (`consentExpiresAt = signedAt + 3 months`). If that date passes before police submission, start a new check rather than reusing the signature.

---

## Case status

Stored on the case. List phase is separate and derived.

| Status | Meaning |
| --- | --- |
| `NOT_STARTED` | Case created, invite not sent |
| `INVITE_SENT` | A live token exists |
| `FACILITATOR_SUBMITTED` | Person has e-signed Section 2 |
| `ADMIN_REVIEW` | Reviewer is editing Section 1 after the person submitted, or after a ready PDF was invalidated |
| `READY_FOR_POLICE` | Completeness passed and the consent PDF is stored |
| `SUBMITTED_TO_POLICE` | Reviewer confirmed portal submission |
| `RESULT_RECEIVED` | Returned police PDF stored |
| `CLEARED` | Reviewer recorded all-clear and an expiry date |
| `WITHDRAWN` | Reviewer cancelled the check |
| `EXPIRED` | Terminal. Reserved for a check closed because consent or clearance lapsed. Do not auto-flip `CLEARED` to this; show overdue from the date instead |

In progress means status is not `CLEARED`, `WITHDRAWN`, or `EXPIRED`.

### Transitions

| Action | From | To | Guards |
| --- | --- | --- | --- |
| Send or resend invite | before police submission | `INVITE_SENT` | Setup complete if this is the first send. Issuing a token does not wipe a submission already stored |
| Person submits Section 2 | token valid and unused | `FACILITATOR_SUBMITTED` | Electronic signature checkbox true. Set `consentSignedAt`, `consentExpiresAt`, mark that invite `completedAt` |
| Save Section 1 | not yet submitted to police | `ADMIN_REVIEW` if current status is `INVITE_SENT` or `FACILITATOR_SUBMITTED`; if current is `READY_FOR_POLICE` and agency inputs changed, drop back to `ADMIN_REVIEW` and clear `generatedPdfKey` | Block edits when status is `SUBMITTED_TO_POLICE`, `RESULT_RECEIVED`, or `CLEARED` |
| Verify identity | any pre-police status | status unchanged | See identity rules. Sets `identityVerifiedAt` and the Section 1.12 ticks |
| Mark ready | pre-police | `READY_FOR_POLICE` | Completeness passes. Generate and store the PDF in the same action |
| Mark submitted | typically `READY_FOR_POLICE` | `SUBMITTED_TO_POLICE` | `agencyReference` is set. Set `submittedToPoliceAt` |
| Upload result PDF | usually after submission | `RESULT_RECEIVED` | Store file, set `resultReceivedAt` |
| Mark cleared | result in hand | `CLEARED` | Reviewer supplies `vetExpiresAt`. Set `clearedAt`. Send cleared email |
| Undo submission | `SUBMITTED_TO_POLICE` only | `READY_FOR_POLICE` | Clear `submittedToPoliceAt`. Append an admin note with optional reason |
| Undo clearance | `CLEARED` only | `RESULT_RECEIVED` if a result PDF exists, otherwise `SUBMITTED_TO_POLICE` | Clear `clearedAt` and `vetExpiresAt`. Do not try to recall the email. Append a note |
| Undo agency sign-off | before police submission | if status was `READY_FOR_POLICE`, set `ADMIN_REVIEW` | Clear electronic signature, typed signature, and date. Clear `generatedPdfKey` |
| Undo identity verification | before you need the files again | status unchanged | Clear `identityVerifiedAt` and the five ID ticks. Files are already gone and are not restored |
| Withdraw | any | `WITHDRAWN` | |
| Correct legal names | after Section 2, before police submission | status unchanged | Update `givenNames`, `familyName`, `subjectAuthFullName`, typed signature if they e-signed, and Section 1.2 full name. Clear `generatedPdfKey` |

After police submission, Section 1 and legal names are locked.

If identity is already verified, later Section 1 saves must keep `primaryIdSighted`, `secondaryIdSighted`, and `photographicIdSighted` true. Do not let a form save untick them.

### List phase (derived, not stored)

Use this on the directory and the manage-checks list.

| Phase | Label | When |
| --- | --- | --- |
| `awaiting_helper` | Awaiting person’s response | Not cleared, not sitting with police, PDF not complete, and `consentSignedAt` is empty |
| `awaiting_staff_signoff` | Awaiting staff sign off | `consentSignedAt` set, PDF not complete, not yet with police |
| `ready_for_police` | Ready to submit to police | Status is `READY_FOR_POLICE`, or completeness already passes |
| `awaiting_police_confirmation` | Awaiting police confirmation | `SUBMITTED_TO_POLICE` or `RESULT_RECEIVED` |
| `completed` | Completed | `CLEARED` |

Evaluate in that priority order: cleared, then with police, then PDF complete, then consent signed, else awaiting the person.

---

## Data model

Four tables. JSON columns hold the form sections so a new PDF question does not need a migration for every tick box.

### `PoliceCheckPerson`

The registry. Independent of product users.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | string pk | |
| `createdAt`, `updatedAt` | datetime | |
| `firstName` | string(120) | Preferred name |
| `legalGivenNames` | string(120) null | First and middle names as on ID. Prefill source |
| `lastName` | string(120) | |
| `email` | string(320) | Lowercased |
| `dateOfBirth` | date null | Prefill only |
| `notes` | text null | Internal |
| `checks` | relation | |

Index `email`, and `(lastName, firstName)`.

### `PoliceCheck`

One lifecycle. A person has many, including history.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | string pk | |
| `personId` | fk | cascade delete |
| `status` | enum | default `NOT_STARTED` |
| `label` | string(200) null | Reviewer label, e.g. `2026 renewal` |
| `agencyReference` | int unique null | Portal “agency reference”. Allocated at create |
| `roleLocation` | string(200) null | Portal only |
| `agencyData` | json null | Section 1 |
| `subjectData` | json null | Section 2 |
| `consentSignedAt` | datetime null | |
| `consentExpiresAt` | datetime null | signed + 3 months |
| `identityVerifiedAt` | datetime null | |
| `adminAcknowledgedFacilitatorAt` | datetime null | Clears the review badge |
| `generatedPdfKey` | string(512) null | Private consent PDF |
| `submittedToPoliceAt` | datetime null | |
| `nzpvsReference` | string(128) null | Optional, if the portal shows one. Not required to advance |
| `resultPdfKey` | string(512) null | |
| `resultReceivedAt` | datetime null | |
| `clearedAt` | datetime null | |
| `vetExpiresAt` | datetime null | Renewal date from the police result |
| `adminNotes` | text null | Append-only lines for undos |
| `createdByOperatorId` | string null | |
| `reviewedByOperatorId` | string null | |

Index `(personId, createdAt)` and `(status, updatedAt)`.

### `PoliceCheckInvite`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | string pk | |
| `checkId` | fk | |
| `tokenHash` | string(64) unique | SHA-256 of `token + ":" + serverSecret`. Never store the raw token |
| `expiresAt` | datetime | now + 14 days |
| `emailedAt` | datetime null | |
| `completedAt` | datetime null | Set on successful Section 2 submit |

Raw token: 32 random bytes, hex (64 chars). URL: `{SITE_ORIGIN}/vetting/{token}`.

Resolve by hashing the URL token and loading the invite. Reject when missing, when `completedAt` is set, or when `expiresAt` is in the past. Resending creates a new invite row. Old unused tokens can stay until they expire; submission completes the invite that was used.

### `PoliceCheckIdentityDocument`

Temporary. Deleted after the reviewer verifies, or by the retention job.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | string pk | |
| `checkId` | fk | |
| `documentType` | `PRIMARY` \| `SECONDARY` \| `NAME_CHANGE` | |
| `fileKey` | string(512) | Private store |
| `mimeType` | string(120) | `image/jpeg`, `image/png`, `image/webp`, `application/pdf` |
| `originalFileName` | string(260) null | |
| `uploadedAt` | datetime | |
| `verifiedAt` | datetime null | |
| `verifiedByOperatorId` | string null | |
| `deletedAt` | datetime null | Soft-delete after the object is removed |

One active file per type per check. A replacement deletes the previous object and sets `deletedAt` on the old row.

Index `(checkId, documentType)` and `(deletedAt, uploadedAt)`.

---

## Form format

Two JSON objects. Dates are strings `dd/mm/yyyy`. Booleans are real booleans. Empty optional strings are `""`, not null.

Electronic signature on this form is **typed full name + date + a checkbox**. NZ Police accept that. Do not draw on the PDF. A drawn signature image may be kept inside `agencyData` for your own audit (`agencyRepSignatureImageDataUrl`) and must not be written into the PDF.

Mutually exclusive pairs are two booleans in the PDF and one value in JSON. When filling the PDF, tick exactly one side.

### Section 1 — `agencyData` (reviewer)

| JSON field | PDF question | Required for a police-ready PDF | Notes |
| --- | --- | --- | --- |
| `agencyName` | 1.1 Name of agency | yes | Always `AGENCY_LEGAL_NAME` |
| `vettingSubjectFullName` | 1.2 Person being vetted | yes | Legal given names + family name |
| `roleDescription` | 1.3 Description of the role | yes | |
| `worksWithChildren` | 1.4 Children / young people | stored | Default `true` for this kind of agency |
| `worksWithVulnerableAdults` | 1.4 Vulnerable adults | stored | Default `false` |
| `roleInvolvesHomeCare` | 1.5 Role involves visiting their home | stored | Default `false`. PDF is a yes/no radio |
| `engagementType` | 1.6 | yes | `PAID_WORKER`, `VOLUNTEER`, or `VOCATIONAL_TRAINING`. PDF is three checkboxes; tick one |
| `isChildrensWorker` | 1.7 Children’s worker | yes | Default `true`. When `false`, leave 1.8 blank |
| `isCoreWorker` | 1.8 Core vs non-core | yes when 1.7 is not false | `true` core, `false` non-core. Missing is incomplete |
| `previouslyVettedByAgency` | 1.9 Previously vetted by this agency | yes | Default `false` |
| `stillInSameRole` | 1.10 Same role as last vet | required only when 1.9 is true | `true` same role, `false` new role |
| `jobTitle` | 1.11 Job title | yes | |
| `primaryIdSighted` | 1.12 Primary ID | yes, unless identity already verified | |
| `secondaryIdSighted` | 1.12 Secondary ID | yes, unless identity already verified | |
| `photographicIdSighted` | 1.12 Photographic ID | yes at verify time | At least one ID shows a photo |
| `nameChangeEvidenceSighted` | 1.12 Name-change evidence | no | Only when IDs do not match the vetting name |
| `realMeIdentityReceived` | 1.12 RealMe assertion | no | Only if the agency accepts RealMe instead of manual sighting |
| `agencyRepFullName` | Representative full name | yes | |
| `agencyRepSignature` | Representative signature text | optional when e-sign is ticked | PDF accepts signature **or** e-sign |
| `agencyRepDate` | Representative date | yes | `dd/mm/yyyy` |
| `agencyRepElectronicSignature` | Representative e-sign checkbox | yes | This is the sign-off milestone |

Seed on create:

```json
{
  "agencyName": "<AGENCY_LEGAL_NAME>",
  "vettingSubjectFullName": "<legal given names> <family name>",
  "roleDescription": "<from setup>",
  "worksWithChildren": true,
  "worksWithVulnerableAdults": false,
  "roleInvolvesHomeCare": false,
  "engagementType": "VOLUNTEER",
  "isChildrensWorker": true,
  "isCoreWorker": true,
  "previouslyVettedByAgency": false,
  "jobTitle": "<from setup>",
  "primaryIdSighted": false,
  "secondaryIdSighted": false,
  "photographicIdSighted": false,
  "nameChangeEvidenceSighted": false,
  "realMeIdentityReceived": false,
  "agencyRepFullName": "",
  "agencyRepSignature": "",
  "agencyRepDate": "",
  "agencyRepElectronicSignature": false
}
```

Identity evidence help text for the reviewer:

- **Primary** — passport, NZ driver licence, or another primary document in the legal name.
- **Secondary** — a different category, such as a bank card, IRD letter, or utility bill.
- **Photographic** — at least one document shows the person’s photo.
- **Name change** — marriage certificate, deed poll, or similar, only when names differ.
- **RealMe** — only when the agency accepts a verified RealMe assertion.

### Section 2 — `subjectData` (person)

| JSON field | PDF question | Required to submit |
| --- | --- | --- |
| `givenNames` | 2.1 Given / first and middle names | yes |
| `familyName` | 2.1 Family name | yes |
| `gender` | 2.1 Gender | yes |
| `dateOfBirth` | 2.1 Date of birth | yes, `dd/mm/yyyy`, not in the future |
| `placeOfBirth` | 2.1 Town/city/state | no |
| `countryOfBirth` | 2.1 Country of birth | yes |
| `nzDriverLicenceNumber` | 2.1 NZ driver licence | no |
| `parentGuardianNameIfMinor` | 2.1 Parent/guardian if aged 14–15 | when the person is 14 or 15 |
| `otherNames` | 2.2 Up to 3 rows of `{ familyName, firstName, middleNames }` | no |
| `streetAddress` | 2.3 Number and street | yes |
| `suburb` | 2.3 Suburb | no |
| `postCode` | 2.3 Post code | no |
| `townCity` | 2.3 Town / city | yes |
| `subjectAuthFullName` | 2.4 Full name | yes |
| `subjectAuthSignature` | 2.4 Signature text | optional when e-sign is ticked |
| `subjectAuthDate` | 2.4 Date | set by the server on submit |
| `subjectElectronicSignature` | 2.4 E-sign checkbox | yes, must be true |
| `parentGuardianAuthFullName` | Parent/guardian name | when 14–15 |
| `parentGuardianAuthSignature` | Parent/guardian signature text | when 14–15 |
| `parentGuardianAuthDate` | Parent/guardian date | when 14–15 |
| `parentGuardianElectronicSignature` | Parent/guardian e-sign | when 14–15 |

Consent statement shown above the signature:

> I confirm that the information provided is about {full name} and is correct. I authorise New Zealand Police to disclose relevant information to {agency name} for vetting purposes.

### When the consent PDF is complete

Collect every gap and return them together. Ready means the list is empty.

Section 1 gaps:

- Person being vetted (full name)
- Role description
- Job title
- Core or non-core, when `isChildrensWorker` is not `false` and `isCoreWorker` is missing
- Still in same role, when previously vetted and `stillInSameRole` is missing
- Representative full name
- Representative date
- Representative electronic signature

Section 2 gaps:

- If `subjectData` fails the schema: “facilitator submission is incomplete”, or “must submit the consent form” when the e-sign checkbox is off
- E-sign checkbox false

Identity gaps:

- If `identityVerifiedAt` is empty and primary or secondary is not sighted: “primary and secondary ID sighted”
- If `identityVerifiedAt` is empty and any identity row has `deletedAt` null: “verify or delete pending files before police submission”

`markReady` runs this check, refuses on any gap, writes the PDF to private storage, saves `generatedPdfKey`, then sets `READY_FOR_POLICE`.

A separate **preview** download may render an incomplete PDF for the reviewer. Label that button incomplete. Do not upload an incomplete PDF to NZ Police. Only a complete render is stored as `generatedPdfKey`.

---

## Identity uploads

Person, while their invite is still open:

- Types: `PRIMARY`, `SECONDARY`, optional `NAME_CHANGE`.
- MIME: JPEG, PNG, WebP, PDF. Max 8 MB. Reject near-empty files.
- Replacing a type deletes the previous object.

Reviewer:

- List active files (no file bytes in the list payload).
- **View** returns a signed URL for that one document, scoped to the case.
- **Verify identity and delete uploads**:
  - Require the reviewer to confirm primary sighted, secondary sighted, and photographic sighted.
  - Name-change and RealMe ticks are optional and stored as given.
  - If primary and secondary files are not both present, require an explicit override. The UI states what they are attesting:
    - No files: they sighted the evidence in person or another way outside the product.
    - Only one of the two types: they sighted both categories from that file and/or in person.
  - Then delete every active object, set `verifiedAt`, `verifiedByOperatorId`, and `deletedAt`, set `identityVerifiedAt`, and write the five ticks into `agencyData`.

Retention job, authenticated with a cron secret:

- Find identity rows with `deletedAt` null, `verifiedAt` null, `uploadedAt` older than 14 days.
- Delete the object and set `deletedAt`.
- Do not delete consent PDFs or result PDFs in this job.

The person can still view their own active uploads through the token until they submit or the reviewer verifies. After verify, the token page shows that identity has been confirmed, with no files left.

---

## Files included with this spec

These files are in this same folder. Put the PDF in the app as a static template. Put the screenshots on the submission-guide modal, one image under each step. Serve them as static files. Do not redraw the form.

| File | Use |
| --- | --- |
| `form/nz-police-vet-request-consent-form.pdf` | Official NZPVS Request & Consent Form. AcroForm. This is the file the app fills. |
| `guide/Step 1.png` | Portal sign-in, open the agency, choose Submit Request |
| `guide/Step 2.png` | Role category screen |
| `guide/Step 3.png` | Role in the home of the person being vetted |
| `guide/Step 4 - Volunteer.png` | Paid-or-volunteer screen with volunteer selected |
| `guide/Step 4 - Paid.png` | Paid-or-volunteer screen with paid worker selected |
| `guide/Step 5.png` | Applicant-details screen the copy-fields map onto |

The screenshots were taken in the Egg Academy Limited portal session. They show which screen is which. The implementing organisation should retake them under their own agency name before go-live, and replace `AGENCY_PORTAL_LABEL` in the step 1 caption.

Nothing else is required. There is no logo, font, or email template file. Email HTML is inline in the spec.

---

## PDF generation

1. Ship `form/nz-police-vet-request-consent-form.pdf` from this folder as the template in the repo.
2. Inspect its AcroForm fields. The official file uses auto-generated names from a Word export (`undefined`, `Text1`, and similar). Those names are an adapter detail.
3. Keep a map from **semantic key** → **template field name**, with metadata: section, form question, page, field type (`text`, `checkbox`, `radio`).
4. Product code fills semantic keys only.
5. Checkbox and radio values must match what the template expects (typically `Yes` / `Off`, confirmed by reading the template).
6. Expand `otherNames[0..2]` into the nine PDF name fields.
7. Expand `engagementType` into the three 1.6 checkboxes, and the yes/no pairs for 1.7, 1.8, 1.9, and 1.10.
8. When `isChildrensWorker` is false, leave 1.8 unticked.
9. A test loads the template, fills a volunteer sample and a paid sample, reads the AcroForm back, and asserts every fillable field has a semantic mapping.

Build a small **PDF lab** for reviewers: load the volunteer sample and the paid sample, edit fields, generate, show the PDF, and show a table of semantic key → value. Use it when the template changes. It is a tool, not a second workflow.

Suggested samples:

- Volunteer: engagement volunteer, job title `Volunteer Session Facilitator`, a one-paragraph role description, fictional person `Alex Example`.
- Paid: engagement paid worker, job title `Session Facilitator`, fictional person `Sam Staff`.

Both samples include a signed agency rep and a signed subject so the lab PDF looks finished.

---

## Police portal copy list

Built from the case for the submission guide. Each row has a label, a value, and a copy button. Empty values show “not set yet”.

| Order | Label | Source |
| --- | --- | --- |
| 0, only if set | Job title (if asked) | `agencyData.jobTitle` |
| 1 | Role location | `roleLocation` |
| 2 | Agency reference | `agencyReference` as a decimal string |
| 3 | Batch number | Always blank. Hint: leave blank on the NZ Police portal |
| 4 | Date consent signed | `subjectData.subjectAuthDate`, else `consentSignedAt` formatted `dd/mm/yyyy` |
| 5 | Family name | `subjectData.familyName` |
| 6 | First / middle name(s) | `subjectData.givenNames` |
| 7 | Gender | `subjectData.gender` |
| 8 | Date of birth | `subjectData.dateOfBirth` |
| 9 | Place of birth (Town/City/State) | `subjectData.placeOfBirth` |
| 10 | Country of birth | `subjectData.countryOfBirth` |
| 11 | NZ driver licence number | `subjectData.nzDriverLicenceNumber` |
| 12 | Number / street | `subjectData.streetAddress` |
| 13 | Suburb | `subjectData.suburb` |
| 14 | Post code | `subjectData.postCode` |
| 15 | Town / city | `subjectData.townCity` |

If `roleLocation` is empty, show a warning on the guide before they submit.

The consent PDF download sits under this list. Button label includes whether completeness passed.

---

## Emails

HTML, escaped names, no tracking pixels on the invite.

### Invite

- To: person email
- Subject: `Police vetting form — required for {agency name}`
- Body: greeting with preferred first name, one sentence that the agency needs the NZ Police vetting consent form and it takes about 10 minutes, a button **Complete vetting form** to the token URL, the raw URL as a fallback, and “this link is for your check only and expires in 14 days”.

### Cleared

- To: person email
- Subject: `{first name}, your police check is cleared`
- Body: the check with the agency is complete and they are cleared. No further action for this check.

Send the cleared email after the status write commits. If mail fails, keep `CLEARED` and surface the send error to the reviewer so they can resend. Do not roll the status back.

---

## Operations checklist

Reviewer operations require `requireOperator()`. Person operations require a valid invite token and nothing else.

| Operation | Who | Effect |
| --- | --- | --- |
| `listPeople(search)` | reviewer | Directory rows with cases enough to compute phase, label, and expiry |
| `createPerson` / `updatePerson` | reviewer | Registry only |
| `createCheck(personId, setup, sendInvite)` | reviewer | Allocate reference, seed JSON, optional invite |
| `updateLabel` / `updateRoleLocation` | reviewer | |
| `getCheck` | reviewer | Case, person, last 5 invites, active identity files, creator and reviewer names |
| `saveAgencySection` | reviewer | Locked after police submission. Identity ticks stick once verified |
| `acknowledgeSubmission` | reviewer | Sets `adminAcknowledgedFacilitatorAt` |
| `sendInvite` | reviewer | New token row, email, status `INVITE_SENT` |
| `verifyIdentity` | reviewer | Attest, delete files |
| `revertIdentity` | reviewer | Clears ticks and `identityVerifiedAt` |
| `revertAgencySignOff` | reviewer | Before police submission only |
| `correctLegalNames` | reviewer | After submit, before police |
| `previewPdf` | reviewer | Returns bytes plus `{ complete, missing }`. Stores only when complete |
| `markReady` | reviewer | Complete PDF stored, status `READY_FOR_POLICE` |
| `signedPdfUrl(kind)` | reviewer | `generated` or `result` |
| `markSubmitted` | reviewer | |
| `revertSubmitted` | reviewer | |
| `uploadResultPdf` | reviewer | |
| `markCleared(vetExpiresAt)` | reviewer | Email |
| `revertCleared` | reviewer | |
| `withdraw` | reviewer | |
| `getByToken` | person | Safe fields only: names, agency, role, subject draft, upload flags, whether identity is already verified. Do not return agency sign-off, notes, or file keys |
| `uploadIdentityByToken` | person | Reject if that invite is completed |
| `viewIdentityByToken` | person | Document must belong to that case and be active |
| `submitSubjectByToken` | person | Transaction: save subject, status, consent dates, invite `completedAt` |

Admin notes for undos are appended, not replaced:

```text
[2026-10-21T02:14:00.000Z] Reverted submitted-to-police status: portal validation failed
```

---

## Suggested module layout

```text
police-check/
  schema.ts              # Section 1 and Section 2 validators, date helpers
  completeness.ts        # missing-field list
  workflow.ts            # milestones + list phase
  portalFields.ts        # copy rows for the police site
  pdfFieldMap.ts         # semantic key → current template field name
  fillPdf.ts             # template + map → bytes
  fillPdf.test.ts
  invites.ts             # random token, hash, expiry, consent +3 months
  identity.ts            # upload, verify-and-delete, retention purge
  mailCopy.ts            # invite and cleared HTML
  service.ts             # transitions above
  screens/
    Directory.tsx
    AddPerson.tsx
    StartCheck.tsx
    CaseWorkspace.tsx
    PersonForm.tsx       # /vetting/:token
    SubmissionGuide.tsx
    PdfLab.tsx
  cron/purge-identity.ts
  template/nz-police-vet-request-consent-form.pdf
  guide/Step 1.png
  guide/Step 2.png
  guide/Step 3.png
  guide/Step 4 - Volunteer.png
  guide/Step 4 - Paid.png
  guide/Step 5.png
```

Keep validators, completeness, phase, and PDF fill free of database and UI imports so they can be unit tested.

---

## Build order

Implement in this order. Each step is usable before the next.

1. **Schema and validators.** Person, check, invite, identity tables. Section 1 and Section 2 parsers. Date helpers (`dd/mm/yyyy` ↔ HTML date).
2. **Directory and add person.** No checks yet.
3. **Create check** with agency reference, seeded JSON, and the setup form.
4. **Invites and the public form.** Hash the token, 14-day expiry, one-shot submit, consent expiry.
5. **Identity upload, reviewer view, verify-and-delete, 14-day purge.**
6. **Section 1 editor, sign-off, completeness list, progress milestones.**
7. **PDF fill against the official template, lab, preview, mark ready.**
8. **Submission guide with copy rows, mark submitted, undo.**
9. **Result upload, mark cleared, cleared email, undo clearance, renewal date on the directory.**
10. **Withdraw, legal-name correction, sign-off undo, identity undo.**

Do not start step 8 before a stored PDF can be downloaded. Do not start step 9 before submitted status exists.

---

## Tests to write with the feature

- Volunteer and paid samples round-trip through the PDF. Every fillable template field is in the map.
- Completeness: missing core-worker answer, missing e-sign, pending identity file, verified identity with no files.
- List phase for each branch in the table above.
- Invite: wrong token, expired token, second submit, hash is not the raw token.
- Verify identity deletes objects and refuses when primary and secondary files are missing unless override is set.
- `markReady` refuses when completeness fails and does not leave a stored PDF.
- Saving Section 1 after `READY_FOR_POLICE` drops status to `ADMIN_REVIEW` and clears `generatedPdfKey`.
- Section 1 save after `SUBMITTED_TO_POLICE` is rejected.
- Undo submission only from `SUBMITTED_TO_POLICE`. Undo clearance restores `RESULT_RECEIVED` when a result file exists.
- Agency reference increments and is unique.

---

## Reviewer acceptance pass

1. Add a person with a real mailbox you control.
2. Start a check with role, title, location, volunteer, core worker. Send the invite.
3. Open the link on a phone. Submit Section 2 with primary and secondary ID photos.
4. As reviewer, view each file, verify, and confirm the files are gone from storage.
5. Sign Section 1. Mark ready. Download the PDF and check 1.1–1.12 and 2.1–2.4 against the on-screen data.
6. Open the submission guide, copy the rows, confirm they match. Mark submitted only after you would actually lodge it.
7. Upload a stand-in result PDF, mark cleared with an expiry date, and confirm the cleared email arrived and the directory shows **Completed** plus the renewal badge.
8. Start a second check for the same person and confirm the first cleared row remains history.
