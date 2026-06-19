# Production Alert Runbooks

This runbook covers the production alerts managed in this repository for:

- `nextjs-prod`
- `nestjs-prod`
- `monitoring`
- Argo CD applications backed by `veecodekub-infra`

Use this document when Alertmanager, Grafana, Discord, or another notification channel reports an alert.

## Golden Rules

1. Confirm whether the alert is still firing before changing anything.
2. Prefer GitOps rollback through `veecodekub-infra` over direct live edits.
3. Capture evidence before restart, rollback, or sync.
4. If user traffic is affected, restore service first and investigate root cause after.

## Quick Triage

Check the current alert state:

```bash
kubectl get prometheusrule -n monitoring
kubectl get pods -n nextjs-prod
kubectl get pods -n nestjs-prod
kubectl get pods -n monitoring
```

Open Grafana:

```text
Dashboards -> Production Overview
Alerting -> Alert rules
Explore -> Loki
```

Useful Loki queries:

```logql
{namespace="nextjs-prod"}
{namespace="nestjs-prod"}
{namespace="nextjs-prod"} |= "error"
{namespace="nestjs-prod"} |= "error"
```

Useful Prometheus queries:

```promql
ALERTS{alertstate="firing"}
probe_success{job="production-uptime"}
kube_deployment_status_replicas_available{namespace=~"nextjs-prod|nestjs-prod"}
increase(kube_pod_container_status_restarts_total{namespace=~"nextjs-prod|nestjs-prod"}[10m])
```

## Standard Evidence To Capture

Run these before taking action when possible:

```bash
kubectl get deploy,pods,svc,ingress,hpa,pdb -n nextjs-prod -o wide
kubectl get deploy,pods,svc,ingress,hpa,pdb -n nestjs-prod -o wide
kubectl get events -n nextjs-prod --sort-by=.lastTimestamp | tail -50
kubectl get events -n nestjs-prod --sort-by=.lastTimestamp | tail -50
```

For a specific pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers --tail=200
kubectl logs <pod-name> -n <namespace> --all-containers --previous --tail=200
```

## Rollback Pattern

Production deployments are promoted through infra PRs and image digests. Prefer rollback by reverting the infra commit that changed the prod image.

1. Find the recent prod promotion commit in `veecodekub-infra`.
2. Revert it in GitHub or locally.
3. Open a PR.
4. Wait for `validate` to pass.
5. Merge the PR.
6. Sync the affected Argo CD application.

Local example:

```bash
git switch main
git pull --ff-only
git revert <infra-promotion-commit-sha>
git push origin main
```

Use direct rollback only for urgent recovery, and reconcile Git afterward.

## ProductionEndpointDown

Impact: A public production endpoint is not returning a successful HTTP response.

Common causes:

- App pod is down or not ready.
- Ingress or TLS is broken.
- Endpoint path is wrong.
- NetworkPolicy blocks traffic.
- External dependency makes readiness fail.

Check uptime targets:

```bash
kubectl get probe production-uptime -n monitoring -o yaml
kubectl get ingress -n nextjs-prod
kubectl get ingress -n nestjs-prod
```

Check from outside the cluster:

```bash
curl -i https://nextjs-prod.veecodekub.dev/nextjs-cicd
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/live
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/ready
```

Check apps:

```bash
kubectl get pods -n nextjs-prod -o wide
kubectl get pods -n nestjs-prod -o wide
kubectl describe ingress nextjs -n nextjs-prod
kubectl describe ingress nestjs -n nestjs-prod
```

Mitigation:

- If pods are not ready, follow `NextJSProdReplicasUnavailable` or `NestJSProdReplicasUnavailable`.
- If only one endpoint path fails, verify `monitoring/uptime-checks/probe.yaml`.
- If a recent image promotion caused the failure, revert the infra promotion PR.
- If TLS failed, inspect cert-manager certificate and challenge resources.

Cert checks:

```bash
kubectl get certificate -A
kubectl describe certificate nextjs-prod-tls -n nextjs-prod
kubectl describe certificate nestjs-prod-tls -n nestjs-prod
kubectl get challenge,order -A
```

## ProductionEndpointSlow

Impact: A production endpoint is responding, but response time is consistently high.

Check endpoint latency:

```promql
probe_duration_seconds{job="production-uptime"}
```

Check resource pressure:

```bash
kubectl top pods -n nextjs-prod
kubectl top pods -n nestjs-prod
kubectl get hpa -n nextjs-prod
kubectl get hpa -n nestjs-prod
```

Check logs:

```logql
{namespace="nextjs-prod"} |= "error"
{namespace="nestjs-prod"} |= "error"
```

Mitigation:

- If CPU or memory is high, follow `ProdContainerHighCPU` or `ProdContainerHighMemory`.
- If NestJS readiness is slow, check database connectivity.
- If traffic spike is expected, consider increasing HPA max replicas through GitOps.

## NextJSProdReplicasUnavailable

Impact: NextJS prod has fewer than 2 available replicas.

Check:

```bash
kubectl get deploy nextjs -n nextjs-prod
kubectl describe deploy nextjs -n nextjs-prod
kubectl get rs,pods -n nextjs-prod -o wide
kubectl get events -n nextjs-prod --sort-by=.lastTimestamp | tail -50
```

Inspect logs:

```bash
kubectl logs deploy/nextjs -n nextjs-prod --tail=200
kubectl logs deploy/nextjs -n nextjs-prod --previous --tail=200
```

Mitigation:

- If image pull fails, verify GHCR image and digest in `apps/nextjs/overlays/prod/kustomization.yaml`.
- If readiness fails, verify `/nextjs-cicd` returns HTTP 200 from inside the pod or service.
- If a recent promotion caused the issue, revert the infra promotion PR.

Inside-cluster check:

```bash
kubectl run curl-nextjs -n nextjs-prod --rm -it --image=curlimages/curl --restart=Never -- \
  curl -i http://nextjs.nextjs-prod.svc.cluster.local/nextjs-cicd
