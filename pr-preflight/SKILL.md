---
name: pr-preflight
description: Runs a pre-push quality check on local changes before a pull request is opened. Uses the repository's existing conventions to suggest and add metrics (counters, gauges, histograms), logs (debug, info, warn, error) and traces, then suggests unit and integration tests, alerts for the new behavior, and performance optimizations. Use when the user says "preflight", "PR sanity check", "is this ready for a PR", "check my changes before I push", or "production-ready check".
---

# PR Preflight

An **author-side** check that runs on local changes *before* a PR is opened. It finds what's missing, proposes concrete changes in the repo's own style, and applies the ones the user approves, so the PR goes up already instrumented, tested and reviewed for performance.

**Golden rule:** follow the repository's existing conventions. Never introduce a new logging, metrics, tracing or test library when the repo already has one. If the repo has none, say so and ask before adding anything.

## Workflow

Run the phases in order. After each phase that proposes code changes, **stop and ask the user which suggestions to apply** before editing files.

### Phase 0. Gather the Changeset

```bash
git rev-parse --abbrev-ref HEAD
git merge-base HEAD origin/main      # fall back to origin/master or local main
git diff --name-only <merge-base>    # committed + uncommitted changes on this branch
git status --porcelain               # include untracked files
git diff <merge-base>                # full diff
```

- The changeset is everything on this branch that isn't on the base branch, plus uncommitted and untracked files.
- Skip generated files, lockfiles, vendored code and snapshots.
- Classify each changed file: **service/business logic**, **API handler**, **data access**, **background job/consumer**, **CLI/script**, **config/infra**, **tests**, **docs**. Most of the later phases only apply to the first four.
- Write a one-paragraph summary of what the change does. Every later suggestion should tie back to it.

If the change is docs-only or config-only, say so, run only Phase 5 (Hygiene), and stop.

### Phase 1. Discover Repo Conventions

