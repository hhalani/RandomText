# Policy Summary API – Current-State Assessment & Case for Read-Replica Sourcing

Oct 5, 2026 · @Hamid Halani

## Executive summary

**Recommendation: retire the REST-pushed Policy Summary database and rebuild the Policy Summary capability on read replicas of every policy system of record (PolicyCenter auto, the property policy system, and the mainframe).**

The current Policy Summary API was built to shield the source policy systems from read traffic. It does not. Its database is populated only for Automobile policies, by the Auto policy system calling a Policy Summary REST endpoint directly on each change. Every Property request, every mainframe-held policy and every Auto policy whose update was missed falls through to the source system. In practice most calls end at the source, so the organisation pays for two read paths and gets the protection of neither — and has tightly coupled its policy system of record to a downstream read service in the process.

The root cause is structural, not a bug to patch:

- **Coverage is partial by design.** Only one line of business (Auto) from one system (the Guidewire policy system) pushes updates. Property and mainframe data have no feed at all.
- **A pushed copy can only be as complete as its pushes.** There is no backfill, no replayable history, no reconciliation and no way to prove the copy matches the source, so the API cannot trust its own store and must keep the source fallback.
- **The REST push creates tight coupling.** PolicyCenter depends on Policy Summary being up, fast and contract-compatible every time a policy changes, while Policy Summary depends on PolicyCenter for every store miss. The two systems are coupled at runtime, in their release cycles and in operations, in both directions.
- **The fallback keeps every coupling the design tried to remove.** Latency, availability and capacity of the API are still set by the source systems, and the source still carries the read load.

Read replicas fix the root cause. A replica is a full, continuously synchronised, read-only copy of each system of record, maintained by the database platform rather than by application-to-application REST calls. It covers every line of business and every historical record from day one, needs no push integration built into source systems, removes PolicyCenter's outbound dependency on Policy Summary, and takes read traffic off the primary databases and application servers entirely.

The approach has real costs, chiefly coupling to each source's physical schema, replication lag, licensing, and, if PolicyCenter runs on Guidewire Cloud, the absence of direct database access. These are named with mitigations in the Risks section. They are manageable; the current design's flaws are not.

**Decisions requested:** approve the target architecture, fund Phase 0 (baseline measurement) and Phase 1 (Auto in shadow mode), and confirm read-replica availability with each source platform team.

## Background and current-state architecture

The Policy Summary API serves a condensed view of a policy (insured, term, status, LOB, key coverages, premium) to downstream consumers. Its store is written only by the Auto policy system, which calls a Policy Summary REST endpoint directly when an Auto policy changes. Every other request is answered by a live call into a system of record.

&#91;embedded content: current state · REST-pushed store, three source fallbacks\]

Only the PolicyCenter Auto path has a store behind it; Property and mainframe requests always take the dashed path.

How a request is served today:

1. A consumer calls the Policy Summary API with a policy number (or account/customer key).
2. The API looks the policy up in the Policy Summary DB.
3. **Hit** (Auto policy whose events were captured): the stored row is returned. Freshness depends on the last successful REST push from PolicyCenter.
4. **Miss** (any Property or mainframe policy, or an Auto policy the events never captured): the API calls the source system synchronously, maps its response into the summary contract and returns it.

How the store is populated:

1. A PolicyCenter transaction on an Auto policy (for example a bind, policy change or renewal) triggers custom integration code in PolicyCenter.
2. That code builds a summary payload and calls the Policy Summary REST write endpoint directly, point to point.
3. The endpoint writes the row to the Policy Summary DB.
4. There is no broker, durable log or replay in between. If the call fails, times out or is never made, the store is wrong until a later change on the same policy happens to overwrite it.
5. Nothing compares the store with PolicyCenter afterwards.

## Original design intent vs. observed behaviour

The API was meant to answer policy summary reads from its own store and leave the source systems alone. Observed behaviour is the reverse: the store answers a minority of calls and the source answers the rest.