```

## NestJSProdReplicasUnavailable

Impact: NestJS prod has fewer than 2 available replicas.

Check:

```bash
kubectl get deploy nestjs -n nestjs-prod
kubectl describe deploy nestjs -n nestjs-prod
kubectl get rs,pods -n nestjs-prod -o wide
kubectl get events -n nestjs-prod --sort-by=.lastTimestamp | tail -50
```

Inspect logs:

```bash
kubectl logs deploy/nestjs -n nestjs-prod --tail=200
kubectl logs deploy/nestjs -n nestjs-prod --previous --tail=200
```

Check health endpoints:

```bash
kubectl run curl-nestjs -n nestjs-prod --rm -it --image=curlimages/curl --restart=Never -- \
  curl -i http://nestjs.nestjs-prod.svc.cluster.local/api/v1/health/ready
```

Mitigation:

- If init container is failing, follow `NestJSMigrationInitContainerFailed`.
- If secrets are missing, inspect ExternalSecret.
- If readiness fails because the database is unavailable, restore database connectivity first.
- If a recent promotion caused the issue, revert the infra promotion PR.

ExternalSecret checks:

```bash
kubectl get externalsecret nestjs-secret -n nestjs-prod
kubectl describe externalsecret nestjs-secret -n nestjs-prod
kubectl get secret nestjs-secret -n nestjs-prod
```

## ProdPodCrashLooping

Impact: A production pod is repeatedly starting and failing.

Check:

```bash
kubectl get pods -n nextjs-prod
kubectl get pods -n nestjs-prod
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers --tail=200
kubectl logs <pod-name> -n <namespace> --all-containers --previous --tail=200
```

Common causes:

- Bad application image.
- Missing environment variable or secret.
- Database connection error.
- Read-only filesystem path issue.
- Migration failure.

Mitigation:

- If caused by a new image, revert the infra promotion PR.
- If missing secret, fix ExternalSecret or upstream secret store.
- If read-only filesystem issue, add an `emptyDir` mount through GitOps only if the app truly needs writable storage.

## ProdContainerRestartingFrequently

Impact: A production container restarted more than expected in a short window.

Check restart count:

```bash
kubectl get pods -n nextjs-prod
kubectl get pods -n nestjs-prod
kubectl describe pod <pod-name> -n <namespace>
```

Check previous logs:

```bash
kubectl logs <pod-name> -n <namespace> --previous --tail=200
```

Mitigation:

- If restarts are from OOMKilled, follow `ProdContainerHighMemory`.
- If restarts are from app exceptions, inspect Loki logs and rollback if needed.
- If restarts happened during a rollout and resolved, keep monitoring.

## NestJSMigrationInitContainerFailed

Impact: NestJS rollout is blocked because the Prisma migration init container failed.

Check:

```bash
kubectl get pods -n nestjs-prod
kubectl describe pod <pod-name> -n nestjs-prod
kubectl logs <pod-name> -n nestjs-prod -c migrate --tail=200
kubectl logs <pod-name> -n nestjs-prod -c migrate --previous --tail=200
```

Common causes:

- Database is unreachable.
- Migration SQL failed.
- `DATABASE_URL` or `DIRECT_URL` is wrong.
- New migration is incompatible with current DB state.

Mitigation:

- If DB connectivity is broken, restore network/credentials first.
- If migration is bad, stop promotion and revert the infra PR to the previous image digest.
- Do not manually edit production schema unless you have a verified rollback plan.
- After rollback, verify pods are ready and create a follow-up fix PR in `nestjs-cicd`.

## ProdHPAAtMaxReplicas

Impact: HPA scaled an app to max replicas for more than 10 minutes.

Check:

```bash
kubectl get hpa -n nextjs-prod
kubectl get hpa -n nestjs-prod
kubectl describe hpa nextjs -n nextjs-prod
kubectl describe hpa nestjs -n nestjs-prod
kubectl top pods -n nextjs-prod
kubectl top pods -n nestjs-prod
```

Mitigation:

- If traffic is expected, consider increasing `maxReplicas` in the prod overlay.
- If traffic is abnormal, inspect ingress logs and rate patterns.
- If app is inefficient, check CPU and memory panels and app logs.

## ProdContainerHighCPU

Impact: A production container has used more than 85% of its CPU limit for more than 10 minutes.

Check:

```bash
kubectl top pods -n nextjs-prod
kubectl top pods -n nestjs-prod
kubectl get hpa -n nextjs-prod
kubectl get hpa -n nestjs-prod
```

Grafana:

```text
Production Overview -> CPU Usage vs Limit
```

Mitigation:

- If HPA can scale, wait for scale-up and verify latency improves.
- If HPA is at max, follow `ProdHPAAtMaxReplicas`.
- If caused by a deploy, rollback through infra PR.
- If normal traffic now needs more CPU, update resources in base or prod overlay through GitOps.

## ProdContainerHighMemory

Impact: A production container has used more than 85% of its memory limit for more than 10 minutes.

Check:

```bash
kubectl top pods -n nextjs-prod
kubectl top pods -n nestjs-prod
kubectl describe pod <pod-name> -n <namespace>
```

Grafana:

```text
Production Overview -> Memory Usage vs Limit
```

Check for OOMKilled:

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" "}{.lastState.terminated.reason}{"\n"}{end}'
```

