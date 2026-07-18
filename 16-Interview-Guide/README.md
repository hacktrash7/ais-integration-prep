# 16 — The AIS Interview Guide

Everything you need to interview for AIS roles at **any** employer (the same interview happens at NTT DATA, Accenture, Cognizant, Capgemini, TCS, Infosys, LTIMindtree, EY/Deloitte/KPMG tech, HCLTech, Wipro, and product companies with Microsoft stacks).

**How interviews for this role typically run:**
1. **Screen (30 min):** resume walk, "why AIS from MuleSoft?", 5–10 rapid-fire fundamentals.
2. **Technical round 1 (60 min):** deep Q&A across Logic Apps / Service Bus / APIM / Functions + a small scenario.
3. **Technical round 2 / design (60 min):** whiteboard an E2E integration; EDI if the role touches B2B (manufacturing/enterprise roles often do); ops & security probing.
4. **Hiring manager (45 min):** behavioral, delivery stories, estimation, team fit; often a few "trap" technical checks.

Below: transition narrative → 200+ Q&A by topic (with crisp answers) → scenario/design questions with model answers → behavioral prep → company notes → question banks/links.

---

## 1. Your transition narrative (prepare this word-for-word)

**"Why are you moving from MuleSoft to AIS?"** — the certain first question. A strong shape:

> "Seven years in MuleSoft taught me integration engineering — API-led design, messaging, B2B, error-handling frameworks, CI/CD. Those fundamentals are platform-independent. My company's strategy (and frankly the market's) is Azure-first, so I invested in mapping every one of those skills to AIS: Logic Apps for orchestration, Service Bus for messaging, APIM for API lifecycle, Integration Account for EDI — and I've built [projects from Module 15] hands-on, including CI/CD with Bicep and monitoring with App Insights and KQL. I'm not starting over; I'm transferring 80% and I've deliberately closed the other 20% — the Azure platform itself and C#."

**"Isn't your Azure experience just personal projects?"** — counter: describe Project 3/6 with production-grade details (archiving strategy, idempotent resubmission, alerting, cost model). Depth of *decisions* — not resource-count — signals seniority. Also: "the discipline of running 24×7 integrations for 7 years is my production experience; the platform is what changed."

**Salary/level positioning:** you're not a fresher in integration — position as *senior integration engineer new to one toolset*, anchored by your 7 years. In **internal transitions** (moving within your current employer), emphasize domain knowledge: you already know the company's systems, partners, and processes — that's months of onboarding the external hire needs.

---

## 2. Rapid-fire Q&A bank by topic

> Format: **Q** → the crisp answer to give. Expand only when asked. Practice out loud; record yourself for the top 30.

### A. AIS fundamentals

1. **What services make up AIS?** Logic Apps (orchestration), Functions (code), Service Bus (messaging), Event Grid (events), APIM (API lifecycle), Integration Account (B2B/EDI); adjacent: Event Hubs, Data Factory, Storage.
2. **Logic Apps vs Functions?** Visual workflow + 1400 connectors + ops visibility vs code-first compute for complex logic; typically combined — "Logic Apps for plumbing, Functions for thinking."
3. **Logic Apps vs Power Automate?** Same engine family; PA = citizen/personal automation licensed per user, Logic Apps = enterprise IT, ARM-deployed, source-controlled.
4. **Logic Apps vs Data Factory?** Workflow/event integration vs bulk ETL pipelines; 100k-row batch = ADF, order-by-order processing = Logic Apps.
5. **What's serverless? Trade-offs?** No server management, pay-per-use, elastic; trade-offs: cold starts, execution limits, less runtime control.
6. **iPaaS vs ESB?** Cloud-hosted, managed, consumption-priced vs self-hosted central broker; AIS/Anypoint are both iPaaS; ESB thinking persists in patterns.

### B. Logic Apps

