# Runbook: PostgreSQL Outage

**Severity:** SEV-1 when writes or escrow state are unavailable; otherwise SEV-2
**Owner:** Backend on-call / database operator
**Related:** [Redis outage runbook](./redis-outage.md) · [Database backup and restore](../database-backup-restore.md) · [Environment variables](../environment-variables.md)

## 1. Detection & Alerts

Treat a failed readiness check, sustained pool waiters, or database connection errors as an incident. The server mounts the health router at `GET /health`; it checks PostgreSQL with `SELECT 1` and a 2-second timeout. The database pool stats endpoint is `GET /health/db`. A failed dependency returns HTTP 503 and `status: "degraded"`.

Prometheus scrapes the backend at `/metrics` every 15 seconds. The database rules in [`monitoring/prometheus/rules/alerts.yml`](../../monitoring/prometheus/rules/alerts.yml) are:

| Alert                        | Exact condition                                                      | Meaning                                                                                                                                                                            |
| ---------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DBPoolExhausted`            | `pg_pool_waiting_connections{job="marketpay-backend"} > 10` for 30s  | More than ten requests wait for a pool connection.                                                                                                                                 |
| `DBPoolUtilizationHigh`      | `(pg_pool_total - pg_pool_idle) / clamp_min(pg_pool_total, 1) > 0.8` | More than 80% of pool connections are in use; no `for` duration is configured.                                                                                                     |
| `SlowDatabaseQueries`        | p95 of `pool_query_duration_ms` > 500 ms for 10m                     | Queries through the shared pool are slow. Histogram labels are `operation` and `status`.                                                                                           |
| `HighDatabaseConnections`    | `marketpay_db_connections{state="total"} > 80` for 10m               | Legacy/cluster-wide threshold; the backend pool defaults to only 10 connections, so confirm the deployed pool size and what is being scraped before treating 80 as the pool limit. |
| `BackendMetricsEndpointDown` | `up{job="marketpay-backend"} == 0` for 5m                            | Scrape failure; also check network and metrics authentication.                                                                                                                     |
| `HighApiErrorRate`           | 5xx ratio > 5% for 10m                                               | Correlate with DB logs and health checks; this is not DB-specific.                                                                                                                 |

Other exported PostgreSQL metrics include `pool_queries_total{operation,status}`, `marketpay_db_connections{state="total|idle|waiting"}`, `pg_pool_total`, `pg_pool_idle`, `pg_pool_waiting`, and `pg_pool_waiting_connections`. The pool defaults are `DATABASE_POOL_MIN=2`, `DATABASE_POOL_MAX=10` (`DATABASE_POOL_SIZE` is a legacy alias), `DATABASE_POOL_CONNECTION_TIMEOUT_MS=5000`, and `DATABASE_POOL_IDLE_TIMEOUT_MS=30000`.

The tested `connectWithRetry()` helper attempts `pool.connect()` up to 10 times: delays after failed attempts are 1, 2, 4, 8, 16, then 30 seconds (30 seconds capped) before the final attempt. The matching test is [`pool.retry.test.js`](../../backend/src/db/pool.retry.test.js). Logs include `Database connection attempt N/10 failed — retrying in ...ms`, `Database connection failed — all retry attempts exhausted, shutting down`, and `[pg] Unexpected pool error:`. **The current backend startup does not call this helper**; do not assume that this backoff is active in production or provides a runtime reconnect guarantee.

First checks:

```bash
curl -fsS -i "$API_URL/health"
curl -fsS "$API_URL/health/db"
psql "$DATABASE_URL" -c 'SELECT now(), pg_is_in_recovery();'
```

Use the deployment's secret injection for `DATABASE_URL`; do not paste credentials into incident chat or command history. Check PostgreSQL service health, storage, max connections, failover status, network/DNS, and recent deploy/config changes. `GET /health/db` reports pool statistics; if that route is not enabled in the deployment, use `/metrics` and database-side `pg_stat_activity`.

## 2. Immediate Mitigations

1. Declare a degraded platform window and publish affected functionality, start time, and next update. Do not report an on-chain escrow transaction as failed solely because its database mirror could not be updated.
2. Stop or reduce nonessential write producers and background consumers at the deployment control plane. Temporarily shed expensive reads and writes at the ingress/load balancer; preserve health and incident-response endpoints. Avoid retry storms from clients.
3. If the database is reachable but saturated, identify long-running or idle-in-transaction sessions before terminating anything. Scale API replicas down only if aggregate pool capacity is overwhelming PostgreSQL; increasing `DATABASE_POOL_MAX` without checking server capacity can worsen the outage.
4. Do not delete database volumes, promote a replica without the database operator, run migrations against an unstable primary, or rewind `indexer_state` as a first response.
5. Preserve PostgreSQL logs, current `pg_stat_activity`, alert timestamps, and the latest known healthy backup/checkpoint for the incident record.

## 3. Recovery & Restore Procedures

### Prefer service recovery over restore

If the database data is intact, restore connectivity or complete the managed failover first. Confirm the primary accepts connections using `psql "$DATABASE_URL" -c 'SELECT 1;'`. A restore replaces the target database and is for confirmed corruption/data loss, not a routine response to a connection outage.

### Restore from the repository backup automation

[`scripts/restore-db.sh`](../../scripts/restore-db.sh) downloads an S3-compatible backup and **drops and recreates `PGDATABASE` on `PGHOST`** after a confirmation prompt. Obtain incident commander and database-owner approval, stop application writers/workers, and verify host, database, and selected backup before continuing. Preserve a provider snapshot of the current database when possible. Never run this against a production target to “see whether it works.”

Inject these values from the approved secret store in the operator shell/container; do not echo them or enable shell tracing:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ENDPOINT_URL (optional; defaults to AWS S3)
AWS_REGION (optional; defaults to auto)
S3_BUCKET
S3_PREFIX (optional; defaults to db-backups)
PGHOST
PGPORT (optional; defaults to 5432)
PGDATABASE (defaults to stellarwork if omitted)
PGUSER
PGPASSWORD
```