| Design goal | What was assumed | What actually happens |
| --- | --- | --- |
| Offload reads from the source systems | The summary store would hold every policy a consumer asks for | Store holds Auto only; Property and mainframe requests always go to source |
| Low, predictable latency | Most reads served from a local, indexed store | Most reads pay store lookup + miss + synchronous source call |
| Isolation from source outages and maintenance | Store answers even when the source is down | Source down = API down for every request that misses the store |
| Loosely coupled systems | The policy system and the summary service evolve and fail independently | PolicyCenter calls the Policy Summary REST endpoint directly; each system's availability, latency and contract now affect the other |
| Single trusted view of a policy | Every change is delivered to the store, in order, every time | Failed, timed-out, unmapped or out-of-order REST calls leave the store stale or incomplete, with no detection and no replay |
| One integration for consumers | Consumers call one API for any policy | Consumers still get different freshness and behaviour depending on which path served them |
| Cheaper to run than calling source | Store cost replaces source read cost | Organisation pays for the write endpoint, the store, the PolicyCenter push integration **and** the source read load |

The gap is not an implementation defect that tuning can close. A cache that holds one line of business from one system cannot become the primary read path for all lines from all systems. The fallback is therefore permanent, and the fallback is where the original problem lives.

## Detailed issue analysis

Thirteen issues follow, grouped by theme. Each states the problem, why it happens, and what it causes. Issues 1–5 are root causes; 6–13 are consequences.

### Data coverage

**Issue 1 — Only Automobile policies are in the store.** Property policies are never written to the summary database, so 100% of Property reads fall through to the source. Any consumer building a customer-level or household view (all policies for a named insured) gets a mixed answer: Auto from the store, Property from a live call, with different latency and freshness.

**Issue 2 — Mainframe-held policies are absent.** Legacy books on the mainframe have no feed into the store at all. Every mainframe read becomes a synchronous call into CICS/IMS transactions or batch-extract files, which carry MIPS/MSU cost, batch-window outages and the slowest response times in the estate.

**Issue 3 — No initial load or backfill.** A push-built store only knows policies that changed after the REST integration went live. Policies bound earlier and untouched since (long-term renewals with no endorsements, cancelled or expired history) are missing unless a one-time load was run and kept in sync. Historical terms and prior versions are generally unavailable, so "what did this policy look like on date X" always goes to source.

### Update-feed integrity and coupling

**Issue 4 — The store cannot prove it matches the source.** The store is correct only if every relevant Auto transaction made the REST call, every call succeeded exactly once, and calls were applied in order. None of these is verified, and a point-to-point REST push gives no durable record to check against. Typical failure modes:

- **Unmapped transactions.** The push is usually wired to specific job types (Submission, PolicyChange, Renewal, Cancellation). Less common paths — Rewrite, Reinstatement, out-of-sequence changes, preemption handling, admin data fixes, batch processes and direct DB corrections — often change the policy without calling the endpoint.
- **Failed or timed-out calls.** If Policy Summary is down, slow or returns an error, the update is lost unless PolicyCenter retries. If the push runs through a Guidewire messaging destination, failures pile up as errored messages that someone must resync from the admin console; if it runs inline, the failure is either swallowed or breaks the policy transaction.
- **No replay or backfill.** Unlike a log or a topic, a REST call leaves no history. A missed update can only be repaired by re-triggering it from PolicyCenter, or by waiting for the next change on that policy.
- **Ordering and retries.** Concurrent transactions on the same policy or account, and retries of earlier calls, can arrive out of order. Unless every write is version-checked, an older payload overwrites newer state.
- **Effective-dating.** PolicyCenter stores policies as effective-dated branches and slices. A back-dated or future-dated change alters the in-force view across a date range; a payload carrying only "the new period" can overwrite the right record with the wrong picture.
- **Thin payloads.** If the push carries only identifiers or a partial snapshot, the endpoint must call back into PolicyCenter to enrich it — a round trip in both directions on every change.
- **Contract drift.** Each PolicyCenter upgrade or product-model change (new coverage, new vehicle attributes) needs a payload change in Gosu and an endpoint change, released in lockstep. Drift shows up as null or missing fields, not errors.

Because none of this is reconciled against the source, the API team cannot tell consumers how stale or incomplete the store is. That uncertainty is precisely why the source fallback was kept — and why it is used so heavily.

**Issue 5 — The REST push tightly couples PolicyCenter to Policy Summary.** The system of record calls a downstream read service directly, and that read service calls back into the system of record on every miss. The two are coupled in four ways:

