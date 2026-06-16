# Contribution #1: Add "Change Email" Feature

**Contribution Number:** 1  
**Student:** Sai Pavan Reddy Doddam Reddy  
**Issue:** https://github.com/321-Vegan/321vegan-api/issues/76  
**Status:** Phase III Complete

---

## Why I Chose This Issue

**Why this issue:** Right now users on 321 Vegan have no way to change their email. If they mistyped it on signup or switched email providers, they're stuck. It's a real gap in a real product. The issue is also well scoped - it's not a vague "improve the account system" request, it's one specific feature with clear boundaries: two endpoints, a database migration, and an email service. I can plan the work from day one without needing weeks of codebase exploration first.

**Why me:** I've been studying FastAPI and Python backend development through coursework and personal projects, and this issue hits most of what I've been learning. I've worked with Pydantic and SQLAlchemy in class projects, I've run Alembic migrations before, and I understand how auth flows are supposed to work even though I haven't built one in a real codebase yet. The maintainer also pointed out that there's an existing password reset flow in the codebase that uses the exact same pattern (token generation, expiry, email confirmation), so I can study working code before writing anything.

**What "fixed" means:** The issue is resolved when a logged-in user can hit PATCH `/me/email` with their current password and a new email address, receive a confirmation link at the new address, click it to confirm the swap, and get a notification at their old address that the change went through. On the backend that means three new database columns on the User model, an Alembic migration, a Pydantic request schema, two CRUD methods, two email service methods, and two route handlers. Wrong password returns 401, duplicate email returns 409, bad or expired token returns 400. If all of that works and the maintainer's 6-item checklist is complete, it's done.

---

## Understanding the Issue

### Problem Description

Users on the 321 Vegan platform have no way to update their email address. If someone needs to change their email, there's no self-service option. This feature adds that.

### Expected Behavior

A logged-in user sends a PATCH request to `/me/email` with their current password and new email. The system checks the password, makes sure the new email isn't already taken, generates a confirmation token, and sends a verification link to the new address. When the user clicks that link (GET `/me/email/confirm?token=...`), the system validates the token, swaps the email, and sends a heads-up to the old address.

### Current Behavior

No endpoint exists for email changes. Users are stuck with whatever email they signed up with.

### Affected Components

- `src/app/models/user.py` (3 new columns: `pending_email`, `email_change_token`, `email_change_expires`)
- `src/app/schemas/auth.py` (new `EmailChangeRequest` Pydantic schema)
- `src/app/crud/user.py` (two new CRUD methods)
- `src/app/services/email.py` (two new email methods)
- `src/app/routes/account.py` (two new endpoints)
- Alembic migration file

---

## Reproduction Process

### Environment Setup

Cloned my fork and set up the project using Docker Compose. Ran into two issues during setup:

1. **Port 5432 conflict:** My Mac already had something using port 5432 so the PostgreSQL container wouldn't start. Fixed it by changing the port mapping in `docker-compose.yml` to `"5433:${POSTGRES_PORT}"` so Docker uses port 5433 externally while the containers still communicate on 5432 internally.

2. **Pydantic validation error on startup:** The API container crashed with `ValidationError: APPLE_APP_ID — Input should be a valid integer, unable to parse string as an integer`. The `.env` file had `APPLE_APP_ID=` set to an empty string but the config expects an integer or null. Fixed by commenting out the line in `.env`.

After that, ran `poetry run alembic upgrade head` and `poetry run python -m scripts.create_admin_user` inside the API container to set up the database and create the admin account. App is live at `http://localhost:8000/docs`.

### Steps to Reproduce

1. Start the project with `docker compose up`
2. Open `http://localhost:8000/docs` (FastAPI Swagger UI)
3. Browse through all available endpoints under the `/me` account section
4. Look for any endpoint that allows changing a user's email address
5. **Expected:** There should be a PATCH `/me/email` endpoint for email changes
6. **Actual:** No such endpoint exists. The only account endpoints are GET `/me` (fetch user) and PUT `/me` (update profile fields like nickname, but not email)

### Reproduction Evidence

- **Working branch:** https://github.com/pavanreddy71000/321vegan-api/tree/feature/change-email
- **My findings:** Confirmed the feature is completely missing. The `src/app/routes/account.py` file only has two endpoints (GET `/` and PUT `/`). There is no email change logic anywhere in the codebase — no schema, no CRUD method, no email service method, and no route handler. The password reset flow in `src/app/routes/auth.py` and `src/app/crud/user.py` provides a working template for the token-based confirmation pattern this feature needs.

---

## Solution Approach

### Analysis

