# Backend Team Plan
### EdTech Platform MVP — Internship Sprint

**Lead:** Kartike  
**Team:** 6 Interns  
**Deadline:** 2 Days  
**Stack:** FastAPI · PostgreSQL · SQLAlchemy · Alembic · JWT · Razorpay · Docker

---

## Project Overview

An EdTech platform where a student can discover a course, purchase it, get enrolled automatically, and track their learning progress. The backend is the single source of truth for all data — frontend and mobile app consume REST APIs only.

**MVP user flow:**
```
User registers → Logs in → Browses courses → Creates payment order
→ Razorpay processes payment → Webhook hits backend → Signature verified
→ Enrollment created → User accesses course → Progress tracked
```

---

## Project Structure

```
project/
├── app/
│   ├── main.py                  ← Intern 1
│   ├── db/
│   │   └── database.py          ← Intern 1
│   ├── models/
│   │   ├── __init__.py
│   │   └── models.py            ← Intern 2
│   ├── core/
│   │   └── auth.py              ← Intern 3
│   ├── schemas/
│   │   └── user.py              ← Intern 3
│   └── routes/
│       ├── user_router.py       ← Intern 4
│       ├── payment_router.py    ← Intern 5
│       └── course_router.py     ← Intern 6
├── alembic/                     ← Intern 2
├── .env                         ← Intern 1
├── .env.example                 ← Intern 1
├── requirements.txt             ← Intern 1
└── README.md                    ← Intern 1
```

---

## Team Task Assignments

### Kajal — Project Skeleton + DB Connection
**Files:** `app/db/database.py`, `app/main.py`, `.env`, `requirements.txt`, `README.md`

**Tasks:**
- Create the full folder structure as shown above
- Write `database.py`: SQLAlchemy engine using `DATABASE_URL` from `.env`, `SessionLocal` via `sessionmaker`, and a `get_db()` dependency that yields a session and closes it
- Write `main.py`: create FastAPI app instance, add CORS middleware, placeholder router includes (commented), `GET /health` returning `{"status": "ok"}`
- Write `.env` with `DATABASE_URL` and `SECRET_KEY` keys
- Write `.env.example` with empty placeholder values
- Write `requirements.txt` with all dependencies (see stack below)
- Write `README.md` with setup steps: clone, venv, `.env` config, `uvicorn` run command

**Definition of done:** Server starts with `uvicorn app.main:app --reload` and `/health` returns 200.

**Dependency:** None. Start immediately. This is the foundation — push before anyone else starts.

---

### Anshul — Database Models + Migrations
**Files:** `app/models/models.py`, `app/models/__init__.py`, `alembic/`

**Tasks:**
- Import `Base` from `app.db.database`
- Define all 5 SQLAlchemy models:

| Model | Key Fields |
|---|---|
| `User` | id, name, email, hashed_password, role, is_active, created_at |
| `Course` | id, title, description, price, instructor_id (FK→User), created_at |
| `Enrollment` | id, user_id (FK→User), course_id (FK→Course), status, enrolled_at |
| `Payment` | id, user_id (FK), course_id (FK), razorpay_order_id, razorpay_payment_id, amount, status, created_at |
| `Progress` | id, user_id (FK), lesson_id, completed (bool), updated_at |

- Define all foreign key relationships
- Run `alembic init alembic` and configure `env.py` to point at `Base.metadata`
- Generate first migration: `alembic revision --autogenerate -m "initial_tables"`
- Apply migration: `alembic upgrade head`
- Export all models in `models/__init__.py`

**Definition of done:** All 5 tables visible in pgAdmin after migration.

**Dependency:** Needs Intern 1's `database.py` for `Base`. Can stub locally until Intern 1 pushes.

---

### Bhoomi — Auth Utilities + Pydantic Schemas
**Files:** `app/core/auth.py`, `app/schemas/user.py`

**Tasks:**

Auth utility functions (`auth.py`):
- `hash_password(plain: str) -> str` — bcrypt hash via passlib
- `verify_password(plain: str, hashed: str) -> bool`
- `create_access_token(data: dict) -> str` — JWT signed with `SECRET_KEY`, 30 min expiry
- `get_current_user(token: str = Depends(...), db: Session = Depends(get_db)) -> User` — decode JWT, fetch user from DB, raise `401` if invalid

