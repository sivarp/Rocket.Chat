# Refactoring Analysis — `livechat` & `lib` backend modules

> Scope: `apps/meteor/app/livechat` and `apps/meteor/app/lib`, the two
> highest-complexity backend modules identified in the module-wide survey.
> This is an analysis document only — no source code is changed by it.
> Each candidate is weighed across three lenses: **ROI**, **complexity
> severity**, and **risk-of-breakage**.

---

## 1. Executive summary

Two findings dominate everything else and should shape any refactoring effort:

1. **Zero co-located unit tests on the critical files.** Every high-value
   candidate below — `Helper.ts`, `QueueManager.ts`, `RoutingManager.ts`,
   `closeRoom.ts`, `createRoom.ts`, `sendMessage.ts`, `saveUser.ts`,
   `deleteUser.ts`, `notifyListener.ts`, `sendNotificationsOnMessage.ts` — has
   **no `*.spec.ts` beside it**. Refactoring them blind is the single largest
   risk in this codebase. The one exception is `ContactMerger.ts`, which *does*
   have specs and is therefore the safest large file to touch.

2. **`notifyListener.ts` has the largest blast radius in either module** — it is
   imported by **150 files**. It is also the most mechanically repetitive file
   (45 near-identical broadcast functions), making it simultaneously the highest
   ROI *and* the highest risk candidate.

**Recommended order of work** (detailed in §6):
1. Quick win — externalize `defaultBlockedDomainsList.ts` (data-as-code).
2. Low-risk mechanical extractions — `sendMessage.ts` validators, then the
   `notifyListener.ts` factory.
3. Test-first — add characterization tests, *then* refactor the god
   functions/files (`Helper.ts`, `QueueManager.ts`, `RoutingManager.ts`,
   `saveUser.ts`, `createRoom.ts`).

---

## 2. Methodology

Complexity was measured from the source directly (not estimated):

- **Size** — `wc -l` per file (server files, specs excluded).
- **Function length / nesting** — located the longest functions and their
  conditional/loop nesting via `grep` of export boundaries.
- **Coupling / blast radius** — counted importers across `apps/meteor` with
  `grep -rl`, plus distinct module imports per file.
- **Test presence** — checked for a co-located `*.spec.ts` per candidate.

Each candidate is scored **1–5** on four axes:

| Axis | Meaning |
|------|---------|
| **Sev** (severity) | Code + functional complexity (size, cyclomatic, mixed concerns). |
| **Risk** | Danger of leaving as-is: blast radius × test-gap × centrality. |
| **Effort** | Work required to refactor *safely*. |
| **ROI** | Value (maintainability/clarity gain) per unit of effort + risk. |

All file:line references below were spot-checked against the current source.

---

## 3. `livechat` candidates

| # | Candidate | LOC | Sev | Risk | Effort | ROI |
|---|-----------|-----|-----|------|--------|-----|
| L1 | `server/lib/Helper.ts` | 964 | 5 | 5 | 5 | 4 |
| L2 | `server/lib/QueueManager.ts` | 532 | 5 | 4 | 4 | 4 |
| L3 | `server/lib/RoutingManager.ts` | 382 | 4 | 4 | 4 | 4 |
| L4 | `server/lib/closeRoom.ts` | 297 | 3 | 4 | 3 | 4 |
| L5 | `server/lib/departmentsLib.ts` | 313 | 3 | 3 | 3 | 3 |
| L6 | `server/api/v1/room.ts` | 541 | 3 | 3 | 3 | 3 |
| L7 | `server/lib/contacts/ContactMerger.ts` | 336 | 3 | 2 | 3 | 3 |

**L1 — `Helper.ts` (964 LOC).** A "god file": 16 exports spanning room creation,
inquiry creation, subscription management, agent normalization, and chat
transfers — all mixing business logic with direct DB access.
`forwardRoomToDepartment` (lines 622–820) is **~198 lines** with deep nesting for
queueing, fallback departments and transfers; `forwardRoomToAgent` (484–579) is
~95 lines. It is the spine of the omnichannel flow and has no tests.
*Target shape:* split into `RoomService`, `InquiryService`, `AgentService`,
`TransferService`; isolate DB access behind the existing model layer.

