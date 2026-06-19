# Supabase Backup, Restore, and Migration Runbook

This runbook covers the production database used by `nestjs-prod`.

Supabase is the system of record for application data. App rollback is not enough if a migration or data change damages the database. Use this runbook before production migrations and during database incidents.

## Recovery Objectives

Current recommended objectives:

- RPO: 24 hours with daily backups, or minutes/seconds if PITR is enabled.
- RTO: depends on Supabase restore duration and database size.
- Restore method: Supabase Dashboard restore or Supabase Management API.
- App rollback method: revert the relevant `veecodekub-infra` promotion PR and sync Argo CD.

## Supabase Backup Facts

- Supabase automatically backs up Pro, Team, and Enterprise projects daily.
- Pro projects can access 7 days of daily backups.
- Team projects can access 14 days of daily backups.
- Enterprise projects can access up to 30 days of daily backups.
- PITR gives finer-grained restore points and can restore to a specific point in time.
- Restoring from backup or PITR makes the project inaccessible during the restore.
- Database backups do not restore deleted Supabase Storage objects, only database metadata.
- Daily backup files do not include passwords for custom roles.

References:

- https://supabase.com/docs/guides/platform/backups
- https://supabase.com/docs/reference/cli/supabase-db-dump

## Production Readiness Decision

For production, prefer:

1. Enable PITR on the Supabase production project.
2. Keep daily logical dumps outside Supabase for extra safety if the data is business-critical.
3. Test restore to a separate project at least once per month.
4. Take or confirm a fresh restore point before high-risk migrations.

If PITR is not enabled, document the accepted data loss window based on daily backups.

## Required Access

People performing restore must have:

- Supabase project owner/admin access.
- Access to Supabase Dashboard `Database -> Backups`.
- Access to `nestjs/prod` database secret source if connection strings need rotation.
- Access to Argo CD for app sync/rollback.
- Access to GitHub `veecodekub-infra` for rollback PRs.

Never commit Supabase database URLs, service role keys, access tokens, or backup files to Git.

## Before Every Production Migration

Use this before merging or syncing a production NestJS image that contains Prisma migrations.

1. Confirm Supabase project health.
2. Confirm a usable backup or PITR restore point exists.
3. Confirm `nestjs-prod` is healthy before starting.
4. Confirm rollback target image digest is known.
5. Confirm the migration has been tested against a staging/test database.

Pre-check commands:

```bash
kubectl get pods -n nestjs-prod
kubectl get deploy nestjs -n nestjs-prod
kubectl get externalsecret nestjs-secret -n nestjs-prod
kubectl get hpa,pdb -n nestjs-prod
```

Health checks:

```bash
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/live
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/ready
```

Record current image:

```bash
kubectl get deploy nestjs -n nestjs-prod -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deploy nestjs -n nestjs-prod -o jsonpath='{.spec.template.spec.initContainers[0].image}{"\n"}'
```

Record infra image pin:

```bash
grep -A4 'ghcr.io/veecodekub/nestjs-cicd' apps/nestjs/overlays/prod/kustomization.yaml
```

## Manual Logical Backup

Use this when you need an extra downloadable backup before a risky change.

Preferred location: an encrypted, access-controlled storage location outside Git.

Example using Supabase CLI:

```bash
supabase db dump \
  --db-url "$SUPABASE_DB_URL" \
  --file "backups/nestjs-prod-$(date +%Y%m%d-%H%M%S).sql"
```

Schema-only backup:

```bash
supabase db dump \
  --db-url "$SUPABASE_DB_URL" \
  --schema public \
  --file "backups/nestjs-prod-schema-$(date +%Y%m%d-%H%M%S).sql"
```

Rules:

- Do not store dumps in Git.
- Encrypt dumps at rest.
- Restrict access to project owners/admins.
- Record where the dump is stored and when it was taken.

## Restore From Supabase Daily Backup

Use this when you need to restore the whole project to a previous daily backup.

Impact:

- The Supabase project is inaccessible during restore.
- Data written after the selected backup may be lost.
- Storage objects deleted after that backup are not restored by database backup.

Steps:

1. Announce maintenance window.
2. Pause risky deploys or merges.
3. In Supabase Dashboard, open the production project.
4. Go to `Database -> Backups`.
5. Pick the closest backup before the incident.
6. Confirm restore.
7. Wait for Supabase to finish restore.
8. Verify database health.
9. Restart or resync `nestjs-prod` if connections need refresh.

