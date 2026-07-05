# Example: Session-Cache OOM Investigation

A worked, end-to-end walkthrough of the 10-phase methodology on a synthetic-but-realistic incident. It exercises every phase, including **Phase 0 (tool readiness)**, the **maximum-pain pass** in Phase 2, and **codebase-memory-first** search in Phase 9.

**Scenario:** A stateless JSON API service (`orders-api`) begins returning 502s during evening traffic. Latency climbs over ~90 minutes until the process is OOM-killed and restarted by the orchestrator, then the cycle repeats. It started "sometime this week."

---

## Phase 0: Line Up Your Tools Before Investigating

Before opening the file, inventory what this needs: request/latency metrics, the process memory curve, the deployed source (not `main`), and a way to trace what allocates. Two gaps found and fixed first:

- The metrics dashboard showed host memory, not **per-process RSS** — added a process-level exporter (5 min). Without it, "the host has RAM free" would have been misleading.
- The **code-index / codebase-memory MCP** had not indexed this repo. Indexed it before starting, so Phase 3/9 could ask structural questions (callers, data-flow) instead of grepping.

Only then: open the investigation.

## Phase 1: Open Investigation

**Symptom:** `orders-api` RSS grows monotonically from ~180MB at start to ~2GB over ~90 min, then OOM-kill (exit 137) + restart. During the climb, p99 latency rises from 40ms to >3s; 502s begin around 1.4GB.
**Impact:** Intermittent checkout failures for ~10 min around each restart, several times per evening. Severity: High (revenue-affecting, recurring).

### Established timeline
| Time (UTC) | Event | Confidence | Source |
|---|---|---|---|
| Tue 14:10 | Release `2025.11.3` deployed | High | deploy log |
| Wed 19:40 | First OOM-kill (exit 137) | High | orchestrator events |
| Before Tue 14:10 | No OOM-kills in prior 30 days | High | orchestrator history |
| "This week, evenings" | User first noticed 502s | Medium | user recollection (unverified) |

## Phase 2: Initial Hypotheses + the maximum-pain pass

First-cut list (8 hypotheses, summing ~100%):

| # | Hypothesis | Category | First-cut P |
|---|-----------|----------|-------------|
| H1 | A leak introduced in release `2025.11.3` (our recent change) | recent-change | 20% |
| H2 | Unbounded in-memory cache growing with unique traffic | memory/design | 15% |
| H3 | Upstream DB driver leaking connections/buffers | dependency | 15% |
| H4 | Traffic simply higher in the evening → legitimately needs more RAM | load | 10% |
| H5 | A third-party client library regressed in a minor bump | dependency | 15% |
| H6 | Container memory limit lowered by an infra change | config | 10% |
| H7 | GC/allocator fragmentation under sustained load | runtime | 10% |
| H8 | Other / unknown | — | 5% |

**Maximum-pain pass.** The comfortable hypotheses here are the exculpatory ones — H3/H5 (a *dependency* is at fault) and H4 (it's *just load*, nobody's bug). The painful ones implicate our own release and our own design: H1 (our commit leaked) and H2 (we built an unbounded cache and never capped it). Correcting for motivated reasoning:

- Raise **H1 → 30%** and **H2 → 25%** (painful; also high base rate — the OOMs started the day after our deploy, which the timeline supports as an ordering fact, not just a hunch).
- Lower **H4 → 5%** (exculpatory; and evening load has been higher for months without OOMs — weak).
- Lower **H5 → 8%** (exculpatory unless a bump lands in the timeline).

Pick the **highest-pain credible** hypothesis to test first: the intersection of H1 and H2 — *"our release added or exposed an unbounded cache."* Cheapest to eliminate, ends the investigation if true.

## Phase 3: Evidence Collection (non-destructive, code read via the graph)

- **E1 (ordering):** OOMs began <30h after `2025.11.3`; none in the prior 30 days. Supports a recent-change cause; consistent with H1/H2. High confidence.
- **E2 (memory shape):** RSS grows *linearly with cumulative distinct requests*, not with concurrent load — it never drops after traffic dips. A leak/accumulation signature, not a working-set signature. Refutes H4 (load) and H7 (fragmentation would plateau). High.
- **E3 (heap):** A heap sample at 1.6GB shows ~78% retained by one map keyed by a per-request string. Points squarely at H2.
- **E4 (code graph, not grep):** Using the code-index — `trace_path` on the request handler → a memoization cache `sessionResultCache.set(key, ...)` added in `2025.11.3`. `search_graph` for reads/writes of that map: **many `.set(...)`, zero eviction, zero `.delete(...)`, no max-size guard.** The key is `userId + ":" + requestHash` — effectively unique per request, so the map only grows. This is the H1∩H2 hypothesis, confirmed structurally.
- **E5 (diff):** the `2025.11.3` diff added the cache "to speed up repeated lookups"; the PR description says *"we can add eviction later."*

