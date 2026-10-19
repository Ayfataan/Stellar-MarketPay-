# Runbook: Redis Outage

**Severity:** SEV-2 when cache/queues are impaired; raise to SEV-1 if rate-limit degradation or queue loss creates a security or customer-impacting incident
**Owner:** Backend on-call / platform operator
**Related:** [PostgreSQL outage runbook](./postgres-outage.md) · [Environment variables](../environment-variables.md) · [Redis session/cache ADR](../adr/adr-008-redis-session-cache.md)

## 1. Detection & Alerts

Redis backs the application cache, Bull queues, and some rate-limit stores. `GET /health` runs a Redis `PING` with a 2-second timeout and returns HTTP 503 with `status: "degraded"` when Redis is down. Capture the `redis` field and compare database, Horizon, and Soroban checks to isolate the dependency.

There is currently **no Redis-specific Prometheus alert or Redis metric exporter configured** in this repository. Do not wait for a Redis alert. The configured general alerts `HighApiErrorRate` (>5% 5xx for 10m), `HighRequestLatencyMs` (p99 >2,000 ms for 10m), and `BackendMetricsEndpointDown` can signal impact but do not identify Redis as the cause. The backend's metrics endpoint is `/metrics`; it does not export Redis availability/queue depth gauges.

Expected log/error fingerprints include:

- `[cache] Redis error: <message>` and `[cache] Failed to create Redis client: <message>` from the cache wrapper.
- ioredis connection errors such as `ECONNREFUSED`, `ETIMEDOUT`, DNS/TLS/authentication failures, or repeated reconnects. Queue libraries may report Redis errors through worker/application logs; no Bull error listener or dead-letter queue is configured centrally.
- Cache reads return a miss and cache writes/deletes silently fall through on errors. Rate-limit stores fall back to a per-process in-memory `Map`. These fail-open behaviors can make the API appear partly healthy while caching is lost and rate limiting is no longer globally consistent.

The backend reads `REDIS_URL` (cache and queues); the queue also supports `REDIS_HOST`/`REDIS_PORT` when no URL is supplied, defaulting to `127.0.0.1:6379`. The Redis rate-limit store only creates a Redis client when `REDIS_URL` is set. Confirm every backend/worker uses the same intended endpoint, TLS/auth options, and logical database. Do not paste a credential-bearing URL into logs or chat.

Run these checks from a trusted operator environment, using the deployment's Redis CLI authentication method and the same endpoint as the application:

```bash
curl -fsS -i "$API_URL/health"
redis-cli PING
redis-cli INFO server
redis-cli INFO replication
redis-cli INFO persistence
redis-cli CLIENT LIST
```

For a remote/authenticated cluster, supply host/port/TLS and credentials from the secret manager rather than adding a password to command arguments. Inspect managed-service failover, memory/evictions, connection limits, persistence health, network/DNS, and recent configuration changes. Use the `Bull Board` UI only if it is enabled and access-controlled in the deployment; it is not proof that Redis itself is healthy.

## 2. Immediate Mitigations

1. Declare a degraded window, identify impacted functions (cache, async jobs, shared rate limits), and publish an update. Do not infer that escrow state in PostgreSQL or on Stellar is lost because Redis is unavailable.
2. Restore connectivity or fail over Redis using the managed service's approved procedure. Check primary/replica role, replication lag, persistence, memory pressure, and client limits before restarting or promoting nodes.
3. Reduce nonessential queue producers and expensive cache-dependent traffic at the ingress. If a queue is stalled, pause its producer/worker through the deployment control plane and inspect its state before resuming; do not blindly delete queue keys or restart every worker.
4. Treat rate limits as degraded: the Redis rate-limit fallback is per backend process, so replica count changes and process restarts produce inconsistent/reset counters. Temporarily enforce conservative limits at the gateway/WAF for sensitive routes, and monitor abuse during the gap. Do not raise application limits to mask Redis errors.
5. Never use `FLUSHALL` or `FLUSHDB` on the shared Redis instance. Queue state and rate-limit keys may share the same Redis database as cache keys.

## 3. Recovery & Restore Procedures

### Restore Redis service

1. Confirm the configured endpoint/TLS/auth secret and network route. Test `PING` and `INFO replication`/`INFO persistence` from the backend network. Resolve managed failover, disk, memory, or ACL issues before changing application configuration.
2. If Redis data is corrupt, use the provider's supported snapshot/AOF/RDB recovery workflow and restore to an isolated endpoint first. Do not run repair utilities against a live data directory. Verify the candidate endpoint, then update `REDIS_URL` (or the queue's `REDIS_HOST`/`REDIS_PORT`) using secret management and roll backend/worker instances gradually.
3. Confirm application clients reconnect: `GET /health` must report Redis healthy; logs must stop emitting Redis connection errors. If health is good but clients continue to fail, perform a controlled backend/worker restart after verifying connection configuration.

