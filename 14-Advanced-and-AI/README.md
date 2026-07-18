# 14 — Advanced Topics & AI-Augmented Delivery

**JD coverage:** "Advanced Features (Parallel processing, Clustering, Design Patterns) and AI augmented design and development (MS Copilot usage in project delivery)", "Reusable Assets, Accelerators, Project Templates, Automation", "Performance tuning".

---

## 1. Parallel processing (the JD keyword, mapped to concrete Azure levers)

| Level | Mechanism | Notes |
|---|---|---|
| Within a workflow | **Parallel branches**; **For-each concurrency** (default ~20, max 50) | Watch shared-variable races → use Select/Compose not variables |
| Across workflow runs | Trigger concurrency settings; **SplitOn** debatching | Each item its own run = natural parallelism |
| Messaging fan-out | Topic subscriptions processed independently; competing consumers per queue | Scale consumers horizontally |
| Functions | Scale-out instances × per-instance concurrency (`maxConcurrentCalls`, `prefetchCount`, batch size for Event Hubs) | The tuning knobs interviews ask about |
| Durable Functions | **Fan-out/fan-in** pattern | Code-level scatter-gather |
| Partitioned scale | Event Hubs partitions; Service Bus partitioned entities; sessions = parallel-but-ordered-per-key | The "ordered *and* parallel" answer: partition by business key |

**The ordered-parallel trick to quote:** *"Parallelize across keys, serialize within a key"* — Service Bus sessions (session ID = customer/order ID) give per-key FIFO while sessions process in parallel.

## 2. "Clustering" translated to Azure

In Mule-land clustering = multiple runtime nodes sharing state. In Azure, you rarely "cluster" yourself — you pick the platform's scale/HA constructs:

- **Scale-out:** App Service plan instances (Logic Apps Standard), Functions elastic scale, APIM units.
- **Zone redundancy:** APIM Premium, Service Bus Premium, Functions AZ support — instances spread across datacenter zones.
- **Active-active multi-region:** APIM multi-region, paired Service Bus namespaces + Front Door.
- Singleton concerns (the "only one node runs this" cluster feature): Logic Apps **singleton workflow** (concurrency 1), Durable Functions singletons, or blob leases for distributed locks.

## 3. Performance tuning checklist (senior answers)

1. **Logic Apps:** built-in connectors over managed (Standard); reduce actions (each is a billed, latency-adding hop); avoid huge run history payloads; stateless workflows for hot paths; batch instead of per-item calls.
2. **Functions:** async I/O everywhere; reuse `HttpClient`/connections (DI singletons); right-size prefetch & concurrency for SB triggers; Premium plan to kill cold starts; measure with App Insights profiler.
3. **Service Bus:** prefetch + batched receives; avoid tiny lock durations; Premium messaging units for predictable latency; partition by key for throughput.
4. **APIM:** response caching; avoid heavy policy expressions on hot paths; scale units; disable chatty logging in prod.
5. **Payloads:** claim-check anything big; compress; trim fields at the edge.
6. **Downstream protection:** rate-limit at APIM, queue-based load leveling, circuit-breaker patterns (retry + timeout + fallback in policies/code).

## 4. Design patterns catalog (name-drop fluency)

From the [Azure cloud design patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) — the ones integration interviews expect you to know *by name with a one-liner*. **This table is the summary; the full deep-dive with diagrams, Azure implementations, and MuleSoft references per pattern is [Module 18 — Integration Patterns](../18-Integration-Patterns/README.md).**

| Pattern | One-liner | Where you've met it |
|---|---|---|
| Queue-based load leveling | Buffer bursts behind a queue | Module 05 |
| Competing consumers | Parallel workers on one queue | Module 05 |
| Publisher/subscriber | Topics decouple producers from N consumers | Module 05 |
| Claim check | Pass reference, not payload | Module 02 |
| Retry + exponential backoff | Transient-fault handling | everywhere |
| Circuit breaker | Stop hammering a dead dependency | Functions/Polly, APIM |
| Saga / compensating transaction | Distributed "transactions" via compensation steps | Durable Functions |
| Outbox | Atomic DB-write + message-send | Module 05 |
| Idempotent consumer | Safe reprocessing under at-least-once | Modules 04/05 |
| Strangler fig | Incrementally replace legacy (e.g., BizTalk→AIS migration!) | migration projects |
| Anti-corruption layer | Canonical model shields you from legacy formats | Module 07 |
| Gateway offloading/aggregation | Cross-cutting stuff at APIM | Module 06 |
| Choreography vs orchestration | Events vs central coordinator | LA vs EG design choice |

## 5. AI-augmented delivery (the JD explicitly asks — have a story)

