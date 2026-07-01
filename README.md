# github-contribution-log

# Contribution [#]: 1845

**Contribution Number:** 2

**Student:** Shuja 

**Issue:** https://github.com/RimSort/RimSort/issues/1845

**Status:** Phase IV Complete

---

## Why I Chose This Issue

Why I Chose This IssueI chose issue #1845 "Add HTTP retry logic with backoff" because it aligns with my Python experience and my goal to improve my understanding of robust networking and error-handling patterns. The issue is explicitly labeled `"good first issue"` and `"Code refactoring"`, making it a perfect entry point for a new contributor.

I'm interested in this because:
1. I understand the basics of handling HTTP requests, but implementing a structured exponential backoff strategy with urllib3 and requests adapters will deepen my practical backend skills.
2. The implementation area is highly centralized—all traffic goes through a single utility file (app/utils/http.py)—meaning the scope is well-contained and won't require navigating a massive, sprawling codebase.The issue outlines clear edge cases to consider, such as handling rate limits ($429$ status codes) via Retry-After headers and ensuring streaming downloads aren't broken by partial reads, which provides a great technical challenge.
3. I want to learn how this project structures thread-safe utility functions without sharing session states.From reading the issue thread, I understand the current problem is that failed network requests immediately raise an error with no retry mechanism, which can cause unnecessary crashes during transient network hiccups. 
   