- **Runtime coupling.** PolicyCenter's transaction processing now depends on Policy Summary. If the call is synchronous in the transaction, Policy Summary latency is added to every bind, change and renewal, and a Policy Summary outage can stall or fail policy transactions. If it is asynchronous through a messaging destination, an outage builds an error backlog that blocks later messages for the same policy until it is cleared.
- **Circular dependency.** PolicyCenter → Policy Summary on every change, and Policy Summary → PolicyCenter on every miss. A slowdown in either system propagates to the other, and retries on both sides can amplify into a retry storm during an incident.
- **Release coupling.** Any change to the summary contract needs a Gosu change, a PolicyCenter build, regression testing and a deployment in PolicyCenter's release window. The summary service cannot evolve faster than the policy system, and PolicyCenter upgrades must re-test a custom integration that exists only to feed a read model.
- **Inverted ownership.** The system of record has to know who reads its data and what shape they want. Every new read model would need another point-to-point push from PolicyCenter, growing N×M integrations. Endpoint URLs, credentials, tokens and certificates for a consumer service live in PolicyCenter configuration, and every incident needs both teams to diagnose.

Open question to confirm with the PolicyCenter team: is the push made inline in the bind/commit path, or through a messaging destination? Inline makes Issue 5 a direct risk to policy transactions; a messaging destination makes it an operational-backlog risk. Both are tight coupling.

### Runtime behaviour

**Issue 6 — The fallback makes the source the primary read path.** For every miss, the request pays for the store lookup, then a synchronous source call. The store saves nothing on those requests and adds a hop. With Property and mainframe always missing, misses are the majority of traffic.

**Issue 7 — Latency is set by the slowest source.** Tail latency (p95/p99) of the API equals the tail latency of the PolicyCenter API layer and the mainframe transaction, plus overhead. Consumers with interactive journeys (agent portal, call-centre desktop, customer self-service, claims FNOL lookups) inherit that tail.

**Issue 8 — Availability is coupled to every source.** The API's effective availability is roughly the store's availability multiplied by each source's availability for the requests that reach it. PolicyCenter maintenance windows, upgrade cutovers, batch-process contention, and mainframe batch windows all become Policy Summary outages.

**Issue 9 — Read load still hits systems of record.** Each fallback call consumes PolicyCenter application-server worker threads, database connections and bean/query cache, in the same JVMs that underwriters and agents use to quote and bind. On the mainframe it consumes billable MIPS. On Guidewire Cloud it counts against Cloud API rate and consumption limits. A spike from a downstream consumer (for example a renewal-notice campaign or a portal outage retry storm) can degrade policy transaction processing.

**Issue 10 — Inconsistent answers for the same question.** The same policy can return different data seconds apart depending on whether it was served from a stale store row or the live source. Responses do not tell consumers which path served them or how fresh the data is, so consumers cannot reason about correctness.

### Design and operations

**Issue 11 — The data model is Auto-shaped.** The summary schema reflects personal auto concepts (vehicles, drivers, auto coverages). Property needs dwellings/locations, Coverage A–D style limits, perils, deductibles by peril and protection details. Mainframe products bring their own codes. Extending an Auto-shaped model one LOB at a time leads to nullable columns, LOB-specific branches in code, and a contract every consumer must re-learn.

**Issue 12 — Two read paths to build, test and run.** The team maintains the REST write endpoint, its error and retry handling, coordination with the PolicyCenter push code, the store schema and its migrations, **plus** a client for every source API, response mapping from each source into the summary contract, and the routing logic between them. Every field must be mapped twice and kept consistent. Testing must cover hit, miss, partial hit and stale-hit paths.

**Issue 13 — Delivery depends on source teams' roadmaps.** Adding Property or mainframe coverage requires those teams to design, build, test and support new point-to-point push integrations from their systems into Policy Summary, with their release cycles and change control. The Policy Summary roadmap is blocked on work that is not in its control and, for mainframe, may never be prioritised.

### Cross-cutting concerns

- **Cost:** the write endpoint and store, the custom push integration in PolicyCenter, plus source read cost, plus duplicated engineering — with a low share of traffic served from the investment.
- **Security and privacy:** customer PII is copied into another store whose retention and deletion depend on PolicyCenter pushing every change; a missed cancellation or purge call can leave data that should have been removed. PolicyCenter holds credentials for a consumer service, and the fallback calls run under a broad service account into the source.
- **Observability:** without a hit/miss ratio, push-failure count, reconciliation mismatch rate and staleness metric, the design's failure stays invisible and arguments about it stay anecdotal (see *Evidence to collect*).

