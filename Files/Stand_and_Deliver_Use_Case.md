# Stand & Deliver Use Case: "Stop customers cancelling flights they've already flown"

**Client (fictional):** Galaxium Travels
**Demo asset:** Galaxium Travels booking system (FastAPI + MCP backend, React/TypeScript frontend)
**Checklist:** IBM Bob Level 3 (4 criteria, each graded 1–5)
**Recording length:** 8–10 minutes

This is one use case sized for a single recording. Galaxium Travels runs the booking platform in this repo and has an AI assistant project on its roadmap.

---

## 1. Pain and value (Criterion 1, about 1.5 min)

**What to say:**
> "Galaxium Travels is about to let an AI assistant cancel bookings through its MCP tools. Today a customer, or an agent, can cancel a booking that is already `completed` and the system gives the seat back on a flight that has already flown. Two of the three engineers who know this code are leaving this quarter."

**Evidence in the code:**
- [`cancel_booking`](booking_system_backend/services/booking.py:57) only rejects bookings with status `cancelled`.
- [`seed.py`](booking_system_backend/seed.py:47) creates bookings with `booked`, `cancelled` and `completed` statuses, so `completed` bookings really exist in the demo data.
- The `cancel_booking` MCP tool in [`server.py`](booking_system_backend/server.py:58) calls the same function. An AI agent would inherit the bug.

**Cost of inaction (illustrative, say so in the video and swap in real numbers if you have them):**

| Item | Calculation | Annual impact |
|---|---|---|
| Wrongly cancelled bookings, refunds and seat inventory errors | 50,000 bookings × 0.5% × $400 | $100k |
| Goodwill and rebooking | 250 cases × $2,000 × 25% | $125k |
| Onboarding drag from outdated docs | 2 hires × 6 weeks × ($150k / 52) | about $35k |
| **Total** | | **about $260k a year** |

**Value line:**
> "Bob lets the engineers who are staying find, prove and fix this in minutes, and every change is a diff they review."

---

## 2. The use case (Criterion 2, about 4–5 min live)

| # | Say and do | Bob action | Expected result |
|---|---|---|---|
| 1 | "First, I ask Bob to explain the system." | Prompt: *"Explain this repo and how REST and MCP share business logic."* | Bob explains that [`server.py`](booking_system_backend/server.py) is a thin layer over `services/*`, so one fix covers both REST and MCP. |
| 2 | "Now I ask Bob to review the cancellation rules." | Prompt: *"Review `cancel_booking` for business-rule gaps."* | Bob flags that `completed` bookings can be cancelled and that the seat is returned anyway. |
| 3 | "Let me prove it is real." | Run `python server.py`. Call `GET /bookings/{user_id}` until you find a `completed` booking, then `POST /cancel/{booking_id}`. | The API accepts the cancellation. Seed data is random, so check ahead of time which user has a `completed` booking. |
| 4 | "Tests first." | Prompt: *"Add a failing test to `tests/test_services.py` for cancelling a completed booking."* Run `pytest`. | The new test fails and confirms the bug. |
| 5 | "Now the fix." | Prompt: *"Fix `cancel_booking` minimally. Reuse the `ErrorResponse` pattern."* | Small diff, for example error code `INVALID_STATUS` for any status other than `booked`. |
| 6 | **Obstacle:** "Bob suggested returning a proper HTTP error, but I want to check what that does to the UI." | Prompt: *"Should this return a 4xx instead of 200?"* | Bob inspects [`api.ts`](booking_system_frontend/src/services/api.ts). The frontend's `isErrorResponse` checks `success === false`, and the response interceptor passes `error.response.data` through. A bare FastAPI `HTTPException` would return `{detail: …}` and break that handling. Bob keeps the existing `ErrorResponse` shape and flags the 4xx change as a follow-up. |
| 7 | "Verify." | Run `pytest`. Repeat the cancel call from step 3. | All tests pass. The same call now returns `INVALID_STATUS` and seat counts are unchanged. |
| 8 | "Last, docs." | Prompt: *"Update the backend README to match the real endpoints."* | `/api/*` references become `/flights`, `/book` and so on, and the stray `doc` on line 134 is removed. |

**Why step 6 matters:** it is the unexpected obstacle that the checklist asks for. Bob checks the impact on another part of the system before changing it, instead of applying the first idea.

---

## 3. AI Maturity Curve (Criterion 3, about 1 min)

**Stage chosen: Delegate.**

> "At this stage the client gives Bob a whole task: understand, review, test, fix, document. The engineers approve each step. That fits a team that is losing people and can't afford long investigations."

**Narrate the hands-on steps** as you do them: explain, review, reproduce, failing test, minimal fix, impact check on the frontend, verification, docs.

**Progression:**
- **Now: Delegate.** Bob closes the cancellation gap and the doc drift.
- **Next: Tailor.** Custom Bob modes and rules encode Galaxium Travels standards, for example "all business-rule failures return an `ErrorResponse` with an `error_code`".
- **Later: Scale & Extend.** The MCP tools become the safe channel for customer-facing AI agents, and the same approach is applied to other platforms.

---

## 4. Call to action (Criterion 4, about 1 min)

**What to say:**
> "I'd like to propose a **2-week Proof of Experience** on your booking service layer. We run this same workflow on your real code, targeting the top booking and cancellation defects. Can we lock in a kickoff date today?"

| Element | Detail |
|---|---|
| Scope | One booking repo: architecture explanation, review of booking and cancellation logic, tests plus fixes for the top defects, README refresh. |
| Success criteria | At least 3 verified defects fixed, more tests on the booking module, docs matching the code. |
| Owners | Galaxium Travels: CTO as sponsor, Lead Engineer as day-to-day owner. IBM: seller as engagement lead, technical specialist for Bob setup. |
| Timing | Pre-work call within 1 week, kickoff next sprint, readout on day 10–14. |
| Tie to pain | The cancellation bug, the unreviewed AI-agent exposure, and the knowledge loss. |
| Securing commitment | *"Which engineer owns this, and does the sprint starting [date] work?"* If they hesitate, offer a 90-minute hands-on workshop on their code first. Confirm by email the same day. |

---

## Recording checklist

- [ ] Backend runs (`pip install -r requirements.txt`, `python server.py`) and baseline `pytest` passes before you record.
- [ ] You know ahead of time which user ID has a `completed` booking. The seed is random, so check after each restart.
- [ ] You have said **Delegate** out loud and why.
- [ ] You have said that the dollar figures are illustrative assumptions.
- [ ] The closing slide or statement names owners, dates and scope.

---

## Checklist mapping

| Criterion | Where it's covered |
|---|---|
| 1. Pain and value | Section 1: code-based evidence, cost model, value line |
| 2. Use case | Section 2: eight steps with the frontend-impact obstacle at step 6 |
| 3. AI Maturity Curve | Section 3: Delegate chosen, then Tailor, then Scale & Extend |
| 4. Call to action | Section 4: 2-week Proof of Experience with owners, timing and scope |

---

## Notes

- No code has been changed. The fixes above are what Bob would produce live.
- [`tests/test_rest.py`](booking_system_backend/tests/test_rest.py) and the frontend page components were not reviewed. Run `pytest` before recording to confirm the baseline.
- The 4xx status-code change and an atomic seat update (`UPDATE … WHERE seats_available > 0`) are good "what's next" talking points.
- A fuller version covering more use cases is in [`IBM_Bob_L3_Solution_Document.md`](IBM_Bob_L3_Solution_Document.md).
