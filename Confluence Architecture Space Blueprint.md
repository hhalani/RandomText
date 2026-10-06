# Confluence Architecture Space Blueprint

Oct 2, 2026 · @Hamid Halani

## Why the current page is slow, and the rules that fix it

Split by decision and by C4 level: one page answers one question, carries at most 2–3 live diagrams, and links down for detail. A single page holding every option and diagram is slow because each draw.io/Gliffy/Lucid macro loads its own editor viewer, every attachment version is pulled into the page, and nested macros (Include Page, Jira issues, Children Display) re-render on every view.

Design principles for the new space:

- **One page, one purpose.** An overview, a decision, a component or an interface gets its own page.
- **Summaries go up, detail goes down.** Parent pages hold a short summary (via Excerpt) and links; diagrams and long tables live on child pages.
- **Each option is a child page.** The option-analysis page holds only the comparison table and the recommendation.
- **Decisions are immutable records.** ADRs are numbered, never rewritten; a new ADR supersedes an old one.
- **Metadata drives the index pages.** Every page carries a Page Properties block (status, owner, review date), and index pages use Page Properties Report instead of hand-maintained tables.
- **Templates enforce the shape.** Space templates (below) make the structure the default, not a discipline.

## Standard page tree

Use ten numbered top-level sections; the numbers fix the sidebar order and make the space feel the same across teams. Where your Confluence Cloud site has folders, use folders for the grouping levels (04 Solutions, domains, 06 Integration); otherwise use a short parent page with a Page Properties Report.

&#91;embedded content: architecture space page tree · 10 top-level sections\]

Shared content (standards, ADRs, the integration catalog, diagrams) lives once at the top and is linked from solutions, never copied into them. The full tree, ready to create:

```text
Architecture (space home)
├── 01 Principles & Strategy
│   ├── Architecture principles
│   ├── Technology radar
│   └── Target-state roadmap
├── 02 Standards & Patterns
│   ├── STD – API design (REST / OpenAPI)
│   ├── STD – Eventing & Kafka
│   ├── STD – Spring Boot service baseline
│   ├── STD – Security & IAM
│   ├── PAT – Transactional outbox / CDC
│   ├── PAT – Saga & compensation
│   └── NFR baseline
├── 03 Reference Architectures
│   ├── Microservice on Kubernetes
│   ├── Guidewire Cloud integration
│   └── Event-driven integration
├── 04 Solutions
│   └── <Domain>  (Policy · Billing · Claims · Customer · Shared)
│       └── SA – <Solution>
│           ├── Components
│           ├── Integrations      (links to INT pages)
│           ├── Data model
│           ├── Security
│           ├── Deployment & environments
│           ├── NFR – <Solution>
│           └── OA – <Problem>
│               ├── Option A: <name>
│               ├── Option B: <name>
│               └── Option C: <name>
├── 05 Decisions (ADR log)
│   ├── ADR-0001 – …
│   └── ADR-0002 – …
├── 06 Integration Catalog
│   ├── APIs              (INT pages)
│   └── Kafka topics      (TOPIC pages)
├── 07 Platforms
│   ├── Guidewire         (PolicyCenter · BillingCenter · ClaimCenter · GW pages)
│   ├── Kafka platform    (clusters · Schema Registry · Connect)
│   └── Microservices     (Spring Boot · Kubernetes · CI/CD)
├── 08 Diagram Library
├── 09 Governance         (ARB · review checklists · waivers)
└── 99 Archive
```

## Diagram and performance conventions

Keep live diagram macros to 2–3 per page; everything else is a lightweight image or a link. The table below sets the budget and the substitute for each heavy element.

| Heavy element | Budget per page | Use instead |
| --- | --- | --- |
| draw.io / Gliffy / Lucid live macro | 2–3 | Exported PNG/SVG image linked to the editable source on a child page |
| Same diagram on several pages | 1 source | Store it once on the Diagram Library page; reuse with draw.io "Embed diagram" (by reference), never copy |
| Option diagrams | 0 on the comparison page | One child page per option, each with its own diagram |
| Include Page / Excerpt Include | 3–4, no nesting | Excerpt Include of a short summary only; never include a page that itself includes |
| Jira Issues macro | 1, filtered, ≤ 20 rows | A link to the saved Jira filter |
| Children Display / Page Tree on large branches | 1, depth ≤ 2 | Page Properties Report filtered by label |
| Large tables (> 50 rows) | 0 | Attach the source spreadsheet or move to a child reference page |
| Attachments | Keep latest version | Delete old attachment versions; keep diagram source files under 5 MB |

Diagram rules:

1. Follow C4 levels: Context and Container diagrams on the solution page, Component diagrams on the component's own page, Code level only when needed.
2. Every diagram carries a title, a legend, a version and the date it was last reviewed.
3. Name diagram files `<domain>-<system>-<view>-<level>`, e.g. `claims-fnol-integration-container`.
4. Sequence diagrams for one flow each, on the integration page that owns the flow.
5. Mark diagrams that show a future state with a "Target state" banner so they are not read as current.

## Naming, labels and governance

Prefixes make pages findable in search; labels make them reportable. Use both on every page.

| Page type | Title pattern | Required labels |
| --- | --- | --- |
| Solution architecture | `SA – <Solution name>` | `solution-architecture`, `<domain>` |
| Architecture decision | `ADR-0042 – <Decision in a few words>` | `adr`, `<domain>`, `status-<status>` |
| Option analysis | `OA – <Problem>` and children `OA – <Problem> – Option A: <name>` | `option-analysis`, `<domain>` |
| Integration / API | `INT – <Source> to <Target> – <Flow>` | `integration`, `api` or `event` |
| Kafka topic | `TOPIC – <topic.name>` | `kafka-topic`, `<domain>` |
| Guidewire design | `GW – <PC/BC/CC> – <Feature>` | `guidewire`, `policycenter` / `billingcenter` / `claimcenter` |
| Standard / pattern | `STD – <Topic>` / `PAT – <Pattern>` | `standard` or `pattern` |
| NFR | `NFR – <Solution>` | `nfr` |

Governance:

- **Page Properties on every page:** Owner, Status (Draft, In review, Approved, Superseded, Retired), Last reviewed, Next review, Related ADRs.
- **Review cadence:** solution pages every 6 months; standards every 12 months; ADRs never edited after approval except the status field.
- **Archive, don't delete:** move retired pages under `99 Archive` and add the `archived` label so they drop out of reports.
- **Restrictions:** edit rights on `02 Standards` and `05 Decisions` limited to the architecture group; everything else open to the delivery teams.

## Template layouts

Load each of these as a Confluence space template (Space settings → Templates) so new pages start in this shape. Text in angle brackets is a placeholder; `{macro}` names the Confluence macro to insert.

### 1. Space home

```markdown
# Enterprise Architecture
{Page Properties Report: label = solution-architecture, columns = Owner, Status, Next review}

## Start here
- Principles · Standards · Patterns · Reference architectures

## Recent decisions
{Page Properties Report: label = adr, sort = Last reviewed desc, max 10}

## Solutions by domain
{Content by Label: label = solution-architecture, group by domain}

## How to contribute
Link to the templates page and the review process.
```

### 2. Solution architecture (parent page, light)

```markdown
# SA – <Solution name>
{Page Properties: Owner | Status | Version | Last reviewed | Next review | Related ADRs}

{Excerpt}
One-paragraph summary: business problem, scope, chosen approach.
{/Excerpt}

## 1. Context
Business drivers, in-scope / out-of-scope, stakeholders.
{draw.io: C4 Context diagram}            <- live diagram 1 of max 3

## 2. Solution overview
{draw.io: C4 Container diagram}          <- live diagram 2 of max 3
One line per container: responsibility and technology.

## 3. Key decisions
{Page Properties Report: label = adr AND <solution-label>}

## 4. Detail (child pages)
- Components · Integrations · Data model · Security · Deployment · NFRs

## 5. Risks, assumptions, open questions
| ID | Type | Description | Owner | Due |
```

### 3. Option analysis (comparison page, no diagrams)

```markdown
# OA – <Problem statement>
{Page Properties: Owner | Status | Decision due | Resulting ADR}

## Problem and constraints
3–5 lines. Link the requirements and NFRs.

## Evaluation criteria
| Criterion | Weight |
| Cost (5-yr TCO) | 25% |
| Delivery risk | 20% |
| Fit with standards | 20% |
| Performance / scalability | 20% |
| Operability | 15% |

## Options (one child page each)
| Option | Summary | Score | Link |
| A – <name> | one line | x.x | child page |
| B – <name> | one line | x.x | child page |
| C – <name> | one line | x.x | child page |

## Recommendation
Chosen option, why, what it costs us. Link to the ADR that records it.
```

Option child page:

```markdown
# OA – <Problem> – Option A: <name>
{Page Properties: Option | Score | Status = Considered / Chosen / Rejected}
## Description
{draw.io: option diagram}                 <- the diagram lives here, not on the parent
## How it meets each criterion
| Criterion | Assessment | Score |
## Cost and effort
## Risks and mitigations
## Pros / cons
```

### 4. Architecture decision record (ADR)

```markdown
# ADR-<0000> – <Decision in a few words>
{Page Properties: Status (Proposed/Accepted/Superseded/Deprecated) | Date | Deciders | Supersedes | Superseded by}

## Context
The forces at play: requirement, constraint, problem. 5–10 lines.

## Decision
"We will …" in one or two sentences.

## Options considered
Link to the OA page. One line per option and why it lost.

## Consequences
Positive · Negative · Follow-up actions (with Jira links).

## Compliance
How we will know it is followed (review checklist item, lint rule, fitness function).
```