7. **Consumption vs Standard?** Multi-tenant pay-per-action vs single-tenant App Service hosting; Standard adds VNet, built-in connectors, many workflows/app, local dev + unit testing, stateless option. Choose Standard for enterprise/secure/high-volume, Consumption for simple/spiky/cheap.
8. **Stateful vs stateless?** Stateful persists every step (replay, run history, long-running); stateless runs in-memory (fast, cheap, short-lived, limited history).
9. **Trigger types?** Request (HTTP), Recurrence, polling (SFTP/SQL), push/webhook (Service Bus, Event Grid).
10. **Try/Catch?** Scopes + run-after: Catch scope configured to run after Try "has failed/timed out"; get details via `result('Try')`; Terminate sets final status.
11. **Retry policies?** Per-action: default 4 retries exponential; fixed; exponential w/ limits; none. For transient faults only.
12. **For-each parallelism gotcha?** Runs parallel (default ~20); shared variables race — use Select/Compose or concurrency 1.
13. **SplitOn?** Trigger-level debatch: array input → one run per element.
14. **Call child workflow sync vs async?** HTTP action to child's Request trigger: child Response = sync; child returns 202 + location = async (HTTP action auto-polls); or decouple entirely via Service Bus.
15. **Secure the HTTP trigger?** Rotate SAS, restrict inbound IP to APIM, front with APIM adding OAuth, Standard: private endpoints.
16. **Content-based routing options?** Condition/Switch in-flow; Service Bus topic subscription filters for decoupled routing (preferred at scale).
17. **Large message handling?** Chunking-capable connectors, claim check via Blob, Standard for higher limits, never loop 100k items in one run.
18. **Expressions you use daily?** `triggerBody()`, `body('X')`, `items()`, `coalesce()`, `concat()`, `if()`, `utcNow()`, `formatDateTime()`, `json()`, `xml()`, `base64()`, null-safe `?`.
19. **Connectors: built-in vs managed vs custom?** In-process (Standard, fast, free per-call) vs Microsoft-hosted (billed per call on Consumption) vs OpenAPI-defined custom.
20. **Resubmit a failed run?** From run history; Standard also from a specific action; ensure idempotency before resubmitting.
21. **B2B in Logic Apps?** X12/EDIFACT/AS2 encode-decode actions bound to Integration Account partners/agreements (full answers in section E).

### C. Messaging

22. **Service Bus vs Event Grid vs Event Hubs?** Commands/business messages with DLQ+ordering vs reactive discrete-event push routing vs high-throughput partitioned streaming. Order processing / blob-created reaction / IoT telemetry.
23. **Queue vs topic?** P2P competing consumers vs pub/sub with filtered subscriptions (virtual queues).
24. **Peek-lock lifecycle?** Lock → Complete/Abandon (deliveryCount++)/Dead-letter/Defer; lock expiry = redelivery; MaxDeliveryCount exceeded = auto-DLQ.
25. **Guarantee order?** Sessions: FIFO per session ID, one consumer per session; partition by business key for ordered-parallel.
26. **Exactly-once?** Doesn't exist over distribution; at-least-once + idempotent consumers + duplicate detection window ≈ effectively-once.
27. **DLQ strategy?** Monitor depth (alert!), diagnose via DeadLetterReason, fix, resubmit via tooling; never let DLQs silently grow.
28. **Standard vs Premium namespace?** 256KB vs 100MB messages; shared vs dedicated messaging units; Premium: VNet/private endpoints, geo-DR, JMS 2.0, predictable latency.
29. **Duplicate detection?** Broker drops same MessageId within window; complements (not replaces) consumer idempotency.
30. **Scheduled/deferred messages?** Future-visible enqueue (delayed retry); defer = park until fetched by sequence number.
31. **Claim check?** Blob for payload, message carries reference — solves size limits.
32. **Outbox pattern?** DB write + "message to send" in one local transaction; dispatcher publishes — fixes dual-write inconsistency.
33. **Event Grid delivery guarantees?** At-least-once push, 24h retry with backoff, then dead-letter to storage.

### D. APIM

34. **Why APIM in front of everything?** Security (OAuth/keys), throttling, stable contracts/versioning, analytics, portal, decoupling consumers from implementations.
35. **Policy sections?** inbound/backend/outbound/on-error, at global/product/API/operation scopes, `<base/>` inherits.
36. **Top policies?** rate-limit, quota, validate-jwt, ip-filter, cors, cache-lookup/store, rewrite-uri, set-header, xml-to-json, mock-response, send-request.
37. **Rate limit vs quota?** Short-window throttle (429) vs long-period allowance.
38. **validate-jwt — what's checked?** Signature (keys via OpenID config), issuer, audience, expiry, required claims/roles.
39. **Versions vs revisions?** Breaking side-by-side vs non-breaking iterations with "make current."
40. **Products/subscriptions?** API bundles with terms; consumer subscribes → keys; model tiers (Bronze/Gold).
41. **APIM→backend security?** Managed identity token, client cert, IP restriction on backend, VNet.
42. **Self-hosted gateway?** Containerized gateway on-prem/other clouds, managed from Azure — data stays local, control plane in cloud.
43. **SOAP handling?** Import WSDL — passthrough or SOAP-to-REST transformation.