**L2 — `QueueManager.ts` (532 LOC).** `requestRoom` is ~107 lines with 7+ nested
conditions (agent availability, department fallbacks, settings).
`startConversation` manually manages a Mongo session + retry; `getInquiryStatus`
is a 6-way branch. Registers `beforeDelegateAgent`/`onNewRoom` hooks, creating a
circular coupling with `hooks.ts`. *Target shape:* extract
`InquiryStatusDeterminer` and `ConversationStarter`; move transaction handling
behind a shared helper.

**L3 — `RoutingManager.ts` (382 LOC).** A singleton object acting as an implicit
state machine. `takeInquiry` is ~103 lines mixing lock acquisition, callback
execution and inquiry state transitions. Behaviour depends on callback ordering
(`beforeDelegateAgent` → `delegateInquiry` → `takeInquiry` → `afterTakeInquiry`),
i.e. temporal coupling. *Target shape:* an explicit inquiry state machine;
separate agent *selection* from agent *assignment*; extract lock management.

**L4 — `closeRoom.ts` (297 LOC).** Transaction orchestration, business logic and
a 5-callback chain are intermingled; DB mutation, message saving, notification
and app-event dispatch all happen in one flow. *Target shape:* a `RoomCloser`
service with transaction boundary separated from business logic.

**L5 — `departmentsLib.ts` (313 LOC).** 15+ inline validation checks interleaved
with persistence; "feature envy" on `Helper.updateDepartmentAgents`; eager
multi-callback emission on state changes. *Target shape:* a `DepartmentValidator`
and a separate `DepartmentAgentManager`.

**L6 — `api/v1/room.ts` (541 LOC).** REST handler with business logic embedded;
parameter validation mixed with routing; calls `QueueManager`/`Helper`/`rooms`
directly with no facade. *Target shape:* thin handler that orchestrates a
`RoomService` facade; move validation to dedicated validators.

**L7 — `contacts/ContactMerger.ts` (336 LOC).** Field-merge logic nested ~5
levels deep. **Notably, this file has co-located specs** (`resolveContactConflicts.spec`,
`mapVisitorToContact.spec`), making it the lowest-risk large refactor in the
module. *Target shape:* extract a field-merge strategy; lean on existing tests as
the safety net.

---

## 4. `lib` candidates

| # | Candidate | LOC | Sev | Risk | Effort | ROI |
|---|-----------|-----|-----|------|--------|-----|
| B1 | `server/lib/notifyListener.ts` | 603 | 4 | 5 | 4 | 5 |
| B2 | `server/functions/saveUser/saveUser.ts` | 261 | 5 | 4 | 4 | 5 |
| B3 | `server/functions/createRoom.ts` | 327 | 5 | 4 | 4 | 5 |
| B4 | `server/lib/sendNotificationsOnMessage.ts` | 430 | 5 | 4 | 3 | 4 |
| B5 | `server/functions/sendMessage.ts` | 298 | 4 | 4 | 3 | 4 |
| B6 | `server/functions/deleteUser.ts` | 186 | 4 | 4 | 3 | 4 |
| B7 | `server/lib/defaultBlockedDomainsList.ts` | 945 | 1 | 1 | 1 | 3 |

**B1 — `notifyListener.ts` (603 LOC, 45 exports).** 45 near-identical broadcast
functions where only the entity, id, client-action and diff vary (e.g.
`notifyOnRoomChanged`, `notifyOnRoomChangedById`,
`notifyOnRoomChangedByUsernamesOrUids`, `notifyOnRoomChangedByContactId`, …).
**Imported by 150 files** — the largest blast radius in either module.
*Target shape:* a `createNotifier<T>(entity)` factory collapsing the 45 functions
to ~6 by entity type, keeping the existing export names as thin wrappers during
migration to limit churn across the 150 call sites.

**B2 — `saveUser/saveUser.ts` (261 LOC).** A god function (~170 lines of
effective body, wrapped in an IIFE at line 249) spanning validation, identity
updates, email, password, roles, custom fields, auditing and transaction
management, with nested conditional field updates and 9+ module imports.
*Target shape:* extract `updateUserIdentity` / `updateUserEmail` /
`updateUserPassword` / `updateUserSettings` handlers; keep `saveUser` as a
transaction coordinator.

**B3 — `createRoom.ts` (327 LOC).** A single ~185-line export (from line 142)
handling federation validation, room property construction, app-event triggering,
subscription + team integration and post-creation callbacks. Imported by 9 files.
*Target shape:* split into `validateRoomCreation`, `buildRoomProperties`,
`createRoomSubscriptions`; keep `createRoom` as orchestrator.