### 5. Integration / API specification

```markdown
# INT – <Source> to <Target> – <Flow name>
{Page Properties: Owner | Status | Pattern (Sync REST / Async event / Batch / File) | Criticality | Related ADRs}

## Purpose
Business event or capability this flow serves.

## Flow
{draw.io: sequence diagram for this one flow}

## Interface contract
| Item | Value |
| Protocol / style | REST (OpenAPI 3) / Kafka / SFTP |
| Contract location | link to OpenAPI spec or schema registry subject |
| Auth | OAuth2 client credentials / mTLS |
| Idempotency key | <field> |
| Volumes | avg / peak per hour |
| Latency SLA | p95 <n> ms |

## Error handling
Retries and backoff · dead-letter route · compensation · alerting.

## Data mapping
Link to mapping sheet (attachment or child page); not inline.

## Security and data classification
PII fields, masking, encryption in transit / at rest.
```

### 6. Kafka topic / event contract

```markdown
# TOPIC – <domain.entity.event.v1>
{Page Properties: Owner team | Status | Environment(s) | Schema subject | Data classification}

## Event meaning
What happened, in business terms. When it is published.

## Producers and consumers
| Application | Role | Consumer group | Contact |

## Topic configuration
| Setting | Value |
| Partitions | <n> |
| Partition key | <field> and why (ordering guarantee) |
| Replication factor / min.insync.replicas | 3 / 2 |
| Retention / cleanup.policy | 7d / delete or compact |
| Schema format / compatibility | Avro / BACKWARD |

## Schema
Link to Schema Registry subject + one example payload (collapsed in an Expand macro).

## Delivery semantics
At-least-once / exactly-once · idempotent consumer rule · DLQ topic name · replay procedure.

## Versioning
How breaking changes are introduced (new .v2 topic, dual publishing window).
```

### 7. Guidewire configuration / integration design

```markdown
# GW – <PolicyCenter | BillingCenter | ClaimCenter> – <Feature>
{Page Properties: Owner | Status | GW version / Cloud release | Jira epic | Related ADRs}

## Business requirement
User story or capability, with link.

## Approach
OOTB vs configuration vs customization, and why (link ADR if customizing).

## Data model changes
| Entity / typelist | Change (ext entity, new field, typecode) | Reason |

## Rules and Gosu
Rule sets touched (Validation, Pre-update, Assignment), Gosu classes/enhancements, plugins.

## UI (PCF) changes
Screens touched; mock-up image (static), not a live diagram.

## Integration
Messaging destinations, App Events / Integration Gateway, REST APIs (Cloud API) used.
Link to INT and TOPIC pages; do not repeat their content.

## Upgrade and Cloud impact
Cloud standards compliance, upgrade risk, Guidewire Cloud Assurance notes.

## Testing
GUnit coverage, test data, regression scope.
```

### 8. Non-functional requirements

```markdown
# NFR – <Solution>
{Page Properties: Owner | Status | Approved by}

| ID | Category | Requirement | Measure / target | How verified | Status |
| NFR-01 | Availability | Core quote API available | 99.9% monthly | Synthetic monitoring | Agreed |
| NFR-02 | Performance | Quote response | p95 < 800 ms at 50 TPS | Load test | Draft |
| NFR-03 | Security | PII encrypted at rest | AES-256 | Pen test | Agreed |
| ... | Scalability · Resilience/RTO-RPO · Observability · Compliance | | | | |
```

## Worked example: Earnix rating via proxy

Store the Earnix integration as one SA page holding the C4 Context and Container diagrams. Give each rating flow its own INT page with its sequence diagram, and give the proxy a component page. Record the proxy decision in an ADR, and keep every diagram source once in the Diagram Library.

```text
04 Solutions
└── Policy
    └── SA – Rating (PolicyCenter × Earnix)        <- C4 L1 Context + L2 Container (2 live diagrams)
        ├── Components
        │   └── Rating Proxy                      <- C4 L3 Component diagram
        ├── Integrations                          <- Page Properties Report of INT pages below
        ├── Data model                            <- PC-to-Earnix field mapping (attached sheet)
        ├── NFR – Rating                          <- latency, availability, timeout budget
        └── OA – Rating integration pattern       <- only if alternatives were assessed
05 Decisions
└── ADR-00xx – Route PolicyCenter rating through a proxy
06 Integration Catalog
└── APIs
    ├── INT – PolicyCenter to Earnix – Quote rating     <- sequence: happy path + error path
    └── INT – PolicyCenter to Earnix – <other flow>     <- e.g. renewal or batch re-rate, if any
07 Platforms
└── Guidewire
    └── GW – PC – Rating plugin                   <- Gosu/plugin design; links to INT page
08 Diagram Library
    ├── policy-rating-earnix-context
    ├── policy-rating-earnix-container
    ├── policy-rating-proxy-component
    └── policy-rating-earnix-seq-quote
```