**Tools & where they help in AIS projects:**
- **GitHub Copilot / Copilot Chat in VS Code:** writing Functions (C#), Bicep, KQL queries, XSLT/Liquid drafts, unit tests, YAML pipelines. Realistic claim: 30–50% faster on boilerplate; you still review everything.
- **Azure Copilot (portal):** ask "why did this Logic App run fail," generate CLI commands, explain resources.
- **Logic Apps AI capabilities:** workflow assistant; **AI/agent actions** connecting workflows to Azure OpenAI (e.g., classify inbound emails/documents inside a workflow) — "agentic" Logic Apps workflows are Microsoft's current flagship demo, worth knowing by name.
- **M365 Copilot:** drafting HLD/LLD/specs, meeting summaries → design notes.
- **AI in integration scenarios (architect flavor):** intelligent document processing (unstructured PDF/email → Azure AI Document Intelligence → structured order), semantic routing of tickets, LLM-assisted mapping-spec generation from sample payloads.

**Interview-ready sentence:** *"I use Copilot as an accelerator for code, IaC, KQL, and test scaffolding, with human review; and I've studied how Logic Apps now embeds AI actions so workflows can call Azure OpenAI for classification/extraction steps — I see clear use cases for supplier document intake."*

**Discipline points (say these too):** never paste secrets/customer data into AI tools; validate generated policy/security code; team standards for AI-generated-code review.

## 6. Reusable assets & accelerators (the architect's leverage)

Ideas you can build once and reuse — also *fantastic* portfolio artifacts:
1. **Error-handling framework** (Module 11 §4) as a deployable Bicep module + shared workflows.
2. **Bicep module library**: standard SB namespace, Function app with App Insights + KV wired, Logic App Standard baseline.
3. **Pipeline templates**: build/deploy YAML templates for each service type.
4. **Canonical schema repo** + versioning conventions.
5. **EDI partner-onboarding automation**: script + template agreements (Module 08).
6. **Postman/test harness kits** per interface pattern.
7. **KQL query pack + workbook** for integration health.
8. **Scaffolding CLI** (dotnet templates) for "new integration" projects.
Microsoft's own: **Logic Apps templates gallery**, **APIM/Integration landing-zone accelerators** ([APIM landing zone accelerator](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/api-management/landing-zone-accelerator)) — know they exist.

## 7. Adjacent tech to be conversant in (other companies ask)

- **Containers/AKS/Container Apps:** where custom microservices land when Functions aren't enough; Logic Apps Standard can run on Arc/containers.
- **Dapr** (Container Apps): pub/sub + bindings in the microservices world — conceptually your EIPs again.
- **Kafka / Confluent:** Event Hubs speaks Kafka protocol; hybrid estates often bridge them.
- **BizTalk migration:** many AIS programs *are* BizTalk→AIS migrations (Integration Account is BizTalk's cloud descendant — schemas/maps/orchestrations map to IA artifacts/Logic Apps). If your employer has BizTalk legacy this is gold: mention the **BizTalk migration tool** and strangler-fig approach.
- **Power Platform:** Power Automate shares the Logic Apps engine — know the governance boundary.

---

## 8. Labs

1. Sessions lab (if skipped in 05): prove "parallel across keys, ordered within key" with a session-enabled queue + 2 consumers.
2. Tune a Service Bus-triggered Function: measure throughput at default settings, then adjust `maxConcurrentCalls`/`prefetchCount`; graph the difference from App Insights.
3. Build one reusable asset end-to-end (recommendation: the error-framework Bicep module + shared error-handler workflow) and document it like a product README.
4. Copilot drill: generate a Bicep module, a KQL query, and an xUnit suite with Copilot; note what it got wrong — that critique is your interview story.
5. (Stretch) Logic Apps + Azure OpenAI action: classify a free-text "order email" into structured JSON inside a workflow.

---

## 9. Video & documentation library

- 📖 [Cloud design patterns catalog](https://learn.microsoft.com/en-us/azure/architecture/patterns/) ⭐
- 📖 [Logic Apps limits & config (throughput sections)](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config) · [Service Bus performance best practices](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-performance-improvements)
- 📖 [AI capabilities in Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-overview#built-in-ai-and-agent-capabilities) · search "Logic Apps agent workflows"
- 📖 [BizTalk to AIS migration guidance](https://learn.microsoft.com/en-us/azure/logic-apps/biztalk-server-to-azure-integration-services-overview)
- 🎥 Microsoft Build/Ignite sessions: search **"agentic Logic Apps"**, **"AIS AI integration Build 2025"**
- 🎥 Search **"Azure Functions performance tuning service bus"**

---

## 10. Interview questions for this module

1. How do you process in parallel while preserving order per customer? *(sessions — the golden answer)*
2. What does "clustering" mean in Azure PaaS terms? How do you run a singleton safely?
3. Name six cloud design patterns you've used and where.
4. A Logic App processes 1 msg/sec but 10k are queued — tuning walkthrough? (trigger concurrency, built-in connector, split work, Functions consumer instead?)
5. How do you protect a fragile downstream from bursts? (load leveling + rate limit + circuit breaker)
6. How do you use Copilot/AI in delivery today, and what guardrails do you apply? *(JD asks literally — have the §5 answer polished)*
7. What reusable assets would you build for a new AIS practice? *(§6 list, pick 4 with reasons)*
8. Have you seen BizTalk-to-AIS migrations? What maps to what? (schemas/maps→IA, orchestrations→Logic Apps/Durable, strangler approach)
9. Where would you *not* use AIS and pick containers instead?
10. What are agentic/AI workflows in Logic Apps? Give an integration use case.
