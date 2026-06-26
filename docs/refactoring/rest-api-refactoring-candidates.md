# REST API Refactoring Candidates — the "easy to sell" business case

> Companion to `livechat-lib-analysis.md` and `helper-ts-business-case.md`.
> This document targets the **REST API layer**, which is the easiest place to
> justify a refactor to management: every endpoint is **customer-facing**,
> behaviour is **already locked by a large contract-test suite**, and the
> improvement is **objectively measurable** (lines of duplication removed).

---

## 1. Why the REST API is the easy sell

A refactor is easy to approve when three things are true. For the core REST API,
all three already hold:

1. **It is critical, customer-facing functionality.** These endpoints are the
   public contract every integration, mobile client, and third-party app depends
   on. Improvements here have visible value.
2. **The safety net already exists.** The endpoints are covered by a large
   end-to-end API suite (counts below). "Prove the old behaviour still works" is
   answered up front — the tests are the proof, and they run in CI on every change.
3. **The win is measurable.** Unlike deep logic refactors, the headline candidate
   here is *duplication*. We can state the result as a hard number: lines and
   endpoints de-duplicated.

**Existing contract-test coverage (the safety net):**

| Endpoint group | Handler size | End-to-end test suite | ~Test cases |
|---|---|---|---|
| `users.ts` | 2,094 LOC | `api/users.ts` (6,531 LOC) | ~292 |
| `rooms.ts` | 1,735 LOC | `api/rooms.ts` (4,959 LOC) | ~222 |
| `channels.ts` | 1,509 LOC | `api/channels.ts` (4,731 LOC) | ~221 |
| `chat.ts` | 1,394 LOC | `api/chat.ts` (5,086 LOC) | ~206 |
| `groups.ts` | 1,318 LOC | `api/groups.ts` (2,779 LOC) | ~118 |

These are not aspirational numbers — they are existing assertions guarding the
exact behaviour any refactor must preserve.

---

## 2. Headline candidate — `channels.ts` ↔ `groups.ts` duplication

**This is the recommendation to take to management.** It is the cleanest,
lowest-risk, highest-clarity refactor in the API layer.

### The finding

`channels.ts` (public channels, room type `c`) and `groups.ts` (private groups,
room type `p`) implement **36 operations that are near-identical**, differing
essentially only in *which room type they target*:

```
addAll  addLeader  addModerator  addOwner  archive  close  convertToTeam
counters  create  delete  files  getIntegrations  history  info  invite
kick  leave  list  members  messages  moderators  online  open  removeLeader
removeModerator  removeOwner  rename  roles  setAnnouncement  setCustomFields
setDescription  setPurpose  setReadOnly  setTopic  setType  unarchive
```

Combined, the two files are **2,827 lines**. A large fraction of that is
copy-paste that diverges only at the "find the room" step.

### Proof it is genuine duplication

`channels.archive` and `groups.archive`, side by side — identical except the
lookup helper; the real work (`executeArchiveRoom`) is **already a shared
function**:

```ts
// channels.archive
const findResult = await findChannelByIdOrName({ params: this.bodyParams });
await executeArchiveRoom(this.userId, findResult._id);
return API.v1.success();

// groups.archive
const findResult = await findPrivateGroupByIdOrName({ params: this.bodyParams, userId: this.userId });
await executeArchiveRoom(this.userId, findResult.rid);
return API.v1.success();
```

This same shape — *find room (by type) → call a shared operation → return* —
repeats across most of the 36 operations. The business logic is already
centralized; only the **endpoint wiring is duplicated**.

### Why it sells

- **Risk is low and already mitigated.** ~221 (channels) + ~118 (groups) = **~339
  end-to-end test cases** assert these endpoints' behaviour today. A
  behaviour-preserving de-duplication keeps them green; any divergence fails CI.
- **The win is a hard number.** "Collapse 36 duplicated endpoint pairs;
  materially reduce ~2,800 lines of near-copy-paste to a single
  room-type-parameterized handler set." That is a metric management can repeat.
- **It fixes a real, ongoing cost.** Today, every change to a channel/group
  operation (a new permission check, a bug fix) must be made **twice**, and the
  two copies silently drift. Bugs fixed in `channels` but missed in `groups` are
  a known class of defect this removes structurally.

---

## 3. Secondary candidates (same safety-net advantage)

If management wants a phased program, these large handlers are the next tier —
each backed by its own contract-test suite:

- **`users.ts` (2,094 LOC)** — the largest API file, touching authentication,
  account management, and admin operations. Highly critical; refactor into
  cohesive sub-routers (profile, admin, presence) behind the existing ~292-case
  suite. Higher severity than channels/groups but a harder, more sensitive
  change.
- **`rooms.ts` (1,735 LOC)** — cross-type room operations (upload, favorite,
  saveNotification, adminRooms). ~222 test cases. Candidate for splitting
  read/admin/media concerns.
- **`chat.ts` (1,394 LOC)** — message send/update/delete/react/report, the
  busiest write path. ~206 test cases. Candidate for extracting per-operation
  handlers and shared validation.

These are noted for completeness; **the channels/groups duplication is the one to
lead with** because the cost (duplication) and the win (its removal) are the
easiest to articulate and measure.

---

## 4. Proposed refactoring (behaviour-preserving)

For the headline candidate — **do not change any endpoint contract**:

1. **Lock the baseline.** Run the existing `api/channels.ts` and `api/groups.ts`
   e2e suites green; that is the acceptance gate.
2. **Introduce a room-type-parameterized handler factory.** A single
   `createRoomEndpoint(operation, { roomType, finder })` that wires the shared
   operation to the correct room finder (`findChannelByIdOrName` vs
   `findPrivateGroupByIdOrName`). The underlying operation functions (e.g.
   `executeArchiveRoom`) are already shared and stay untouched.
3. **Migrate one operation pair at a time.** Replace `channels.X` and `groups.X`
   with factory-generated handlers; keep route names, params validators, and
   responses byte-for-byte identical.
4. **Verify after each pair.** The ~339 contract tests must stay green; each pair
   is an independently shippable, revertible commit.

Result: the 36 duplicated pairs collapse to one declarative table of
`(operation, roomType)` wiring, with no behavioural change and no contract change.

---

## 5. Business framing

| | Headline (channels/groups) |
|---|---|
| **Severity** | Medium-high — duplication, drift, double-maintenance |
| **Risk of refactor** | **Low** — ~339 existing contract tests; behaviour preserved |
| **Effort** | Moderate, and *incremental* (one endpoint pair per commit) |
| **ROI** | **High** — visible LOC reduction, removes a whole defect class |
| **"Is current behaviour safe?"** | **Yes, already** — answered by the existing e2e suites |

### KPIs to report

| Metric | Today | Target |
|---|---|---|
| Duplicated endpoint pairs | 36 | 0 (single parameterized set) |
| `channels.ts` + `groups.ts` size | 2,827 LOC | Materially reduced |
| Places to change a channel/group op | 2 (drift-prone) | 1 |
| Contract test suites | Green | Stay green (no regressions) |

---

## 6. Recommendation

Lead the business case with the **`channels.ts` / `groups.ts` de-duplication**.
It is the rare refactor where the problem is a number (36 duplicated operations /
~2,800 lines), the risk is already retired (~339 contract tests in CI), and the
outcome is measurable and customer-safe. Use `users.ts`, `rooms.ts`, and
`chat.ts` as the follow-on phases once the pattern and the safety story are
proven on the headline candidate.

---

*All figures measured directly against the current source
(`apps/meteor/app/api/server/v1/*` and `apps/meteor/tests/end-to-end/api/*`).*
