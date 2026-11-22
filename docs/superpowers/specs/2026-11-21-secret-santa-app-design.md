# Secret Santa web app: design

- Status: approved by the owner on 2026-11-21. Nothing is implemented.
- Date: 2026-11-21
- Repository: `rubicon/rubicon-secret-santa` (public, AGPL-3.0-only)
- Supersedes: an earlier uncommitted draft that framed this as a mode of a chat skill. That framing
  is dropped; this is a standalone web app.

Every address, hostname and name here is an invented placeholder (`example.com`). Real values live
in local deployment config and Worker secrets, never in this repository.

Platform claims carry the documentation URL beside them. A claim not confirmed from Cloudflare's
documentation is marked `UNVERIFIED:` and collected in section 15.

Vocabulary: a **drawing** is one Secret Santa exchange (one set of people, one draw). An
**organizer** runs drawings. A **participant** is a person in a drawing. A **giver** is assigned a
**recipient**. The **admin** is an organizer with extra powers over the whole deployment. "Session"
is used only for sign-in sessions, never for a drawing.

## 1. Intent

A Secret Santa site the owner is happy to send his family to, and later to run as a free public
service. The reason it exists is trust: the common sites carry ads, affiliate links, upsells, or
quietly keep data. This one is built so a skeptical person can check what happens to their data by
reading the source, and run their own copy.

What makes it different:

1. **Privacy by mode.** In Private mode the organizer can take part without ever seeing anyone's
   assignment in the app.
2. **Stated, enforced deletion.** Every drawing and everything in it is deleted from the app
   automatically 90 days after it is created, and is unrecoverable within 120 (section 11).
3. **Open source, self-hostable, no ads, no affiliate links, no upsell, no third-party tracking.**
4. **Editable after the draw** without starting over: resend, add or remove one person, or redraw.
5. **A polished Christmas look**, not a utility page.

Success for Phase 1: the owner runs the family exchange on it this season, every participant gets
exactly one correct email, the organizer (who also takes part) never learns an assignment, and
nothing is left in storage after 90 days.

## 2. Phases

- **Phase 1 (this spec):** family use on the owner's Cloudflare account. Built config-driven, so
  nothing about the owner is in the code and anyone can deploy their own copy.
- **Phase 2 (out of scope, designed for):** a free public site the owner hosts. This spec keeps
  Phase 2 possible (per-organizer data scoping, caps, anti-relay rules) but does not build it.
  Known Phase 2 blockers are listed in section 14.

## 3. Goals, non-goals, anti-goals

**Goals (Phase 1)**

1. Organizers sign in, keep a saved list of people, and run many drawings.
2. Each drawing has a mode: **Private**, **Display**, or **Display & Email** (section 6).
3. Each drawing has a toggle, **allow pairs to draw each other** (section 7).
4. The draw is server-side, uniform over every valid assignment, and "impossible" is only ever
   reported when it is proved (section 7).
5. Participants get one email with their recipient and a private link to view it again; no sign-in
   (section 8).
6. Editing after the draw: resend without redraw, add or remove one person by splicing, and full
   redraw (section 9).
7. Email templates with tokens, editable by the admin (section 10).
8. Automatic deletion 90 days after creation, plus an admin list (drawing names and dates only)
   to view and delete drawings early (section 11).
9. Admin-configurable caps (section 12).
10. Setup automated as far as is safe (section 13).

**Non-goals**

- Hiding data from whoever controls the Cloudflare account. Anyone who can deploy code or read
  Durable Object storage can read the pairings. The app states this plainly (section 12).
- Wish lists, gift tracking, budgets beyond a free-text field, reminders, chat, gift purchasing.
- The `secret-santa-generator` chat skill. It is a separate product and is not in this repository.
- Phase 2 public hosting (section 14).

**Anti-goals: failures even if everything else works**

- Ads, affiliate links, upsell, analytics or any third-party script or asset.
- An assignment visible to the organizer in Private mode through any screen, email, status
  indicator, export, error message or timing the app controls.
- An assignment, participant name, email address or private link in a URL path or query string,
  or in Worker logs.
- The app sending mail to an address that is not on the drawing it is sending for (open relay).
- A draw that silently favors some assignments, or that says "impossible" when it is not.
- Data that outlives its stated deletion date inside the app.

