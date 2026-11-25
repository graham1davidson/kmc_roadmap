# KMC Punch List — spec v0.2

For Graham's sign-off · 2026-11-25 · KMC brand only

## What it does

A PM or super walks a job with an iPhone and photographs each deficiency. Each
photo becomes one punch item. AI drafts the fields; the PM confirms or corrects
them. Items go to a vendor, the vendor fixes them, KM staff photograph the fix,
and the PM closes the item. Filter to one vendor or trade and send them a PDF.

Simplicity is the rule: one photo, one item, a few taps to confirm.

## Decisions so far (from the 10 questions)

| Topic | Decision |
|---|---|
| Vendors and trades | Read from the KM Trade Desk database (`kmc-desk`). No second vendor list. |
| Organization | By project, with a location. Floor plans can be uploaded; room names are read off them. |
| Item ID | Short job code plus item number: `SMITH-045` |
| Statuses | Open → Assigned → Ready for Review → Closed |
| Closing | Anyone can add the completed photo; only the project's PM (or an admin) closes |
| Vendors | Receive the PDF only. No vendor login. |
| Sending | Email from the app, or download the PDF |
| Devices | iPhone (works with or without signal) and desktop |
| Sign-in and hosting | Same as Trade Desk: Cloudflare Access, a Cloudflare Worker, the same roster |
| AI | Suggests trade, description, priority, location and vendor. The PM finalizes. |
| Structure | A separate app now; combined with the other KMC apps later |
| Priority | Low · Medium · High. No due dates in v1. |
| Report recipient | PM picks a Trade Desk contact; defaults to the vendor's main email |
| Code home | A new GitHub repo (e.g. `kmc_punch_list`) |

## How it works

**1. Capture.** Pick the project, then take or choose photos (several at once).
Each photo is one new item. With no signal, photos queue on the phone and upload
when signal returns.

**2. AI draft.** For each uploaded photo the server asks Claude for:

- trade (chosen from Trade Desk's trade list, e.g. Electrical, Plumbing, Trim/Millwork)
- a one-to-two sentence description of the deficiency
- priority
- location, only when the photo shows it (a unit number, a room's fixtures);
  otherwise left blank for the PM
- vendor: the vendor on this project in that trade, from Trade Desk's job
  history. If none, the best-rated active vendor in that trade. Never a vendor
  flagged Do Not Use.

Drafts land in a **To review** stack. They are not on the punch list and never
appear on a report until the PM confirms them.

**3. PM review.** One card per draft: the photo, the suggested fields (marked as
suggestions), and Confirm. Every field is editable. Extra photos can be added to
explain the item. Confirming gives the item its ID and puts it on the list as
Open, or Assigned if a vendor is set.

**4. Work the list.** Filter by project, trade, vendor, status, priority and
location. Sort by any of them, or by age. Filters are remembered per person.

**5. Report.** With the list filtered (say, one vendor, open items), tap
**Report**. A branded PDF is built: project, vendor, date, and per item the ID,
location, description, priority, photos and a plan pin if there is one. Then
**Download**, or **Email** it to a vendor contact the PM picks from Trade Desk
(the vendor's main email by default), with the PM copied and replying to the PM. Every report sent is saved, with who sent it, to
whom and which items.

**6. Close.** Anyone adds a completed photo, which moves the item to Ready for
Review. The PM sees the before and after side by side and either closes the item
or sends it back to Assigned with a note. **No item can close without a
completed photo.** The database enforces this, not only the screen.

## Floor plans

Upload a plan (PDF or image) per project. AI reads the room names and numbers off
it, and the PM checks the list once. After that, location is a pick list. The PM
can also tap the plan to drop a pin, and the pin appears on the vendor report.

Honest limit: from a close-up photo alone, AI usually can't tell which room it
was taken in, because a cracked outlet cover looks the same everywhere. So AI
only fills location when the photo gives it away. Otherwise the PM picks it.

## Rules carried over from Trade Desk

- Nothing goes to the team or to any vendor until Graham has tested it. During
  testing every outgoing email goes to Graham only.
- One owner per field: vendors, contacts, trades, projects and people stay in
  Trade Desk's tables. This app only adds punch tables.
- No secret in the page. The Claude API key, the email key and photo storage
  are reached only through the Worker.
- Never invent contact data. A vendor with no email shows "needs a contact" and
  the report can still be downloaded.
- KMC brand v2.2: Warm Paper ground, light only, the KMC lockup, Gotham stack,
  Roboto for numbers, 48px touch targets, verified at 393px wide.

## Technical shape

- **App:** its own Cloudflare Worker (`kmc-punch`) and installable phone web app,
  with no framework and no build step, matching Trade Desk.
- **Database:** the same `kmc-desk` D1 database. New tables only:
  - `punch_items`: project, number (for the ID), status, trade, vendor, location,
    plan pin, description, priority, the AI's original suggestion (kept so we
    can see how often the PM changes it), and who did what and when
  - `punch_photos`: item, kind (`deficiency` or `completion`), storage key, who took it, when
  - `floor_plans`: project, name, file, confirmed room list
  - `punch_reports`: what was sent, to whom, by whom, the PDF itself
- **Photos and PDFs:** a Cloudflare R2 bucket. Photos are resized on the phone
  before upload, so they upload fast on weak signal.
- **AI:** the Claude API, called from the Worker. The key is a Worker secret.
- **Email:** Resend, the same service Trade Desk's weekly digest uses.
- **Offline:** the page and today's lists are cached on the phone. New photos and
  edits queue in the phone's storage. Each carries an ID made on the phone, so a
  retry on bad signal can never create the same item twice (Trade Desk does the
  same for scorecards).
- **Audit:** every change is written to Trade Desk's existing `audit_log`.

## Build order (each step tested with Graham before the next)

1. Capture, AI draft, PM review, and the list with filtering and sorting (online)
2. Completed photo, PM close, and the PDF report with download and email
3. Floor plans: room list and pins
4. Offline capture and sync

## Open questions

1. **Job codes.** Each project gets a short code (e.g. `SMITH`), stored in a new
   `job_code` column on Trade Desk's `projects` table and entered when the
   project is set up. Codes must be unique and can't change once a punch item
   uses them. Up to 8 letters and digits?
2. **Schema home.** The punch tables share one database with Trade Desk, so
   their migration should live in Trade Desk's `migrations/` folder to keep one
   schema history. OK?
3. **Repo.** Waiting on the empty `kmc_punch_list` repo (or whatever you name it).