### E. EDI / B2B

44. **X12 envelope structure?** ISA/IEA interchange → GS/GE group → ST/SE transaction set; control numbers at each level for tracking/dedup.
45. **Common X12 docs?** 850 PO, 855 POA, 856 ASN, 810 invoice, 997/999 acks, 820 payment, 940/945 warehouse.
46. **TA1 vs 997 vs 855?** Envelope ack vs syntax/functional ack vs *business* response document.
47. **Integration Account?** Artifact store + B2B engine: partners, agreements, schemas (XSD), maps (XSLT), certificates; links to Logic Apps.
48. **What's in an agreement?** Host/guest partners + qualifiers, protocol settings both directions: validation, acks, control numbering, batching, character sets.
49. **AS2 — what do signing/encryption/MDN give?** Authenticity/integrity, confidentiality, non-repudiation receipt (sync or async MDN).
50. **Inbound 850 end-to-end?** (Narrate Module 08 §3: AS2 decode → archive → X12 decode/validate/dedup → 997 out → map to canonical → SAP → 855 back.)
51. **Duplicate interchange handling?** Agreement-level control-number dedup + business idempotency on PO number.
52. **Archiving approach?** Raw + decoded + as-sent to Blob, partner/doctype/date/control-number naming, lifecycle tiers, retention per compliance.
53. **Partner onboarding at scale?** Templated agreements + IaC/scripts; test-indicator phase; cert rotation calendar.
54. **EDIFACT vs X12?** UNB/UNH vs ISA/ST; international vs North America; different separators/versioning (D96A etc.).
55. **SWIFT?** Financial messaging (MT/ISO 20022 MX); Logic Apps has SWIFT encode/decode; used in treasury integrations.

### F. Functions & C#

56. **Triggers/bindings?** Event sources (HTTP, SB, timer, blob, Event Grid) / declarative I-O (Cosmos, Blob, SB out) without SDK plumbing.
57. **Hosting plans?** Consumption (scale-to-zero, cold starts, time-limited) / Flex / Premium (pre-warmed, VNet) / Dedicated.
58. **Cold start mitigation?** Premium pre-warmed, keep-alive strategies, lighter deps, .NET isolated w/ ReadyToRun.
59. **Isolated vs in-process?** Isolated worker = separate process, current model, latest .NET; in-process retired for new work.
60. **Poison message handling in SB-triggered function?** try/catch → Complete/Abandon/DeadLetter explicitly; idempotency; DLQ alerts.
61. **async/await in one line?** Non-blocking I/O — thread released during waits; never block with `.Result` (deadlock risk).
62. **LINQ = your DataWeave?** `Select/Where/GroupBy/Aggregate/SelectMany` — map/filter/groupBy/reduce/flatten.
63. **Durable Functions patterns?** Chaining, fan-out/fan-in, async HTTP API, monitor, human interaction, saga; state checkpointed in Storage.
64. **DI in Functions — why?** Testability + lifetime management (singleton HttpClient!).
65. **Idempotent function — how?** Business-key dedup store, upserts, dup detection, deterministic side effects.

### G. Security

66. **Managed identity?** Azure-managed service principal per resource; token via platform; **no stored secrets**; system vs user-assigned (shared across resources).
67. **Client credentials flow?** App → Entra token endpoint (id+secret/cert) → JWT → API validates signature/issuer/audience/roles.
68. **Key Vault integration patterns?** App-setting references (Functions/LA Standard), APIM named values, Bicep references — always via managed identity.
69. **TLS vs mTLS vs message-level?** Server-auth channel encryption vs mutual cert auth vs payload-level sign/encrypt (AS2) surviving intermediaries/storage.
70. **Private endpoint vs VNet integration?** Inbound private IP for the service vs outbound app traffic through your VNet.
71. **PII in run history?** secureData flags on actions, log hygiene, data-classification-driven design.
72. **Service Bus auth?** Prefer Entra RBAC (Sender/Receiver) over SAS; private endpoints; disable public access if Premium.

