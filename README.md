# Contribution #1: Add "Change Email" Feature

**Contribution Number:** 1  
**Student:** Sai Pavan Reddy Doddam Reddy  
**Issue:** https://github.com/321-Vegan/321vegan-api/issues/76  
**Status:** Phase II Complete

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

### Unit Tests

- [ ] Correct password + valid new email returns "Confirmation email sent"
- [ ] Wrong password returns HTTP 401
- [ ] New email already in use returns HTTP 409
- [ ] Valid token confirms the email change and updates the user record
- [ ] Expired token returns HTTP 400
- [ ] Invalid token returns HTTP 400

### Integration Tests

- [ ] Full flow: request change, confirm token, verify email is updated in DB
- [ ] Old email receives notification after successful change

### Manual Testing

[To be completed in Phase III]

---

## Implementation Notes

### Week 1 Progress

[To be completed]

### Code Changes

- **Files modified:** [To be updated during implementation]
- **Key commits:** [To be updated during implementation]
- **Approach decisions:** [To be updated during implementation]

---

## Pull Request

**PR Link:** [To be added in Phase IV]

**PR Description:** [To be drafted in Phase IV]

**Maintainer Feedback:**
- [Pending]

**Status:** Not yet submitted

---

## Learnings & Reflections

[To be completed after submission]

---

## Resources Used

- [321vegan-api repository](https://github.com/321-Vegan/321vegan-api)
- [Issue #76: Change email feature](https://github.com/321-Vegan/321vegan-api/issues/76)
- [Maintainer implementation guidance](https://github.com/321-Vegan/321vegan-api/issues/76#issuecomment-4642792577)
