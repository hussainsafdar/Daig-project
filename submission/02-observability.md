# Phase 2 — Observability

## Task 1 & 2: Cross-service trace

make obs and make load were run. In Grafana Explore (Tempo datasource), filtering by duration > 200ms surfaced trace ID 133591ffd57cbeb0beaa1ef436016b:

orders: POST /api/orders
Total duration: 410.81ms
Status: 201 (order placed successfully)
50 spans total

Waterfall breakdown:
- orders (top level) - majority of the 410.81ms is spent here in sequential Postgres queries
- kitchen POST (child span) - 16.27ms
- dispatch POST (grandchild span) - 8.13ms

The service that owns most of the total duration is orders itself, not kitchen or dispatch. This matches the architecture: orders commits its own database transaction before calling kitchen, so an order survives a downstream failure.

## Task 3: SLOs

daig defines two SLOs: 99.9% availability of order requests over a 30-day window (about 43 minutes downtime per month), and p95 of POST /api/orders under 1 second. Under make load (4335 orders, 100x baseline), measured p95 was 617-709ms, well inside budget, with 0% error rate against a 0.10% SLO allowance.

## Task 4 & 5: Chaos latency injection

Ran ./chaos/day4-latency.sh break with CHAOS_SLOW_DISPATCH and CHAOS_HOT_SURGE_LOOP true. Under PROFILE=spike load, p95 hit 5107-10409ms against the 1s SLO, error rate up to 97.74% against 0.10% allowed.

Diagnosis trail:
1. Metric: p95 far exceeds the 1s SLO under spike load.
2. Trace: the orders trace showed most time inside the kitchen span, unlike the baseline trace where orders dominated.
3. Logs: kitchen logs showed open_orders around 6880 on every decision, with 1-2 second gaps between decisions under load.
4. Culprit function by code inspection: computeSurgeScore() in services/kitchen/src/server.js, gated behind CHAOS_HOT_SURGE_LOOP.

Gap found: Pyroscope profiling could not be used for this diagnosis. docker exec daig-kitchen-1 env grep pyroscope returned nothing, grep for pyroscope across services/ returned nothing, and the Pyroscope UI application list only shows pyroscope itself, not kitchen, orders, or dispatch. The Pyroscope container runs and the compose overlay wires it up, but no service was ever instrumented with a Pyroscope client, and PYROSCOPE_SERVER_ADDRESS is missing from the x-app-env anchor in docker-compose.yml. The README's own path (profile shows computeSurgeScore dominating kitchen CPU) cannot be reproduced as shipped. Profiling is the fourth observability pillar the architecture advertises but never actually connects.

Reverted with ./chaos/day4-latency.sh fix, then re-exported Vault credentials for kitchen and dispatch, since the chaos script's plain docker compose up --no-deps drops OpenBao-sourced credentials - the same env-file precedence issue flagged in VERIFICATION.md.
