# Research: What observability engineering involves, and what an `observability-engineer` agent should own

Open-ended question: what work does observability engineering involve (its practices, the role, and the lens an expert
applies when reviewing code, designs, and plans), so that Han can define an `observability-engineer` agent?

Evidence mode: **strict** (default). Every claim that bears on the recommendation is corroborated by an independent
source or by codebase evidence, or is marked `[single-source]` and does not stand alone.

## Summary

Observability engineering is the work of making a running system answer questions about itself. In practice that means
deciding what the software records, how those records connect across services, what gets kept and what gets thrown away
to control cost, how "working well enough" is defined for users, and which of those signals should wake a person up. The
role, as employers describe it, builds and runs that tooling, sets the standards other teams follow, and goes looking for
blind spots after incidents.

Han already covers part of this. Two existing agents each hold a slice: the DevOps reviewer runs a short observability
checklist as one of many readiness checks, and the on-call reviewer checks that new code logs enough to be debugged.
Neither goes deep on the design questions at the center of the discipline: what to sample, how to keep costs from
exploding, how to name and connect telemetry, and how to tune alerts so they fire for real user pain.

The recommendation is to build a focused `observability-engineer` that owns telemetry design across both code and
configuration, and to move the observability depth out of the DevOps reviewer so the two do not compete. It should not
copy the whole job description: running the platform, picking vendors, and training people are things a person does,
not things a read-only reviewer can do. Several popular ideas in the field (the "wide events" school, cost-reduction
advice, and maturity models) come mainly from vendors with a product to sell, so the agent should treat them as
vocabulary rather than as rules.

The practice areas and the role are well-corroborated; the case for a separate agent rests on codebase evidence plus a
softer role distinction.

- **Confidence:** Medium
- **Web search:** used

## Research Results

### Observability means answering new questions, not only watching known ones

The neutral definition, from the OpenTelemetry project, is understanding a system from the outside by asking it
questions, including ones nobody predicted (A1). Monitoring, as the Google SRE book describes it, watches known
conditions and pages a person when users feel pain (A4). Charity Majors, a founder of the vendor Honeycomb, draws the
line more sharply: monitoring handles the problems you anticipated, and observability lets you investigate the ones you
did not, without shipping new code (A2).

The two framings agree on the goal and disagree on the requirements. Majors argues that real observability needs one
detailed, structured record per request (a "wide event") with many high-variety fields, aggregated at query time (A2,
A3). The OpenTelemetry definition does not demand that design (A1). The sharper position comes from a vendor that sells
it, and a practitioner talk on Grafana's site calls it "more of a North Star than a silver bullet" (A24)
[single-source]. The research does not settle which framing is canonical, so an agent should know both.

### The work splits into eight practice areas

The two research angles converged on the same body of work. Each area below is something an observability engineer
designs, reviews, or maintains.

- **Choosing signals.** Systems emit traces (the path of one request across services), metrics (cheap numeric
  aggregates), and logs, with profiles still emerging (A1, A10). Each fits different questions. Averages hide slow
  outliers, so latency needs histograms and high percentiles (A4, A6).
- **Instrumenting code consistently.** OpenTelemetry, the open standard most of the industry instruments with, supplies
  libraries plus shared naming rules called semantic conventions, so telemetry from your code, libraries, and platforms
  lines up (A12, A1).
- **Connecting signals across services.** Trace context (a trace ID and span ID) must travel with every request between
  services, by default in the W3C `traceparent` header, and logs should carry the same IDs (A10, A18). OpenTelemetry also
  warns that baggage, its side channel for key-value data, must not carry secrets, and that inbound context is untrusted
  (A10) [single-source].
- **Controlling cardinality.** Cardinality is the number of distinct values a field can take. Every distinct label value
  on a metric creates a new time series, so a per-user label multiplies cost (A14, A15). Prometheus advises keeping label
  values under about ten and treating more than a hundred as a reason to rethink (A14) [single-source on the numbers].
