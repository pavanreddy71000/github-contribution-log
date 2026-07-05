# Contribution #2: Add Pagination to Alerts Endpoint

**Contribution Number:** 2  
**Student:** Sai Pavan Reddy Doddam Reddy  
**Issue:** https://github.com/RobertoDeLaCamara/ML-IDS/issues/2  
**Status:** Phase I Complete

---

## Why I Chose This Issue

**Why this issue:** The ML-IDS project's `GET /api/alerts` endpoint currently returns every alert in the database at once. As the system detects more intrusions over time, this becomes a real problem, response payloads grow unbounded, the API slows down, and any frontend consuming the data has to handle an unpredictable amount of results. Pagination is one of those features that every production API needs eventually, and this issue asks for a clean implementation: `limit` and `offset` query parameters, input validation with proper 400 errors, response headers for total count and navigation, and backward compatibility so existing clients don't break.

**Why me:** In my first contribution I built a brand-new feature from scratch, a token-based email change flow for the 321Vegan API. That taught me how to create endpoints, write Pydantic schemas, and work with SQLAlchemy models. This issue builds on that same FastAPI/SQLAlchemy stack but tackles a different kind of problem: modifying an existing endpoint rather than creating a new one. That's a skill I haven't practiced yet, and it's closer to what most real-world backend work actually looks like improving what's already there while making sure nothing breaks for current users.

**What "fixed" means:** When this is done, the `GET /api/alerts` endpoint will accept optional `limit` and `offset` query parameters, return a bounded set of results instead of everything, include `X-Total-Count` and pagination headers in the response, reject invalid parameter values with a 400 status code, and continue to work exactly as before if no pagination parameters are passed.