Run from the repository root with AWS CLI, `pg_restore`, `psql`, and `gzip` installed:

```bash
./scripts/restore-db.sh --list
./scripts/restore-db.sh stellarwork-<UTC timestamp>.sql.gz
```

Select a known-good timestamp from the listing and substitute the actual database-specific key. The script downloads the backup, terminates connections to that database, drops/recreates it, and runs `pg_restore --clean --if-exists --exit-on-error`; type `yes` only after rechecking the target. `./scripts/restore-db.sh --latest` is also supported, but only use it after verifying the `-latest.sql.gz` object is the intended recovery point. The `.sql.gz` suffix is the script's object naming convention: its dump uses PostgreSQL custom format; do not manually gunzip it before `pg_restore`.

[`scripts/backup-db.sh`](../../scripts/backup-db.sh) creates/uploads a current dump only when PostgreSQL is readable. Its required database and S3 settings are the same; it also needs `pg_dump`, `aws`, and `gzip`. It prunes timestamped backups older than `BACKUP_RETENTION_DAYS` (default 30), so do not run it as a substitute for preserving a failed database snapshot.

After restore, keep writes paused until schema and data checks pass. Start one backend instance first so its normal migration/bootstrap path completes; then restore the normal replica and worker count. If startup logs show the 10-attempt retry exhaustion, correct connectivity before repeated restarts.

## 4. Verification & Post-Recovery Reconciliation

1. Confirm `psql "$DATABASE_URL" -c 'SELECT now(), pg_is_in_recovery();'` succeeds against the intended writable primary. Check the expected schema/migration version and row counts, and spot-check recent jobs, escrows, and contract events against the recovery point.
2. Confirm `GET /health` returns 200 with database, Redis, Horizon, and Soroban healthy. Confirm `/health/db` has no sustained waiters and `/metrics` shows `pg_pool_waiting_connections` at zero or baseline, stable `pg_pool_total`, and query latency/error rates returned to normal.
3. Verify the indexer checkpoint and compare it with the live Testnet ledger (or the configured network):

   ```sql
   SELECT synced, last_processed_ledger, last_transaction_at, updated_at
   FROM indexer_state WHERE id = 1;
   SELECT max(ledger) AS latest_mirrored_contract_event FROM contract_events;
   ```

   ```bash
   curl -fsS "$HORIZON_URL/ledgers?limit=1&order=desc"
   ```

   `HORIZON_URL` defaults to `https://horizon-testnet.stellar.org`; verify `STELLAR_NETWORK`, `HORIZON_URL`, `SOROBAN_RPC_URL`, and `CONTRACT_ID`/`ESCROW_CONTRACT_ID` are for the same network and deployed contract. On bootstrap, `indexerService.start()` resumes the **platform-wallet transaction stream** from `indexer_state.last_processed_ledger`; its contract transaction stream and Horizon contract-event stream start at cursor `now`. Observe `/health`'s `indexer` object and logs for progress/errors before reopening writes.

4. Important limitation: this repository does not persist a contract-event cursor or provide a supported command/API to replay historical contract events or reconcile every escrow row against Soroban state. Restarting the service resumes its transaction streams but does **not** backfill missed contract events. Do not manually lower `last_processed_ledger`: transaction processing includes non-idempotent donor-stat increments, so replay can corrupt aggregates. If the restored checkpoint predates the outage or an event gap is suspected, keep escrow-sensitive workflows degraded and have an engineer perform a reviewed, idempotent Soroban/Horizon event reconciliation before declaring mirror state caught up. Record the gap and the last verified ledger.
5. Run read-only smoke checks for representative funded, in-progress, released, refunded, and disputed escrows; compare `jobs`/`escrows` status and `contract_events` to the live ledger/transaction evidence. Re-enable producers gradually, monitor 5xx and pool alerts, then close the status window and record RPO/RTO and the backup key used.

## Related documentation

- [Redis outage runbook](./redis-outage.md)
- [Database backup and restore guide](../database-backup-restore.md)
- [Database pool tuning](../database-pool-tuning.md)
- [Environment variables](../environment-variables.md)
