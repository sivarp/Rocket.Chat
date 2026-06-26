# Business Case — Refactoring `livechat/server/lib/Helper.ts`

> Prepared for engineering management. Audience: decision-makers approving
> investment. This document justifies refactoring a single, specific file and
> answers the question management raised: *"Do we have test coverage proving the
> current behaviour works, so a refactor won't silently break it?"*

---

## 1. The ask in one line

Approve a time-boxed, **behaviour-preserving** refactor of
`apps/meteor/app/livechat/server/lib/Helper.ts`, executed **test-first** so that
existing functionality is provably protected before any code moves.

---

## 2. Why this file (and not another)

We scored every large backend file in the two most complex modules on four axes:
complexity severity, risk-of-breakage, effort, and ROI (see
`livechat-lib-analysis.md`). One file — and only one — scores the maximum on
**both** severity **and** risk:

| | Severity | Risk-of-breakage |
|---|:---:|:---:|
| **`Helper.ts`** | **5 / 5** | **5 / 5** |

That combination is precisely what management asked us to target: the most
complex *and* the most dangerous-to-leave-as-is candidate.

**What makes it severity 5:**
- **964 lines** in a single file holding **16 exported functions** that mix five
  unrelated concerns — room creation, inquiry creation, subscription management,
  agent assignment, and chat transfers — *plus* direct database access.
- The two transfer functions alone are oversized and deeply nested:
  `forwardRoomToDepartment` is **~198 lines** (lines 622–820) and
  `forwardRoomToAgent` is **~95 lines** (lines 484–579), with logic that branches
  on queueing, fallback departments, agent limits, and callback ordering.

**What makes it risk 5:**
- It is **core to every Omnichannel/Livechat conversation** — room setup,
  queueing, and agent/department transfers all flow through it.
- **6 production modules import it directly**, on top of heavy use within the
  livechat module itself; a regression here radiates across the feature.
- It has **zero unit tests** (see §4), so today nothing catches a logic
  regression at the function level.

---

## 3. The business problem (cost of doing nothing)

This is not "tidy code for its own sake." The current state has measurable
business cost:

- **Slower delivery of Omnichannel features.** Every change to transfers,
  queueing, or routing means reading and re-reasoning about a 198-line function
  with hidden callback-ordering dependencies. Estimation inflates and lead time
  grows for anything touching this area.
- **Elevated defect risk in a customer-facing path.** Livechat transfer/routing
  bugs are visible to end customers and support agents in real time. A single
  tangled function with no unit safety net is a recurring incident source.
- **Onboarding drag.** New engineers cannot safely modify this file without
  significant ramp-up, concentrating knowledge (and bus-factor risk) in a few
  people.
- **Compounding interest.** The file already merged multiple concerns; without
  intervention each new requirement is bolted onto the largest function, making
  the next change more expensive than the last.

---

## 4. The coverage question (the part management cares about most)

**Question:** *Can we assert that current functionality works before we refactor?*

**Answer:** Yes — at the behavioural level, which is the correct safety net for a
refactor — but there is a unit-level gap we must close first.

### What protects us today

Although `Helper.ts` has **no unit tests**, its behaviour is exercised by a
substantial **black-box / behavioural** suite that asserts the *outcomes* of the
flows it powers:

- **28 end-to-end API test files** for Livechat. The transfer/forward/queue
  surfaces are covered with real assertions, including:
  - `livechat/room.forward` — success path **and** authorization failure
    (missing `transfer-livechat-guest` permission).
  - Forwarding to **an agent already at their limit** (must not forward, must not
    add system messages).
  - Forwarding **with the waiting queue disabled**.
  - **Fallback department** validation: not-an-id, non-existent department, and
    same-department error cases.
  - `livechat/visitor/department.transfer` and `livechat/transfer.history/:rid`
    (including empty-history and populated-history cases).
  - The `11-livechat`, `07-queue`, `05-inquiries`, and `01-agents` suites alone
    contain **~279 `it`/`describe` blocks**.