Post-restore checks:

```bash
kubectl rollout restart deployment nestjs -n nestjs-prod
kubectl rollout status deployment nestjs -n nestjs-prod
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/ready
```

## Restore With PITR

Use PITR when you need to restore to a point shortly before a bad migration, bad deploy, or destructive data operation.

Steps:

1. Identify the incident time in timezone `Asia/Bangkok`.
2. Choose a restore timestamp before the damaging operation.
3. In Supabase Dashboard, open `Database -> Backups -> Point in Time`.
4. Select the timestamp.
5. Confirm restore.
6. Wait until Supabase reports restore completion.
7. Verify app health and data integrity.

Example Management API shape:

```bash
export SUPABASE_ACCESS_TOKEN="<redacted>"
export PROJECT_REF="<project-ref>"
export RECOVERY_TIME_TARGET_UNIX="<unix-timestamp>"

curl -X POST "https://api.supabase.com/v1/projects/$PROJECT_REF/database/backups/restore-pitr" \
  -H "Authorization: Bearer $SUPABASE_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"recovery_time_target_unix\":\"$RECOVERY_TIME_TARGET_UNIX\"}"
```

Never paste access tokens into tickets, PRs, or chat.

## Restore To A New Supabase Project

Use this for validation or forensic recovery when you do not want to overwrite production.

Recommended for:

- Monthly restore tests.
- Inspecting data before deciding whether to restore production.
- Recovering a few rows manually.

High-level process:

1. Create or duplicate a Supabase project.
2. Restore backup into the new project.
3. Run application smoke tests against the restored database.
4. Compare critical data.
5. Destroy the temporary project when finished.

Do not point production `nestjs-prod` at a temporary restored database unless this is an intentional failover plan.

## Bad Migration Response

Use this when `NestJSMigrationInitContainerFailed` fires or a migration damages data.

Immediate triage:

```bash
kubectl get pods -n nestjs-prod
kubectl describe pod <pod-name> -n nestjs-prod
kubectl logs <pod-name> -n nestjs-prod -c migrate --tail=200
kubectl logs <pod-name> -n nestjs-prod -c migrate --previous --tail=200
```

If migration failed before changing data:

1. Revert the infra promotion PR to the previous image digest.
2. Merge rollback PR.
3. Sync Argo CD `nestjs-prod`.
4. Create a fix PR in `nestjs-cicd`.

If migration changed data or schema:

1. Stop further deploys.
2. Identify exact incident time.
3. Decide between PITR restore, daily backup restore, or manual repair.
4. Prefer PITR if available and data loss must be minimized.
5. Restore.
6. Roll back app image if the new app requires the bad schema.
7. Verify health endpoints and critical user flows.

Do not run ad hoc destructive SQL in production unless the SQL has been reviewed and the backup/restore path is confirmed.

## App Rollback After DB Restore

If the restored database schema matches the previous app version, roll back NestJS image through GitOps:

1. Revert the most recent NestJS prod promotion commit in `veecodekub-infra`.
2. Wait for `validate`.
3. Merge PR.
4. Sync Argo CD `nestjs-prod`.
5. Verify pods, health endpoints, and logs.

Checks:

```bash
kubectl get pods -n nestjs-prod
kubectl rollout status deployment nestjs -n nestjs-prod
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/live
curl -i https://nestjs-prod.veecodekub.dev/api/v1/health/ready
```

## Monthly Restore Test

Run monthly or before major releases.

1. Restore a backup to a separate Supabase project.
2. Point a local or temporary test deployment at the restored database.
3. Run smoke tests:
   - NestJS health endpoints.
   - Login/auth flow if applicable.
   - Main read/write API flow.
   - Prisma migration status.
4. Record:
   - Backup timestamp.
   - Restore start time.
   - Restore finish time.
   - Issues found.
   - Whether restored app worked.

Success criteria:

- Restore completes.
- App can connect.
- Critical tables exist.
- Critical data is present.
- Smoke tests pass.

## Incident Checklist

During incident:

1. Confirm alert and impact.
2. Capture evidence.
3. Stop risky deploys.
4. Choose restore or rollback path.
5. Announce downtime if restore is required.
6. Restore or rollback.
7. Verify app and data.

After incident:

1. Confirm alerts resolved.
2. Confirm Argo CD apps are `Healthy` and `Synced`.
3. Add incident notes to an issue or PR.
4. Update this runbook if any step was missing.
5. Add tests or migration safeguards to prevent recurrence.