- **Sampling.** Head sampling decides at the start of a request whether to keep it. It is cheap but cannot promise to keep
  the errors. Tail sampling decides after the request finishes, so it can keep errors and slow requests, but it holds
  state, costs more to run, and can itself fail under load (A9, A2). Teams often combine the two (A9).
- **Running the telemetry pipeline.** The OpenTelemetry Collector is a service that receives, processes, and forwards
  telemetry. It batches, retries, filters sensitive data, and can send to several backends (A11) [single-source]. Sending
  straight to a backend is fine for small setups (A11).
- **Keeping sensitive data out.** OpenTelemetry cannot know what is sensitive, so the team owns it. Collect only what you
  need, review what instrumentation libraries add on their own, scrub in the pipeline, and do not trust hashing to hide
  IDs drawn from a small set of possible values (A13, A18).
- **Defining service levels and alerting on them.** A service level indicator (SLI) measures what users feel, a service
  level objective (SLO) sets the target, and the error budget is how much failure the target allows (A6). Alerts should
  fire on symptoms users feel, and every page should be urgent, actionable, and new (A4). The SRE workbook recommends
  alerting on how fast the error budget is burning, over two time windows at once (A5). For a 99.9% target, its example
  pages at 14.4 times the normal burn rate over one hour and 6 times over six hours, and opens a ticket at 1 times over
  three days (A5) [single-source on the thresholds]. The workbook names the costs: many parameters, poor fit for
  low-traffic services, and trouble with extreme targets like 99.999% (A5).

Three standard checklists frame service dashboards. The Four Golden Signals are latency, traffic, errors, and saturation
(A4). RED (rate, errors, duration) applies per service, and USE (utilization, saturation, errors) applies per resource;
the RED method's author recommends using the two together (A7, A8).

### The weakest-evidenced areas come from vendors

Four practice areas have only vendor or secondary support, and an agent should carry them as vocabulary with that caveat.

- **Cost management.** Named levers are cardinality control, dropping or hashing labels in the pipeline, sampling,
  tiered retention, and trimming debug and duplicate logs (A15, A16, A9). Every cost source found sells tooling in this
  space, and no independent benchmark surfaced.
- **Observability-driven development.** The practice is to instrument while writing the code, ask "how will I know when
  this isn't working?", and check production after each deploy (A25). Two vendors agree on the practice; nobody
  published independent evidence of outcomes.
- **Maturity models.** Honeycomb's model scores five capabilities: incident resilience, code quality, release cadence,
  complexity and technical debt, and understanding user behavior (A26) [single-source]. The level definitions could not
  be retrieved.
- **"Three pillars are not enough."** Cindy Sridharan's book argues that having logs, metrics, and traces does not make a
  system observable (A23) [single-source, secondary report only; the primary text was paywalled or blocked].

### The role builds the platform and sets the standards

Job postings describe the observability engineer as a platform-and-standards role (A19, A20). The recurring duties are:

- Design, roll out, and improve the monitoring, logging, and tracing stack (A19, A20).
- Set instrumentation standards and drive adoption across teams (A19, A20).
- Build dashboards and alerts tied to SLOs, and cut alert noise (A19, A20).
- Lead root cause analysis and post-incident reviews to find visibility gaps (A19, A20).
- Wire observability into CI/CD and work alongside development, SRE, and platform teams (A19, A20).
- Monitor the observability system itself, write runbooks, and train engineers to read the data (A19, A20).

Postings commonly ask for five or more years in SRE, DevOps, or platform work (A19, A20). The boundary with neighboring
roles is soft. Search results treat observability as a duty inside SRE rather than a separate title (A22). A Honeycomb
podcast frames the role as speaking for the user across product, support, and engineering (A21) [single-source, vendor].
No authoritative source draws a hard line between the observability engineer and the SRE.

### An expert reviewer looks for twelve recurring problems

Across both angles, these are the concerns an observability expert raises against a code change, a design, or a plan.