| Diagram | Type | Live on | Referenced from | Source file |
| --- | --- | --- | --- | --- |
| System context | C4 L1 | SA – Rating | Space home domain view (image) | `policy-rating-earnix-context` |
| Containers: PolicyCenter, proxy, Earnix | C4 L2 | SA – Rating | ADR-00xx (embed by reference) | `policy-rating-earnix-container` |
| Rating Proxy internals | C4 L3 | Rating Proxy component page | — | `policy-rating-proxy-component` |
| Quote rating flow | Sequence | INT – Quote rating | GW – PC – Rating plugin (link) | `policy-rating-earnix-seq-quote` |
| Timeout / error flow | Sequence | INT – Quote rating, inside an Expand | — | `policy-rating-earnix-seq-quote-error` |

No page carries more than two live diagrams; everywhere else the diagram is a link or a by-reference embed, so one edit updates it everywhere.

**Where the diagram source lives.** Prefer text sources (C4-PlantUML for C4, PlantUML or Mermaid for sequences) over hand-drawn shapes: they diff in Git, review in PRs, and render lighter. Keep the `.puml` files in the proxy's repo under `docs/architecture/`, and render them in Confluence with a PlantUML/Mermaid macro or by importing into draw.io (Arrange → Insert → Advanced → PlantUML). Link each page to its source file.

Starter sources follow. Technology labels marked `<…>` and the audit store are assumptions to replace with your actual proxy design.

C4 Container (goes on SA – Rating):

```plain
@startuml policy-rating-earnix-container
!include <C4/C4_Container>
title Container view – PolicyCenter rating through the Rating Proxy

Person(user, "Underwriter / Agent", "Creates and quotes policies")
System_Boundary(gw, "Guidewire InsuranceSuite") {
  Container(pc, "PolicyCenter", "Guidewire, Gosu", "Policy transactions; rating plugin calls the external rater")
}
System_Boundary(intl, "Integration layer") {
  Container(proxy, "Rating Proxy", "<Spring Boot / API gateway>", "Validates, maps PC model to Earnix, auth, retries, logging")
  ContainerDb(audit, "Rating audit store", "<database>", "Request/response for audit and replay (if used)")
}
System_Ext(earnix, "Earnix", "Rating / pricing engine")
System_Ext(idp, "Identity provider", "Issues service tokens")

Rel(user, pc, "Quotes policy", "HTTPS")
Rel(pc, proxy, "Rate request", "REST/JSON")
Rel(proxy, earnix, "Rating call", "<REST / SOAP>")
Rel(proxy, idp, "Gets token", "OAuth2")
Rel(proxy, audit, "Writes", "<JDBC>")
@enduml
```

Sequence – quote rating, happy and error paths (goes on INT – Quote rating):

```plain
@startuml policy-rating-earnix-seq-quote
title Quote rating – PolicyCenter → Rating Proxy → Earnix
actor "Underwriter" as U
participant "PolicyCenter\n(rating plugin)" as PC
participant "Rating Proxy" as P
participant "Earnix" as E

U -> PC : Quote
PC -> PC : Build rating request from PolicyPeriod
PC -> P : POST /rate (policy, coverages, correlationId)
P -> P : Validate, map to Earnix model
P -> E : Rating request
alt Rated
  E --> P : Premium per coverage
  P --> PC : 200 rating result
  PC -> PC : Create cost entities, set premium
  PC --> U : Quote displayed
else Timeout or Earnix error
  E --> P : Error / no response
  P --> PC : 5xx / 504 with error code
  PC --> U : Quote blocked, rating error shown
end
@enduml
```

## Migrating the current page

Break the existing page apart in place so links keep working: the old page becomes the new solution page and sheds its content to children.

1. Create the top-level page tree and load the eight templates as space templates.
2. Inventory the current page: list every diagram, option and table, and tag each with its target page type.
3. Create one option child page per option and move its diagram there (draw.io: copy the diagram to the new page, then delete it from the old one so the attachment is not duplicated).
4. Reduce the original page to the Solution architecture template: context and container diagrams only, plus links.
5. Record the chosen option as an ADR and link it from the option analysis page.
6. Move integration and Kafka detail to INT and TOPIC pages; replace them on the parent with a Page Properties Report.
7. Delete superseded attachment versions and check page load time before and after (target: under 3 seconds).
8. Add labels and Page Properties to every new page so the index pages fill themselves.