Mitigation:

- If memory leak started after deploy, rollback through infra PR.
- If steady-state memory is expected, increase memory requests/limits through GitOps.
- If OOMKilled, check previous logs before the container restarts again.

## KubePodNotReady

Impact: A pod, including monitoring components such as Grafana, stayed non-ready for more than 15 minutes.

Check:

```bash
kubectl get pods -A | grep -v Running
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers --tail=200
```

For Grafana:

```bash
kubectl get pods -n monitoring | grep grafana
kubectl rollout status deployment kube-prometheus-stack-grafana -n monitoring
kubectl logs deploy/kube-prometheus-stack-grafana -n monitoring --all-containers --tail=200
```

Mitigation:

- If a rollout is in progress, wait for it to finish and confirm the alert resolves.
- If the alert sends `[RESOLVED]`, no action is needed.
- If Grafana is unavailable, alerts may still fire through Alertmanager, but dashboards may be unavailable.

## Argo CD Sync Issues

If an app is OutOfSync or Degraded:

```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```

In Argo CD UI:

```text
Application -> Refresh -> Diff -> Sync
```

Recommended sync options:

- Use normal sync for most changes.
- Use `Prune` only when intentionally deleting resources.
- Avoid `Force` and `Replace` unless you understand the resource impact.

## After Incident Checklist

1. Confirm alert is resolved in Grafana.
2. Confirm affected Argo CD app is `Healthy` and `Synced`.
3. Confirm production endpoints return HTTP 200.
4. Add a short incident note to the PR or issue.
5. If a runbook step was missing, update this file.