## Business and technical impact

The impact lands on four groups: consumers of the API, the policy platform, the business, and the teams that run it. Severity reflects how directly each issue blocks the API's stated purpose.

| Area | Impact | Driven by | Severity |
| --- | --- | --- | --- |
| Policy transaction stability | PolicyCenter bind, change and renewal depend on Policy Summary being up and fast; an outage can slow or fail transactions, or build a message backlog that needs manual resync | Issue 5 | High |
| Customer & agent journeys | Slow or failed policy lookups in portals, call-centre desktop and FNOL whenever the source is slow or down | Issues 6, 7, 8 | High |
| Policy system load | Quote/bind/endorse performance in PolicyCenter degraded by read spikes from downstream consumers | Issue 9 | High |
| Enterprise customer view | No single call returns all of a customer's policies with consistent data; household and cross-sell use cases blocked | Issues 1, 2, 11 | High |
| Data correctness | Stale or incomplete Auto data served without any signal to the consumer; decisions made on wrong coverage or status | Issues 3, 4, 10 | High |
| Mainframe cost | Every mainframe policy lookup consumes billable MIPS; cost grows with consumer adoption | Issues 2, 9 | Medium |
| Delivery speed | Summary contract changes wait on PolicyCenter release windows; new LOBs wait on source-team push work; changes need coordinated releases across teams | Issues 5, 11, 13 | Medium |
| Run cost & toil | Two read paths, push-failure triage and resyncs, and source-client maintenance for a store that serves a minority of traffic | Issues 4, 12 | Medium |
| Privacy & compliance | Retention and deletion of copied PII depend on every push succeeding; gaps are undetected | Issue 4, cross-cutting | Medium |

The compounding effect matters most. Each new consumer onboarded to the API adds load to the source systems rather than to the store, so success of the API makes the platform less stable — the opposite of the original intent.

## Options considered

Five options were assessed against the problems above. Read replicas (Option D) are the only one that covers every source on day one without new work from source teams and without putting read load on systems of record.

| Option | Description | Coverage (Auto / Property / Mainframe) | Source load removed | Coupling | Dependency on source teams | Data completeness | Effort | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A. Status quo | Keep REST-pushed store + source fallback | Partial / None / None | Low | Tight, two-way | n/a | Unverified | None | Rejected — the problem |
| B. Replace push with events and extend | Move Auto from REST push to published events (Kafka); add Property and mainframe event feeds, backfill, reconciliation, replay tooling | Full, eventually | High, eventually | Loose at runtime; contract coupling remains | Very high — new publishing in every source | Good only with continuous reconciliation | High, multi-team | Rejected — slowest, blocked on others |
| C. Direct source calls | Drop the store; always call source APIs | Full | None — all reads hit source | Tight, one-way | Low | Exact | Low | Rejected — makes Issues 7–9 worse |
| **D. Read replicas** | **Query a read-only replica of each system of record through a unified service** | **Full from day one** | **Full — primaries untouched** | **None at application level** | **Low — platform/DBA enablement only** | **Exact, minus replication lag (seconds)** | **Medium** | **Recommended** |
| E. CDC into a central store | Stream DB changes (e.g. Debezium, IBM IIDR, Precisely) into a canonical operational data store | Full | Full | None at application level | Low–medium | Good; needs initial snapshot and schema handling | High | Future evolution of D |

Two points shape the choice.

- **Option B fixes one symptom with a similar mechanism.** Moving Auto from REST to Kafka would remove the runtime coupling of Issue 5, but every source would still have to publish complete, ordered, effective-date-aware events forever, and the store would still need reconciliation against the source to be trusted. Reconciliation against the source requires reading the source in bulk — which is what a replica provides.
- **Option E is D's natural successor, not its competitor.** Change-data-capture reads the same database logs that replication uses. Starting with replicas proves the per-source query models and canonical contract first; a CDC-fed store can replace a replica later, behind the same service contract, if cross-system query needs or schema-isolation needs justify it.

## Target architecture: read-replica-sourced Policy Summary

The new service reads every policy, for every line of business, from a read-only replica of its system of record. There is no event-built store to keep in sync and no fallback into source applications.

&#91;embedded content: target state · one service, three adapters, three replicas\]