Before suggesting anything, learn how this repo already does observability and testing. See [REFERENCE.md](REFERENCE.md#convention-discovery) for search patterns per language.

Record, with a file:line example for each:

| Convention | What to find |
|------------|--------------|
| Logger | Library, how it's obtained (global, injected, from context), structured vs printf, standard field names (`user_id`, `request_id`, `err`) |
| Metrics | Library (Prometheus, OpenTelemetry, StatsD, Datadog, Micrometer...), where metrics are registered, naming scheme (`service_subsystem_name_unit`), common labels, unit suffixes |
| Tracing | Library, how spans are started, span naming scheme, common attributes, how errors are recorded on spans |
| Tests | Framework, file naming/location, unit vs integration split (build tags, folders, markers), fixtures/mocks/testcontainers in use |
| Alerts | Where alert rules live (Prometheus rules, Terraform monitors, Datadog JSON, Grafana, SLO files), if they live in this repo at all |

Show this table to the user briefly. If a convention can't be found, say "none found" rather than guessing.

### Phase 2. Observability: Metrics, Logs, Traces

For each changed function/flow of the relevant types, check what's already instrumented and suggest only what's missing. Proportionality matters: a small helper doesn't need a span and three metrics.

#### Metrics

| Type | Use for | Examples |
|------|---------|----------|
| **Counter** | Things that only go up: events, requests, errors, retries | `orders_created_total`, `payment_failures_total{reason}` |
| **Gauge** | Current values that go up and down | queue depth, in-flight requests, cache size, pool connections in use |
| **Histogram** | Distributions: latency, payload size, batch size | `http_request_duration_seconds`, `batch_size` |

- Match the repo's naming, unit suffixes and label set exactly.
- **Check label cardinality:** never use user IDs, emails, request IDs, raw URLs or free-form error messages as labels. Use bounded values (status code class, error type, operation name).
- Prefer one histogram over separate counter + timer when the library gives a count for free.
- Record outcome (`success`/`failure`) as a label rather than separate metrics, if that's the repo convention.

#### Logs

| Level | Use for |
|-------|---------|
| **debug** | Detailed flow information useful only while diagnosing; safe to be noisy because it's off in prod |
| **info** | Significant business events and state changes (job started/finished, entity created) — not every function call |
| **warn** | Something unexpected that was handled: retries, fallbacks, degraded mode, deprecated input |
| **error** | A failure that needs attention and was not recovered from. Include the error and enough context to act on it |

**Every log line must carry enough context to act on it without reading the code.** Someone seeing it in a log search at 3am should be able to tell what happened, to which entity, in which request, and why. For each suggested log (and each existing log in the diff), check it includes the fields that apply:

| Context | Examples |
|---------|----------|
| Correlation | `request_id`, `trace_id`/`span_id`, `job_id`, `message_id` — however the repo attaches them (context logger, MDC, middleware) |
| Operation | what was being attempted: `op="reserve_inventory"`, handler/route, job name |
| Entity identifiers | the IDs of the things being acted on: `order_id`, `account_id`, `tenant_id`, `batch_id` |
| Relevant inputs/state | the bounded, non-sensitive values that drove the decision: `status`, `item_count`, `plan`, `previous_state` → `new_state` |
| Outcome | `result="skipped"`, `reason="already_processed"`, HTTP/status code |
| Error details | the error object itself (so stack/cause chain is kept), error type/code, and the upstream dependency name if a call failed |
| Retry/timing | `attempt`, `max_attempts`, `backoff_ms`, `duration_ms` |

- Keep the message a short, static string and put the variable data in fields (`"inventory reservation failed"` + fields, not `"failed to reserve 3 items for order 123"`), so logs are searchable and groupable. If the repo uses printf-style logging, include the same context values in the message.
- When a function already has context available (a request-scoped logger, an enriched context), use it instead of re-adding fields by hand.
- If an existing log in the diff is missing context (e.g. `log.Error("failed")` or an error logged without the ID it failed for), suggest enriching it.
- **Never log secrets, tokens, passwords, or PII** (emails, phone numbers, addresses, full card numbers). Flag existing ones in the diff.
- Don't log and re-throw the same error at every layer; log once where it's handled.
- No logging inside tight loops at info level or above.

#### Traces

- Add spans around meaningful units of work: inbound handlers, outbound calls (HTTP, DB, queue, cache), and expensive internal steps.
- Make sure context is propagated: new goroutines/threads/async tasks and outbound calls must carry the parent context.
- Add attributes for the key dimensions (same bounded values as metric labels); record errors and set span status on failure.
- If the repo uses auto-instrumentation for HTTP/DB, don't add duplicate manual spans.

#### Output for this phase

Present a numbered list of suggestions grouped by file:

```
### Observability suggestions

**src/orders/service.go**
1. [counter] `orders_created_total{channel}` in CreateOrder after successful insert (line 84) — matches `payments_processed_total` pattern in src/payments/metrics.go:12
2. [histogram] `order_create_duration_seconds` around CreateOrder (lines 60-95)
3. [log/warn] "inventory reservation retrying" on retry (line 72), fields: request_id, order_id, sku_count, attempt, max_attempts, backoff_ms, err
4. [trace] child span "orders.reserve_inventory" around the inventory client call (line 70)
5. [log/error] enrich existing `log.Error("create failed")` (line 91) with request_id, order_id, customer_tier, err — currently says nothing about which order failed

Apply which? (all / numbers / none)
```

Apply the approved ones, keeping edits minimal and consistent with surrounding code. Register new metrics wherever the repo registers existing ones.

### Phase 3. Tests

Find the test conventions from Phase 1, then for each changed behavior check whether a test covers it.

- **Unit tests:** new/changed functions with logic, branches and error paths. List the specific cases: happy path, each error branch, boundary values (empty, nil/null, zero, max), and any bug this change fixes (a regression test).
- **Integration tests:** suggest when the change touches a DB query, an external API client, a message queue, a cache, an HTTP route, or wiring between components. Use the repo's existing integration setup (testcontainers, docker-compose, fakes, test server) — don't invent one.
- If metrics/logs were added in Phase 2 and the repo tests instrumentation, suggest asserting on them too.
- Run the existing tests for the affected packages (and the linter/type checker if configured) and report failures.

Present test suggestions as a numbered list with the file each test would go in, then ask which to write.

#### Write each test failing first (red → green)

A test that has never failed proves nothing: it may not exercise the change at all. For every approved test, write it in the style of neighboring test files, then:

1. **Red — prove it fails without the behavior.** Run the test against code that doesn't have the behavior and confirm it fails **for the expected reason** (an assertion on the behavior, not a typo, import error or broken fixture).
   - *Behavior not written yet* (a fix or instrumentation this preflight is about to apply): write the test before making the code change and run it. It should fail.
   - *Behavior already in the changeset*: back up the changed file to the scratch/temp directory, temporarily undo only the relevant hunk (or restore that file from the merge base with `git show <merge-base>:<path>`), run the test, then restore the file from the backup. Confirm with `git diff` that the file is exactly as the user left it. Never use `git stash`, `git checkout` or `git reset` on the user's work for this.
   - *Brand-new function or file* (no prior version to run against): break the behavior the test asserts on (e.g. return the wrong value) instead, confirm the failure, then restore.
2. **Green — run it with the behavior in place** and confirm it passes.
3. If the test **passes in the red step**, it doesn't cover the change. Rewrite it until it fails without the change, or tell the user why it can't.

Behavior-preserving refactors and performance changes are the exception: their tests should pass both before and after, which proves behavior didn't change. Say which tests fall into this group.

Report each test with its red and green results, e.g. `TestCreateOrder_RetriesOnInventoryTimeout — red: failed (expected 2 attempts, got 1) → green: passed`.

### Phase 4. Alerts

Using the metrics now present (existing + newly added) and what the change actually does, suggest alerts. Every alert must be **actionable**: someone woken up by it knows what's broken and what to look at. See [REFERENCE.md](REFERENCE.md#alert-patterns) for templates.

For each alert, give:

- **Name** and **severity** (page vs ticket)
- **What it detects** in user-impact terms
- **Expression/condition** in the repo's alerting format if found (e.g. PromQL rule, Datadog query, Terraform monitor), otherwise PromQL-style pseudocode
- **Threshold and window** with a short justification, and a note that thresholds need tuning against real traffic
- **First debugging step** (dashboard, log query, runbook link to create)

Base them on the change's use case, e.g.:

| Change type | Alerts worth considering |
|-------------|--------------------------|
| New API endpoint | Error rate > X% over 5m, p99 latency > SLO |
| Background job / cron | Job hasn't succeeded in N intervals (absence alert), failure count |
| Queue consumer | Consumer lag / queue depth growing, DLQ messages > 0 |
| External dependency call | Dependency error rate, timeout rate, circuit breaker open |
| Cache | Hit ratio drop |
| Resource pool | Saturation (in-use / max > 90%) |

If alert rules live in this repo, offer to add them there. Otherwise output them for the user to create in their monitoring tool. Don't suggest "FYI" alerts or alerts on metrics that don't exist.

### Phase 5. Performance & Hygiene

Look at the changed code (not the whole repo) for:

**Performance**
- N+1 queries or calls inside loops that could be batched
- Missing pagination/limits on queries or list endpoints
- Queries filtering on columns that likely lack an index (check migrations/schema)
- Unbounded in-memory collections, reading whole files/responses into memory
- Repeated expensive work that could be cached or hoisted out of a loop
- Sequential independent I/O that could run concurrently
- Missing timeouts on outbound calls; retries without backoff/jitter
- Unnecessary allocations or copies in hot paths; inefficient serialization
- Holding locks or DB transactions across network calls

**Hygiene** (quick wins before pushing)
- Leftover debug output (`console.log`, `print`, `fmt.Println`, `dbg!`), commented-out code, stray TODOs without a ticket
- Hard-coded secrets, credentials, or environment-specific URLs
- New config/env vars not documented
- Public API or behavior changes without doc/README/changelog updates

Only report performance issues you can point to in the code; label speculative ones as "worth measuring" rather than stating them as problems. Offer to apply the straightforward fixes. If a fix changes behavior (e.g. adds a missing timeout or limit), write a failing test for it first, as in Phase 3.

### Phase 6. Preflight Report

Finish with a summary:

```
## PR Preflight: <branch name>

**Change summary:** <one paragraph>

| Check          | Status        | Notes |
|----------------|---------------|-------|
| Metrics        | ✅ / ⚠️ / ❌ / n/a | e.g. 2 added, 1 skipped by user |
| Logs           | ...           | |
| Traces         | ...           | |
| Unit tests     | ...           | e.g. 4 added, each failed first then passed |
| Integration    | ...           | |
| Alerts         | ...           | e.g. 3 suggested, not created |
| Performance    | ...           | |
| Hygiene        | ...           | |
| Tests/lint run | ...           | |

**Applied:** <list of changes made, with files>
**Still open:** <suggestions the user skipped or that need their action, e.g. creating alerts in Datadog>

**Verdict:** Ready for PR / Ready with follow-ups / Not ready (<blocking reason>)
```

Don't commit or push; leave that to the user.

## Principles

- **Repo conventions over best practices.** If the repo's pattern differs from textbook advice, follow the repo and mention the tradeoff once at most.
- **Cite locations.** Every suggestion names a file and line, and every convention cites an existing example.
- **Be proportional.** A 10-line bug fix needs a regression test, not a dashboard.
- **Ask before editing.** Suggestions are a menu; the user picks.
- **Red before green.** Every new test is seen failing before it is seen passing.
- **Don't over-instrument.** Too many metrics, noisy logs and over-sensitive alerts are defects too.