1. Alerts on causes (CPU, disk) instead of symptoms users feel, or pages nobody can act on (A4).
2. Naive SLO alerting: one short window, a fixed delay before firing, or a single burn rate (A5).
3. No instrumentation for user-visible failures, the golden signals, or the SLO's own measurements (A4, A17).
4. Trace context or request IDs dropped at a service boundary, so logs, traces, and metrics cannot be joined (A10,
   A18).
5. Unbounded identifiers such as user IDs used as metric labels (A14, A15). This one is contested: the wide-events camp
   says those same fields belong on events (A3).
6. Wrong metric types, values baked into metric names, or heavy instrumentation on hot paths (A14) [single-source].
7. Sampling that throws away the interesting requests, or a tail sampler nobody monitors (A9).
8. Personal or regulated data in telemetry, including attributes that libraries add on their own (A13, A18).
9. Dashboards that need someone watching them, or that do not show the SLO measurements (A4, A17).
10. Alerts with no runbook link or routing, and alert paths nobody has tested (A18) [single-source on testing].
11. Telemetry cost blowups from unbounded labels, histograms, and duplicate metrics (A14, A15).
12. Treating "we have logs, metrics, and traces" as proof of observability (A23) [single-source].

Production readiness reviews at Google include several of these, such as whether every user-visible failure is
instrumented and alerted, and whether dashboards cover the SLO measurements (A17, A18).

### Practitioners disagree about metrics versus events

The main live disagreement is where detailed, high-variety data should live. Majors argues metrics handle it poorly and
that wide events should be the single source of truth (A2, A3). Prometheus keeps metrics at the center and bounds their
labels tightly (A14). The SRE books are metrics- and SLO-oriented (A4, A6). Sampling is the one point the camps share,
since both use it to control cost (A9, A3). An agent that picks a side will give advice wrong for half its users, so it
should name the trade-off rather than rule on it.

### Han already splits observability between two agents

The codebase shows observability is a sub-scope of two existing agents, with no single owner.

- **The DevOps reviewer holds the platform side.** Its description lists the Four Golden Signals and SLO discipline as
  one item among a dozen readiness concerns, and its observability sweep is a ten-bullet checklist (A27). That checklist
  names traces, structured logs, SLOs, burn-rate alerts, and OpenTelemetry, but it carries no guidance on head versus
  tail sampling, semantic conventions, pipeline processors, or the metrics-versus-events trade-off (A27).
- **The on-call reviewer holds the source-code side.** It checks that new code emits enough to debug, propagates
  correlation IDs, uses structured fields, and keeps personal data out of logs, metric labels, and trace attributes
  (A28). It explicitly hands alert rules, dashboards, and platform configuration to the DevOps reviewer (A28).
- **Skills route observability to the DevOps reviewer.** Code review, plan review, feature planning, change planning,
  and gap analysis all name the DevOps reviewer as the agent for observability (A29, A30). Code review notes that no
  checklist category substitutes for it on observability (A29).
- **Two skills dispatch neither agent.** The investigate skill and the runbook skill dispatch no agent that covers
  observability (A33).
- **The DevOps reviewer guards against over-building.** Its YAGNI section names dashboards, log fields, trace spans,
  SLOs, and error budgets as things to defer when telemetry is not yet flowing or traffic does not yet exist (A27).

Han has done this kind of carve-out before. The research behind the on-call reviewer found the DevOps reviewer already
covered much of the territory, and adversarial validation narrowed the new agent to the slice nobody owned (A31).

## Options to Consider

### O1: Deepen the DevOps reviewer instead of adding an agent

- **What it is:** Expand the DevOps reviewer's observability sweep with the missing practice areas (sampling, semantic
  conventions, the pipeline, cardinality and cost, alert tuning) and build no new agent.
- **Trade-offs:** This adds no new boundary and no dispatch-table changes, since every skill already routes
  observability there (A29, A30). The cost is depth: the DevOps reviewer already spans a dozen concern areas, and its
  description must fit a 1024-character budget (A27, A32). Observability would stay one checklist among many, and the
  investigate and runbook skills would still have nobody to ask (A33).
- **Rests on:** (A27), (A29), (A30), (A32), (A33)
- **Evidence status:** corroborated (codebase)