The primaries keep transacting; summary reads stop at the replica layer and never reach them.

### Components

| Component | Responsibility | Implementation notes |
| --- | --- | --- |
| Canonical summary contract | One LOB-agnostic core (policy number, LOB, term, status, insureds, total premium, source system) plus LOB extension blocks (Auto: vehicles, drivers; Property: locations, dwelling coverages, perils) | OpenAPI-first; versioned; designed for all three sources before build |
| Aggregation layer | Resolves which source(s) hold a policy or customer, fans out in parallel, merges results | Spring Boot; `CompletableFuture` or virtual threads (Java 21) for fan-out; per-source timeouts |
| Resilience | Isolates a slow or failed replica so other sources still answer | Resilience4j circuit breaker + bulkhead per datasource; partial response with per-source status instead of total failure |
| Source adapters | Translate each source's schema into the canonical model | One module per source; SQL via jOOQ or Spring JDBC against versioned views; mainframe code-translation tables |
| Datasources | Read-only connections to each replica | Separate HikariCP pool per replica, read-only accounts, `SET TRANSACTION READ ONLY` / `ApplicationIntent=ReadOnly` as applicable |
| Cache (optional) | Absorb hot-key traffic | Caffeine or Redis, short TTL (seconds); never the source of truth |
| Freshness metadata | Tell consumers how current the data is | `asOf` timestamp and replica-lag indicator on every response |

### Design principles

- **The replica is the source of truth for reads.** The service never writes, and never calls a source application in the normal path.
- **Dependencies run one way.** Source systems no longer push to Policy Summary. PolicyCenter's REST call to the summary endpoint is removed, so PolicyCenter has no runtime, release or configuration dependency on the summary service.
- **Schema coupling is contained.** All knowledge of PolicyCenter, Property and mainframe tables lives in database views and one adapter per source; the canonical contract never leaks source structure.
- **Effective-dating is solved once.** The PolicyCenter adapter's views encode in-force-as-of logic (branch, slice and model selection) so consumers can ask "as of date X" correctly.
- **Fail partially, not totally.** A replica problem degrades one source's data, clearly flagged, not the whole API.
- **Change notifications are optional.** If consumers need to know that a policy changed, published policy events (where they exist) can invalidate cache or notify them, but they never carry the data the service serves.

## Issue-to-resolution traceability

Every issue in the current design maps to a specific property of the read-replica architecture. Two issues are fully removed only with additional work, noted under residual risk.

| # | Current issue | How the read-replica design resolves it | Residual risk |
| --- | --- | --- | --- |
| 1 | Auto only | Each LOB's system of record has a replica; Property is queried from its own replica | None once Property adapter is built |
| 2 | Mainframe absent | Mainframe data replicated to a distributed copy (DB2 replica or CDC target) and queried there; no CICS/IMS calls | Replication tooling and licensing for z/OS sources |
| 3 | No backfill or history | A replica is a full copy, including all terms, versions and history | None |
| 4 | Store unprovable vs source | Replica is maintained by the database engine from the transaction log, not by application REST calls; correctness is the platform's guarantee | Replication lag must be monitored |
| 5 | REST push couples PolicyCenter to Policy Summary | Push removed; PolicyCenter no longer calls the summary service, and the service never calls PolicyCenter's application tier | Coupling moves to the database schema (see Risks) |
| 6 | Fallback is the primary path | No fallback to source applications; every read is served from a replica | None |
| 7 | Latency set by slowest source | Indexed SQL on replicas; source app tier removed from the request path | Complex effective-dated queries need tuning and indexes |
| 8 | Availability coupled to sources | Replicas stay readable during app maintenance; per-source circuit breakers give partial results instead of total failure | Replica outage still affects its source's data |
| 9 | Read load on systems of record | Zero application or primary-DB load from summary reads | Replica sizing must absorb peak read traffic |
| 10 | Inconsistent answers | One path per source; response carries `asOf` / replica-lag metadata | Lag of seconds remains visible to consumers |
| 11 | Auto-shaped model | Canonical LOB-agnostic core plus LOB extension blocks, designed once for all three sources | Model design effort up front |
| 12 | Two read paths | One path per source; write endpoint, store, push integration and source clients retired | Per-source SQL/view maintenance |
| 13 | Blocked on source-team roadmaps | Needs only replica enablement and read access, not new push code in source systems | Schema changes on source upgrades (see Risks) |

