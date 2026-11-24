---
name: resiliency
description: "Evaluate code changes for resilience, observability, and error handling. Apply logging, metrics, rate limiting, retries, and back-off strategies. Uses discover-scoped file change analysis."
disable-model-invocation: false
---

# Resiliency

Evaluate and harden code changes for resilience, observability, and error handling. This skill ensures code can handle failures gracefully, is observable in production, and implements production-grade patterns.

## Invocation

This skill is invoked by the issue-orchestrator after `hunt-bugs` completes. It receives:
- `issue` — the issue number (e.g., `42`)
- `branch` — the integration branch (e.g., `feature/issue-42`)
- `base_branch` — the base branch (e.g., `main`)

## Process

### 1. Discover Changed Files

First, discover which files differ between `base_branch` and `branch`:

```bash
git diff <base_branch>...<branch> --stat
```

This gives you the list of all changed files. Only evaluate the changed files and their direct dependencies.

### 2. Evaluate Resilience

For each changed file, check:

- **Failure modes** — What happens if dependencies fail? (DB down, API timeout, network error, disk full)
- **Graceful degradation** — Does the code fail gracefully or crash?
- **Input validation** — Are external inputs validated? What about malformed data?
- **State consistency** — If a partial failure occurs, is the system in a consistent state?
- **Idempotency** — Can the operation be safely retried without side effects?
- **Resource exhaustion** — Can the code be triggered to exhaust memory, file handles, or connections?
- **Timeout handling** — Are external calls (DB, API, file system) properly timed out?
- **Backpressure** — Does the code handle backpressure (e.g., too many concurrent requests)?

### 3. Evaluate Observability

Check that the code is observable in production:

- **Structured logging** — Are logs structured (JSON) with consistent fields (timestamp, level, message, context)?
- **Log levels** — Appropriate use of `debug`, `info`, `warn`, `error`?
- **Correlation IDs** — Are request IDs propagated through the call chain?
- **Missing log points** — Are there critical paths with no logging?
- **Sensitive data** — Are passwords, tokens, PII excluded from logs?
- **Log completeness** — Do logs capture enough context to debug issues?

### 4. Evaluate Error Handling

- **Error boundaries** — Are errors caught at the right level? (Not too broad, not too narrow)
- **Error propagation** — Are errors propagated correctly (not swallowed)?
- **Custom error types** — Are domain-specific errors used (not generic `Error`)?
- **Retry logic** — Are transient errors retried with backoff?
- **Fallback values** — Are default/fallback values provided where appropriate?
- **User-facing errors** — Are user-facing errors friendly and actionable?
- **Stack traces** — Are stack traces captured for debugging (but not exposed to users)?

### 5. Apply Production Patterns

For each changed file, apply these patterns where missing:

#### Logging
```typescript
// Before
console.log("Processing request");

// After
logger.info("Processing request", { userId, requestId, timestamp: Date.now() });
```

#### Metrics
- Add counters for key operations (e.g., `requests_total`, `errors_total`)
- Add histograms for latency-sensitive operations
- Add gauges for resource usage
- Use the project's existing metrics library (e.g., Prometheus, OpenTelemetry)

#### Rate Limiting
- Are external API calls rate-limited?
- Are user-facing endpoints rate-limited?
- Is there a sliding window or token bucket implementation?
- Are rate limit headers returned to the client?

#### Retries with Back-off
- Implement exponential backoff with jitter for transient failures
- Use the project's existing retry library (e.g., `p-retry`, `retry-axios`)
- Set appropriate max retries (usually 3-5)
- Distinguish between retriable (network, timeout) and non-retriable (validation, auth) errors
- Add a circuit breaker for repeated failures

#### Timeout Configuration
- All external calls should have explicit timeouts
- Timeouts should be configurable (not hardcoded)
- Timeout errors should be categorized (e.g., `ConnectionTimeoutError`)

### 6. Verify

- All changed files are evaluated
- Missing patterns are applied
- No new observability gaps are introduced
- The code is production-ready

### 7. Document

Document only significant changes that affect understanding.

## Output

After evaluation and hardening, commit the changes to the integration branch (`branch`). Then signal `success` to the orchestrator so it transitions to `ready-for-spec-review`.

## Key Principle

You are the last technical checkpoint before human review. Your job is to ensure the code can handle the real world — failures, edge cases, and production conditions — not just the happy path. You never say "looks good" without actually evaluating resilience, observability, and error handling.
