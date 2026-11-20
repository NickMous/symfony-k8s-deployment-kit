# Upgrading to Postgres 18

Kit v0.2.0 runs `postgres:18-alpine` (was `postgres:16-alpine`). Postgres
can't open a data directory from an older major version, so a database that
already holds data has to be dumped and restored once. Bumping the kit
without doing that is safe but not silent: Postgres 18 refuses to start with

```
FATAL:  database files are incompatible with server
DETAIL:  The data directory was initialized by PostgreSQL version 16, which is not compatible with this version 18.x.
```

so there is no risk of the app starting against a new, empty database.

The procedure below keeps the old data on the same volume (renamed to
`pgdata-16`), which makes rollback a rename instead of a restore. It needs
free space on the volume for a second copy of the data; check with `df -h`
in the Postgres pod first.

Expect a few minutes of downtime: the app is stopped while the data moves.

## Before you start

Set these for your project:

```sh
CTX=my-cluster                    # kubectl context
NS=myproject-production           # namespace
FLUX_KS=myproject-production      # Flux Kustomization that applies the overlay
# Resource names with the overlay's namePrefix applied (check with
# `kubectl get deploy,statefulset`); these are the kit defaults:
APP=myproject-app
WORKER=myproject-app-messenger
PG=myproject-app-postgres
POD=${PG}-0
PVC=postgres-data-${PG}-0
k() { kubectl --context "$CTX" -n "$NS" "$@"; }
# Waits until a Deployment has no pods left (including terminating ones)
wait_no_pods() {
  local sel
  sel=$(k get deploy "$1" -o go-template='{{range $k, $v := .spec.selector.matchLabels}}{{$k}}={{$v}},{{end}}')
  while [ -n "$(k get pods -o name -l "${sel%,}")" ]; do sleep 2; done
}
# Exact row count per table
count_rows() {
  k exec "$POD" -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Atc "select format(\$\$select %L, count(*) from %I.%I;\$\$, relname, schemaname, relname) from pg_stat_user_tables order by 1" | psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -At'
}
```

1. **Take a backup outside the cluster** (Longhorn backup to S3, or similar),
   and check that it completed.
2. Open the project PR that moves the kit ref to `v0.2.0`, but **don't merge
   it yet**. If Flux applies the overlay from your main branch, merging is
   what makes the change permanent (step 9).

   With per-branch staging, pushing that PR branch already moves every
   staging environment to Postgres 18, because they share the synced base.
   Handle the staging databases first (see the last section).

## Procedure

```sh
# 1. Stop Flux from undoing the manual steps below
flux --context "$CTX" suspend kustomization "$FLUX_KS"

# 2. Stop everything that writes to the database. With replicas at 0 the
#    HorizontalPodAutoscaler stays inactive.
k scale deploy "$APP" "$WORKER" --replicas=0
wait_no_pods "$APP"; wait_no_pods "$WORKER"

# 3. Dump the database and record row counts to compare after the restore
k exec "$POD" -- sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" --no-owner' > dump.sql
count_rows > counts-before.txt

# 4. Stop Postgres
k scale statefulset "$PG" --replicas=0
k wait --for=delete pod "$POD" --timeout=120s

# 5. Keep the old data on the volume as pgdata-16
k run pgdata-rename --rm -i --restart=Never --image=alpine:3 \
  --overrides='{"spec":{"containers":[{"name":"pgdata-rename","image":"alpine:3","stdin":true,"command":["sh","-c","cd /data && mv pgdata pgdata-16 && ls -la"],"volumeMounts":[{"name":"d","mountPath":"/data"}]}],"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"'"$PVC"'"}}]}}'

# 6. Start Postgres 18; it initializes an empty pgdata with the same user,
#    password and database name from the existing secret and config
k set image statefulset "$PG" postgres=postgres:18-alpine
k scale statefulset "$PG" --replicas=1
k wait --for=condition=Ready pod "$POD" --timeout=180s

# 7. Restore, refresh planner statistics, and compare row counts
k exec -i "$POD" -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -v ON_ERROR_STOP=1 -q' < dump.sql
k exec "$POD" -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -qc "analyze"'
count_rows > counts-after.txt
diff counts-before.txt counts-after.txt && echo "row counts match"

# 8. Start the app again
k scale deploy "$APP" "$WORKER" --replicas=2

# 9. Merge the PR that moves to kit v0.2.0, then let Flux take over again
flux --context "$CTX" resume kustomization "$FLUX_KS"
```

`dump.sql` contains all application data. Keep it somewhere safe until
you're confident in the upgrade, then delete it.

After a few days without problems, remove the old data:

```sh
k exec "$POD" -- rm -rf /var/lib/postgresql/data/pgdata-16
```

## Rollback

Until `pgdata-16` is removed, going back takes a minute and doesn't need the
dump. Changes written after the upgrade are lost.

```sh
flux --context "$CTX" suspend kustomization "$FLUX_KS"
k scale deploy "$APP" "$WORKER" --replicas=0
k scale statefulset "$PG" --replicas=0
k wait --for=delete pod "$POD" --timeout=120s
k run pgdata-rollback --rm -i --restart=Never --image=alpine:3 \
  --overrides='{"spec":{"containers":[{"name":"pgdata-rollback","image":"alpine:3","stdin":true,"command":["sh","-c","cd /data && mv pgdata pgdata-18-failed && mv pgdata-16 pgdata && ls -la"],"volumeMounts":[{"name":"d","mountPath":"/data"}]}],"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"'"$PVC"'"}}]}}'
k set image statefulset "$PG" postgres=postgres:16-alpine
k scale statefulset "$PG" --replicas=1
k scale deploy "$APP" "$WORKER" --replicas=2
```

Revert the kit ref to `v0.1.9` in the project before resuming Flux.

## Databases without data worth keeping

For staging or preview databases, delete the PVC together with the pod and
let the StatefulSet create a new one. The app's `run-migrations` init
container creates the schema on the next start.

```sh
k delete pvc "$PVC" --wait=false
k delete pod "$POD"
```
