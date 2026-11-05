# Meeting RAIDDer: Internal Teams Pilot (Revised Spec + Build Plan)

Status: transcript route changed to **Recall.ai (meeting-bot API)**. Ready for Gate R (section 3). Not yet approved for full build, and not approved for any real meeting until data security signs off.

**Change log:** The earlier draft made native Teams live transcripts (Work IQ) the primary route. Microsoft documents the admin controls but no delivery mechanism for live segments (the Work IQ A2A interface is a question-and-answer API), so that route is now a watch item, not the plan. Recall.ai is the primary route to try. Its onboarding guide (`recall://guides/onboarding` on the Recall MCP server) could not be read when this was written, because the server was not yet authorized. **Where the guide and this spec disagree, the guide wins for Recall setup steps.**

## 1. Goal and scope

A Teams meeting app with a chat bot that listens to a live meeting transcript, proposes Risks, Assumptions, Issues, Decisions and Dependencies (RAIDD) in the meeting chat, lets a designated facilitator confirm, edit or dismiss each proposal, and writes confirmed items to an Excel table in SharePoint.

**Pilot scope:** one organization, scheduled internal meetings, English, one facilitator per meeting, one project workbook per meeting. Separate application from the EPUB pipeline.

**Non-goals:** email-address "participant" invitations, external or ad hoc meetings, audio capture or storage, multi-language, multiple facilitators, auto-confirmation of any item.

**Unverified external facts.** Everything below about Microsoft APIs, permissions and policies is a working assumption until confirmed in Gate 0 against current Microsoft documentation and our tenant. Do not build past Gate 0 on an assumption that has not been tested.

## 2. Transcript input routes

