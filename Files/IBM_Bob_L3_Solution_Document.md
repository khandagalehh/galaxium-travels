# IBM Bob Level 3 – Stand & Deliver Solution Document

**Basis:** Evaluation Checklist – IBM Bob Level 3 (4 criteria, each graded 1–5)
**Demo environment:** `galaxium-travels` (FastAPI + MCP backend, React/TypeScript frontend)
**Target grade:** 5 on all four criteria

> The client, figures and timelines below are **fictional and illustrative**. The checklist allows a fictional client. Replace the assumption numbers with the client's own data before using them live. Everything described about the code comes from the files in this repo.

---

## 0. Scenario at a glance

| Item | Value |
|---|---|
| Fictional client | **Galaxium Travels**, a mid-size interplanetary travel operator that runs its own booking platform |
| Sponsor persona | VP Engineering / CTO |
| Bob "story" | Bob helps the team understand, fix, test, document and extend a booking system that has no clear owner |
| AI Maturity Curve stage chosen | **Delegate** (primary), with **Tailor** as the next step |
| Demo asset | Galaxium Travels booking system |

### What the code base gives you

| Layer | File(s) | Relevance to the pitch |
|---|---|---|
| REST + MCP server | [`server.py`](booking_system_backend/server.py) | One service layer exposed over REST (`/flights`, `/book`, `/bookings/{user_id}`, `/cancel/{booking_id}`, `/register`, `/user`) and MCP tools (`list_flights`, `book_flight`, `get_bookings`, `cancel_booking`, `register_user`, `get_user_id`). Shows AI-ready architecture. |
| Business logic | [`services/booking.py`](booking_system_backend/services/booking.py), [`services/user.py`](booking_system_backend/services/user.py), [`services/flight.py`](booking_system_backend/services/flight.py) | Booking rules and structured error codes. This is where Bob's analysis and fixes are shown. |
| Data model | [`models.py`](booking_system_backend/models.py), [`db.py`](booking_system_backend/db.py), [`seed.py`](booking_system_backend/seed.py) | SQLite and SQLAlchemy. Seed data is 10 users, 10 flights and 20 bookings. |
| Contracts | [`schemas.py`](booking_system_backend/schemas.py) | Pydantic models, including `ErrorResponse` with `error_code`. |
| Tests | [`tests/test_services.py`](booking_system_backend/tests/test_services.py), [`tests/test_rest.py`](booking_system_backend/tests/test_rest.py) | Existing safety net that Bob extends. |
| Frontend | [`src/services/api.ts`](booking_system_frontend/src/services/api.ts), `src/pages`, `src/components` | React/TypeScript UI. `api.ts` ends with `// Made with Bob`, so Bob generated or touched it. |
| Docs | [`README.md`](README.md), [`booking_system_backend/README.md`](booking_system_backend/README.md) | Have drifted from the code (see §2.3). |

---

## 1. Criterion 1 – Client pain points and value proposition

### 1.1 Pain-point narrative (target: Grade 5)

**Client:** Galaxium Travels runs the booking platform in this repo. It has about 3 engineers who know it. Two of them are leaving or moving teams.

**Pain points, each tied to evidence found in the code:**

| # | Pain | Evidence in the code base |
|---|---|---|
| P1 | **Knowledge loss and slow onboarding.** Docs no longer match the system. | The README documents `/api/flights`, `/api/book` and similar, but [`server.py`](booking_system_backend/server.py) serves `/flights` and `/book` with no `/api` prefix. The README also references `start.bat`, which is not in the repo, and a stray word `doc` at line 134 of the backend README. |
| P2 | **Revenue and trust risk from booking logic.** Rules are incomplete, so bad bookings reach customers. | [`book_flight`](booking_system_backend/services/booking.py) checks seats and then decrements them in separate steps with no row lock. Two simultaneous requests for the last seat can both succeed. `cancel_booking` also accepts `completed` bookings and returns a seat for a flight that has already flown. There is no check that a flight is in the future. |
| P3 | **Weak API contract, which causes support tickets and integration bugs.** | REST endpoints return `ErrorResponse` with **HTTP 200** because errors are returned, not raised. The frontend works around this with `isErrorResponse` in [`api.ts`](booking_system_frontend/src/services/api.ts). |
| P4 | **Security and compliance exposure.** | "Login" is name + email only, with no password or token. CORS is `allow_origins=["*"]` together with `allow_credentials=True`. `seed()` runs on every start in the lifespan hook and **deletes all bookings, users and flights** (`db.query(...).delete()`), which is dangerous if it ever points at real data. |
| P5 | **Slow path to AI-enabled channels.** | The business wants an AI agent to book and cancel on behalf of customers. Errors are raised as bare `Exception(result.details)` in the MCP tools, which gives agents poor structure. |