My contribution will build on top of a recent security audit (PR #1841) to make the application's network layer significantly more resilient.

Left a comment on the issue introducing myself to express my interest in working on it and to ask for initial pointers on getting started in the codebase.

---

## Understanding the Issue

## Problem Description
The core application code relies heavily on making direct HTTP web requests (such as checking for software updates, pulling mod information, or hitting Steam APIs). Currently, these requests are executed without any safety nets. If a user experiences a minor, split-second network drop or if the remote server returns a temporary error, the network layer instantly gives up and throws an unhandled exception, causing the application to crash or fail key tasks unnecessarily.

## Expected Behavior
When a network request encounters a transient failure (e.g., a physical connection timeout, a `500 Internal Server Error`, or a `429 Too Many Requests` rate limit), the application should gracefully pause, wait briefly, and try the request again. It should use an exponential backoff strategy so it waits slightly longer between each attempt before finally giving up.

## Current Behavior
The application invokes standard requests (`requests.get`) directly. It immediately raises an error on the very first sign of a network hiccup or bad status code. No retry mechanism or backoff delays exist.

## Affected Components
- `app/utils/http.py`: This is the centralized network utility file where all HTTP operations pass through. The core functions (`get`, `post`, etc.) need to be refactored here so that the entire codebase inherits the safer retry mechanics automatically.

---

## Reproduction Process



## Steps to Reproduce
1. Create a isolated test script named `test_http.py` in the root directory of the repository to call the HTTP utility directly.
2. Add code to execute `app.utils.http.get()` pointing to a completely non-existent domain (e.g., `https://this-domain-does-not-exist-at-all-xyz.com`) or call `.raise_for_status()` on a forced failing endpoint to trigger a clear exception.
3. Run the script inside the project's virtual environment using `uv run python test_http.py`.
4. **Observed result:** The script immediately throws a `ConnectionError` or an unhandled HTTP exception and terminates instantly. There is no pause, delay, or subsequent attempt to retry the connection, proving that the application has zero fallback mechanisms for transient network bugs.



### Reproduction Evidence

- **Commit showing reproduction:** [[Branch wit Bug Reproduced]](https://github.com/shuja-waraich-03/RimSort.git)
- **Screenshots/logs:** 
  
``` Testing RimSort HTTP client...
Sending request to: https://this-domain-does-not-exist-at-all-xyz.com

--- RESULT ---
The request failed with error type: ConnectionError
Error message: HTTPSConnectionPool(host='this-domain-does-not-exist-at-all-xyz.com', port=443): Max retries exceeded with url: / (Caused by NameResolutionError("HTTPSConnection(host='this-domain-does-not-exist-at-all-xyz.com', port=443): Failed to resolve 'this-domain-does-not-exist-at-all-xyz.com' ([Errno 8] nodename nor servname provided, or not known)")) 
```


---

## Solution Approach

## Analysis
The root cause is that the HTTP utilities inside `app/utils/http.py` do not implement an explicit session engine configured with structural fault handling. By default, the `requests` module executes calls in isolation. Without mounting an `HTTPAdapter` armed with explicit `urllib3` `Retry` logic, the client possesses zero internal loops to intercept, evaluate, and retry failed tcp/status hooks.

## Proposed Solution
The fix involves building a centralized, thread-safe configuration directly inside `app/utils/http.py`. Instead of invoking bare requests, we will configure a `requests.Session()` within the request cycle. We will import `Retry` from `urllib3.util` and mount an `HTTPAdapter` to this session. This adapter will be explicitly instructed to intercept structural failure status codes (like `500`, `502`, `503`, `504`, and `429`) and safely retry up to 4 times (for idempotent `GET`/`HEAD` methods) using an exponential backoff factor.

## Implementation Plan
Using UMPIRE framework (adapted):

*Understand:* The application fails immediately on any transient network drop or temporary server error because `app/utils/http.py` does not possess a retry framework.

*Match:* Standard Python resilience design patterns rely on configuring `urllib3.util.Retry` structures wrapped cleanly inside a `requests.adapters.HTTPAdapter` session registry.

*Plan:* 
- **Centralize an HTTP Session Wrapper:** Modify `app/utils/http.py` to utilize a persistent or cleanly instantiated `requests.Session()` object instead of making bare `requests.get()` calls.
- **Configure urllib3 Retry Logic:** Import `Retry` from `urllib3.util` and mount a custom `HTTPAdapter` to the session.
- **Implement Exponential Backoff:** Configure the adapter with a maximum of 4 retries, a backoff factor of `1`, restrict retries to idempotent methods (`GET`/`HEAD`), and add status fault hooks specifically listening for `429` (Rate Limited) and `5xx` server-side errors.
- **Verification:** Re-run the reproduction test script to visually and programmatically confirm that the execution pauses and attempts backoffs multiple times before raising a final failure.

---

## Testing Strategy

I added a new test file at `tests/utils/test_http.py`, modeled on the existing
patterns in `tests/utils/` (e.g. `test_url_validation.py`), which uses
`unittest.mock` to patch `requests` rather than hitting the network. The retry
behavior itself is configuration on the mounted adapter, so the tests assert
that configuration is present — this is exactly the regression that would have
existed before the fix (no retry adapter mounted at all).

### Unit Tests

- [x] Test case 1: `get()` applies `DEFAULT_TIMEOUT` and dispatches the `GET` method to the correct URL.
- [x] Test case 2: An explicitly passed `timeout=` is preserved and not overwritten by the default.
- [x] Test case 3: `post()` and `head()` dispatch to the `POST` and `HEAD` methods respectively.
- [x] Test case 4: A retry-enabled `HTTPAdapter` is mounted for both `http://` and `https://` with `total=4`, `backoff_factor=1`, and `allowed_methods` restricted to `GET`/`HEAD`.
- [x] Test case 5: The transient status codes `429, 500, 502, 503, 504` are all in the retry `status_forcelist`.
- [x] Test case 6: `raise_on_status` is `False`, preserving the existing contract that callers decide when to call `response.raise_for_status()`.

### Integration Tests

- [x] Ran the existing HTTP-adjacent suites to confirm no regressions: `tests/utils/test_http_downloader.py` and `tests/utils/test_http_download_worker.py` (19 tests, all passing). These exercise the streaming download paths that depend on `http.get(..., stream=True)`.

### Manual Testing

- Re-ran the Phase I reproduction script (`uv run python test_http.py`) against the dead domain. A DNS-resolution failure (`NameResolutionError`) is not a transient/retryable condition, so it still surfaces quickly — this is correct behavior; retries target connection timeouts and 429/5xx responses, not nonexistent hosts.
- Verified linting/formatting/typing locally:
  - `uv run ruff check app/utils/http.py tests/utils/test_http.py` → all checks passed
  - `uv run ruff format --check ...` → already formatted
  - `uv run mypy app/utils/http.py` → no issues found
  - `uv run pytest tests/utils/test_http.py -q` → 6 passed

---

## Implementation Notes

### Week Progress (Jun 23)

**What I built:**
- Refactored `app/utils/http.py` so the three public wrappers (`get`, `post`, `head`) route through a single private `_request()` helper instead of calling `requests.get/post/head` directly.
- `_request()` builds a `requests.Session` via a `_new_session()` helper that mounts an `HTTPAdapter` configured with `urllib3.util.retry.Retry`:
  - `total=4` retries with `backoff_factor=1` (exponential backoff — wait times grow ~1s → 2s → 4s → 8s across attempts).
  - `status_forcelist=(429, 500, 502, 503, 504)` so rate limits and server errors are retried.
  - `allowed_methods=frozenset(["GET", "HEAD"])` — retries are restricted to idempotent methods; `POST` is intentionally excluded so requests with side effects are never silently duplicated.
  - `respect_retry_after_header=True` so rate-limited responses honor the server's `Retry-After` header.
  - `raise_on_status=False` so the final response is returned and callers keep using `response.raise_for_status()` as before.
- Pulled the retry parameters out into module-level constants (`DEFAULT_RETRIES`, `DEFAULT_BACKOFF_FACTOR`, `RETRY_STATUS_CODES`) so they're documented and easy to tune.
- Added `tests/utils/test_http.py` with 6 unit tests (see Testing Strategy).

**Challenges faced / decisions made:**
- **Thread safety:** the issue specifically calls out wanting thread-safe utilities "without sharing session states." A `requests.Session` is not safe to share across threads, so instead of one module-level session I create a *fresh* session per request inside `_request()`. This keeps the retry config centralized while avoiding shared mutable state.
- **Streaming downloads:** the issue flags that streaming must not break. I audited the callers first (`grep` for `http.get`/`http.post`/`http.head`) and found several `stream=True` users — `http_downloader.py:175`, `update_utils.py:1413`, `installer.py:145`, `steamcmd/wrapper.py:525` (used as `with http.get(...) as rx:`), plus a couple of views. I confirmed that closing the session on return does **not** kill an in-flight streamed response, because the connection is checked out of the pool and only released once the body is consumed — this is the same behavior as `requests`' own top-level `requests.get(stream=True)`, which also wraps the call in `with Session() as ...`. Verified by running the existing download-worker tests.
- **Preserving the existing contract:** the previous code returned the response without raising on bad status. Setting `raise_on_status=False` keeps that contract so I didn't have to touch any of the ~15 call sites.
- **DNS failures aren't retryable:** when I re-ran the reproduction script the dead-domain test still fails fast. I initially expected a delay, then realized a `NameResolutionError` is not a transient condition urllib3 retries — which is the correct, intended behavior.

### Code Changes

- **Files modified:**
  - `app/utils/http.py` — added retry/backoff session, helper functions, and config constants.
  - `tests/utils/test_http.py` — new unit test file (6 tests).
- **Key commits (branch `feature/http-retry`):**
  - `feat(http): add retry logic with backoff for transient failures`
  - `test(http): cover retry configuration and request wrappers`
- **Approach decisions:** Per-request session for thread safety; constants for tunable retry params; retries restricted to idempotent `GET`/`HEAD` (POST excluded to avoid duplicating side effects); `respect_retry_after_header=True`; `raise_on_status=False` to avoid changing the response contract for existing callers.

---

## Pull Request

**PR Link:** https://github.com/RimSort/RimSort/pull/2280

**Branch:** `feature/http-retry`

**Summary of Contribution:**
Added automatic retry-with-backoff to RimSort's shared HTTP utility so that
transient network failures no longer surface as hard errors to the user. All
HTTP traffic already routes through `app/utils/http.py`, so the fix is inherited
across the codebase without touching call sites.

- Added a `_new_session()` helper that mounts a `urllib3` `Retry` adapter on both
  `http://` and `https://`, and a `_request()` helper that the public
  `get()`/`post()`/`head()` wrappers now route through.
- Retries transient failures — connection errors and status codes
  `429, 500, 502, 503, 504` — using exponential backoff (`DEFAULT_RETRIES = 4`,
  `DEFAULT_BACKOFF_FACTOR = 1`).
- Restricts retries to idempotent methods (`GET`, `HEAD`); `POST` is not retried
  automatically, to avoid duplicate side effects.
- Honors the `Retry-After` header for rate-limited responses.
- Sets `raise_on_status=False` so existing callers keep relying on
  `response.raise_for_status()` — the public API is unchanged.
- Creates a fresh `Session` per request, so no state is shared between threads.
- Added `tests/utils/test_http.py` covering the retry configuration and request
  wrappers.
- Impact: 2 files changed, +127 / -10.

**Maintainer Feedback:**
- PR is open and awaiting maintainer review; no feedback received yet.
- Next steps if requested: address review comments (e.g. tuning retry
  count/backoff defaults or expanding test coverage) and iterate.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

- **How `requests` and `urllib3` layer together.** I learned that `requests`
  delegates its actual retry/connection-pooling behavior to `urllib3`, and that
  the seam for configuring retries is a `urllib3.util.retry.Retry` object mounted
  on a `requests.adapters.HTTPAdapter`, which is in turn mounted onto a
  `Session` per URL scheme. Understanding that layering was the key to a small,
  centralized fix.
- **Idempotency as a design constraint.** I learned *why* automatic retries must
  be restricted to idempotent methods — retrying a `POST` can duplicate a
  side effect (a second write, a double submission). That's why I set
  `allowed_methods=frozenset(["GET", "HEAD"])` and deliberately left `POST` out.
- **Exponential backoff and `Retry-After`.** I understood how `backoff_factor`
  translates into exponentially growing sleep intervals between attempts, and how
  `respect_retry_after_header=True` lets a rate-limited server dictate the wait.
- **Which failures are actually transient.** A DNS `NameResolutionError` is *not*
  retryable, whereas connection timeouts and `429`/`5xx` responses are — so
  retries shouldn't mask a genuinely dead host.
- **Preserving a public contract.** Using `raise_on_status=False` let me add
  retries without changing the return contract of `get`/`post`/`head`, so none
  of the ~15 existing call sites needed edits.
- **Testing configuration, not the network.** I learned to assert on the mounted
  adapter's retry configuration with `unittest.mock` instead of making real
  network calls, which keeps the tests fast and deterministic.

### Challenges Overcome

- **Thread safety without shared session state.** The issue explicitly asked for
  thread-safe utilities "without sharing session states," but a `requests.Session`
  is not safe to share across threads. I resolved this by creating a *fresh*
  session per request inside `_request()` (via `_new_session()`), keeping the
  retry config centralized while avoiding shared mutable state.
- **Not breaking streaming downloads.** The issue flagged that streaming must not
  break. I audited the callers (`grep` for `http.get`/`post`/`head`) and found
  several `stream=True` users. I confirmed that closing the session on return does
  not kill an in-flight streamed response — the connection is checked out of the
  pool and only released once the body is consumed, which mirrors `requests`' own
  top-level `requests.get(stream=True)` behavior. I verified this by running the
  existing download-worker tests with no regressions.
- **Interpreting the reproduction result correctly.** When I re-ran the
  reproduction script against a dead domain, it still failed fast and I initially
  thought retries weren't working. I then realized a `NameResolutionError` is not
  a transient condition urllib3 retries — the fast failure was correct behavior,
  not a bug in my change.

### What I'd Do Differently Next Time

- **Pin down final parameter values earlier.** My planning notes started with
  `total=3` / `backoff_factor=0.5` (and briefly allowed `POST`), but the submitted
  code settled on `total=4` / `backoff_factor=1` / `GET`/`HEAD` only. Next time I'd
  lock the constants in before writing up the plan so the log stays consistent
  end to end.
- **Add a behavioral retry test.** My tests assert the retry configuration is
  present. If I revisit this, I'd add a test that simulates a `503`-then-`200`
  sequence (e.g. with `responses` or a mocked adapter) to prove the retry loop
  actually fires, not just that it's configured.
- **Open the upstream issue conversation sooner.** Confirming the maintainers'
  preferred defaults (retry count, backoff, whether `POST` should ever be retried)
  before implementing would reduce the chance of iterating on those values during
  review.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
