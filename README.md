# Contribution #1: Add "Change Email" Feature

**Contribution Number:** 1  
**Student:** Sai Pavan Reddy  
**Issue:** https://github.com/321-Vegan/321vegan-api/issues/76  
**Status:** Phase I Complete  

---

## Why I Chose This Issue

**Why this issue:** Right now users on 321 Vegan have no way to change their email. If they mistyped it on signup or switched email providers, they're stuck. It's a real gap in a real product. The issue is also well scoped, it's not a vague "improve the account system" request, it's one specific feature with clear boundaries: two endpoints, a database migration, and an email service. I can plan the work from day one without needing weeks of codebase exploration first.

**Why me:** I've been studying FastAPI and Python backend development, and this issue hits most of what I've been learning. I've worked with Pydantic and SQLAlchemy in some projects, I've run Alembic migrations before, and I understand how auth flows are supposed to work. The maintainer also pointed out that there's an existing password reset flow in the codebase that uses the exact same pattern (token generation, expiry, email confirmation), so I can study working code before writing anything.

**What "fixed" means:** The issue is resolved when a logged-in user can hit PATCH `/me/email` with their current password and a new email address, receive a confirmation link at the new address, click it to confirm the swap, and get a notification at their old address that the change went through. On the backend that means three new database columns on the User model, an Alembic migration, a Pydantic request schema, two CRUD methods, two email service methods, and two route handlers. Wrong password returns 401, duplicate email returns 409, bad or expired token returns 400. If all of that works and the maintainer's 6-item checklist is complete, it's done.

---
