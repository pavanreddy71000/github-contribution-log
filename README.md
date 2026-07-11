# Contribution #2: Add Pagination to Alerts Endpoint

**Contribution Number:** 2

**Student:** Sai Pavan Reddy Doddam Reddy

**Issue:** https://github.com/RobertoDeLaCamara/ML-IDS/issues/2

**Status:** Phase II Complete

---

## Why I Chose This Issue

**Why this issue:** The ML-IDS project's `GET /api/alerts` endpoint currently returns every alert in the database at once. As the system detects more intrusions over time, this becomes a real problem, response payloads grow unbounded, the API slows down, and any frontend consuming the data has to handle an unpredictable amount of results. Pagination is one of those features that every production API needs eventually, and this issue asks for a clean implementation: `limit` and `offset` query parameters, input validation with proper 400 errors, response headers for total count and navigation, and backward compatibility so existing clients don't break.

**Why me:** In my first contribution I built a brand-new feature from scratch, a token-based email change flow for the 321Vegan API. That taught me how to create endpoints, write Pydantic schemas, and work with SQLAlchemy models. This issue builds on that same FastAPI/SQLAlchemy stack but tackles a different kind of problem: modifying an existing endpoint rather than creating a new one. That's a skill I haven't practiced yet, and it's closer to what most real-world backend work actually looks like improving what's already there while making sure nothing breaks for current users.

**What "fixed" means:** When this is done, the `GET /api/alerts` endpoint will accept optional `limit` and `offset` query parameters, return a bounded set of results instead of everything, include `X-Total-Count` and pagination headers in the response, reject invalid parameter values with a 400 status code, and continue to work exactly as before if no pagination parameters are passed.

---

## Reproduction Process

### Environment Setup

Cloned my fork and set up a Python virtual environment with `venv` and `pip install -r requirements.txt`. The install went smoothly with no errors — much simpler than my Module 1 project since this one uses pip and requirements.txt instead of Poetry and Docker.

The project uses SQLite by default (not PostgreSQL), so no database server setup was needed. All 4 existing tests passed on the first run with `pytest tests/test_inference_server.py -v`.

No environment gotchas to report for this project.

### Steps to Reproduce

This is a feature request, not a bug, so "reproduction" means confirming the current endpoint lacks what the issue asks for:

1. Open `src/inference_server/routers/alerts.py` and examine the `list_alerts` endpoint (line 19)
2. The endpoint has `limit` and `offset` query parameters, but:
   - Default limit is 100 (issue wants 20) and max is 1000 (issue wants 100)
   - The response is `List[AlertResponse]` — a bare JSON array with no pagination metadata
   - No `X-Total-Count`, `X-Page-Size`, or `X-Page-Offset` response headers exist
   - No total count query is performed, so clients have no way to know how many alerts exist
   - `limit=0` is silently accepted (returns empty results instead of a 400 error)
3. The issue's expected response format — `{"alerts": [...], "total": 1523, "limit": 20, "offset": 0}` — is completely absent from the current implementation

**Current behavior:** Returns a bare JSON array of alerts with no pagination metadata or headers.
**Expected behavior:** Returns a wrapped response with `alerts`, `total`, `limit`, `offset` fields, plus `X-Total-Count`, `X-Page-Size`, and `X-Page-Offset` headers.

### Reproduction Evidence

Working branch: https://github.com/pavanreddy71000/ML-IDS/tree/feature/alerts-pagination

---

## Solution Approach

### Implementation Plan (UMPIRE Framework)

**Understand:** The `GET /api/alerts` endpoint currently returns all matching alerts as a flat JSON array. It has basic `limit`/`offset` query parameters wired to SQL, but lacks the pagination metadata (total count, headers, wrapped response body) that clients need to navigate through pages of results. As the alerts table grows, this makes the API impractical for frontend dashboards or any client that needs to display paginated data.

**Match:** The existing endpoint already demonstrates the query-building pattern I'll extend — filters are composed with `select()`, `where()`, and `order_by()`. I'll follow the same async SQLAlchemy pattern for the count query. The `AlertResponse` schema in `schemas.py` shows how Pydantic models are structured in this project, which I'll use as a template for the new paginated response schema.

**Plan:**
1. Add a `PaginatedAlertResponse` Pydantic schema in `schemas.py` with fields: `alerts` (list of AlertResponse), `total` (int), `limit` (int), `offset` (int)
2. Modify `list_alerts` in `routers/alerts.py`:
   - Change default limit from 100 to 20, max from 1000 to 100
   - Add a `SELECT COUNT(*)` query (using `func.count()`) before the paginated query to get total matching alerts
   - Return a `JSONResponse` with the wrapped body and `X-Total-Count`, `X-Page-Size`, `X-Page-Offset` headers
   - Add validation: return 400 for `limit < 1` or `offset < 0`
3. Write unit tests covering: first page, middle page, last page, invalid parameters (negative limit, negative offset), and backward compatibility (no params returns first page with default limit)

**Files to modify:**
- `src/inference_server/schemas.py` — add `PaginatedAlertResponse`
- `src/inference_server/routers/alerts.py` — modify `list_alerts` endpoint
- `tests/` — add pagination tests (new test file or extend existing)

**Review:** Will follow the project's CONTRIBUTING.md conventions: PEP 8 style, async/await for all I/O, type hints on all functions, docstrings on public functions, and conventional commit messages (`feat: add pagination to alerts endpoint`).

**Evaluate:** All existing tests must continue to pass. New pagination tests will verify correct page slicing, total count accuracy, response header presence, 400 errors for invalid params, and backward compatibility. Manual verification by calling `GET /api/alerts?limit=20&offset=0` and confirming the response shape matches the issue's expected format.

---