Pydantic schemas (`schemas/user.py`):
- `SignupRequest` — name, email, password
- `LoginRequest` — email, password
- `TokenResponse` — access_token, token_type
- `UserResponse` — id, name, email (never include hashed_password in responses)

**Definition of done:** `hash_password` + `verify_password` unit tested manually in a Python shell. `create_access_token` returns a decodable JWT.

**Dependency:** Needs Intern 2's `User` model for `get_current_user`. Stub the return type until Intern 2 pushes.

---

### Krishna — Auth Routes
**Files:** `app/routes/user_router.py`

**Tasks:**
- `POST /auth/signup` — validate input via `SignupRequest`, check for duplicate email (raise `400`), call `hash_password`, save `User` to DB, return `UserResponse`
- `POST /auth/login` — validate via `LoginRequest`, call `verify_password`, call `create_access_token`, return `TokenResponse`
- `GET /auth/me` — protected route using `Depends(get_current_user)`, return `UserResponse` of current user

**Rules:**
- Call Intern 3's functions directly — do not rewrite hash or JWT logic
- Use Intern 3's schemas for all request/response types
- Test all 3 endpoints in Postman before marking done

**Definition of done:** Signup → Login → `/auth/me` full flow works end-to-end in Postman.

**Dependency:** Needs Intern 2 (User model) and Intern 3 (auth.py + schemas).

---

### Khushi — Razorpay Payment Routes
**Files:** `app/routes/payment_router.py`

**Tasks:**
- `POST /payment/create-order` *(protected)* — accept `course_id`, call Razorpay API to create an order, save a `Payment` record with `status=pending`, return `order_id` and `amount` to frontend
- `POST /payment/webhook` — receive Razorpay webhook payload, **verify HMAC-SHA256 signature** (mandatory), update `Payment` status to `success`, create `Enrollment` record automatically
- `GET /payment/status/{order_id}` — return payment status and enrollment status from DB

**Critical rules:**
- Signature verification on the webhook is **non-negotiable**. The task is considered incomplete without it. Use `hmac.compare_digest` for the comparison
- `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET` must live in `.env` only — never hardcoded in any file
- Protect `create-order` with `Depends(get_current_user)`
- Never trust frontend payment confirmation — backend verification is the only source of truth

**Definition of done:** Webhook receives a test payload, verifies signature, and creates an Enrollment record in the DB.

**Dependency:** Needs Intern 2's `Payment` + `Enrollment` models and Intern 3's `get_current_user`.

---

### Nandini — Course + Progress Routes
**Files:** `app/routes/course_router.py`

**Tasks:**
- `GET /courses` *(public)* — list all courses with title, description, price
- `GET /courses/{id}` *(public)* — full course details
- `GET /my-courses` *(protected)* — return courses the current user is enrolled in, joined via `Enrollment` table
- `POST /progress/update` *(protected)* — accept `lesson_id`, mark `Progress` record as completed for current user
- `GET /progress/{course_id}` *(protected)* — return completion percentage: `(completed lessons / total lessons) × 100`

Pydantic schemas to write:
- `CourseResponse` — id, title, description, price
- `ProgressUpdate` — lesson_id
- `ProgressResponse` — course_id, percent_complete

**Definition of done:** `/courses` returns seeded course data. `/my-courses` returns an enrolled course after a test payment. Progress updates correctly.

**Dependency:** Needs Intern 2's `Course`, `Enrollment`, `Progress` models and Intern 3's `get_current_user`.

---

## Timeline

### Day 1

| Time | Intern 1 | Intern 2 | Intern 3 | Intern 4 | Intern 5 | Intern 6 |
|---|---|---|---|---|---|---|
| 0–2 hrs | Build skeleton, push to GitHub | Write models (stub Base locally) | Write auth utils + schemas (stub User) | Study Intern 3's code, set up env | Study Razorpay docs, set up env | Study Intern 2's models, set up env |
| 2–4 hrs | ✅ Done. Available for review | Integrate real Base from Intern 1, run Alembic | Integrate real User model from Intern 2 | Wait for Intern 2+3, plan routes | Stub payment routes with mock Razorpay | Start course listing routes (public) |
| 4–6 hrs | Code review + assist team | ✅ Done — migrations applied | ✅ Done — auth utils tested in shell | Build all 3 auth endpoints | Implement create-order + DB write | Implement my-courses + progress |
| 6–8 hrs | Integration review | Seed DB with test data (courses, users) | Test `get_current_user` dependency | Test signup/login/me in Postman | Implement webhook + signature verify | Test progress endpoints end-to-end |