### H. DevOps

73. **Bicep vs ARM?** DSL compiling to ARM; cleaner, modular; idempotent desired-state deployments.
74. **Logic Apps CI/CD — the catch?** Consumption: definition in ARM + API connections need per-env parameterization/auth; Standard: infra & code separate, connections via managed identity = clean.
75. **Pipeline auth to Azure?** Service connection — SPN or (better) workload identity federation, no stored secrets.
76. **Env config strategy?** Same artifact all envs; parameter files + app settings + Key Vault per env.
77. **APIM as code?** Policies as XML files in repo, Bicep/APIOps deploy, revisions for safe rollout.
78. **Zero-downtime Function deploy?** Staging slot + swap.

### I. Monitoring & ops

79. **Monitor vs App Insights vs Log Analytics?** Umbrella (metrics/alerts) / APM SDK-level telemetry / the log store you query with KQL; AI data lives in LAW.
80. **KQL basics?** `where`, `project`, `summarize by bin()`, `join`, `render`; e.g., failed runs by workflow last 24h.
81. **Trace one transaction across services?** Correlation ID at edge + propagate (headers/SB CorrelationId) + tracked properties + App Insights distributed tracing/App Map.
82. **Standard error framework?** Uniform error schema → error topic → central handler: log store, severity-based alerting, payload archive, resubmission path.
83. **Key alerts for messaging solutions?** DLQ depth, queue depth, failure-rate, latency P95, heartbeat "no runs in N hours," cert expiry.
84. **Splunk/Cribl with Azure?** Diagnostic settings → Event Hubs → Splunk/Cribl ingestion — central SIEM/observability alongside LAW.

### J. Architecture

85. **E2E design question** — see §3 scenarios below.
86. **HA vs DR for each service?** Zones for HA; SB geo-DR = *metadata only*; APIM multi-region; Logic Apps = redeploy from IaC; archives on GRS.
87. **Cost drivers?** Per-action (LA Consumption), enterprise connector rates, APIM units, SB Premium MUs; design levers: trigger conditions, batching, built-in connectors, right tiers.
88. **When NOT AIS?** Heavy custom compute → containers/AKS; giant ETL → ADF/Synapse; UI automation → Power Platform.
89. **Estimation approach?** T-shirt sizing per interface on drivers (pattern/transformation/systems/NFR/error handling) + platform foundations as separate line items.

---

## 3. Scenario & system-design questions (with model answers)

**S1. "Design supplier EDI onboarding for a manufacturer (200 partners, AS2+SFTP, orders→SAP)."**
Model: Module 13 §1 architecture; walk transport→decode→ack→canonical→SAP; archiving + reprocessing + partner-management story (Module 08 §4); IaC-driven partner onboarding; tracking dashboard; DLQ + heartbeat alerts; cost tiers. Close with rollout: pilot 5 partners → template → wave migrations.

**S2. "An order must update SAP, WMS, and CRM. SAP is slow and flaky."**
Model: intake → Service Bus topic; three subscriptions; SAP subscriber with retry/backoff + circuit-breaker + DLQ + resubmission; WMS/CRM unaffected (isolation = the point); idempotent SAP writes (order-number dedup); monitor DLQ.