### 1.2 Quantifying the cost of inaction (illustrative model – replace with client data)

| Assumption | Value |
|---|---|
| Engineers on the platform | 3 |
| Fully loaded cost per engineer | $150k / year |
| Average booking value | $2,000 |
| Bookings per year | 50,000 |
| Booking-defect or double-booking rate | 0.5% |
| Support and refund cost per defect | $400 |
| Onboarding time for a new engineer today | ~6 weeks |

| Cost of doing nothing | Calculation | Annual impact |
|---|---|---|
| Double-booking and refund leakage | 50,000 × 0.5% × $400 | **$100,000** |
| Revenue protection at risk (rebooking and compensation) | 250 defects × $2,000 × 25% goodwill | **$125,000** |
| Onboarding drag | 2 hires × 6 weeks × ($150k / 52) | **~$35,000** |
| Delayed AI channel (opportunity) | Not quantified, but each quarter of delay lets competitors get there first | **qualitative** |
| **Total quantified** | | **~$260,000 / year** |

**Make inaction feel more expensive than change:**
- The knowledge-loss clock is running: two engineers are leaving this quarter.
- Every release made without tests and docs adds risk.
- Bob's demo is quick (see §2). A short proof of value costs far less than one quarter of the leakage above.

### 1.3 Value proposition for IBM Bob

| Client need | How Bob addresses it | Where it shows in the demo |
|---|---|---|
| Understand an unfamiliar system | Bob reads the repo, explains architecture and flags README drift | Walk through `server.py` and the READMEs |
| Fix risky logic quickly | Bob finds the booking-logic gaps and proposes small, reviewable diffs | `services/booking.py` |
| Raise quality | Bob writes and runs tests | `tests/test_services.py` |
| Keep docs true | Bob regenerates docs from code | Both READMEs |
| Prepare for AI agents | Bob improves MCP tool contracts | `server.py` MCP section |
| Stay in control | Human reviews every diff, with modes and permissions | Throughout |

**One-line value statement:** *"Bob lets your remaining engineers understand, harden and extend the booking platform in days instead of quarters, without hiring or a rewrite."*

---

## 2. Criterion 2 – Use cases tied to pain points

### 2.1 Primary use case (robust, end-to-end): "Harden the booking engine without breaking it"

Run this live in the Galaxium repo. It includes an unanticipated obstacle, which is what separates a 5 from a 3.

| Step | Prompt / action (Bob) | Expected outcome | Pain solved |
|---|---|---|---|
| 1. Understand | *"Explain the architecture of this repo and how REST and MCP share business logic."* | Service-layer explanation: `server.py` calls `services/*` for both protocols. | P1 |
| 2. Find risk | *"Review `services/booking.py` for correctness, concurrency and business-rule gaps."* | Bob identifies the double-booking race, cancellation of `completed` bookings, no future-date check, and `datetime.utcnow()` usage. | P2 |
| 3. Baseline | Run `pytest` in `booking_system_backend` | Existing tests pass. This is the safety net. | P2 |
| 4. Test first | *"Add failing tests for: cancelling a completed booking, booking a departed flight."* | New tests fail and prove the gaps are real. | P2 |
| 5. Fix | *"Fix `booking.py` minimally so those tests pass. Keep `ErrorResponse` codes consistent."* | Small diff, for example `INVALID_STATUS` and `FLIGHT_DEPARTED`. | P2 |
| 6. **Obstacle** | Seat decrement is non-atomic. *"Make the seat decrement atomic."* | See §2.2. | P2 |
| 7. Contract | *"Return proper HTTP status codes from the REST endpoints while keeping MCP behavior."* | Endpoints raise `HTTPException` (404/409/400) instead of returning 200 with an error body. The frontend `isErrorResponse` path is updated. | P3 |
| 8. Docs | *"Update both READMEs to match the real endpoints."* | `/api/*` references corrected, stray `doc` removed. | P1 |
| 9. Verify | Re-run `pytest` and `npm run build` | Green. | All |