### Day 2

| Time | All Interns | Kartike (Lead) |
|---|---|---|
| 0–2 hrs | Fix issues from Day 1 testing | Review all PRs, merge to main |
| 2–4 hrs | Write Postman collections for own routes | Wire all routers into `main.py`, run full migration |
| 4–6 hrs | Cross-test: each intern tests another's endpoints | End-to-end flow test: signup → purchase → enroll → progress |
| 6–8 hrs | Bug fixes, clean up code, add inline comments | Final review, deployment check |

---

## Dependency Map

```
Intern 1 (database.py)
    └── Intern 2 (models.py)
            ├── Intern 3 (auth.py)  ← also needs Intern 1
            │       ├── Intern 4 (user_router.py)
            │       ├── Intern 5 (payment_router.py)
            │       └── Intern 6 (course_router.py)
            ├── Intern 5 (Payment + Enrollment models)
            └── Intern 6 (Course + Progress models)
```

**Rule:** No intern merges to `main` until their direct dependency is merged first.

---

## Git Workflow

```
main
├── feature/skeleton-db          ← Intern 1
├── feature/models-migrations    ← Intern 2
├── feature/auth-utils           ← Intern 3
├── feature/auth-routes          ← Intern 4
├── feature/payment-routes       ← Intern 5
└── feature/course-progress      ← Intern 6
```

**Branch rules:**
- One branch per intern, named exactly as above
- No direct pushes to `main`
- All merges via Pull Request
- Kartike reviews and merges all PRs
- Commit messages: `feat: add get_db dependency`, `fix: handle duplicate email on signup`

---

## API Contract Summary

| Method | Endpoint | Auth | Owner |
|---|---|---|---|
| GET | `/health` | Public | Intern 1 |
| POST | `/auth/signup` | Public | Intern 4 |
| POST | `/auth/login` | Public | Intern 4 |
| GET | `/auth/me` | Bearer token | Intern 4 |
| GET | `/courses` | Public | Intern 6 |
| GET | `/courses/{id}` | Public | Intern 6 |
| GET | `/my-courses` | Bearer token | Intern 6 |
| POST | `/payment/create-order` | Bearer token | Intern 5 |
| POST | `/payment/webhook` | Razorpay signature | Intern 5 |
| GET | `/payment/status/{order_id}` | Bearer token | Intern 5 |
| POST | `/progress/update` | Bearer token | Intern 6 |
| GET | `/progress/{course_id}` | Bearer token | Intern 6 |

---

## Requirements

```
fastapi
uvicorn[standard]
sqlalchemy
alembic
psycopg2-binary
python-dotenv
python-jose[cryptography]
passlib[bcrypt]
pydantic[email]
razorpay
```

---

## Environment Variables

```
# .env.example

DATABASE_URL=postgresql://user:password@localhost:5432/edtech_db
SECRET_KEY=your_secret_key_here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxx
RAZORPAY_KEY_SECRET=your_razorpay_secret
```

---

## Evaluation Criteria (per the assignment doc)

| Criteria | What to check |
|---|---|
| Task completion | Does their endpoint work end-to-end? |
| Code understanding | Can they explain every line — why `Depends(get_db)`, what `yield` does, why HMAC verification? |
| Explanation skills | Can they walk through the request lifecycle for their route? |
| Contribution level | Did they only do their file, or did they help others debug? |
| Team involvement | Did they communicate blockers early? Did they test others' work? |

---

## Lead Checklist (Kartike)

- [ ] GitHub repo created, all interns added as collaborators
- [ ] Branch protection on `main` — no direct push
- [ ] Intern 1 pushes skeleton within first 2 hours
- [ ] Alembic migration verified in pgAdmin before Day 1 ends
- [ ] Intern 5 webhook has signature verification — reviewed personally
- [ ] All routers wired in `main.py` by Day 2 midpoint
- [ ] Full flow tested: signup → login → browse → purchase → enroll → progress
- [ ] All branches merged, final code on `main` before deadline