**S3. "Real-time inventory API for customers, backend is on-prem SQL, 500 rps peak."**
Model: APIM (OAuth, rate limits, caching 5–30s TTL — discuss staleness trade-off) → Function → on-prem via VNet+ExpressRoute (or read-replica sync to Azure SQL via change-feed/ADF for scale & resilience — better answer: cache/replicate, don't hammer on-prem). Load-test; P95 SLO; App Insights.

**S4. "A Logic App run failed at 2 AM; business found out at 9 AM. Fix the process."**
Model: it's an observability gap — diagnostic settings, failure + heartbeat alerts to on-call (action groups), error framework with severity routing, DLQ-based durable capture (not just run failure), runbook + resubmission tooling, weekly ops review of alert quality.

**S5. "Migrate 300 BizTalk/MuleSoft interfaces to AIS."**
Model: inventory & classify (T-shirt + pattern), strangler-fig coexistence (route per-interface at APIM/edge), platform foundations first (error framework, CI/CD, monitoring, standards), wave planning by business risk, regression packs per interface (golden files), parallel-run validation period, decommission gates.

**S6. "Message ordering matters for stock updates but you need throughput."**
Model: sessions keyed by SKU/store — ordered within key, parallel across keys; explain lock, session-aware consumers, and why global FIFO is an anti-goal.

**S7 (C# check). "Given an order list, produce total value per customer for orders > $100, sorted desc."**
```csharp
var result = orders
    .Where(o => o.Amount > 100)
    .GroupBy(o => o.CustomerId)
    .Select(g => new { Customer = g.Key, Total = g.Sum(o => o.Amount) })
    .OrderByDescending(x => x.Total)
    .ToList();
```
Be ready to write ~this level of LINQ live, plus a simple async HTTP call with error handling.

---

## 4. Behavioral & delivery questions (have STAR stories ready)

Prepare 6 stories from your MuleSoft years (they transfer 100%):
1. A production incident you led through (root cause via logs, comms, prevention).
2. A design you fought for / got overruled on and what happened.
3. Handling a difficult trading partner / system owner.
4. Delivering under an impossible deadline (scope trade-offs).
5. Mentoring/upskilling someone.
6. **Your self-driven AIS upskilling itself** — the discipline of it is a story interviewers respect.

For the JD's soft-skill rows ("logical and reasoning skills, communication, leadership qualities to manage a team"): expect a live problem decomposition ("how many EDI messages might a manufacturer process daily? estimate it") — practice thinking aloud in structured steps.

---

## 5. Company-specific notes

**Manufacturing / building-tech enterprises (common AIS profile):** HVAC, security, or industrial products — expect supplier/customer EDI (orders/invoices/ASNs), SAP-centric ERP flows, field-service platforms, IoT-adjacent telemetry. Frame examples around supply chain + manufacturing. **Internal transition:** name systems/processes you already know — that's your moat. The JD ladder means they'll calibrate you Developer/Sr./Architect *during* the interview — answer with architecture framing wherever you can to bias upward.

**GSIs (NTT DATA, Accenture, Cognizant, Capgemini, TCS, Infosys, Wipro, HCLTech, LTIMindtree):** breadth-first rapid fire (the §2 bank is exactly their style), often BizTalk-migration projects, client-facing communication checks, and "which certs do you hold?" (AZ-204 moves the needle — Module 17).

**Consulting/product (EY, Deloitte, KPMG, ISVs):** heavier on design rounds (S1–S6 style), cost/licensing fluency, and DevOps maturity; expect APIM + landing-zone vocabulary.

**What everyone asks that many enterprise JDs understate:** Event Grid vs Service Bus vs Event Hubs; Durable Functions; Bicep/IaC; "how do you monitor"; managed identity. All covered — don't skip Modules 05/09/10/11.

---

## 6. Question banks & prep links

- MS Learn practice: [AZ-204 practice assessment](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/practice/assessment?assessment-type=practice&assessmentId=35) (free, closest thing to an official question bank)
- Search: **"Azure Logic Apps interview questions"**, **"Azure Service Bus interview questions"**, **"Azure Integration Services interview questions"** — cross-check anything you read against the MS docs links in each module (many blog answers are outdated; noticing that *is* a skill).
- [Azure Architecture Center patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) — design-round ammunition.
- YouTube: search **"Azure integration services mock interview"**, **"AZ-204 exam cram John Savill"**.
- For C# rounds: LeetCode Easy in C# (10–15 problems) purely to get fluent typing C# under pressure — AIS roles rarely go beyond that.

---

## 7. Final checklist before any interview

```text
[ ] Transition narrative rehearsed (60–90 seconds)
[ ] Top-30 rapid-fire answers rehearsed out loud
[ ] 2 scenario walkthroughs whiteboarded from memory (S1 + one other)
[ ] Portfolio repos polished; capstone demo video ready
[ ] 6 STAR stories written down
[ ] Questions to ask THEM prepared (platform maturity? BizTalk legacy? team standards?
    EDI partner count? Logic Apps Standard vs Consumption estate? on-call model?)
[ ] Re-read the target company's JD; map every line to your evidence
```