### 2.2 The "unanticipated obstacle" to narrate

At step 6, a naive fix such as `flight.seats_available -= 1` still has a read-then-write race. In SQLite there is no `SELECT … FOR UPDATE`.

What to show:
1. Bob explains the limitation of SQLite in this environment.
2. Bob proposes an **atomic conditional update**, for example `UPDATE flights SET seats_available = seats_available - 1 WHERE flight_id = ? AND seats_available > 0`, and checks `rowcount`.
3. Bob adds a test for the "last seat" case and runs it.

Message for the client: Bob does not stop at the first answer. It adapts to environment constraints, tests the result, and leaves the human in control.

### 2.3 Supporting use cases (pick one or two if time is short)

| Use case | Evidence in repo | Bob value |
|---|---|---|
| **Documentation repair** | README says `/api/*`; code serves `/flights` etc. | Doc and code drift closed in minutes |
| **Security review** | Wildcard CORS with credentials, name/email "auth", destructive `seed()` on startup | Prioritised findings with fixes, such as guarding `seed()` behind an env flag |
| **AI-agent readiness (MCP)** | Six MCP tools already in [`server.py`](booking_system_backend/server.py) | Bob improves tool descriptions and error structure so agents recover well |
| **Test generation** | Tests exist for services and REST | Bob extends coverage for edge cases (cancel completed, duplicate cancel, bad email) |
| **Modernisation** | `declarative_base` from `sqlalchemy.ext.declarative` and `datetime.utcnow()` | Bob migrates to current SQLAlchemy 2.x and timezone-aware datetimes |

---

## 3. Criterion 3 – AI Maturity Curve

### 3.1 Chosen stage: **Delegate**

| Stage | Meaning | Fit to Galaxium Travels |
|---|---|---|
| Assist | AI suggests and answers | They already use ad-hoc chat tools. This is not the destination. |
| **Delegate** | **AI carries out multi-step tasks under human review** | **Chosen. The team is short of people. They need Bob to do the investigation, edits, test runs and docs, with engineers approving.** |
| Tailor | AI configured to the organisation's own standards and workflows | **Next step**: custom modes, rules and skills encoding their conventions |
| Scale & Extend | AI embedded across teams and connected to enterprise systems and agents | **Later step**: MCP-connected agents booking through the platform |

### 3.2 Why Delegate fits this client

- The pain is capacity and knowledge loss, not lack of ideas. Delegating bounded tasks fits that.
- The repo already has the building blocks for safe delegation: a service layer, tests, typed schemas and a clear error-code vocabulary.
- Every Bob action is reviewable (diffs, test output), which suits a risk-aware CTO.

### 3.3 Narrate the hands-on steps (required by the checklist)

Explain what you did in the lab and why. Use first-person wording during the recording:

1. *"I asked Bob to explain the repo. It mapped REST and MCP to one shared service layer."*
2. *"I delegated a review of `booking.py`. Bob found the race condition and the status-handling gap."*
3. *"I had Bob write failing tests first, then the fix, then I ran `pytest` to confirm."*
4. *"I asked Bob to align the READMEs with the real endpoints."*
5. *"I reviewed every diff before accepting it."*

### 3.4 Progression story (target: Grade 5)

| Horizon | Stage | What happens | Outcome |
|---|---|---|---|
| Now | Assist → **Delegate** | Bob hardens the booking engine, tests and docs | Fewer defects, documented system |
| Next (30–90 days) | **Tailor** | Create custom Bob modes and rules, for example "always return `ErrorResponse` codes", "no `utcnow()`", and a house review checklist | Consistent quality for every engineer |
| Later (6–12 months) | **Scale & Extend** | Roll out to other platforms and connect MCP agents (the Galaxium MCP tools are the template) | AI-assisted channel for customers |

---

## 4. Criterion 4 – Call to action