---

*Deadline: 2 days from start date. All tasks submitted as GitHub PR links.*

---

# Sprint 2
# Backend Team — Sprint 2 Task Assignments
### Baoiam EdTech Platform — Website Go-Live
**Lead:** Kartike  
**Team:** Kajal, Anshul, Bhoomi, Krishna, Khushi, Nandini  
**Deadline:** 7 Days  
**Goal:** Auth (Phone OTP), Courses API, Inquiry API live and integrated

---

## What Changed From Sprint 1

| Intern | Sprint 1 | Sprint 2 |
|---|---|---|
| Kajal | Project skeleton | No change — maintain infra |
| Anshul | DB models | Update User model + add OTP model |
| Bhoomi | Email/password auth utils | Rewrite for OTP flow + MSG91 |
| Krishna | Email/password routes | Rewrite for send-otp + verify-otp |
| Khushi | Razorpay routes | Course catalog + detail APIs |
| Nandini | Course + progress routes | Inquiry form + platform stats API |

---

## New Auth Flow

```
User enters phone number
→ POST /auth/send-otp     → generate 6-digit OTP, store in DB with 10 min expiry, send via MSG91
→ User enters OTP
→ POST /auth/verify-otp   → verify OTP, mark used, create User if new, return JWT
→ All protected routes use same Bearer token as before
```

**Rule:** OTP expires in 10 minutes. Used OTPs cannot be reused. Never log OTP values.

---

## Task Assignments

---

### Kajal — Infrastructure + Environment
**Same files as Sprint 1. New responsibility: keep everything running for others.**

Tasks:
- Add `MSG91_API_KEY`, `MSG91_SENDER_ID`, `MSG91_TEMPLATE_ID` to `.env` and `.env.example`
- Add `twilio` or `requests` (for MSG91 HTTP call) to `requirements.txt`
- Ensure DB connection still works after Anshul's model changes
- Run `alembic upgrade head` after Anshul pushes new migration
- Keep `GET /health` live at all times — others depend on it for testing

**Definition of done:** Server starts cleanly with new env vars. Migration applies without errors.

---

### Anshul — Database Model Updates + New OTP Model
**Files:** `app/models/models.py`, new Alembic migration

**Changes to User model:**
- Remove: `hashed_password`
- Add: `phone_number` (String, unique, not null)
- Add: `is_verified` (Boolean, default False)
- Keep: id, name, email (optional now), role, created_at

**New OTP model:**
```
OTP
├── id           (Integer, primary key)
├── phone_number (String, indexed)
├── otp_code     (String 6)
├── expires_at   (DateTime)
├── is_used      (Boolean, default False)
└── created_at   (DateTime)
```

**Also:**
- Generate new Alembic migration: `alembic revision --autogenerate -m "otp_auth_update"`
- Apply: `alembic upgrade head`
- Export `OTP` model from `models/__init__.py`

**Definition of done:** Both tables updated/created in pgAdmin. No orphan columns.

---

### Bhoomi — OTP Utilities + Updated Schemas
**Files:** `app/core/auth.py`, `app/schemas/user.py`

**Remove entirely:**
- `hash_password()`
- `verify_password()`

**Keep as-is:**
- `create_access_token(data: dict) -> str`
- `get_current_user(token, db) -> User`

**Add — OTP utility functions:**
- `generate_otp() -> str` — returns random 6-digit string
- `send_otp_msg91(phone: str, otp: str) -> bool` — HTTP POST to MSG91 API, returns True if sent
- `is_otp_valid(otp_record: OTP) -> bool` — checks `is_used == False` and `expires_at > now()`

**MSG91 API call (inside `send_otp_msg91`):**
```python
import requests

def send_otp_msg91(phone: str, otp: str) -> bool:
    url = "https://api.msg91.com/api/v5/otp"
    payload = {
        "mobile": f"91{phone}",
        "otp": otp,
        "template_id": os.getenv("MSG91_TEMPLATE_ID"),
        "authkey": os.getenv("MSG91_API_KEY"),
    }
    response = requests.post(url, json=payload)
    return response.status_code == 200
```