## Risks, constraints and mitigations

The read-replica approach trades event-pipeline fragility for schema coupling and platform dependencies. Those are well understood and have standard mitigations; the first row is a go/no-go check.

| Risk | Why it matters | Mitigation | Likelihood / Impact |
| --- | --- | --- | --- |
| **No direct DB access on Guidewire Cloud** | If PolicyCenter runs on Guidewire Cloud Platform, customers cannot attach to the operational database or create a replica | Confirm hosting model first. If on GWCP, use Guidewire Cloud Data Access / Data Platform as the "replica" equivalent and validate its latency against freshness needs; keep Cloud API only for the rare real-time-critical field | Depends on hosting / Critical |
| Coupling to physical schema | Guidewire tables (e.g. `pc_policyperiod`, `pc_policy`, line and coverage tables) and mainframe layouts are internal and change on upgrade | Isolate all SQL behind a per-source adapter and a set of versioned database views owned by the Policy Summary team; contract tests run against each upgrade candidate in the source team's pipeline | High / Medium |
| Effective-dated data model | PolicyCenter branches, slices, `FixedID`, effective/expiration dates and `MostRecentModel` flags make "in-force as of date" queries easy to get wrong | Encode the as-of logic once in views; validate against PolicyCenter UI/API results for a sample set during shadow mode | Medium / High |
| Replication lag | Consumers could read data seconds behind a just-bound transaction | Monitor lag per replica; return `asOf` in responses; alert above SLO; offer a narrowly scoped source read only for documented read-your-writes flows | Medium / Low |
| Replica capacity | Summary traffic could saturate the replica and slow other replica users (reporting) | Dedicated replica or resource governor for the service; connection pooling (HikariCP) limits; short-TTL cache for hot policies | Medium / Medium |
| Licensing and infrastructure cost | Readable secondaries (e.g. SQL Server Always On, Oracle Active Data Guard) and mainframe replication tools carry licence cost | Price per source in Phase 0; offset against retired Kafka/store infra, MIPS savings and reduced PolicyCenter app-tier load | Medium / Medium |
| Mainframe replication | z/OS data (DB2, VSAM, IMS) needs CDC/replication tooling to a distributed target; code values need decoding | Use existing enterprise replication if present; otherwise phase mainframe last; build code-translation tables in the adapter | Medium / Medium |
| Security and access | Direct read access to systems-of-record data, including PII | Least-privilege read-only accounts on views only; column masking for fields the API does not expose; access logging; DBA and InfoSec sign-off | Low / High |
| Cross-system queries | A replica per system cannot join across systems in SQL | Federate in the service: parallel per-source queries joined on a shared customer/account key; move to Option E (CDC into a central store) if cross-system query volume demands it | Medium / Low |
| Business logic only in the application tier | Some derived values (e.g. computed premium views, display names, status rules) exist only in Gosu / application code | Inventory derived fields in Phase 0; replicate the rule in the adapter or source it from a persisted column; flag any field that cannot be derived | Medium / Medium |

## Migration roadmap and decommission plan

Migrate one source at a time behind the same consumer contract, starting with Auto because it is the only source where old and new can be compared side by side. Durations are to be set after Phase 0.

&#91;embedded content: migration roadmap · 6 phases, 5 gates (not to scale)\]

Each phase starts only when the previous gate is met; the old store stays in place until Gate 4.

| Phase | Scope | Exit gate |
| --- | --- | --- |
| 0. Baseline and feasibility | Instrument the current API (hit rate, source calls, latency, mismatch rate); confirm PolicyCenter hosting model; confirm replica or replicated-copy availability for all three sources; price licences; inventory derived fields that exist only in application code; draft the canonical contract | Replica path confirmed for every source; baseline metrics published |
| 1. Auto in shadow mode | Build PolicyCenter adapter and as-of views; run the new service in parallel and compare every response with the current API and with PolicyCenter | Field-level match rate and latency SLO met over an agreed window |
| 2. Auto cutover | Route Auto traffic to the new service; turn off Auto fallback calls; remove the REST push from PolicyCenter (Gosu integration code, endpoint configuration and credentials) | Zero Auto summary calls into PolicyCenter application tier; zero calls from PolicyCenter to Policy Summary |
| 3. Property | Build Property adapter and LOB extension; onboard consumers needing Property; customer-level view across Auto and Property | Property requests served from replica; no Property source calls |
| 4. Mainframe | Stand up z/OS replication to a distributed target; build adapter with code translation | Mainframe requests served from replica; MIPS from lookups eliminated |
| 5. Decommission | Retire the summary write endpoint, the Policy Summary DB and all source API clients; remove service-account access to source APIs | Old components shut down; cost and access removal confirmed |