### Inspect Bull queues before resuming workers

The queues created in [`backend/src/utils/queue.js`](../../backend/src/utils/queue.js) are `emailQueue`, `auditQueue`, and `linkVerificationQueue`. The link-verification queue uses 3 attempts with exponential backoff starting at 5 seconds and retains up to 5,000 failed jobs; Bull does not create a separate dead-letter queue here. Inspect counts without printing job payloads or personal data:

```bash
cd backend
node <<'NODE'
const { emailQueue, auditQueue, linkVerificationQueue } = require("./src/utils/queue");
const queues = { emailQueue, auditQueue, linkVerificationQueue };
(async () => {
  try {
    for (const [name, queue] of Object.entries(queues)) {
      console.log(name, await queue.getJobCounts("waiting", "active", "delayed", "failed", "completed"));
    }
  } finally {
    await Promise.all(Object.values(queues).map((queue) => queue.close()));
  }
})().catch((error) => { console.error(error.message); process.exitCode = 1; });
NODE
```

After confirming Redis is stable, resume workers gradually and watch waiting/active/delayed/failed counts. Review failed-job IDs and sanitized error reasons with the owning team before retrying or cleaning them. There is no built-in dead-letter routing in this configuration; do not `clean()` failed jobs or delete `bull:*` keys until the jobs are accounted for and an owner approves removal.

### Clear only corrupted application cache entries

Cache invalidation is usually unnecessary after Redis recovery because cache writes have TTLs and the wrapper fails open. If corruption is confirmed, preserve queue and rate-limit data and delete only the affected cache namespace. The application cache uses namespaced keys such as `jobs:list:*`, `job:<id>`, `profile:<id>`, `escrow:<id>`, and other feature-specific prefixes. From an authenticated operator shell, scan first, then remove only the approved pattern (never use `KEYS` on production):

```bash
redis-cli --scan --pattern 'jobs:list:*'   # review matches first
redis-cli --scan --pattern 'jobs:list:*' | xargs -r -n 100 redis-cli UNLINK
```

Repeat only for a specifically identified cache prefix. Adapt Redis CLI connection/auth options to the deployment for both commands without exposing credentials. Do not sweep broad patterns such as `*`, `bull:*`, or rate-limit prefixes. To invalidate through the application cache wrapper instead, use its `delPattern()`/`invalidateJobListCache()` functions from a controlled backend maintenance context.

## 4. Verification & Post-Recovery Reconciliation

1. Confirm Redis `PING` succeeds from an application-network host, replication is healthy, persistence has no current error, memory/evictions are within policy, and the Redis service is not rejecting clients.
2. Confirm `GET /health` returns 200 and the `redis` check is healthy across backend replicas. Confirm `[cache] Redis error:` and connection retry logs stop. Test a cacheable read twice and verify normal response behavior without relying on cache contents for correctness.
3. Verify the queue counts are moving as expected. Check that no active jobs are stuck, delayed retries progress, failed-job retention is understood, and queue producers/workers return to their normal replica counts. Ensure no queue keys were removed during recovery.
4. Confirm shared rate limiting is back on Redis by verifying the configured Redis rate-limit client is `ready` in the running application/diagnostics and by exercising an approved low-risk test route. Because the fallback `Map` is process-local and the rate-limit window is 60 seconds by default, allow at least one full window after Redis reconnect and account for counters created during the outage; retain gateway/WAF protections until all replicas have rejoined Redis.
5. Redis recovery does not itself reconcile PostgreSQL escrow mirrors. The backend indexer is bootstrapped independently and should continue/reconnect based on its database checkpoint. If database restore or a missed Stellar event is also suspected, follow the [PostgreSQL outage runbook](./postgres-outage.md), including its warning that historical contract-event replay is not currently exposed as a supported command. Compare mirrored escrow state to the live configured Stellar ledger before restoring escrow-sensitive features.
6. Remove temporary gateway controls gradually, close the degraded window after queue/rate-limit/cache checks pass, and record outage duration, queue backlog/failures, cache invalidations, and rate-limit exposure.

## Related documentation

- [PostgreSQL outage runbook](./postgres-outage.md)
- [Redis session/cache ADR](../adr/adr-008-redis-session-cache.md)
- [Environment variables](../environment-variables.md)
- [Database backup and restore guide](../database-backup-restore.md)