- **66 Omnichannel UI end-to-end (Playwright) specs**, including four dedicated to
  the exact behaviour this file drives: `omnichannel-chat-transfers`,
  `omnichannel-rooms-forward`, `omnichannel-transfer-to-another-agents`, and
  `omnichannel-auto-transfer-unanswered-chat`.

**Translation for management:** the *contract* of these flows — "forward succeeds,
unauthorized is rejected, an over-limit agent is not assigned, a bad fallback
department errors out" — is already asserted automatically. A refactor that
preserves behaviour will keep these green; if it breaks behaviour, **these tests
will catch it in CI before release.**

### The gap, stated honestly

- **No unit tests** isolate the internal branches of `forwardRoomToDepartment`,
  `forwardRoomToAgent`, `createLivechatInquiry`, etc. Integration/e2e tests prove
  *what* the system does, but they are slower and do not pin down *every*
  internal decision path.

### How we close the gap (and turn it into an asset)

Step 1 of the work is to **add characterization (golden-master) unit tests**
around `Helper.ts`'s public functions *before changing any logic* — capturing
current behaviour exactly as it is today. This:

1. Directly satisfies management's requirement to prove prior functionality works.
2. Becomes the regression net the file has never had — a **permanent asset** that
   outlives the refactor.
3. Lets us refactor with confidence: red test = behaviour changed, full stop.

The existing 28 API suites + 66 e2e specs serve as the **outer** safety net; the
new characterization tests are the **inner** net. Refactoring only proceeds once
both are green.

---

## 5. Proposed approach (low-risk, reversible)

A **behaviour-preserving** refactor — no functional change, no API change:

1. **Lock behaviour:** add characterization unit tests for the 16 exported
   functions; confirm the full existing API + e2e suites pass as the baseline.
2. **Split by concern** using the strangler pattern — extract `RoomService`,
   `InquiryService`, `AgentService`, and `TransferService`, moving functions out
   one at a time while keeping the original export signatures as thin
   pass-throughs so the 6 importers need no immediate change.
3. **Tame the giants:** decompose `forwardRoomToDepartment` /
   `forwardRoomToAgent` into named, individually testable steps.
4. **Verify after each step:** characterization + API + e2e suites must stay
   green; each extraction is independently shippable and revertible.

Because every step is incremental and guarded by tests, the work can pause or roll
back at any commit without leaving the system half-migrated.

---

## 6. Effort, and what success looks like

- **Effort:** scored 5/5 (the largest in the module) — plan for a focused,
  multi-sprint initiative, front-loaded by the test-writing phase. Recommend
  time-boxing phase 1 (characterization tests) first; its output is valuable
  *even if the refactor is later deferred*.
- **Sequencing:** this is the *highest-value, highest-effort* item. The companion
  analysis recommends doing low-risk quick wins first; this business case is for
  the strategic, high-severity target management specifically asked about.

### Success metrics (KPIs to report back)

| Metric | Today | Target |
|---|---|---|
| Largest function length | ~198 lines | < 50 lines |
| File size / responsibilities | 964 LOC, 16 mixed fns | Split into ≥4 focused services |
| Unit-test coverage of these flows | 0% | Characterization tests on all public fns |
| Behavioural (API + e2e) suites | Green | Stay green (no regressions) |
| Lead time for a transfer/routing change | Baseline | Reduced (measure over next quarter) |

---

## 7. Recommendation

Approve phase 1 (characterization tests) immediately — it is low-risk, directly
answers the "is current behaviour safe?" question, and delivers a lasting
regression net regardless of what follows. Then proceed with the incremental,
behaviour-preserving split of `Helper.ts`. The existing 28 API suites and 66
Omnichannel e2e specs make this the rare high-severity refactor we can perform
*with an outer safety net already in place* — we are adding the inner net, not
working without one.

---

*Companion document: `livechat-lib-analysis.md` (full candidate scoring across
both modules). All figures here were measured directly against the current
source.*