**Updated Pydantic schemas:**
- Remove: `SignupRequest`, `LoginRequest`
- Add: `SendOTPRequest` — phone_number (10 digit string, validated)
- Add: `VerifyOTPRequest` — phone_number, otp_code
- Add: `OTPResponse` — message: str
- Keep: `TokenResponse`, `UserResponse`

**Definition of done:** `generate_otp()` returns 6 digits. `is_otp_valid()` correctly rejects expired/used OTPs. MSG91 call tested with a real phone number.

---

### Krishna — Auth Routes (Full Rewrite)
**Files:** `app/routes/user_router.py`

**Remove entirely:**
- `POST /auth/signup`
- `POST /auth/login`

**New endpoints:**

`POST /auth/send-otp`
```
Input:  SendOTPRequest { phone_number }
Logic:
  1. Validate phone (10 digits, numeric)
  2. Call generate_otp() → get 6-digit code
  3. Save OTP record to DB: { phone_number, otp_code, expires_at = now + 10min, is_used = False }
  4. Call send_otp_msg91(phone, otp)
  5. Return { "message": "OTP sent successfully" }
  
Errors:
  - Invalid phone format → 400
  - MSG91 failure → 503 "SMS service unavailable"
```

`POST /auth/verify-otp`
```
Input:  VerifyOTPRequest { phone_number, otp_code }
Logic:
  1. Fetch latest unused OTP for this phone from DB
  2. Call is_otp_valid(otp_record) — reject if expired or used
  3. Mark otp_record.is_used = True
  4. Check if User with this phone exists:
       - Yes → fetch user
       - No  → create new User { phone_number, is_verified=True }
  5. Call create_access_token({ "sub": str(user.id) })
  6. Return TokenResponse { access_token, token_type: "bearer" }

Errors:
  - OTP not found → 404
  - OTP expired   → 400 "OTP expired"
  - OTP already used → 400 "OTP already used"
  - Wrong OTP     → 400 "Invalid OTP"
```

`GET /auth/me` — keep exactly as before (protected via `get_current_user`)

**Definition of done:** Full flow tested in Postman — send OTP to real number, receive SMS, verify, get JWT, call `/auth/me` with token.

---

### Khushi — Course Catalog + Detail APIs
**Files:** `app/routes/course_router.py` (new responsibility)

**Context from design:**
4 course categories visible: Web Development, Data Science, UI/UX Design, Digital Marketing.
Each course has: title, description, category, price, avg salary, avg time to hire, student rating, tech stack tags, and modules list.

**Endpoints:**

`GET /courses`
```
Query params: ?category=web-development (optional filter)
Returns: list of courses with { id, title, category, price, avg_salary, 
         avg_time_to_hire, student_rating, tech_tags[] }
Public — no auth required
```

`GET /courses/{id}`
```
Returns: full course detail including modules list
{ id, title, description, category, price, avg_salary, avg_time_to_hire,
  student_rating, tech_tags[], modules: [{ number, title, duration_hours }] }
Public — no auth required
```

**Also:**
- Add `Module` model to Anshul's models if not already there: id, course_id (FK), number, title, duration_hours
- Write Pydantic schemas: `CourseListResponse`, `CourseDetailResponse`, `ModuleResponse`
- Seed at least 2 courses with modules in DB for testing

**Definition of done:** `GET /courses` returns seeded courses. `GET /courses?category=data-science` filters correctly. `GET /courses/1` returns full detail with modules.

---

### Nandini — Platform Stats API + Inquiry API
**Files:** `app/routes/stats_router.py`, `app/routes/inquiry_router.py`

**Stats API** (from hero section of design):
`GET /stats`
```
Returns hardcoded for now (update from DB later):
{
  "students": "200K+",
  "courses": "500+", 
  "industry_connections": "200+",
  "hiring_partners": "100+",
  "satisfaction_percent": "95%"
}
Public — no auth required
```

**Inquiry API:**

New DB model needed (tell Anshul):
```
Inquiry
├── id         (Integer, primary key)
├── name       (String)
├── phone      (String)
├── email      (String, optional)
├── message    (String)
└── created_at (DateTime)
```

`POST /inquiry`
```
Input: { name, phone, email (optional), message }
Logic: validate input, save to DB, return { "message": "Inquiry submitted successfully" }
Public — no auth required
```