### O2: Build a focused telemetry-design reviewer

- **What it is:** A new read-only `observability-engineer` that owns telemetry design across both code and
  configuration: signal choice, instrumentation and naming, context propagation, cardinality, sampling, the pipeline,
  sensitive data in telemetry, SLI and SLO definition, alert design, dashboards, and telemetry cost. The DevOps reviewer
  keeps rollout, infrastructure, and overall readiness and points to the new agent for observability depth. The on-call
  reviewer keeps code-level resilience and its "how will I know when this breaks?" gate.
- **Trade-offs:** It gives the discipline's eight practice areas and twelve review concerns one owner (A27, A28). It
  requires boundary edits in both directions, because Han's guidance says every "use this other agent" pointer must be
  matched by one coming back (A32). That means changes to the DevOps reviewer, the on-call reviewer, and the dispatch
  tables in at least five skills (A29, A30). The on-call reviewer's personal-data and correlation-ID checks overlap the
  new agent's territory and need a clear split (A28). The new agent also needs a long-form doc, a plugin README line,
  and an agents-index entry in the same change (A34).
- **Rests on:** (A4), (A5), (A9), (A10), (A14), (A19), (A20), (A27), (A28), (A32), (A34)
- **Evidence status:** corroborated (practice areas and role duties by independent web sources; current coverage gaps by
  codebase)

### O3: Model the full job description

- **What it is:** An agent that mirrors the job as employers post it: platform ownership, tool selection, maturity
  assessment, root cause analysis, standards, and coaching.
- **Trade-offs:** It matches the role as hiring managers describe it (A19, A20). But Han agents read and report; they
  change nothing (A27, A28), so running the platform, picking vendors, and training people fall outside what an agent can
  do. The maturity model's levels could not be verified (A26), and the platform-ownership duties overlap the DevOps
  reviewer almost completely (A27).
- **Rests on:** (A19), (A20), (A26), (A27)
- **Evidence status:** corroborated for the duties; single-source (caveated) for the maturity model

## Recommendation

- **Recommendation:** O2. Build a focused telemetry-design reviewer, and move observability depth out of the DevOps
  reviewer so each agent has one clear territory. Give it the eight practice areas as its protocols and the twelve review
  concerns as its named anti-patterns. Carry the vendor-sourced ideas (wide events, cost levers, observability-driven
  development, maturity scoring) as vocabulary with their caveat, and have it name the metrics-versus-events trade-off
  rather than rule on it. Inherit the DevOps reviewer's YAGNI guard, so it does not ask a solo engineer for tail sampling
  or burn-rate alerts on a service with no traffic yet.
- **Evidence basis:** The practice areas rest on corroborated primary sources: SRE alerting and SLO practice (A4, A5,
  A6), OpenTelemetry sampling and propagation (A9, A10), and cardinality guidance (A14, A15). The role's duties rest on
  two corroborating sets of job postings (A19, A20). The gap in Han's current coverage rests on codebase evidence (A27,
  A28, A29, A30, A33), which is the trusted anchor for what Han does today. The bidirectional-boundary cost rests on
  codebase guidance (A32). The specific burn-rate thresholds (A5) and pipeline details (A11) are single-source and should
  appear as examples, not rules. The role's separateness from SRE is soft (A21, A22), so the case for a separate agent
  rests on the codebase gap, not on the job title.

## Validation

_Pending adversarial validation._

## Sources