**B4 — `sendNotificationsOnMessage.ts` (430 LOC).** `sendMessageNotifications`
(from line 275) builds a Mongo `$or` query via nested `forEach` over
`['desktop','mobile','email']` × per-kind settings (lines ~317–353) — very high
cyclomatic complexity. *Target shape:* extract a pure `buildNotificationQuery()`
with explicit parameters.

**B5 — `sendMessage.ts` (298 LOC).** Six inline attachment validators
(`validFullURLParam`, `validPartialURLParam`, `validateAttachmentsFields`,
`validateAttachmentsActions`, `validateAttachment`, `validateBodyAttachments`,
lines 33–150) bundled with prepare → store → hooks. Imported by 13 files.
**The validator extraction is isolated and low-risk** — a good early win.
*Target shape:* move validators to `messageValidators.ts`; reduce `sendMessage`
to core logic.

**B6 — `deleteUser.ts` (186 LOC).** 13 imports; mixes file-upload cleanup,
livechat department updates, integration cleanup, three message-erasure
strategies, auditing and app events. *Target shape:* extract per-concern cleanup
handlers behind a coordinator.

**B7 — `defaultBlockedDomainsList.ts` (945 LOC).** ~87% of the file is a single
constant array of domains — data masquerading as code. **Quick win, near-zero
risk.** *Target shape:* move the list to a JSON asset and load it.

---

## 5. Three ranked views

The four-axis scores let the same candidates be ordered differently depending on
what you optimize for.

### Lens A — ROI (value vs. effort + risk)
Best return first:
1. **B7** — externalize blocked-domains list (trivial effort, ~0 risk).
2. **B5** — extract `sendMessage` validators (isolated, low risk).
3. **B1** — `notifyListener` factory (mechanical, high value; manage the 150-file churn with wrappers).
4. **B2 / B3** — god functions; high value, plan carefully.
5. **L4** — `closeRoom` separation.
6. **L1** — highest value overall but the most effort/risk.

### Lens B — Pure complexity severity
Most complex first:
1. **L1** `Helper.ts` (964, god file).
2. **B2 / B3** (god functions, mixed concerns).
3. **B4** (cyclomatic query building).
4. **L2** `QueueManager`.
5. **L3** `RoutingManager`.
6. **B5 / B6**.

### Lens C — Risk of breakage (danger of leaving as-is)
Most dangerous first:
1. **B1** — 150 importers, no spec.
2. **L1** — central to every omnichannel flow, no spec.
3. **B3 / B5** — 9–13 importers, no spec.
4. **L2 / L3** — core routing/queueing, no spec.
5. **B2 / B4** — central but narrower reach.
6. **L7** — lowest risk; it *has* tests.

---

## 6. Cross-cutting risks

- **Missing tests.** No critical candidate (except `ContactMerger.ts`) has
  co-located unit tests. **Prerequisite:** write characterization / golden-master
  tests *before* refactoring L1, L2, L3, B2, B3.
- **Callback / temporal coupling.** Livechat flows depend on callback execution
  order (`RoutingManager`, `QueueManager`, `closeRoom`). Refactors must preserve
  ordering or replace it with an explicit, typed event mechanism.
- **Manual transaction handling.** Mongo sessions/retries are hand-rolled in
  `QueueManager` and `closeRoom` with no shared abstraction — duplicate logic and
  deadlock/recovery risk. A shared transaction helper would de-risk several
  candidates at once.
- **Data-as-code.** `defaultBlockedDomainsList.ts` inflates the module and review
  diffs for no functional reason.

---

## 7. Recommended sequencing

1. **Quick win:** B7 — move the blocked-domains list to a JSON asset.
2. **Low-risk mechanical extractions:** B5 (`sendMessage` validators) → B1
   (`notifyListener` factory, keeping old export names as wrappers).
3. **Establish safety nets:** add characterization tests around the god
   functions/files.
4. **Test-then-refactor the god code:** B2 (`saveUser`) and B3 (`createRoom`) →
   L1 (`Helper.ts`) → L2/L3 (`QueueManager`/`RoutingManager`) → L4 (`closeRoom`).
5. **Opportunistic:** L5, L6, B4, B6 as the relevant areas are touched.

---

*Generated as an analysis artifact. No production code, configuration, or tests
were modified by this document.*