`GET /inquiry` (admin only — protect with get_current_user + role check)
```
Returns: list of all inquiries (for admin panel later)
```

Pydantic schemas: `InquiryRequest`, `InquiryResponse`

**Definition of done:** `POST /inquiry` saves to DB. `GET /stats` returns all 5 values. Verified in Postman.

---

## Updated API Contract

| Method | Endpoint | Auth | Owner |
|---|---|---|---|
| GET | `/health` | Public | Kajal |
| POST | `/auth/send-otp` | Public | Krishna |
| POST | `/auth/verify-otp` | Public | Krishna |
| GET | `/auth/me` | Bearer token | Krishna |
| GET | `/courses` | Public | Khushi |
| GET | `/courses/{id}` | Public | Khushi |
| GET | `/stats` | Public | Nandini |
| POST | `/inquiry` | Public | Nandini |
| GET | `/inquiry` | Bearer token (admin) | Nandini |
| POST | `/payment/create-order` | Bearer token | Khushi (Sprint 1) |
| POST | `/payment/webhook` | Razorpay signature | Khushi (Sprint 1) |
| GET | `/payment/status/{id}` | Bearer token | Khushi (Sprint 1) |

---

## Dependency Map

```
Kajal (infra + env vars)
    └── Anshul (model updates + Inquiry model)
            ├── Bhoomi (OTP utils — needs OTP model)
            │       └── Krishna (auth routes — needs OTP utils)
            ├── Khushi (needs Course + Module models)
            └── Nandini (needs Inquiry model)
```

---

## 7-Day Timeline

| Day | Kajal | Anshul | Bhoomi | Krishna | Khushi | Nandini |
|---|---|---|---|---|---|---|
| 1 | Update .env, requirements.txt | Update User model, add OTP model | Remove old utils, write generate_otp + is_otp_valid | Study new flow, set up MSG91 test account | Read Course model, plan schemas | Plan stats + inquiry endpoints |
| 2 | Run migration after Anshul | Generate + apply migration | Write send_otp_msg91, update schemas | Write send-otp endpoint | Write GET /courses with filter | Write GET /stats |
| 3 | Test DB connection | Seed 2 courses with modules | Test OTP utils in Python shell | Write verify-otp endpoint | Write GET /courses/{id} | Write POST /inquiry + model |
| 4 | Code review support | Help Khushi with Module model | Test full OTP flow with real SMS | Test full flow in Postman | Seed data, test both endpoints | Write GET /inquiry (admin) |
| 5 | Integration — wire all routers in main.py | — | — | Fix issues from integration | Fix issues from integration | Fix issues from integration |
| 6 | End-to-end test full flow | — | — | Re-test auth after integration | Re-test course APIs | Re-test stats + inquiry |
| 7 | Buffer — bug fixes + deployment to Railway/Render | — | — | — | — | — |

---

## MSG91 Setup (Bhoomi + Krishna)

1. Create account at msg91.com
2. Get API key from dashboard
3. Create an OTP template — note the `template_id`
4. Add to `.env`:
```
MSG91_API_KEY=your_key_here
MSG91_SENDER_ID=BAOIAM
MSG91_TEMPLATE_ID=your_template_id
```
5. Test with a real Indian mobile number before integrating with Krishna's route

---

## Git Branches

```
main
├── fix/otp-auth-models       ← Anshul
├── fix/otp-auth-utils        ← Bhoomi  
├── fix/auth-routes-otp       ← Krishna
├── feature/course-apis       ← Khushi
└── feature/stats-inquiry     ← Nandini
```

---

## Lead Checklist (Kartike)

- [ ] MSG91 account created, API key shared with Bhoomi securely (not over chat)
- [ ] Anshul's migration applied cleanly before others start
- [ ] Bhoomi tests OTP send with real phone before Krishna starts routes
- [ ] Krishna's verify-otp correctly marks OTP as used — test this personally
- [ ] All routers wired in main.py on Day 5
- [ ] Full flow tested: send OTP → verify → get JWT → call /auth/me → browse courses
- [ ] Deployed to Railway/Render on Day 7

---

*Sprint 2 — 7 day deadline. All tasks submitted as PRs to main repo.*