Consumer impact is kept minimal by versioning the API: the new service exposes the canonical contract as v2 and, during Phases 1–2, a v1-compatible façade for existing Auto consumers.

## Evidence to collect and success metrics

The argument above is structural; numbers from production make it undeniable. Fill the *Current* column during Phase 0 and use the same measures as exit criteria for each later phase.

| Metric | How to measure | Current | Target (new system) |
| --- | --- | --- | --- |
| Store hit rate (% of calls served without source) | API logs/metrics tagged by serving path, split by LOB and source system | To measure | 100% served from replicas |
| Source calls per day from the API | API gateway logs + PolicyCenter/mainframe access logs for the API's service account | To measure | \~0 (only documented read-your-writes exceptions) |
| Latency p50 / p95 / p99, by serving path | APM traces (e.g. Dynatrace, AppDynamics, OpenTelemetry) | To measure | Agree SLO, e.g. p95 well below today's fallback path |
| API availability vs source availability | Synthetic checks + incident records; correlate outages with source maintenance windows | To measure | Independent of source app-tier maintenance |
| Push mode (inline vs messaging destination) | PolicyCenter integration code review with the PolicyCenter team | To confirm | Push removed |
| PolicyCenter transaction time spent on the push | APM spans on bind/change/renewal that include the call to the summary endpoint | To measure | Zero |
| Push failures and resyncs | Failed/timed-out calls to the write endpoint; errored messages and manual resyncs in PolicyCenter per month | To measure | Push removed |
| Policy transactions affected by Policy Summary incidents | Incident records where a Policy Summary outage slowed, failed or backlogged PolicyCenter transactions | To measure | Zero |
| Store vs source mismatch rate | Sample N Auto policies daily; compare store fields with PolicyCenter for the same as-of date | To measure | Mismatch eliminated (replica = source, minus lag) |
| Policies missing from store | Count in-force Auto policies in PolicyCenter vs store | To measure | n/a — full copy |
| PolicyCenter load from API | App-server threads, DB CPU and query count attributable to the API account | To measure | Zero on primary |
| Mainframe MIPS from policy lookups | SMF/chargeback reports for the relevant transactions | To measure | Zero on mainframe online regions |
| Replication lag | Replica monitoring per source | n/a | Within agreed SLO (e.g. seconds), alerting above it |
| Run cost | Endpoint, store, compute, push maintenance and source-read cost vs replica, licence and service cost | To measure | Lower total cost of ownership |

The first two metrics alone usually settle the debate: if most calls already reach the source, the store is not doing its job.

## Recommendation and decisions requested

Build the new Policy Summary service on read replicas of all policy systems of record, and retire the REST-pushed store once Auto, Property and mainframe are live. The current design cannot meet its own goal: it covers one line of business from one system, cannot prove its data is correct, sends most traffic to the systems it was meant to protect, and ties PolicyCenter to a downstream read service through a direct REST push.

Decisions requested:

- [ ] Approve read-replica sourcing as the target architecture for Policy Summary.
- [ ] Confirm PolicyCenter hosting model (self-managed vs Guidewire Cloud) and the resulting replica mechanism.
- [ ] Confirm how PolicyCenter calls the Policy Summary endpoint today (inline in the transaction or via a messaging destination).
- [ ] Agree with the PolicyCenter team to remove the REST push at Auto cutover (Phase 2), and to add no new pushes to Policy Summary meanwhile.
- [ ] Request read-replica (or replicated copy) availability and read-only access from the PolicyCenter, Property and mainframe platform teams.
- [ ] Fund Phase 0 (baseline measurement and replica feasibility) and Phase 1 (Auto in shadow mode).
- [ ] Freeze new feature work on the REST-pushed store; fix only defects until retirement.
- [ ] Agree consumer-facing freshness SLO (maximum replication lag) with the main API consumers.
