# github-contribution-log

# Contribution [#]: 1845

**Contribution Number:** 2

**Student:** Shuja 

**Issue:** https://github.com/RimSort/RimSort/issues/1845

**Status:** Phase I 

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
The fix involves building a centralized, thread-safe configuration directly inside `app/utils/http.py`. Instead of invoking bare requests, we will configure a `requests.Session()` within the request cycle. We will import `Retry` from `urllib3.util` and mount an `HTTPAdapter` to this session. This adapter will be explicitly instructed to intercept structural failure status codes (like `500`, `502`, `503`, `504`, and `429`) and safely retry up to 3 times using an exponential backoff factor.

## Implementation Plan
Using UMPIRE framework (adapted):

*Understand:* The application fails immediately on any transient network drop or temporary server error because `app/utils/http.py` does not possess a retry framework.

*Match:* Standard Python resilience design patterns rely on configuring `urllib3.util.Retry` structures wrapped cleanly inside a `requests.adapters.HTTPAdapter` session registry.

*Plan:* 
- **Centralize an HTTP Session Wrapper:** Modify `app/utils/http.py` to utilize a persistent or cleanly instantiated `requests.Session()` object instead of making bare `requests.get()` calls.
- **Configure urllib3 Retry Logic:** Import `Retry` from `urllib3.util` and mount a custom `HTTPAdapter` to the session.
- **Implement Exponential Backoff:** Configure the adapter with a maximum of 3 retries, a backoff factor (e.g., `0.5` or `1`), and status fault hooks specifically listening for `429` (Rate Limited) and `5xx` server-side errors.
- **Verification:** Re-run the reproduction test script to visually and programmatically confirm that the execution pauses and attempts backoffs multiple times before raising a final failure.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