### 4.1 Specific ask (target: Grade 5)

> *"I'd like to propose a **2-week Proof of Experience** on your booking platform. Bob will run the same hardening workflow I just showed you against your real service layer. Can we lock in the kickoff date today?"*

| Element | Proposal |
|---|---|
| **Offer** | Proof of Experience (PoX) on one booking-related repository |
| **Scope** | (1) architecture explanation and doc refresh, (2) review of booking and cancellation logic, (3) tests for the top 5 defects, (4) fixes via reviewed diffs, (5) one MCP or API contract improvement |
| **Success criteria** | Test coverage up on the targeted module, documented endpoints match code, at least 3 verified defects fixed, onboarding time for a new engineer measured |
| **Duration** | 2 weeks |
| **Owners (client)** | CTO as sponsor, Lead Engineer as day-to-day owner, Security representative for one review session |
| **Owners (IBM)** | Seller as engagement lead, Technical Specialist/Client Engineer to set up Bob |
| **Timeline** | Pre-work call within 1 week. Kickoff at the start of next sprint. Readout at day 10 or 14. |
| **Pre-requisites** | Read access to one repo, a non-production environment, 2 engineers for ~3 hours/week |
| **Tie to pain** | Directly addresses P1–P3 and starts P4 and P5 |

### 4.2 Secure the commitment

Close with an explicit question and fallback:

1. *"Which of your engineers should own this on your side?"*
2. *"Does the sprint starting [date] work for kickoff?"*
3. If the client hesitates: *"A smaller option is a 90-minute hands-on workshop on your own code this month, and we decide on the PoX afterwards."*
4. Confirm by email the same day with owners, dates and scope.

---

## 5. Suggested demo script (~8–10 minutes)

| Time | Section | Checklist criterion |
|---|---|---|
| 0:00–1:30 | Introduce Galaxium Travels. State pains P1–P3 and the ~$260k/year cost. Mention the two departing engineers. | 1 |
| 1:30–2:15 | Bob's value statement. Explain why you are positioning at **Delegate**. | 1, 3 |
| 2:15–7:00 | Live use case (§2.1 steps 1–9), including the atomic-update obstacle. | 2, 3 |
| 7:00–8:00 | Maturity progression: Delegate → Tailor → Scale & Extend, with MCP as the bridge. | 3 |
| 8:00–9:30 | Call to action: 2-week PoX, owners, dates, and the fallback workshop. | 4 |

### Pre-recording checklist

- [ ] Backend runs: `cd booking_system_backend; pip install -r requirements.txt; python server.py` (port 8080)
- [ ] Baseline `pytest` is green before you start
- [ ] Frontend optional: `cd booking_system_frontend; npm install; npm run dev` (port 5173)
- [ ] You have chosen and can say your single maturity stage
- [ ] Assumption numbers adjusted to your fictional client
- [ ] Next-step slide or statement has owner, date and scope

---

## 6. Self-assessment against the checklist

| # | Criterion | Grade 5 requirement | How this plan meets it |
|---|---|---|---|
| 1 | Pain and value | Specific pain, quantified or evidenced, inaction costs more than change | §1.1 code-evidenced pains, §1.2 cost model of ~$260k/year, §1.3 value statement |
| 2 | Use cases | Robust use case tied to Bob and the client's pain, with obstacles | §2.1 nine-step use case, §2.2 atomic-update obstacle |
| 3 | AI Maturity Curve | Stage chosen, tied to the client's current position and next steps | §3 Delegate chosen with reasoning, then Tailor, then Scale & Extend, and lab steps narrated |
| 4 | Call to action | Specific next step with owners, timing, scope, and commitment secured | §4 2-week PoX, named owners, dates, success criteria, fallback and confirmation |

---

## 7. Caveats

- Financial figures in §1.2 are assumptions, not IBM or client data. Say so in the recording, or replace them with the client's own numbers.
- The code review findings (race condition, `completed` bookings being cancellable, HTTP 200 on errors, wildcard CORS with credentials, destructive `seed()` on startup, README drift) were read directly from the source files. The fixes in §2.1 are proposals for Bob to produce live and have **not** been applied to this repo.
- `start.bat` is referenced in the README but only `start.sh` exists in the repository.