## 4. Constraints

- **Platform:** Cloudflare Workers (Workers Paid, already active on the owner's account), Durable
  Objects with SQLite storage, Email Sending via the `send_email` binding from an onboarded
  sending subdomain, Cloudflare Access for organizers only. Cloudflare only; no third-party API.
- **Language:** TypeScript.
- **Never onboard an apex domain** that has other mail (the owner's apex has Google Workspace and
  DMARC quarantine). Send from a dedicated sending subdomain (placeholder
  `santa@mail.example.com`).
- **Public repository:** no real email, Cloudflare account, zone, application or identity-provider
  ID, or deployment hostname in any committed file.
- **Access seats:** every user who authenticates through Access consumes a Zero Trust seat;
  service tokens do not
  (https://developers.cloudflare.com/cloudflare-one/faq/getting-started-faq/). Participants must
  therefore never go through Access. The free-plan seat count is UNVERIFIED.
- **Brand:** not chosen. The display name is a config value; code and docs use a neutral name.
- **Visual direction:** taken from a private reference the owner chose. Nothing from it (names,
  text, images, branding) is copied into the repository; the builder reviews it during UI work.
  The reference is a set of static HTML files on the owner's machine; its location is kept in the
  owner's local notes, not here.

## 5. Approaches considered

**Storage.** Three options were weighed; the owner picked A.

| | A. A Durable Object per drawing (chosen) | B. One D1 database | C. One Durable Object for everything |
| --- | --- | --- | --- |
| Isolation | Physical: one drawing's data is in its own object | Every query must filter by organizer | Every query must filter by organizer |
| 90-day delete | Each drawing's own alarm wipes its own storage | Daily scheduled job deletes expired rows | Alarms inside one shared object |
| Double draw / double send | One object owns each drawing; storage calls never interleave, but awaits on email do, so durable state guards are still required (section 10) | Must be built with transactions and status guards | Same as A, but for the whole site |
| Public site | Scales per drawing | Fine | Bottleneck; one bug exposes everything |

A was chosen because it gives each drawing one owner for its state and its own deletion alarm, and
it keeps drawings apart physically. It does not by itself make concurrent operations safe: a
Durable Object can run other requests while one awaits a network call
(https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/), so
section 10 specifies the guards.

**Participant identity.** Two options; the owner picked the private link.

- **Private link in each participant's email (chosen):** no sign-in, no Access seats, the same in
  both phases. A forwarded link reveals that person's match, exactly as a forwarded email would.
- **Access sign-in for participants:** a stronger check, but one seat per participant, and it does
  not work for a public site.

**Rendering (builder's call within this constraint):** server-rendered HTML with a small amount of
client script. The participant page needs script (section 8). No client framework is required.

## 6. Modes

Set per drawing at creation; changing it later is allowed only before the draw.

| Mode | Participants emailed? | Organizer sees pairings in the app? | Organizer's confirmation email |
| --- | --- | --- | --- |
| **Private** | Yes | **No.** Never, by any path the app controls. | Counts only; no list |
| **Display** | No | Yes, on screen | Includes the full list |
| **Display & Email** | Yes | Yes, on screen | Includes the full list |

Display mode is for organizers who hand out assignments themselves. Participants in Display mode
get no email and no private link.

**What "hidden from the organizer" means in Private mode.** The organizer can see participant
names, emails and delivery status, can trigger every edit in section 9, and receives counts. No
screen, email, status change, error message or export reveals who gives to whom, including
indirectly (section 9 covers the indirect cases). The account owner can still technically read
storage; the app's privacy page says so.

## 7. The draw

### Inputs

Participants (display name and email), exclusions (unordered pairs who must not give to each
other, typically couples), and the **allow pairs to draw each other** toggle.

### Rules

- Every participant gives to exactly one other participant and receives from exactly one.
- No one draws themselves. Exclusions are symmetric: if A and B are excluded, neither gives to the
  other.
- **Toggle on:** any assignment satisfying the rules is valid, including two people who draw each
  other.
- **Toggle off:** the assignment must be **a single chain**: one cycle through everyone
  (A → B → C → … → A). With two participants this is impossible, since a two-person chain is a pair
  drawing each other.

### Guarantees

1. **Uniform:** every valid assignment is equally likely.
2. **"Impossible" is a proof.** It is reported only when no valid assignment exists.
3. **Honest third outcome.** If the draw cannot finish within its compute budget, it returns
   *undecided*, never *impossible*, and tells the organizer to remove an exclusion or try again.
4. Randomness is `crypto.getRandomValues`, turned into integers without modulo bias.

### Method (proposed; the builder may substitute anything that keeps the guarantees and passes the
tests in section 16)

- **Toggle on, feasibility:** a valid assignment exists exactly when the bipartite graph
  (giver → allowed recipient) has a perfect matching. This is decided exactly in polynomial time.
- **Toggle on, sampling:** draw a uniformly random permutation and accept it if it satisfies the
  rules; repeat. Accepted results are exactly uniform over valid assignments. With few exclusions
  the acceptance rate is around one in three to one in eight, so this is fast. If a large attempt
  budget is exhausted on a feasible input, fall back to exact counting-based sampling for small
  drawings (up to about 20 people); above that, return *undecided*.
- **Toggle off, sampling:** fix one person, shuffle the rest uniformly to get a uniformly random
  chain, and accept it if no link is excluded; repeat. Exactly uniform over valid chains.
- **Toggle off, feasibility:** deciding whether any valid chain exists is a Hamiltonian-cycle
  problem, which has no known fast general method. With few exclusions a valid chain is
  guaranteed by a simple degree test (if everyone may give to at least half the group). Otherwise
  use exhaustive search with a time budget; if the budget runs out, return *undecided*.
- The participant cap (section 12) bounds all of this. Default 50, hard ceiling 100.

### Storage of draw inputs

Exclusions and the toggle are stored with the drawing until it is deleted, because splicing and
redraws need them.

## 8. Participant experience

### The email

One email per participant per assignment, from the configured sender, built from the Santa
template (section 10). It names only that participant's recipient, plus the drawing name, the
optional exchange date and budget, and their private link.

### The private link

- A 256-bit random token, base64url-encoded, stored only as its SHA-256 hash, bound to one
  participant in one drawing.
- **The token travels in the URL fragment**, for example
  `https://santa.example.com/m#<drawing-id>.<token>`. The fragment is never sent to the server, so
  the token cannot appear in request logs or in a `Referer` header. Page script reads the fragment
  and posts it in a request body.
- The match page: `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, `X-Robots-Tag:
  noindex`, a strict Content Security Policy, and no third-party assets.
- The token never appears in logs, error messages or analytics.
- A token is voided when the participant is removed, when a full redraw happens, and when the
  drawing is deleted.
- **On resend**, the new link is valid at once, and the old link stays valid until the new email's
  send record reaches `sent`; only then is the old link voided. If the send ends `failed` or
  `unknown`, the old link stays valid, so a failed resend never locks a participant out. At most
  two links per participant are live at a time; a further resend while one is pending replaces the
  pending one.

### The match page

Shows: the drawing name, exchange date and budget if set, and "You're buying for: <recipient>".
One button: **Email this to me again**, which sends the Santa email to the address on file for
that participant (never to an address in the request), with a fresh link (old-link rules above).

**Self-resend is capped at 20 per participant per day** (owner's decision, admin-adjustable). No
real participant reaches it; it stops a script holding a forwarded link from flooding one inbox or
using up the account's shared daily email quota. A drawing-wide ceiling also applies: self-resends
across a whole drawing are capped at twice its participant count per day. Organizer-triggered
emails (draw, edits, resends) do not count against either self-resend cap. Self-resend is never a
relay: it sends only this participant's own assignment to this participant's own stored address.

**Lost links.** If a participant deletes their email, the organizer resends it. A public "email me
my link" form is deferred to Phase 2: it would need several live links per participant (links are
stored only as hashes, so an existing one cannot be re-sent), an index of emails across drawings
(which breaks per-drawing isolation), and a bot check, which would load a third-party script.

## 9. Organizer experience

### Sign-in

Organizers sign in through Cloudflare Access (passkey, or one-time PIN by email). The Access
application covers only organizer and admin paths (section 12). The Worker also validates the
Access token itself and checks the email against a configured organizer list. There are no service
tokens.

### Saved people

Each organizer keeps a list of people (display name, email) to pick from when creating drawings.
Emails are stored lowercase. The list is the organizer's own and lasts until the organizer removes
entries; it is outside the 90-day rule, and the privacy page says so. The admin can delete an
organizer's whole record.

Each organizer also has a display name, asked for on first sign-in and editable, because Access
sign-ins carry no reliable name. It fills `{{organizer_name}}`.

### Creating a drawing

Name (60 characters max), mode, the pairs toggle, optional exchange date, optional budget (short
free text), participants (from saved people or added inline), exclusions. Validation before the
draw:

- at least two participants (three with the toggle off);
- no duplicate emails; duplicate display names need a distinguishing label because the recipient is
  shown by name;
- every email well-formed; exclusions name participants in this drawing; no self-exclusion;
- participant cap.

Then **Draw**. Draw and first send are idempotent: a repeated request (double click, retry) never
draws twice.

### After the draw

The drawing page shows participants with delivery status, and in Display modes the pairings.

**Resend without redraw.** To one participant or to all whose email failed or is uncertain. Reuses
the stored assignment; issues a fresh link, and the old link is voided once the new email is sent
(section 8).

**Add one person (splice).** Choose uniformly at random a link A → B in the current assignment such
that A → new and new → B are both allowed (respecting exclusions and, with the toggle off, keeping
a single chain). Set A → new → B. Emails sent: the new person gets their assignment, and A gets an
"your recipient has changed" email. That is two emails. If no link qualifies: in Display modes the
organizer is told a splice isn't possible and is offered a full redraw; in Private mode see
"Private mode: add and remove" below.

**Remove one person (splice).** If X sits in A → X → B, set A → B, provided A → B is allowed and,
with the toggle off, the chain stays valid with at least three people. A gets one "your recipient
has changed" email; X's link is voided. If A → B is not allowed: in Display modes the organizer is
told "this change needs a full redraw" and asked to confirm; in Private mode see below.

**Private mode: add and remove** (owner's decision). The organizer confirms up front that the change
"may give everyone a new assignment". The app splices when it can; when it can't, it performs a full
redraw automatically. The organizer's screen and confirmation are identical in both cases ("X has
been removed. Everyone affected has been emailed."), with no email count that would distinguish
them.

**When the organizer is also a participant** (their email matches a participant in the drawing), a
splice that would change the organizer's own recipient is never used, because the update email
would reveal a live pairing. Removing the organizer's own recipient X would give the organizer X's
former recipient; adding a person at the organizer's own link would tell the organizer the
newcomer's recipient. So in Private mode:

- **Remove:** if the organizer gives to the removed person, always do a full redraw.
- **Add:** choose the insertion link only from links that do not start at the organizer; if none
  qualifies, do a full redraw.

Accepted residual leak: after a successful splice the organizer could, at most, infer that the
removed person's giver and recipient are not an excluded pair; after a redraw anything inferred
concerns a voided assignment. An organizer who is also a participant can tell from their own inbox
whether a redraw ran, but never gains a live pairing. Probing is no longer free: every blocked
splice redraws and emails the whole drawing.

**When no valid redraw exists** (too few people left, or the remaining exclusions make every
assignment impossible, or the draw returns *undecided*), the edit is rejected atomically: the
assignment, links and send records stay exactly as they were, no email is sent, and the organizer
is told "This change can't be made with the current exclusions." This reveals nothing about the
assignment, because a redraw failing depends only on the people and exclusions, and when a redraw
is impossible a splice is impossible too.

**Full redraw.** Requires explicit confirmation once anyone has been emailed. Draws fresh, voids
every old link, and emails everyone a new assignment whose subject and opening say the earlier one
is void.

**Correcting a participant's email after the draw** is done as remove plus add, not an in-place
edit, so an organizer cannot silently redirect someone's assignment to another mailbox while
keeping the old assignment.

### Private mode: indirect leaks the app must prevent

- After a splice, the organizer must not be able to tell who A is, or whether a splice or a redraw
  ran. Per-person status rows show only the status of each person's *own original delivery*;
  update emails are reported only as "all sent" or "some failed: retry", with a retry button that
  names no one and no count.
- Error messages for splice and redraw never name a participant other than the one being added or
  removed.
- Confirmation emails carry counts only.
- Exports (if any are built) in Private mode contain no pairings.

### Confirmation emails to the organizer

1. **Draw confirmation** after the draw: in Private mode, counts only; in Display modes, the full
   list.
2. **All sent** once every participant email for the current assignment has a `sent` status.

## 10. Email

### Sending

- `env.EMAIL.send({ to, from, subject, html, text })` returns `{ messageId }`
  (https://developers.cloudflare.com/email-service/api/send-emails/workers-api/). One recipient
  per message, never `cc` or `bcc`. Both `html` and `text` are sent.
- The binding is restricted to the configured sender with `allowed_sender_addresses`
  (https://developers.cloudflare.com/email-service/configuration/send-bindings/). Recipients
  cannot be restricted at the binding, because they are arbitrary; the code enforces it instead.
- **Anti-relay rules, enforced in code:** a send goes only to an email that is a participant of the
  drawing being sent for, or to the signed-in organizer; `from` is the configured constant;
  subject and body come only from templates; no caller-supplied headers or attachments.

### Templates

Three templates, each with a subject, a text body, and an HTML body rendered into a fixed,
app-controlled layout:

1. **Santa email** (to each participant).
2. **Organizer draw confirmation** (two variants: counts only for Private, full list for Display
   modes).
3. **Organizer all-sent confirmation.**

Plus the system variants the edits need: "your recipient has changed" and "your earlier assignment
is void". These use the Santa template's look with fixed wording.

Tokens (exact set is the builder's call; these are the minimum): `{{participant_name}}`,
`{{recipient_name}}`, `{{drawing_name}}`, `{{exchange_date}}`, `{{budget}}`, `{{match_link}}`,
`{{organizer_name}}`, `{{sent_count}}`, `{{participant_count}}`, `{{pairing_list}}`.

The admin edits **plain text with tokens**, never raw HTML. The app renders that text into its
fixed HTML layout and the text body. Allowed tokens per template:

| Template | Allowed tokens |
| --- | --- |
| Santa email | all except `{{sent_count}}`, `{{participant_count}}`, `{{pairing_list}}` |
| Draw confirmation, Private | `{{drawing_name}}`, `{{exchange_date}}`, `{{budget}}`, `{{organizer_name}}`, `{{sent_count}}`, `{{participant_count}}` |
| Draw confirmation, Display modes | the Private set plus `{{pairing_list}}` |
| All sent | the Private confirmation set |

Saving a template with a token not allowed for it is refused.

Rules:

- Every token value is HTML-escaped in HTML and stripped of CR and LF in subjects.
- `{{recipient_name}}` and `{{pairing_list}}` are not allowed in any subject, so inbox previews
  and provider-side logs of subjects never show an assignment.
- `{{pairing_list}}` is accepted only in the Display-mode confirmation template.
- Templates are edited by the admin, for the whole deployment, and stored in the admin's Durable
  Object. Defaults ship in the repository.

### Delivery state

Each send record moves through `pending → sending → sent | failed`. A record found still `sending`
on resume becomes `unknown` and is **never** resent automatically; the organizer sees it and
chooses. The documentation shows no idempotency key on `send()`, so delivery is at-most-once by
default with visible exceptions.

Error handling, by code
(https://developers.cloudflare.com/email-service/api/send-emails/workers-api/):

| Code | Handling |
| --- | --- |
| `E_RATE_LIMIT_EXCEEDED`, `E_EMAIL_SENDING_NOT_READY` | Retry with backoff, small fixed cap |
| `E_DAILY_LIMIT_EXCEEDED` | Mark `failed`; tell the organizer to resend later |
| `E_RECIPIENT_SUPPRESSED` | Mark `failed`; tell the organizer that address is suppressed |
| Any other code, or an unknown one | Mark `failed` with the code; retried only on request |
| An exception with no code (transport or runtime failure) | Mark `unknown`, not `failed`: acceptance can't be ruled out, so it is never resent automatically |
| A `TypeError` raised before the call is made (malformed arguments) | A bug: mark `failed` and surface it |

`failed` is used only when the error establishes that the message was not accepted.

### Concurrency guards

A Durable Object runs other requests while one is awaiting a send or another object, so the
Drawing object never relies on "one request at a time":

- Every state change (draw, edit, redraw, start of a send batch) is a single synchronous storage
  transaction that checks and advances a `version` number and an `operation` field. A second
  draw or edit that arrives while an operation is in progress is refused, or returns the result of
  the earlier request when it carries the same idempotency key.
- Each send record is claimed by moving it from `pending` to `sending` in a synchronous write
  before the network call. Only the request that made that move sends that message.
- Before writing a send result or applying an edit, the code re-checks `version`. A result that
  belongs to an assignment a redraw has since voided is recorded against the old version and
  triggers nothing.
- Draw, edit and send logic never holds `blockConcurrencyWhile()` across a network call.

A newly onboarded sending domain starts with a conservative daily quota that rises with history
(https://developers.cloudflare.com/email-service/platform/limits/).

## 11. Data and deletion

### Durable Objects

| Object | Key | Holds |
| --- | --- | --- |
| **Drawing** | a new unique id per drawing | settings, participants, exclusions, assignment, link hashes, send records, `delete_at` |
| **Organizer** | the organizer's lowercase email | saved people, an index of the organizer's drawings (id, name, created, `delete_at`) |
| **Admin registry** | a single fixed name | templates, caps, and an index of every drawing (id, name, organizer, created, `delete_at`) |

Recipients are stored as a reference to a participant row, never as an email address in a
separate copy.

### The 90-day rule

- `delete_at` = creation time + 90 days (owner's decision). It is fixed at creation and never
  extended by edits or by the exchange date.
- Each Drawing object sets an alarm for `delete_at`. When it fires, the object calls
  `deleteAll()`, which is atomic for SQLite-backed objects and, with a compatibility date of
  2026-02-24 or later, also clears the alarm
  (https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/). The Worker's
  compatibility date must be on or after that date.
- Alarms are retried only six times on an uncaught exception
  (https://developers.cloudflare.com/durable-objects/api/alarms/). Every alarm handler therefore
  catches its own errors and sets a new alarm (with backoff) before returning, so retries never
  run out.
- **The Admin registry is the durable deletion work list.** Its entry for a drawing is removed only
  after the Drawing object confirms its storage is empty. The registry's own daily alarm (with the
  same catch-and-reschedule rule) re-triggers deletion of every entry past `delete_at` and records
  each attempt. Organizer indexes are cleaned the same way.
- An entry more than 24 hours past `delete_at` and still not confirmed is shown at the top of the
  admin list as "deletion overdue", so a stuck deletion is visible rather than silent.
- Any request to a drawing past `delete_at` deletes it first and answers "not found".
- **Delete now:** the organizer for their own drawings; the admin for any drawing. It uses the same
  work-list path.

### What the deletion promise covers

The app's privacy page states it exactly (owner's decision):

- Everything the app stores about a drawing is deleted from the app 90 days after the drawing is
  created.
- **It is gone permanently within 120 days.** Cloudflare keeps 30 days of point-in-time recovery
  history for every SQLite-backed Durable Object, so for up to 30 days after deletion someone with
  access to the Cloudflare account could restore it
  (https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/#pitr-point-in-time-recovery-api).
  The app never uses that recovery API.
- Copies outside the app are not covered: emails already in participants' mailboxes, and whatever
  the email provider retains in its own logs (UNVERIFIED: what Cloudflare's Email Sending activity
  log keeps and for how long).

### Admin list

Shows drawing name, organizer, created date and deletion date. No participants, no pairings.
Actions: open nothing; delete now.

## 12. Security

### Who can do what

| Route group | Caller | Checks |
| --- | --- | --- |
| Public pages, static assets | anyone | none |
| Match page and its API | holder of a private link | token hash matches a live link |
| Organizer pages and API | organizer | Access at the edge, plus in-Worker JWT validation, plus email on the organizer list |
| Admin pages and API | admin | the organizer checks, plus email on the admin list |

- **Access is path-scoped.** The Access application covers the organizer and admin paths only
  (https://developers.cloudflare.com/workers/configuration/cloudflare-access/ and
  https://developers.cloudflare.com/cloudflare-one/access-controls/policies/app-paths/). Note: the
  owner's existing Access application is on a hostname used only by this app and currently covers
  the whole hostname. Setup finds it by its application domain and narrows it to the organizer and
  admin paths; nothing else loses protection. Phase 1 needs one seat per organizer (two people);
  the free-plan seat count is UNVERIFIED but this is far below any plausible limit.
- **In-Worker JWT validation** on every organizer and admin request: header
  `Cf-Access-Jwt-Assertion`; keys from `https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`;
  check `iss`, `aud`, `exp`, `nbf`
  (https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/authorization-cookie/validating-json/).
  A request to an organizer path without a valid token is refused even if Access were
  misconfigured. UNVERIFIED: that the `email` claim is present for passkey sign-ins; must be tested
  live before launch.
- Every organizer action is scoped to that organizer's own drawings; the Drawing object checks the
  caller is its organizer (or the admin).
- Static assets use `run_worker_first` so every request reaches Worker code first
  (https://developers.cloudflare.com/workers/static-assets/).
- `workers_dev: false` and `preview_urls: false`, set explicitly
  (https://developers.cloudflare.com/workers/configuration/previews/).
- State-changing requests require POST, a same-origin check, and a CSRF token on organizer forms.

### Threat model

| Actor | Protected? | How |
| --- | --- | --- |
| Organizer, Private mode | Yes | Section 9's leak rules; redaction tests in section 16 |
| Another participant | Yes | Links are per-participant; the match API returns only the caller's row |
| Someone with no link and no organizer sign-in | Yes | Nothing reachable but public pages |
| A forwarded link or email | No | It shows that one person's match. A successful resend voids the old link |
| A compromised participant mailbox | No | The email contains the match |
| The Cloudflare account owner, anyone with deploy rights | **No** | Can read storage or deploy other code. Stated on the privacy page |
| Cloudflare itself | No | Out of scope |
| Spam through the app | Yes (Phase 1) | Anti-relay rules, organizer allow-list, caps |

### Logging

- `observability.logs.invocation_logs = false`; no `console.log` of names, emails, assignments or
  tokens. Request URLs never carry personal data or tokens.
- UNVERIFIED: whether invocation logs capture bodies; checked live.

### Caps (admin-configurable, with defaults)

| Cap | Default |
| --- | --- |
| Participants per drawing | 50 (hard ceiling 100) |
| Active drawings per organizer | 10 |
| Drawings created per organizer per day | 10 |
| Organizer-triggered emails per organizer per day | 300 |
| Participant self-resend | 20 per participant per day, and 2 × participant count per drawing per day |

## 13. Configuration and setup

### Configuration (local, never committed)

| Setting | Placeholder |
| --- | --- |
| Display name of the app | `Secret Santa` |
| Hostname | `santa.example.com` |
| Sender | `santa@mail.example.com` |
| Organizer emails | `organizer@example.com`, … |
| Admin emails | subset of organizers |
| Access team domain and application audience tag | from the Access application |

A committed `wrangler.template.jsonc` and `.dev.vars.example` carry placeholders only. A setup
script renders the real Wrangler config from a local, git-ignored file.

### Setup automation

A `setup` script that is safe to re-run, shows each account change and asks before making it:

1. checks the sending domain is onboarded and active (it does not onboard one);
2. creates or updates the path-scoped Access application and its policy from the organizer list;
3. renders the config and deploys the Worker with its custom domain;
4. confirms `workers.dev` and preview URLs are off and that an organizer path without sign-in
   redirects to Access.

It needs a Cloudflare API token with the narrowest scopes that cover these steps; the exact scopes
are the builder's to determine from the documentation and list in the README. The token is read
from the environment and never written to disk or printed.

## 14. Phase 2 (not built here): known blockers

1. **Organizer sign-in at public scale.** Access charges a seat per authenticated user, so it
   cannot serve unlimited public organizers. Phase 2 needs another organizer sign-in.
2. **Spam.** Any public organizer could email arbitrary addresses with template text they control.
   Needs per-organizer quotas, Turnstile, organizer-editable templates restricted or removed, abuse
   reporting, and a suppression path.
3. **The account-wide email quota** is shared by every organizer.
4. Legal pages (privacy, terms) for a public service.

Phase 1 keeps every data path scoped per organizer so these can be added without migrating data.

## 15. UNVERIFIED, collected

1. Free-plan Access seat count.
2. The `email` claim in the Access JWT for passkey sign-ins.
3. What the Email Sending activity log stores and for how long.
4. Whether Workers invocation logs include request or response bodies.
5. The minimum Cloudflare API token scopes for the setup script.

## 16. Testing

Test-driven throughout. No tests of mocked behavior and no end-to-end tests with the platform
replaced by a mock.

- **Framework:** the Workers Vitest integration, package `@cloudflare/vitest-plugin` (renamed from
  `@cloudflare/vitest-pool-workers` in August 2026), configured with the `cloudflareTest()` plugin;
  tests run inside the Workers runtime with real bindings and per-test-file storage isolation
  (https://developers.cloudflare.com/workers/testing/vitest-integration/ and
  https://developers.cloudflare.com/changelog/post/2026-08-19-vitest-plugin/). Integration tests
  may use `createTestHarness()` from Wrangler
  (https://developers.cloudflare.com/workers/testing/). Versions are pinned and verified against
  the registry at build time.
- **Email in tests:** locally the `send_email` binding is simulated and logs instead of sending
  (https://developers.cloudflare.com/email-service/observability/local-development/sending/).
  Tests never use `remote: true`.
- **Pure core** (no platform calls): parsing, validation, feasibility, draw, splice, template
  rendering and escaping, token generation and hashing, send state machine, error classification,
  authorization decisions. Randomness is an injected parameter (a seeded generator in tests), not a
  mock.
- **Draw properties:** for many seeds and sizes 2 to 30, every result satisfies every rule; known
  impossible inputs return *impossible*; inputs where a chain exists but random search struggles
  still find one or return *undecided*, never *impossible*.
- **Uniformity:** for small inputs (for example 4 to 6 people, with and without exclusions, both
  toggle settings), enumerate every valid assignment and run a chi-square test over many draws.
- **Splice:** add and remove keep every rule, and the insertion point is uniform over qualifying
  links.
- **Private-mode redaction contract:** for generated drawings in Private mode, serialize every
  organizer-facing response, page and email, before and after each kind of edit, and assert no
  pairing is present, that the organizer-visible output of an add or remove is byte-identical
  whether a splice or a redraw ran (apart from the person's name), and that nothing identifies
  which participant received an update email. The one accepted inference is the one stated in
  section 9.
- **Organizer as participant:** in Private mode, removing the organizer's own recipient, or an add
  whose only qualifying link starts at the organizer, always takes the redraw path, and no
  organizer-received email reveals a live pairing.
- **Atomic edit failure:** an add or remove with no valid redraw leaves assignment, links and send
  records byte-identical and sends nothing.
- **Durable Object behavior** in the Workers runtime: concurrent draws produce one draw; a draw or
  edit racing an in-flight send batch is refused or ordered by `version`; resume after a crash
  leaves `unknown`, not a resend; a resend whose email fails leaves the old link working; alarm
  deletion clears storage and indexes, and a deletion whose first attempts throw is still completed
  by the rescheduled alarm.
- **Live checks** (owner-run, with addresses the owner controls): SPF, DKIM and DMARC pass;
  delivery to two mail providers; passkey sign-in and the `email` claim; organizer paths refused
  without sign-in; match pages work without sign-in; `workers.dev` and preview URLs unreachable;
  logs contain no names, emails or tokens.

## 17. Build order (owner's decision: core first)

Each stage is usable when it lands.

1. Draw, Private-mode emails, private links and self-resend, organizer resend, auto-delete.
2. Display and Display & Email modes; splice add and remove; full redraw.
3. Admin template editor, caps screen, admin list.
4. Setup script.

Adding a person necessarily emails two people (the newcomer and the giver whose recipient changed);
removing a person emails one.

No open questions remain for the owner.

## 18. Calls made by the author (check these)

- Storage layout, Durable Object names and what each holds (section 11).
- Server-rendered pages with light script; the token in the URL fragment.
- Token size, hashing and voiding rules.
- The draw method, the *undecided* outcome, and the cap defaults.
- Email-address correction as remove-plus-add.
- Template tokens and the subject restrictions.
- Templates are deployment-wide and admin-edited (not per organizer) in Phase 1.
- Status display rules for Private mode after a splice.
- The setup script asks before every account change and does not onboard a sending domain.
- Lost links are recovered by the organizer resending; a public recovery form is deferred.
- Saved people persist until removed, outside the 90-day rule; the admin can delete an organizer's
  whole record.
- Organizers set a display name; templates are plain text with per-template token allow-lists.
