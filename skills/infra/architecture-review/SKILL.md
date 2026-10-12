---
name: architecture-review
description: Reviewing system designs, ADRs, and architecture proposals — judging them against quality-attribute scenarios, one-way-door decisions, failure modes, and boundary soundness rather than code taste.
---

# Architecture Review

Architecture review is a discipline separate from code review. Code review
asks whether a change keeps the codebase healthy; architecture review asks
whether the *structure* will carry the stated requirements, what it costs
later, and how it fails. The verdict shapes months of work, so it earns more
rigour than taste. Apply it to design docs, ADRs, RFCs, diagrams, and running
estates — at the design gate before building, for single significant
decisions, and as periodic health checks.

## Review against scenarios, not adjectives

No design is good in the abstract; it is fit *for something*. Force the "for
what" into the open first: name the top 3–5 quality attributes (performance,
scalability, availability, reliability, security, modifiability, operability,
cost) and state each as a concrete, numbered scenario — e.g. "a tenant
uploads a 50k-row file; import finishes within 5 minutes while UI latency
stays under 500 ms p95". Adjectives like "scalable" with no numbers are not
requirements. If the author cannot rank the attributes, running that ranking
session is the review's most valuable output.

## Make the trade-offs explicit

Every structural choice buys some attributes with others: distribution buys
deployability but pays latency and partial failure; strong consistency buys
correct invariants but pays availability under partition; caching buys
latency but pays staleness; buying SaaS buys speed but pays lock-in. A
proposal with one option, no rejected alternatives, and no stated costs is
not reviewable — send it back. Watch for one parameter that dominates an
attribute (queue depth, cache TTL, pool size): those are where incidents and
tuning will happen. Where one decision pulls two attributes in opposite
directions (replication factor: durability up, write latency up), that is the
point deserving the most explicit sign-off.

## Spend scrutiny on one-way doors

Judge reversibility before depth. Data models, wire formats, public API
contracts, ID schemes, auth boundaries, build-vs-buy choices, and module
topology get the deep look; reversible details get a light touch and a
"two-way door" label. Capture decisions in ADRs (Architecture Decision
Records): context, alternatives honestly weighed, the decision, and the
consequences owned — including which door it is.

## Judge the seams, not the boxes

Coupling, cohesion, and data ownership at the boundaries carry the design.
Ask: who owns each piece of data, how many seams does one user journey cross,
and do the seams align with team ownership (the org chart will win). A
distributed system with entangled boundaries is a distributed monolith — all
of the operational cost, none of the autonomy. Reject speculative complexity
(multi-service topologies, event sourcing, multi-region) for imagined scale:
every increment of complexity must buy a named, ranked quality attribute.

## Walk the failure modes

The happy path is the least interesting thing about a design. For every
dependency, ask what happens when it is down, slow, wrong, or duplicated —
especially *slow*, which piles up threads and connections until the caller
dies too. For every component, ask what state it holds and what is lost if
it dies right now. Look for the classic amplifiers: retries without backoff
and budgets, timeouts longer than the caller's own budget, unbounded queues,
no back-pressure, synchronised fleet-wide behaviour. Demand designed
degraded modes (cached reads, feature flags, read-only mode) rather than
accidental ones. Before a large build, run a pre-mortem: "it is a year later
and this failed — write the incident summary" — the fears nobody volunteers
in critique mode surface fastest that way.

## Deliver a weighted verdict

Findings are risks phrased as scenarios — "if X happens, Y fails, because
Z" — locatable, arguable, testable. Label each blocking vs advisory, and
name the non-risks too (what you checked and found sound — it saves the
next review). The verdict is approve / approve-with-conditions / revise;
record the decisions as ADRs and turn load-bearing constraints into automated
fitness functions so the review outlives the meeting. For an existing
estate, verify claims against the repository, dependency graph, and
telemetry — review what runs, not what is drawn.

## Input → Process → Output

**INPUT:** Proposal: "Move the notification fan-out from a synchronous loop
in the request handler to an async job queue (Redis Streams)."

**PROCESS:**
1. Quality-attribute scenarios: "100k events/min must deliver within 30s
   p99" (scalability), "a queue outage must not lose events" (reliability),
   "duplicate deliveries are acceptable, lost ones are not" (at-least-once).
2. One-way vs two-way door: adding the queue is one-way (data format and
   consumer contracts lock in) — spend scrutiny here; the consumer code
   itself is two-way and can iterate.
3. Failure-mode walk: Redis down → handlers block on publish; poison message
   → consumer crash loop; clock skew → delayed retries fire early.
4. Trade-off record: gain throughput and isolation; pay for a new stateful
   dependency, duplicate-delivery handling, and dead-letter ops.
5. Verdict: approve-with-conditions — dead-letter queue, idempotency keys
   on consumers, Redis persistence + replica.

**OUTPUT:** An ADR capturing the decision, its scenarios, the conditions,
and the failure-mode mitigations — plus a fitness function (queue-depth
alert, consumer-lag dashboard) so the review keeps working after the
meeting.

## Proven pattern vs avoid pattern

- **Proven — numbered quality-attribute scenarios:** "deliver 100k
  events/min within 30s p99" is reviewable; "the system should be scalable"
  is not. Scenarios turn vague requirements into pass/fail checks that
  survive into monitoring.
- **Avoid — reviewing the diagram instead of the runtime:** approving an
  architecture from boxes and arrows without checking the repository,
  dependency graph, and production telemetry. The review must cover what
  runs, not what is drawn — stale diagrams are where one-way doors hide.

## Telemetry, testing & debugging

- **Instrument:** turn each scenario into a metric — queue depth, consumer
  lag, delivery latency p99, dead-letter rate. Fitness functions alert when
  the architecture's promises break, not just when the service is down.
- **Test:** load-test the queue path at the scenario rate before launch;
  chaos-test the failure modes (kill Redis, inject a poison message) in a
  staging environment.
- **Debug:** when delivery latency spikes, trace one event end to end
  (publish → stream → consumer → ack) to find which hop owns the delay;
  check consumer-lag first, code second.