| ID  | Source | Link / location | Retrieved | Trust class | Summary (one line) | Evidence status |
| --- | ------ | --------------- | --------- | ----------- | ------------------ | --------------- |
| A1 | OpenTelemetry, Observability primer | https://opentelemetry.io/docs/concepts/observability-primer/ | 2026-10-14 | web | Defines observability as asking a system new questions from outside; defines traces, metrics, logs, and instrumentation. | corroborated by A2, A12 |
| A2 | Majors, "Build observable systems" (Honeycomb, 2017) | https://honeycomb.io/blog/2017/06/build-observable-systems/ | 2026-10-14 | web (vendor) | Monitoring covers known problems, observability unknown ones; argues for wide per-request events, high cardinality, read-time aggregation, and sampling that keeps errors. | definition corroborated by A1; wide-events thesis contested by A14, A24 |
| A3 | Majors, "It's time to version observability" (Honeycomb) | https://www.honeycomb.io/blog/time-to-version-observability-signs-point-to-yes | 2026-10-14 | web (vendor) | Calls three pillars "1.0" and wide events "2.0"; says metrics handle high cardinality poorly; names sampling as the cost lever. | contested by A14, A24; same vendor camp as A2 |
| A4 | Google SRE book, "Monitoring Distributed Systems" | https://sre.google/sre-book/monitoring-distributed-systems/ | 2026-10-14 | web | Four Golden Signals; alert on symptoms; every page urgent, actionable, and new; averages hide the tail; dashboards answer basic questions. | corroborated by A5, A7, A17 |
| A5 | Google SRE workbook, "Alerting on SLOs" | https://sre.google/workbook/alerting-on-slos/ | 2026-10-14 | web | Recommends multiwindow, multi-burn-rate alerts; for 99.9% pages at 14.4x/1h and 6x/6h, tickets at 1x/3d; names weaknesses of simpler alerts and costs of this one. | approach corroborated by A4, A6; thresholds single source (caveated) |
| A6 | Google SRE book, "Service Level Objectives" | https://sre.google/sre-book/service-level-objectives/ | 2026-10-14 | web | Defines SLI, SLO, SLA, and error budget; pick few user-centric indicators; prefer percentiles; set targets from user needs. | corroborated by A4, A5 |
| A7 | Wilkie, "The RED Method" (Grafana) | https://grafana.com/blog/the-red-method-how-to-instrument-your-services/ | 2026-10-14 | web (vendor; method's author) | RED is rate, errors, duration per service; USE is utilization, saturation, errors per resource; use both together. | corroborated by A4, A8 |
| A8 | RED and USE search digest (FOSDEM 2018, Gregg slides) | https://archive.fosdem.org/2018/schedule/event/the_red_method/ ; https://www.brendangregg.com/Slides/EuroBSDcon2017_SystemMethodology.pdf | 2026-10-14 | web (search summaries) | Gregg created USE and Wilkie created RED; the two are complementary. | corroborates A7 |
| A9 | OpenTelemetry, "Sampling" | https://opentelemetry.io/docs/concepts/sampling/ | 2026-10-14 | web | Head sampling is cheap but can drop errors; tail sampling keeps errors and slow traces but is stateful and can degrade under load; combine them. | corroborated by A2, A3 |
| A10 | OpenTelemetry, "Context propagation" and "Signals" | https://opentelemetry.io/docs/concepts/context-propagation/ ; https://opentelemetry.io/docs/concepts/signals/ | 2026-10-14 | web | Trace context crosses services via W3C traceparent; logs can carry trace IDs; baggage must not hold secrets; profiles listed as emerging. | propagation corroborated by A18; baggage and profiles single source (caveated) |
| A11 | OpenTelemetry Collector | https://opentelemetry.io/docs/collector/ | 2026-10-14 | web | Vendor-neutral service that receives, processes, and exports telemetry; batches, retries, filters sensitive data; direct export fine for small setups. | single source (caveated) |
| A12 | OpenTelemetry, "Semantic conventions" | https://opentelemetry.io/docs/concepts/semantic-conventions/ | 2026-10-14 | web | Shared naming vocabulary that makes telemetry consistent and correlatable across code, libraries, and platforms. | single source (caveated); project's own spec |
| A13 | OpenTelemetry, "Handling sensitive data" | https://opentelemetry.io/docs/security/handling-sensitive-data/ | 2026-10-14 | web | Team owns what is sensitive; minimize collection; review library-emitted attributes; scrub in the Collector; hashing small ID spaces may be reversible. | corroborated by A18 |
| A14 | Prometheus, "Instrumentation practices" | https://prometheus.io/docs/practices/instrumentation/ | 2026-10-14 | web | Each label set is a new series; keep cardinality under ~10, rethink over 100; per-user labels, values in names, type misuse, and hot-path overhead are anti-patterns. | cardinality corroborated by A15; numeric thresholds and other anti-patterns single source (caveated) |
| A15 | Third-party Datadog cost guides (SigNoz, Finout) | https://signoz.io/guides/how-to-reduce-datadog-costs/ ; https://www.finout.io/blog/datadog-cost-optimization | 2026-10-14 | web (secondary, competing vendors; search summaries) | Custom metrics billed per unique tag combination, so a user_id tag multiplies cost; histograms and duplicates add cost. | cardinality corroborated by A14 |
| A16 | Observability cost vendor digest (Coralogix, Last9, OpenObserve) | https://coralogix.com/guides/observability-cost-optimization/ ; https://last9.io/guides/high-cardinality/cost-optimization-and-emergency-response-surviving-cardinality-spikes-without-breaking-the-budget/ | 2026-10-14 | web (vendors; search summaries) | Cost levers: cardinality control, dropping or hashing labels, sampling, retention tiers, trimming debug and duplicate logs. | agrees with A15; all vendors, weak |
| A17 | Google SRE book, "Evolving SRE Engagement Model" (PRR) | https://sre.google/sre-book/evolving-sre-engagement-model/ | 2026-10-14 | web | Production readiness review asks whether user-visible failures are instrumented and alerted and whether dashboards cover SLO metrics. | corroborated by A18 |
| A18 | Production readiness checklist roundup | https://getdx.com/blog/production-readiness-checklist/ (and search results page) | 2026-10-14 | web (secondary, mixed; search summaries) | Checklists include structured logs, request IDs end to end, tracing, SLOs, scrubbed sensitive data, routed alerts with runbook links, tested alert paths. | core items corroborated by A17; alert-path testing single source (caveated) |
| A19 | Observability Engineer posting, Transaction Network Services (Built In) | https://builtin.com/job/observability-engineer/6430381 | 2026-10-14 | web | Duties: monitoring/logging/tracing stack, instrumentation standards, dashboards and alerts, CI/CD, RCA, tooling, coaching; asks 5+ years SRE/DevOps and SLO design. | corroborated by A20 |
| A20 | Observability engineer postings digest | https://builtin.com/job/observability-engineer/3400943 ; https://jobs.lever.co/weekdayworks/ac6aae20-e79f-40e0-bd08-1b9ad4fedd5d ; https://www.dice.com/job-detail/664f6a54-4dd2-4276-a038-7f7e8660786e | 2026-10-14 | web (search summaries of postings) | Common duties: build the stack, set instrumentation standards, SLO dashboards, alert noise reduction, RCA, CI/CD, self-monitoring, runbooks, training. | corroborated by A19; individual postings not opened |
| A21 | O11ycast ep. 83, "Observability isn't just SRE on steroids" (Honeycomb) | https://www.honeycomb.io/resources/podcasts/ep-83-observability-isn-t-just-sre-on-steroids-with-dan-ravenstone | 2026-10-14 | web (vendor) | Frames the observability engineer as cross-functional and speaking for the user, beyond SRE. | single source (caveated) |
| A22 | SRE vs platform engineering search digest (Tigera and others) | https://www.tigera.io/learn/guides/devsecops/platform-engineering-vs-sre/ | 2026-10-14 | web (secondary; search summaries) | Observability appears as a duty inside SRE, not a separate title; responsibilities blur across SRE, DevOps, and infrastructure. | blurry boundary consistent with A21; otherwise single source (caveated) |
| A23 | Sridharan, "Distributed Systems Observability" (secondary reports) | https://www.oreilly.com/library/view/distributed-systemsobservability/9781492033431/ ; https://www.infoq.com/news/2019/02/rethinking-observability | 2026-10-14 | web (secondary) | Reported as arguing that having logs, metrics, and traces does not by itself make a system observable. | single source (caveated); primary text not read |
| A24 | "Observability 2.0" pushback (SimpliSafe talk on Grafana's site) | https://grafana.com/events/obsessions/2025/boston/are-we-ready-for-observability-2-0-pragmatic-observability-at-simplisafe/ | 2026-10-14 | web (secondary; search summary) | Calls wide events "more of a North Star than a silver bullet"; trade-offs against lightweight metrics. | single source (caveated) |
| A25 | Honeycomb, "A next step beyond test-driven development" | https://dev.to/honeycombio/a-next-step-beyond-test-driven-development-h05 | 2026-10-14 | web (vendor) | Observability-driven development: instrument while coding, ask how you will know it broke, check production after deploy. | agrees with a CNCF post seen in search only; no outcome evidence |
| A26 | Honeycomb observability maturity model webinar | https://www.honeycomb.io/resources/webinars/honeycomb-presents-a-new-framework-for-an-observability-maturity-model | 2026-10-14 | web (vendor) | Model scores five capabilities: incident resilience, code quality, release cadence, complexity and tech debt, user behavior understanding. | single source (caveated); levels not retrieved |
| A27 | `devops-engineer` agent definition | `han-core/agents/devops-engineer.md:4-10`, `:74-75`, `:124-129`, `:284-299` | n/a | codebase | Observability is one readiness concern; a ten-bullet sweep covers golden signals, SLOs, burn-rate, traces, logs, OTel; no sampling design, semantic conventions, or pipeline processors; YAGNI defers dashboards and SLOs without traffic. | codebase anchor |
| A28 | `on-call-engineer` agent definition | `han-core/agents/on-call-engineer.md:22-26`, `:118-122`, `:347-366` | n/a | codebase | Audits observability only as expressed in application source (correlation IDs, structured fields, PII); hands alert rules, dashboards, and platform config to devops-engineer. | codebase anchor |
| A29 | Code review agent dispatch table | `han-coding/skills/code-review/references/agent-dispatch.md:32-33`, `:261` | n/a | codebase | Routes observability config to devops-engineer; notes no checklist category substitutes for it on observability. | codebase anchor |
| A30 | Planning and gap-analysis dispatch tables | `han-planning/skills/iterative-plan-review/SKILL.md:267-268`; `han-planning/skills/plan-a-feature/references/review-team-briefs.md:22-23`; `han-planning/skills/plan-a-change/references/team-selection.md:64`; `han-research/skills/gap-analysis/SKILL.md:177-181` | n/a | codebase | Each names devops-engineer as the agent for observability. | codebase anchor |
| A31 | Prior on-call-engineer research | `docs/research/on-call-engineer-research.md:11`, `:119-124` | n/a | codebase | Found devops-engineer already covered observability and SLOs; validation narrowed the new agent to the unowned slice. | codebase anchor |
| A32 | Agent description guidance | `han-plugin-builder/skills/guidance/references/agent-building-guidelines/agent-description-length.md:33`, `:75-78`, `:87` | n/a | codebase | Descriptions fit 1024 characters; "Does not X — use Y" names the sibling; boundaries must be bidirectional. | codebase anchor |
| A33 | Investigate and runbook skills | `han-coding/skills/investigate/SKILL.md`; `han-documentation/skills/runbook/SKILL.md` | n/a | codebase | Neither dispatches devops-engineer or on-call-engineer; runbook dispatches no agent. | codebase anchor |
| A34 | Long-form doc coverage rule and index convention | `docs/templates/coverage-rule.md:1-12`; `CLAUDE.md` ("Indexes stay complete") | n/a | codebase | Every new agent needs a long-form doc, a plugin README scent line, and an agents-index entry in the same PR. | codebase anchor |

### A5: Google SRE workbook, "Alerting on SLOs" — recommendation-bearing

- **Link / location:** https://sre.google/workbook/alerting-on-slos/
- **Retrieved:** 2026-10-14
- **Trust class:** web (outside the trust boundary; primary practitioner source)
- **Summary:** Compares six ways to alert on an SLO and recommends multiwindow, multi-burn-rate alerting for most
  services. Burn rate is how fast the error budget is being spent relative to the target. For a 99.9% SLO the example
  pages at 14.4x over one hour (with a five-minute short window) and 6x over six hours (thirty-minute short window), and
  opens a ticket at 1x over three days. It names the costs: many parameters, poor fit for low-traffic services, and
  trouble with very high targets.
- **Evidence status:** approach corroborated by A4 and A6; specific thresholds single source (caveated)

### A9: OpenTelemetry, "Sampling" — recommendation-bearing

- **Link / location:** https://opentelemetry.io/docs/concepts/sampling/
- **Retrieved:** 2026-10-14
- **Trust class:** web (outside the trust boundary; project's own docs)
- **Summary:** Head sampling decides early, for example by trace-ID probability, and cannot guarantee keeping error
  traces. Tail sampling decides after seeing the whole trace, so it can keep errors and slow requests, but it holds state,
  uses more resources, is harder to maintain, and can degrade under load. The two are often combined. The page also lists
  cases where sampling is not appropriate, such as low volume and regulatory limits.
- **Evidence status:** corroborated by A2 and A3

### A14: Prometheus, "Instrumentation practices" — recommendation-bearing

- **Link / location:** https://prometheus.io/docs/practices/instrumentation/
- **Retrieved:** 2026-10-14
- **Trust class:** web (outside the trust boundary; project's own docs)
- **Summary:** Every distinct set of label values creates a new time series. Keep label cardinality under about ten where
  possible and look for alternatives above a hundred. Per-user labels, values encoded in metric names, misused counter
  and gauge types, and heavy instrumentation in hot paths are named anti-patterns.
- **Evidence status:** cardinality cost corroborated by A15; numeric thresholds and the other anti-patterns single source
  (caveated)

### A19: Observability Engineer posting, Transaction Network Services — recommendation-bearing

- **Link / location:** https://builtin.com/job/observability-engineer/6430381
- **Retrieved:** 2026-10-14
- **Trust class:** web (outside the trust boundary; employer's own description, listing since removed)
- **Summary:** Lists leading the design and rollout of monitoring, logging, and tracing; setting and enforcing
  instrumentation standards; dashboards and alerts; CI/CD integration; leading root cause analysis and post-incident
  reviews to find observability gaps; evaluating tools; and helping teams interpret data. Requires five or more years in
  SRE, DevOps, or observability and SLO/SLI design experience.
- **Evidence status:** corroborated by A20

### A27: `devops-engineer` agent definition — recommendation-bearing

- **Link / location:** `han-core/agents/devops-engineer.md:4-10`, `:74-75`, `:124-129`, `:284-299`
- **Retrieved:** n/a
- **Trust class:** codebase (trusted current-state anchor)
- **Summary:** The description lists the Four Golden Signals and SLO discipline among a dozen readiness concerns. Protocol
  6, the observability sweep, is ten bullets: latency, traffic, errors, saturation, SLIs and SLOs with multi-window
  burn-rate alerts, traces with correlation IDs, structured logs without personal data, OpenTelemetry with vendor SDKs
  isolated at the collector, and business metrics. A search of the file finds no guidance on head versus tail sampling,
  semantic conventions, pipeline processors, or the metrics-versus-events trade-off. Its YAGNI section defers dashboards,
  log fields, trace spans, SLOs, and error budgets when telemetry is not flowing or traffic does not exist.
- **Evidence status:** codebase anchor

### A28: `on-call-engineer` agent definition — recommendation-bearing

- **Link / location:** `han-core/agents/on-call-engineer.md:22-26`, `:118-122`, `:347-366`
- **Retrieved:** n/a
- **Trust class:** codebase (trusted current-state anchor)
- **Summary:** Reviews application source only. Its observability protocol checks whether a change emits enough to be
  debugged, propagates correlation IDs, uses structured fields, and keeps personal data, tokens, and credentials out of
  log statements, metric labels, and trace attributes. It states that the observability platform, alert rules, and
  dashboard configuration belong to `devops-engineer`.
- **Evidence status:** codebase anchor
