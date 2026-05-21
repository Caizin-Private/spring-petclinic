# Runbook — Database Unavailable (HikariCP connection failures)

> **Trigger**: petclinic logs show `HikariPool-1` "Failed to validate connection ... This connection has been closed" followed by `java.net.ConnectException: Connection refused` against `demo-db:5432`.
>
> **First seen**: Incident `petclinic-exc-20260521` (2026-05-21). See full RCA: `rca-output/rca-petclinic-exc-20260521-2026-05-21-09-45.md` in `platform-agent-framework`.

## Symptom signature

The application surface looks like a generic 5xx burst. The log signature is what disambiguates this from a transient network blip:

```
WARN  com.zaxxer.hikari.pool.PoolBase : HikariPool-1 - Failed to validate connection
      org.postgresql.jdbc.PgConnection@... (This connection has been closed.)
ERROR com.zaxxer.hikari.pool.HikariPool : HikariPool-1 - Connection is not available,
      request timed out after 30037ms (total=0, active=0, idle=0, waiting=1)
ERROR o.s.web.servlet.DispatcherServlet : Could not open JPA EntityManager for transaction;
      root cause: java.net.ConnectException: Connection refused
```

The defining marker is `(total=0, active=0, idle=0)` — the pool has been emptied and cannot replenish. This means the database is unreachable at the TCP layer, not slow or overloaded.

## Diagnosis (Kubernetes)

Run these commands against the cluster — they take <30s and confirm or refute the most likely cause (the database tier was removed).

```bash
# 1. Is there a backing pod for the demo-db Service?
kubectl get pods -n petclinic -l app=demo-db -o wide

# 2. If no pods, check the Deployment replica count
kubectl get deployment demo-db -n petclinic

# 3. Confirm the Service has no Endpoints (no IPs in the subset)
kubectl describe endpoints demo-db -n petclinic

# 4. Check recent events for the namespace — look for "ScalingReplicaSet ... from 1 to 0"
kubectl get events -n petclinic --sort-by=.lastTimestamp
```

If `kubectl get deployment demo-db` reports `READY 0/0` and the Endpoints describe shows no `Subsets`, the database has been scaled to zero. This is the documented failure mode for this runbook.

## Mitigation

```bash
kubectl scale deployment/demo-db -n petclinic --replicas=1
```

Then watch for recovery (typically <60s once the Postgres pod is Ready):

```bash
# Postgres pod reaches Ready
kubectl get pods -n petclinic -l app=demo-db -w

# Petclinic stops emitting new ConnectException entries
kubectl logs deploy/petclinic -n petclinic --tail=50 -f
```

Validation:

- HikariPool metrics show `active > 0` (or new `HikariPool-1 - Added connection` lines appear in petclinic logs)
- `/owners` or any DB-backed endpoint returns 200
- No new `ConnectException` entries in the last minute

## Why the symptom lags the cause (~11 minutes)

When the database pod is gracefully terminated, the application's HikariCP pool does **not** see connection failures immediately — it continues to hand out cached connections that the kernel still considers ESTABLISHED. The pool only discovers the breakage when:

1. A connection is validated (HikariCP `keepaliveTime`, default ~30s but only after `idleTimeout` interactions), or
2. A new query is issued and the underlying socket returns ECONNRESET / EOF.

In the 2026-05-21 incident the delay was ~11 minutes. Don't be misled by the time gap — correlate the **k8s scaling event**, not the petclinic logs, when identifying T0.

## Follow-up actions (not covered by this runbook)

These are tracked separately in the RCA (`§5.2` and `§5.3`) and should not be done as part of mitigation:

- Prometheus alert on `kube_deployment_status_replicas{deployment="demo-db",namespace="petclinic"} == 0`
- `PodDisruptionBudget minAvailable=1` for `demo-db`
- Expose DB connectivity in `/actuator/health` and add graceful degradation in the petclinic app
- Document and approve (or eliminate) the scale-down policy that triggered the incident

## Related

- RCA: `rca-output/rca-petclinic-exc-20260521-2026-05-21-09-45.md`
- Service: `demo-db` in namespace `petclinic` (Deployment, single replica, PVC-backed, image `postgres:18.3`)
- App: `petclinic` Spring Boot, HikariCP defaults, JPA EntityManager

<!-- TODO: link the Grafana dashboard for HikariCP metrics once one exists. -->