| Route | Status |
|---|---|
| **Recall.ai meeting bot** (third-party API) | **Primary. Prove in Gate R.** A bot joins the meeting by URL, and live transcript segments are delivered to our service. Exact delivery mechanism, latency and Teams support to be confirmed from Recall's guide. |
| Native live transcript access (Work IQ) | Watch item. No documented live-delivery mechanism yet. Revisit if Microsoft publishes one. |
| Self-hosted application media bot | Fallback only (C#/.NET, Windows hosting, developer-preview platform). Separately scoped. |
| Post-meeting transcript via Graph | Recap only. Does not satisfy live detection. |

**Trade-off accepted for the pilot:** meeting audio and transcripts pass through a third-party vendor. That is acceptable only after the data-security decision in Gate R item 1. If it is refused, fall back to the self-hosted media bot or the watch item.

If Gate R fails, stop the pilot path and document the blocker.

## 3. Gate R: Recall.ai feasibility spike (do first)

Exit with a written go/no-go. Follow Recall's onboarding guide for setup, and record evidence for each item.

1. **Data-security decision (blocks real meetings).** Written approval or refusal for sending meeting audio and transcripts to Recall: data residency, retention and deletion settings, vendor review, participant notice. Until approved, use rehearsal meetings with test accounts only.
2. **Workspace.** Confirm the Recall workspace is Western Governors University in US West (Oregon) before creating keys or webhooks. Stop if it differs.
3. **Teams joining.** Can a Recall bot join a scheduled Teams meeting in our tenant by join URL? Record what the organizer must allow: lobby admission, external-bot policies, any tenant policy blocking third-party bots (check with the Teams admin).
4. **Delivery mechanism.** How are live transcript segments delivered (webhook, websocket, polling), what is the segment shape (speaker, text, timestamps, partial versus final), and how are retries and ordering handled?
5. **Latency.** Measure speech-to-segment delay over at least 5 rehearsal meetings of mixed pace. Report median and worst case.
6. **Bot lifecycle.** Join, admitted, in-call, left, and failure states and their webhooks. What happens on kick, meeting end, or transcript errors?
7. **Calendar scheduling.** Does Recall's calendar integration support our calendar provider (Microsoft 365 / Outlook)? What authorization is required, and who grants it?
8. **Cost.** Per-hour bot and transcription cost at expected pilot volume.

**Gate decision:** go only if items 1–6 are satisfied and median latency leaves room for the delivery target in section 7. Item 7 may be deferred to M2 without blocking go.

Keep the native Work IQ check as a low-priority side test: run the Work IQ sample during a live rehearsal meeting and record whether it can supply live text. It does not block Gate R.

## 4. Meeting experience

1. The facilitator opens the RAIDDer web console, pastes a Teams meeting URL, and selects the project workbook. (Calendar-based scheduling comes later: see M2.) The facilitator is the signed-in user.
2. The facilitator launches the bot. The console shows the bot state (**joining, waiting for admission, in meeting, paused, unavailable, ended**), and the meeting organizer or a participant admits it from the lobby if required. The bot's participant name states that it is transcribing for RAIDD notes.
3. The detector proposes RAIDD items. For the first milestones they appear as cards in the web console; posting them as cards in the Teams meeting chat is a later milestone (M4) and needs the Teams app. Card example:

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

- Meeting activation is explicit. Participants see the bot join with a name that says it is transcribing, and the organizer is told before the first pilot meeting.
- **Third-party processing.** Recall receives meeting audio and produces the transcript. Use only after the Gate R item 1 decision. Configure Recall to retain recordings and transcripts for the shortest period allowed, and delete after processing where the guide permits. Do not request media storage from Recall that we do not need.
- Recall API keys and webhook secrets live in the managed secret store, never in the repository. Verify webhook signatures on every inbound request.
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
Web console ─ meeting URL ─► Bot Launcher ─► Recall API ─► Recall bot joins Teams meeting
                                                                │ (live transcript, bot status)
                                                                ▼
                              Webhook endpoint (signature check) ─► Segment buffer
                                                                │
                                                                ▼
                                                  Detector (LLM, approved endpoint)
                                                                │
                                                                ▼
Web console cards ◄─ Card Publisher ◄─ Proposal Store ◄─ Dedupe/merge
   (Teams chat cards later)                    ▲
        │ (confirm / edit / dismiss)           │
        ▼                                      │
   API ─► Authz check ─► state machine ─► Outbox ─► Excel Writer ─► SharePoint workbook

Calendar (later): Recall calendar integration ─► scheduled bots ─► same ingest path
```

Components:

- **Web console and API** (facilitator sign-in, meeting URL entry, bot launch and status, proposal cards, workbook selection). The Teams app (manifest, meeting-chat cards) is added in M4.
- **Bot launcher:** creates and tracks Recall bots, one per meeting, with an idempotency key so a repeated click does not launch two bots. Stores bot ID, meeting URL, state.
- **Webhook endpoint:** receives Recall transcript and bot-status events. Verifies the signature, acknowledges fast, persists the raw event before processing, and de-duplicates by event ID.
- **Transcript ingest:** adapter interface `TranscriptSource` with a `RecallTranscriptSource` implementation (and a fake and a replay source for tests), so the route can be swapped.
- **Detector:** takes a sliding window of segments plus open proposals; returns structured candidates (category, summary, excerpt, timestamps, optional owner/due date, merge target). Strict JSON schema output, validated.
- **Proposal store:** durable state per meeting and proposal, status machine below.
- **Card publisher:** batches and posts or updates adaptive cards using stored conversation references.
- **Outbox and Excel writer:** transactional outbox, idempotent row upsert keyed on item ID, retry with backoff.
- **Auth service:** per-facilitator delegated token storage and refresh.

## 11. Stack (recommended, pending Gate 0)

- TypeScript on Node.js with a small HTTP API and web console. Recall is called over its REST API, so no media hosting is needed. The Teams bot SDK is added in M4, and the .NET/Windows requirement applies only to the self-hosted media-bot fallback.
- Webhooks need a publicly reachable HTTPS endpoint, so local development uses a tunnel or a deployed dev instance.
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

- **M0 — Gate R spike** (section 3). Output: go/no-go, measured latency, and the data-security decision. Nothing below runs against a real meeting without go. Work that needs no live Recall access (interfaces, fakes, state machine, evaluation set) may start before go.
- **M1 — Thin vertical slice.** Paste a meeting URL in the console, launch a bot, receive live transcript, detect one category (Risk), show one card, Confirm only, write one Excel row. Proves the loop: spoken statement, transcript, card, facilitator confirmation, Excel row.
- **M2 — Core loop hardening and calendar scheduling.** Edit and Dismiss, bot-state display, proposal store with state machine, outbox and retry, idempotency, facilitator authorization, meeting-end summary, and calendar-based scheduling through Recall's calendar integration (if Gate R item 7 supports it).
- **M3 — Detector.** All five categories, structured output, dedupe and merge, batching, evaluation set and scorecard, latency instrumentation.
- **M4 — Resilience, privacy and Teams cards.** Facilitator reassign, token re-auth flow, reconnect and webhook-replay handling, redaction step, retention purge, audit log, full failure-behavior test pass (section 9), and the Teams app with meeting-chat cards.
- **M5 — Pilot.** Install in real meetings per section 7 criteria, collect feedback, report.

## 14. First actions

1. Authorize the Recall MCP server (`/mcp` in Claude Code, signing in to the Western Governors University workspace, US West (Oregon)), then read `recall://guides/onboarding` and follow it. Record any difference from this section.
2. Create the Gate R checklist as a tracked task list and assign an owner for each item (data security for item 1, Teams admin for item 3).
3. Run items 3–6 with a rehearsal meeting and test accounts only.
4. In parallel, write the evaluation set (section 7) and the `TranscriptSource` and `Detector` interfaces with fakes and a replay source, which do not depend on Gate R results.
5. Record the Gate R decision at the top of this file before starting M1.

## 15. Open questions

- Will data security approve Recall, and under what retention and residency settings?
- Does Recall's calendar integration cover our Microsoft 365 calendars, and who authorizes it?
- Which participants must be told a bot is joining, and how?
- Which model endpoint is approved, and what are its rate and cost limits?
- Retention period for excerpts and audit logs.
- Is excerpt storage acceptable at all for meetings that may mention students, or must the workbook hold summaries only?
- Who owns the project workbook template and its permissions?
