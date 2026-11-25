# webhook_delivery
A service that accepts events from a client via API and reliably delivers them to registered customer URLs (like Stripe or GitHub webhooks).

Webhook Delivery Service: 8-Week Plan
The project in one page

What it is: A service that accepts events from a client via API and reliably delivers them to registered customer URLs (like Stripe or GitHub webhooks).

Guarantees (your contract):

Every accepted event is delivered at least once, even if workers or the database crash.
A slow or dead customer endpoint never slows down the API or other customers.
Failed deliveries retry with exponential backoff plus jitter, then land in a dead-letter queue.
Consumers can dedupe using an event ID, since at-least-once means duplicates are possible.

Out of scope: a UI, billing, multi-region, exactly-once delivery (you'll write a section explaining why that's mostly a myth).

Stack: FastAPI, Postgres, Redis (cache, rate limiting, Redis Streams for the queue), your own worker processes, Docker Compose, pytest, and Locust or k6 for load tests.

Core endpoints:

POST /endpoints registers a destination URL and a signing secret
POST /events ingests an event (requires an Idempotency-Key header)
GET /events/{id} shows delivery status and attempt history
POST /events/{id}/replay re-queues a dead-lettered event
Weekly plan

Each week has build, experiment, read, and done when items. Keep a DECISIONS.md where every major choice gets three lines: chosen, rejected, why.

Week 1: Baseline API and database
Build: FastAPI app, Postgres schema (endpoints, events, delivery_attempts), migrations, Docker Compose. Deliver events synchronously inside the request on purpose.
Experiment: Point an endpoint at a deliberately slow test server (sleep 5s). Watch your API stall. That's your motivating problem.
Read: DDIA ch. 2-3 (data models, storage and indexing).
Done when: the sync version works end to end and you can explain why it fails under a slow customer.
Week 2: Load testing and database tuning
Build: Locust/k6 script for POST /events. Add indexes, connection pooling (e.g. PgBouncer or SQLAlchemy pool tuning).
Experiment: Record a baseline (p50/p95/p99, RPS, error rate). Change one thing, rerun, record. Always run the load generator on a separate machine or at least note when it shares the box.
Read: your database's EXPLAIN ANALYZE docs, plus the pool-size section of the PgBouncer docs.
Done when: you have a results table in the README with before/after numbers you produced.
Week 3: Caching and rate limiting
Build: Redis cache for endpoint lookups (cache-aside), plus a per-API-key rate limiter (token bucket or sliding window).
Experiment: Compare latency with and without the cache. Then cause a cache stampede (expire a hot key under load) and fix it (request coalescing or jittered TTLs).
Read: DDIA ch. 5 (replication, for the staleness concepts), the Redis docs on expiry.
Done when: you can say what the cache costs you (staleness, invalidation) and when you'd remove it.
Week 4: Async queue and workers (the core)
Build: POST /events now writes the event to Postgres and enqueues to a Redis Stream, then returns 202. Worker processes use consumer groups to read, deliver via HTTP with timeouts, and ack on success.
Experiment: Rerun your week 1 slow-endpoint scenario. API latency should be unaffected.
Read: DDIA ch. 11 sections on message brokers, plus the Redis Streams docs (consumer groups, pending entries list).
Done when: slow customers no longer affect API latency and you can explain what "acked" means and what happens if a worker dies before acking.
Week 5: Reliability semantics
Build: retries with exponential backoff and jitter, max attempts, dead-letter handling, idempotency keys on ingest, an event ID header on delivery, HMAC signatures so customers can verify authenticity, and reclaiming stuck messages from crashed workers.
Experiment: Compare Postgres-as-a-queue (SELECT ... FOR UPDATE SKIP LOCKED) with Redis Streams for the same workload. Write up the tradeoffs.
Read: DDIA ch. 7 (transactions) and the "outbox pattern" (look for a good write-up). Think about the gap between "committed to Postgres" and "enqueued to Redis": what if the process dies between them?
Done when: you can describe the dual-write problem and how you solved it, either with an outbox table or by making Postgres the source of truth.
Week 6: Break it on purpose
Build: automated failure tests (pytest plus Docker commands).
Experiment: under load, kill workers, kill Redis, kill Postgres, return 500s and timeouts from customer endpoints, send duplicate ingest requests. For each: what happened, was anything lost, what did you fix.
Read: DDIA ch. 8 (trouble with distributed systems).
Done when: docs/postmortems/ has one short entry per real failure you found (impact, root cause, fix, how you verified it).
Week 7: Security, observability, and AWS deploy
Build: API key auth (hashed in DB), per-key authorization, input validation, SSRF protection (block private IP ranges in destination URLs; this is a real concern for webhook services), secrets via environment/secret manager, dependency and container scans in CI. Add Prometheus metrics, a Grafana dashboard, and one alert (e.g. DLQ growth or delivery failure rate).
Deploy: a simple AWS version (e.g. ECS/Fargate or one EC2 instance, with RDS and ElastiCache or just containers). Set a budget alert first, and check current free-tier terms. Skip this part if time is tight; local Compose plus a cheap VPS is fine.
Write: a half-page threat model (who attacks what, and what you did about it).
Done when: you can list your top three threats and the mitigation for each.

Week 8: Polish and interview prep
Build: README with architecture diagram, guarantees, measured results table, and a "what I'd do differently / what breaks at 100x" section.
Prep: record yourself explaining the design in 5 minutes, then in 20. Practice the follow-ups below.
Done when: a stranger can run it with one command and you can defend every box in the diagram.
Weekly habits (all 8 weeks)
One system design problem out loud per week (URL shortener, rate limiter, notification service, news feed, chat). Ask me to play interviewer and push back.
Decisions log updated after every experiment.
Rule: don't commit code you can't explain line by line.
Realistic time: 10-15 hours a week. If you're short, cut week 7's AWS deploy first, then week 3's rate limiter, but never weeks 4-6.
Interview questions this project should prepare you for
What happens if a worker crashes after delivering but before acking?
How do you avoid losing an event between the database commit and the enqueue?
Why at-least-once and not exactly-once? How do customers cope with duplicates?
How would you handle one customer with a huge backlog starving everyone else?
What breaks first at 100x traffic, and what do you change?
Why Redis Streams over Kafka or SQS here? When would you switch?
How do you stop someone registering http://169.254.169.254/ as a webhook URL?
Resume bullets (fill in only after you've measured)
Built a webhook delivery service with at-least-once guarantees (Python, FastAPI, Postgres, Redis Streams), cutting p99 ingest latency from [X] ms to [Y] ms at [N] RPS by moving delivery off the request path.
Verified no event loss across [N] injected failures (worker, Redis, and database crashes) using automated chaos tests, documenting [N] fixed failure modes in public postmortems.
Hardened the service with HMAC-signed deliveries, SSRF protection, and CI vulnerability scanning; instrumented with Prometheus/Grafana alerting.