**Web-research checkpoint:** searched the cache library's docs — the constructor accepts a `maxSize`/TTL that was simply not passed. Known, documented; our omission, not the library's bug (further deflating H5).

## Phase 4: Revised Hypotheses

| # | Hypothesis | First-cut P | Revised P | Evidence |
|---|-----------|-------------|-----------|----------|
| **H2/H1** | **Unbounded memoization cache added in `2025.11.3`, keyed near-uniquely, never evicts** | 25%/30% | **96%** | E2, E3, E4, E5 all converge |
| H3 | DB driver leak | 15% | 1% | E3 shows retention is the cache map, not driver buffers |
| H4 | Just load | 5% | 0% | E2 refutes (no drop after dips) |
| H5–H8 | others | — | 0–1% | not supported |

H2/H1 exceeds 80% from passive evidence. **Skip Phase 5** — an experiment can't add certainty the heap sample + call-graph already give. (If the heap had been ambiguous, the experiment would be: deploy the same build with the cache disabled behind a flag and watch the RSS curve flatten.)

## Phase 6: Final Revision
H2/H1 stays at 96%. Proceed to blame.

## Phase 7: Blame Assignment (three levels)

- **Level 1 (lines):** the `sessionResultCache = new Map()` module global + the unconditional `.set(key, value)` in the handler, with a near-unique key and no eviction/cap. Critical.
- **Level 2 (anti-pattern):** *unbounded accumulation keyed by unbounded input* — a cache whose key space scales with request volume and whose size is bounded by nothing. "We'll add eviction later" is architecture-by-TODO. Same class: any collection that only ever grows.
- **Level 3 (style):** no size/'TTL is required' review rule for caches; a cache API that lets you construct it *without* a bound (the unsafe default is reachable); no memory-regression test; latency/RSS not gated in CI.

## Phase 8: Immediate Fix
Pass the library's `maxSize` + TTL (bounded LRU); add a metric for cache size and evictions; add a failure-path test that fills the cache past the cap and asserts eviction + flat RSS. Deploy. RSS now plateaus at ~260MB under the same evening traffic; no OOM.

## Phase 9: Anti-Pattern Search (code graph first)
Spawn parallel search agents, each instructed to use the codebase-memory MCP as the **primary** tool (grep only as fallback): `search_graph` for collection-typed module globals; `query_graph` (Cypher) for "every `Map`/`Set`/`Vec`/list field that has a `.set`/`.push`/`.insert` writer but no eviction/`delete`/cap in any method." Found 7 more unbounded collections: 2 Critical (a metrics label map keyed by raw URL; a per-tenant buffer with no cap), 3 High, 2 Low (bounded-by-config, documented). Grep alone would have found the *names*; the graph found *which ones lacked the paired eviction* — the actual finding.

## Phase 10: Remediation Plan (three levels → three scopes)
- **L1 → Phase 1 (today):** cap this cache + fix the 2 Critical siblings.
- **L2 → Phase 2 (this week):** a shared bounded-cache helper that *cannot* be constructed without a size/TTL (make the unsafe default unrepresentable); apply it to the 3 High findings; a CI memory-regression check on the service.
- **L3 → Phase 3 (next sprint):** a review checklist item — "every cache/collection has a documented bound"; an allocator/RSS budget gate in load tests.
- **What NOT to do:** don't just bump the container memory limit (moves the cliff, doesn't remove it); don't disable the cache entirely (the speedup is real — bound it, don't delete it).

---

## Key Lessons From This Investigation

1. **The maximum-pain pass pointed at the answer before any evidence.** The exculpatory hypotheses (dependency, "just load") were comfortable and wrong; the painful one (our own release added an unbounded cache) was correct — and the timeline already supported testing it first.
2. **"Add eviction later" was the root cause.** The author knew the cache needed a bound, wrote it in the PR, and shipped without it. The comment/PR note *is* the smoking gun (Phase 3 E5).
3. **The memory *shape* eliminated half the list cheaply.** RSS that grows with cumulative distinct requests and never drops is an accumulation signature — it refuted "just load" and "fragmentation" with a single graph, no experiment.
4. **The code graph found the real finding; grep would have found only names.** "Which growing collection lacks the paired eviction" is a structural question — exactly what `trace_path`/`query_graph` answer and text search does not.
5. **Fix the class, not the instance.** One OOM revealed 7 more unbounded collections. The durable fix was a bounded-cache helper that makes the unsafe construction impossible — turning a style rule into a compile-time/API guarantee.
