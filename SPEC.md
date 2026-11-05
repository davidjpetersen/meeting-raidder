# Meeting RAIDDer: Internal Teams Pilot (Revised Spec + Build Plan)

Status: ready for Gate 0 (feasibility spike). Not yet approved for full build.

## 1. Goal and scope

A Teams meeting app with a chat bot that listens to a live meeting transcript, proposes Risks, Assumptions, Issues, Decisions and Dependencies (RAIDD) in the meeting chat, lets a designated facilitator confirm, edit or dismiss each proposal, and writes confirmed items to an Excel table in SharePoint.

**Pilot scope:** one organization, scheduled internal meetings, English, one facilitator per meeting, one project workbook per meeting. Separate application from the EPUB pipeline.

**Non-goals:** email-address "participant" invitations, external or ad hoc meetings, audio capture or storage, multi-language, multiple facilitators, auto-confirmation of any item.

**Unverified external facts.** Everything below about Microsoft APIs, permissions and policies is a working assumption until confirmed in Gate 0 against current Microsoft documentation and our tenant. Do not build past Gate 0 on an assumption that has not been tested.

## 2. Transcript input routes

| Route | Status |
|---|---|
| Native live transcript access (Work IQ) | Primary. Prove in Gate 0. |
| Application-hosted audio media bot | Fallback only. Separately scoped (C#/.NET, Windows hosting, developer-preview platform). Not part of this pilot. |
| Post-meeting transcript via Graph | Recap only. Does not satisfy live detection. |

If Gate 0 fails, stop the pilot path and document the blocker. Do not start the media-bot route without a new scope decision.

## 3. Gate 0: feasibility spike (do first)

Exit with a written go/no-go. All items must be answered with evidence from our tenant.

1. **Access policy.** Is live transcript access enabled for our tenant? Which admin controls apply, including any recently announced or upcoming transcript-access controls? Record who owns each switch and the current state.
2. **Identity model.** Which identity reads the live transcript (facilitator, organizer, or app) and with which delegated scopes? How does a server hold that delegation during the meeting (token acquisition, refresh, expiry)?
3. **Delivery mechanism.** How does the service receive live transcript segments (push, polling, notifications)? What is the segment shape (speaker, text, timestamps)?
4. **Latency.** Measure speech-to-transcript delay over at least 5 meetings of mixed pace. Report median and worst case.
5. **Prerequisites.** Must transcription be running? Who must start it? What happens when it stops or restarts? What does the bot see when it does?
6. **Installation.** Can the app be installed into a scheduled meeting by its organizer under our app policies (custom app upload or admin-approved catalog)?
7. **Privacy review.** Meeting speech may include protected student or personal information. Get a written decision from data security on transcript handling, excerpt storage, model endpoint, and retention (see section 8) before any real meeting is used.

**Gate decision:** go only if items 1–3 and 6 are satisfied and median speech-to-transcript latency leaves room for the delivery target in section 7.

## 4. Meeting experience

1. The organizer adds Meeting RAIDDer to a scheduled meeting and selects the project workbook and facilitator.
2. The facilitator starts monitoring. A status card shows **active**, **paused** or **unavailable** transcript input, and updates when it changes.
3. The detector proposes RAIDD items. The bot posts cards in the meeting chat:

   > **Possible Risk**
   > Supplier approval may arrive too late for the planned launch.
   > Suggested owner: unassigned
   > **Confirm · Edit · Dismiss**

4. Only the designated facilitator can confirm or edit. Confirmation updates the card and saves the item. Dismissal is recorded and the proposal is not re-raised for the same discussion.
5. Silence never counts as confirmation. Unresolved proposals stay open until the meeting ends.
6. At meeting end the bot posts confirmed-item counts, unresolved proposals, and the workbook link.

Repeated discussion updates an existing proposal rather than creating a new one.

**Facilitator lifecycle.** If the facilitator leaves or loses authorization, monitoring pauses and the status card says so. The organizer may reassign the facilitator. Pending proposals persist until a facilitator confirms or the meeting ends.

## 5. Detection rules

- **Risk:** a future uncertainty that could harm the objective. **Issue:** an existing problem. **Assumption:** a stated condition taken as true. **Dependency:** reliance on a person, team or deliverable outside the speaker's control. **Decision:** an agreed outcome, not a suggestion.
- A suggestion ("we could use vendor B") is not a Decision until agreement is evident. If unsure, propose as a note-worthy suggestion or do not propose.
- Keep a short supporting excerpt and timestamp. Leave owner and due date blank unless explicitly stated by a speaker or entered by the facilitator.
- Corrected decisions: a later correction in the meeting updates the proposal, or supersedes a confirmed item (the original row is marked superseded, never deleted).

**Duplicate definition.** A new candidate is a duplicate of an existing proposal if it has the same category and its summary is judged equivalent by the detector's comparison step within the same meeting. Duplicates add evidence to the existing proposal and do not create a new card.

## 6. Data and workbook

SharePoint Excel table columns: ID, project, category, description, owner, due date, status, meeting link, evidence timestamp, evidence excerpt, confirmer, confirmation time, source proposal ID.

- Stable item ID: derived from meeting ID plus proposal ID. Unique key in the table. Used for idempotent writes.
- Writes use the facilitator's delegated Microsoft authorization (Graph Excel row-add is assumed not to support application permissions; confirm in Gate 0).
- Failed writes enter a durable queue. The card shows "Confirmed — Excel sync pending" until the write succeeds, then updates. If delegated authorization can no longer be refreshed, the card prompts the facilitator to sign in again, and items are not lost.
- Workbook access follows the SharePoint site permissions. The facilitator must have edit access, which is verified when the workbook is selected.

## 7. Quality and performance targets

**Delivery latency.** Measured from our service receiving the transcript segment to the card being posted: median ≤ 30 seconds and 95th percentile ≤ 60 seconds. Batching is part of this budget: proposals are grouped into at most one chat update per minute, but a proposal must not wait longer than the batch window after it is ready. Speech-to-transcript delay is measured separately and reported alongside.

**Detection quality.** Before any pilot meeting, build an evaluation set of at least 20 labeled snippets covering all five categories, near-misses (Risk versus Issue, suggestion versus Decision), and no-item chatter. Initial bar: of cards posted, at least 70% are judged useful by the facilitator, and at least 60% of labeled items are surfaced. These are starting targets to be revisited after the first live meetings, not a promise of accuracy.

**Pilot success criteria.** Run at least 5 real meetings with 2 or more facilitators. Report cards posted, confirm/edit/dismiss rates, duplicate rate, sync failures, and facilitator feedback.

## 8. Privacy, security and compliance

- Meeting activation is explicit. Participants see that the assistant is analyzing the meeting transcript when monitoring starts.
- No raw audio is stored by the app.
- Transcript text is held in memory and in the proposal store only as long as needed. Default retention for stored excerpts: set by data security in Gate 0; until then excerpts are kept only for confirmed items and a short-lived working set for open proposals, purged at meeting close.
- Excerpts must not capture student identifiers. Apply a redaction or exclusion step for obvious personal identifiers before storage, and document its limits. This is mitigation, not a guarantee.
- Use only an organization-approved model endpoint, with no training on our data.
- Secrets in a managed secret store. Delegated refresh tokens encrypted at rest and scoped per facilitator.
- Confirmation authorization checks the acting user's Entra ID against the meeting's designated facilitator on the server, never on card contents alone.
- Audit log of confirm, edit, dismiss, reassign and sync events.

## 9. Failure behavior (to test)

All five categories; rejected suggestions; corrected decisions; duplicates; unauthorized confirmation attempts; transcript stopping and resuming; service restart and reconnect; repeated button clicks; facilitator leaves; delegated token expiry; Excel write failure and recovery; workbook locked or permission removed.

---

# Build plan

## 10. Architecture

```
Teams meeting ─(live transcript)─► Transcript Ingest ─► Segment buffer
                                                          │
                                                          ▼
                                             Detector (LLM, approved endpoint)
                                                          │
                                                          ▼
Teams meeting chat ◄─ Card Publisher ◄─ Proposal Store ◄─ Dedupe/merge
        │ (card actions)                       ▲
        ▼                                      │
   Bot endpoint ─► Authz check ─► Confirm/Edit/Dismiss ─► Outbox ─► Excel Writer ─► SharePoint workbook
```

Components:

- **Bot and meeting app** (Teams app manifest, bot endpoint, meeting chat tab/dialog for edit and setup).
- **Transcript ingest:** adapter interface `TranscriptSource` so the mechanism proven in Gate 0 (or a fake for tests) can be swapped in.
- **Detector:** takes a sliding window of segments plus open proposals; returns structured candidates (category, summary, excerpt, timestamps, optional owner/due date, merge target). Strict JSON schema output, validated.
- **Proposal store:** durable state per meeting and proposal, status machine below.
- **Card publisher:** batches and posts or updates adaptive cards using stored conversation references.
- **Outbox and Excel writer:** transactional outbox, idempotent row upsert keyed on item ID, retry with backoff.
- **Auth service:** per-facilitator delegated token storage and refresh.

## 11. Stack (recommended, pending Gate 0)

- TypeScript on Node.js, using the current Teams bot SDK. The native route needs no media hosting, so the .NET/Windows requirement applies only to the fallback.
- Hosting: one managed container service in our cloud tenant. Single instance is acceptable for the pilot, with state in the database so restarts are safe.
- Storage: a managed relational database (Postgres or Azure SQL) for proposals, outbox, meeting state and audit log.
- Secrets: managed key vault.
- Model: organization-approved endpoint behind a thin `Detector` interface with a recorded-fixture mode for tests.
- Tests: Vitest for unit tests, scripted transcript replays for detector evaluation, a fake Teams and Graph layer for integration tests.

## 12. State machine

Proposal: `proposed → posted → (confirmed | dismissed | superseded | expired)`.
Confirmed item sync: `pending → synced | failed(retrying) | needs_reauth`.

Transitions are enforced in one module and guarded by version checks, so repeated clicks and reconnect races resolve to a single outcome. Item ID is derived deterministically, so re-running a write is an upsert.

## 13. Milestones (ordered, no dates)

- **M0 — Gate 0 spike** (section 3). Output: go/no-go and measured latency. Nothing below starts without go.
- **M1 — Thin vertical slice.** One category (Risk). Real transcript source, hard-coded facilitator, one card, Confirm only, one Excel row. This proves the full loop: spoken statement, transcript, chat card, facilitator confirmation, Excel row.
- **M2 — Core loop hardening.** Edit and Dismiss, status card (active/paused/unavailable), proposal store with state machine, outbox and retry, idempotency, facilitator authorization, meeting-end summary.
- **M3 — Detector.** All five categories, structured output, dedupe and merge, batching, evaluation set and scorecard, latency instrumentation.
- **M4 — Resilience and privacy.** Facilitator reassign, token re-auth flow, reconnect handling, redaction step, retention purge, audit log, full failure-behavior test pass (section 9).
- **M5 — Pilot.** Install in real meetings per section 7 criteria, collect feedback, report.

## 14. First actions

1. Create the Gate 0 checklist as a tracked task list and assign an owner for each item (tenant admin for items 1, 6; data security for item 7).
2. Stand up a throwaway Teams app and test meeting to run items 2–5.
3. In parallel, write the evaluation set (section 7) and the `TranscriptSource` and `Detector` interfaces with fakes, which do not depend on Gate 0 results.
4. Record the Gate 0 decision at the top of this file before starting M1.

## 15. Open questions

- Which model endpoint is approved, and what are its rate and cost limits?
- Retention period for excerpts and audit logs.
- Is excerpt storage acceptable at all for meetings that may mention students, or must the workbook hold summaries only?
- Who owns the project workbook template and its permissions?