The codebase already has a password reset flow that does basically the same thing: generate a token, store it with an expiry, email a link, validate when the user clicks it. The email change feature is the same pattern with different field names, plus one extra step up front (verifying the current password and making sure the new email isn't already taken).

### Proposed Solution

Build a token based email change flow that follows the same approach as the existing password reset code. User submits their current password and new email, the system saves the new email as pending with a temporary token, emails a confirmation link, and when the user clicks it the system swaps the email over.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Users need a way to change their email. The change should require password confirmation and email verification before it takes effect. This is a common pattern in web apps and a good one to learn hands-on.

**Match:** The password reset flow (`create_password_reset_token` / `reset_password`) uses the same token generation, expiry, and confirmation pattern. I'll follow that structure.

**Plan:**
1. Add `pending_email`, `email_change_token`, `email_change_expires` columns to the User model and generate an Alembic migration
2. Create `EmailChangeRequest` schema in `schemas/auth.py`
3. Add `request_email_change` and `confirm_email_change` methods in `crud/user.py`
4. Add `send_email_change_confirmation` and `send_email_change_notification` in `services/email.py`
5. Add PATCH `/me/email` and GET `/me/email/confirm` endpoints in `routes/account.py`

**Implement:** Working branch: https://github.com/pavanreddy71000/321vegan-api/tree/feature/change-email

**Review:** Follow the project's CONTRIBUTING.md guidelines, match existing code style, run any existing tests.

**Evaluate:** Verify password rejection returns 401, duplicate email returns 409, expired token returns 400, and the happy path swaps the email and notifies the old address.

---

## Testing Strategy

### Manual Testing (via Swagger UI)

- [x] Correct password + valid new email → 200, "A confirmation email has been sent to your new email address."
- [x] Wrong password → 401 Unauthorized, "Incorrect password."
- [x] Valid token confirms the email change → 200, "Your email address has been updated successfully."
- [x] Invalid token → 400 Bad Request, "Invalid or expired email change token."
- [x] Database verified: email swapped from `admin@example.com` to `test@example.com`, pending fields cleared to NULL after confirmation

### Notes

- SMTP credentials are not configured in the local dev environment, so the email service logs "Would send email to..." instead of actually sending. The code path still executes fully — it just skips the SMTP connection when credentials are empty.
- Duplicate email (409) test was not run separately because `request_email_change()` in the CRUD layer handles that check, and the happy path test already exercises the non-duplicate path.

---

## Implementation Notes

### What I Built

Implemented the full email change feature across 5 existing files plus 1 new migration file. The implementation follows the existing password reset flow as a structural template, as the maintainer suggested.

I broke the work into four coding phases to keep each session focused:

**Phase A — Database layer:**
Added three columns (`pending_email`, `email_change_token`, `email_change_expires`) to the User model in `models/user.py`. Used Alembic's `--autogenerate` to create the migration — Alembic compared my updated model to the live database and generated the `add_column` calls automatically. Added the `EmailChangeRequest` Pydantic schema to `schemas/auth.py` with `new_email` (EmailStr) and `current_password` (str) fields.

**Phase B — Business logic:**
Added two CRUD methods to `crud/user.py`: `request_email_change()` checks email uniqueness, generates a token via `generate_reset_token()`, stores the pending data with an expiry, and returns the token. `confirm_email_change()` looks up the user by `email_change_token`, checks expiry, swaps `pending_email` into `email`, and clears the temporary fields.

**Phase C — Email service and endpoints:**
Added `send_email_change_confirmation()` and `send_email_change_notification()` to `services/email.py`, following the same HTML/text email template pattern as the existing `send_password_reset_email()`. Added `PATCH /me/email` and `GET /me/email/confirm` endpoints to `routes/account.py`. The PATCH endpoint verifies the password, triggers the change request, and sends the confirmation email. The GET endpoint validates the token, applies the email swap, and notifies the old address.

**Phase D — Testing:**
Tested the full happy path and error cases through the Swagger UI. Reset test data afterward.

### Challenges Faced

1. **Port 5432 conflict kept returning:** Every time I synced my fork or did `docker compose down && up`, the `docker-compose.yml` port would reset back to 5432 because that's what's in the repo. I had to change it to 5433 each time. The fix is local-only and not committed, since it would break things for other contributors.

2. **SMTP credentials causing 500 errors:** The first time I tested `PATCH /me/email`, it returned a 500 "Failed to send confirmation email." The `.env` had `SMTP_PASSWORD=putPasswordHere` — a fake value that made the email service try (and fail) to connect to the SMTP server. Clearing `SMTP_PASSWORD=` triggered the fallback path that skips sending and just logs.

3. **Understanding `get_one` and SQLAlchemy filters:** I initially tried reusing `verify_reset_token()` in `confirm_email_change()`, but that searches the wrong column (`reset_token` instead of `email_change_token`). Had to learn that `self.get_one(db, User.email_change_token == token)` is a SQLAlchemy filter expression that gets translated to SQL, not a Python comparison.

4. **Mixing up column names in expiry check:** Wrote `user.email_change_token < datetime.now()` instead of `user.email_change_expires < datetime.now()` — comparing a string to a datetime, which would crash at runtime. Caught it during code review before testing.

### Code Changes

- **Files modified:**
  - `src/app/models/user.py` — 3 new columns
  - `src/alembic/versions/073f10aae432_add_email_change_fields_to_users.py` — new migration (auto-generated)
  - `src/app/schemas/auth.py` — new `EmailChangeRequest` schema
  - `src/app/crud/user.py` — 2 new CRUD methods
  - `src/app/services/email.py` — 2 new email methods
  - `src/app/routes/account.py` — 2 new endpoints + 3 new imports
- **Key commits:**
  - `8879a87` — Add database columns and schema for email change feature
  - `21bbe09` — Add request and confirm email change CRUD methods
  - `12b9c4b` — Add email change endpoints and email service methods
- **Approach decisions:** Followed the maintainer's recommendation to model everything on the existing password reset flow. Used `generate_reset_token()` (the same token generator) rather than creating a separate one for email changes, keeping the codebase consistent.

---
