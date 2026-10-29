# QuickFeed on Kubernetes: the database

This folder runs QuickFeed on the UiS Kubernetes cluster with a PostgreSQL
database that **is not stored in the pod**. Data survives pod restarts, crashes,
image upgrades and node failures.

QuickFeed uses only PostgreSQL. The one-time Job in `jobs/` copies an existing
SQLite database (`qf.db`) into PostgreSQL (see "Moving from SQLite to PostgreSQL").

| | PostgreSQL in the cluster |
|---|---|
| Folder | `base/` (shared) + `overlays/test/` (namespace `quickfeed`, image tag) |
| Where the data is | a PostgreSQL server (StatefulSet) with its own persistent volume |
| QuickFeed pods | 1 now; more is possible later (see "Scaling") |
| Upgrade | short downtime now; rolling upgrades possible later |
| Code changes | small, in `database/` (see "Code changes") |

```
deploy/k8s/
├── overlays/
│   └── test/kustomization.yaml     qf-test.ux.uis.no: namespace quickfeed + image tag (set by image.yml)
├── base/
│   ├── kustomization.yaml          shared parts; no namespace, no tag, no Secrets
│   ├── data-pvc.yaml               volume for Let's Encrypt certificates and logs
│   ├── deployment.yaml             QuickFeed: 1 replica, strategy Recreate, waits for PostgreSQL
│   ├── service.yaml
│   ├── ingressroutetcp.yaml        Traefik routes for qf-test.ux.uis.no (TLS passthrough on 443, HTTP on 80)
│   ├── postgres-statefulset.yaml   PostgreSQL 17 with its own volume
│   ├── postgres-service.yaml       address postgres:5432, inside the cluster only
│   └── postgres.env.example        database name, user, password (copy to postgres.env)
└── jobs/
    └── migrate-sqlite-to-postgres.yaml  one-time copy of a SQLite database (qf.db) into PostgreSQL
```

## Why the data survives

On the UiS cluster (Talos on OpenStack), a PersistentVolumeClaim (PVC) creates
an **OpenStack Cinder volume**. It is a network disk that exists outside the
pod and outside the node:

- **Pod restart / crash / new image:** the new pod mounts the same volume.
- **Node failure:** Kubernetes starts the pod on another node, and Cinder
  attaches the same volume there. (This can take a few minutes, because the
  volume must first be detached from the failed node.)
- **`kubectl delete pod`, `kubectl rollout restart`:** no data is lost.
- Data is lost only if the PVC itself is deleted (`kubectl delete pvc`, or
  deleting the namespace). Never do that without a backup.

## Before you start

1. `kubectl get nodes` works against the UiS cluster.
2. There is a StorageClass for Cinder:

   ```sh
   kubectl get storageclass
   ```

   One should be marked `(default)`. If none is, write its name into
   `storageClassName` in `base/data-pvc.yaml` and `base/postgres-statefulset.yaml`.
3. The QuickFeed image is built and pushed to GHCR by `.github/workflows/image.yml`.
   After every merge to `master`, its `update-tag` job writes the new tag (the commit SHA)
   into `overlays/test/kustomization.yaml` and commits it. You do not set it by hand.
   (For the one-time Job, set the same tag in `jobs/kustomization.yaml`.)

   If the GHCR package is private, the cluster also needs an image pull secret.
4. QuickFeed's own configuration (GitHub App, `DOMAIN`, secrets; see
   `.env-template`) is in a Secret named `quickfeed-env`:

   ```sh
   kubectl create namespace quickfeed
   kubectl -n quickfeed create secret generic quickfeed-env --from-env-file=.env
   ```

QuickFeed is exposed through Traefik by `base/ingressroutetcp.yaml`.
QuickFeed gets its own Let's Encrypt certificate, so Traefik passes TLS
through on port 443 and forwards port 80 for certificate challenges and
redirects. The host name there must match `DOMAIN` in `quickfeed-env`.
This requires Traefik v2.10 or newer with the Kubernetes CRD provider enabled,
and entry points named `websecure` (port 443) and `web` (port 80).
Check that the CRD exists with `kubectl get crd ingressroutetcps.traefik.io`.
After deploying, verify passthrough: the certificate issuer must be Let's Encrypt,
not Traefik's default certificate:

```sh
openssl s_client -connect qf-test.ux.uis.no:443 -servername qf-test.ux.uis.no </dev/null | grep issuer
```

## Deploy QuickFeed with PostgreSQL

```sh
cp deploy/k8s/base/postgres.env.example deploy/k8s/base/postgres.env
# edit postgres.env: set POSTGRES_PASSWORD (letters and digits only), e.g. openssl rand -hex 24
kubectl -n quickfeed create secret generic postgres --from-env-file=deploy/k8s/base/postgres.env
kubectl apply -k deploy/k8s/overlays/test
kubectl -n quickfeed get pods
```

QuickFeed waits until PostgreSQL is ready (init container `wait-for-postgres`),
then logs `Using PostgreSQL database from QUICKFEED_DATABASE_URL` and creates
its tables. The volume at `/data` stays, for the Let's Encrypt certificates
and the logs.

### Moving from SQLite to PostgreSQL (keeping the data)

If there is SQLite data to keep (`/data/qf.db` on the volume `quickfeed-data`), copy it once:

```sh
# 1. Stop QuickFeed, so nothing writes to qf.db during the copy
kubectl -n quickfeed scale deploy/quickfeed --replicas=0

# 2. Start PostgreSQL (QuickFeed stays stopped)
kubectl apply -k deploy/k8s/overlays/test
kubectl -n quickfeed scale deploy/quickfeed --replicas=0
kubectl -n quickfeed rollout status statefulset/postgres

# 3. Copy the data
kubectl apply -k deploy/k8s/jobs
kubectl -n quickfeed logs -f job/migrate-sqlite-to-postgres
```

The job prints the number of rows in each table in both databases and fails
if any count differs. It refuses to copy into a database that already has
data. `qf.db` is mounted read-only and is not changed.

```sh
# 4. Start QuickFeed on PostgreSQL
kubectl -n quickfeed scale deploy/quickfeed --replicas=1
```

Note: if QuickFeed was started on PostgreSQL before the copy, it has already
created (empty) tables, and the copy refuses to run if any of them has rows.

## Checking that the data survives (for the demo / report)

```sh
# 1. Log in to QuickFeed and create something (e.g. a course), or check row counts:
kubectl -n quickfeed exec postgres-0 -- sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB" -c "select count(*) from users"'

# 2. Pod failure: delete the QuickFeed pod and the database pod
kubectl -n quickfeed delete pod -l app.kubernetes.io/name=quickfeed
kubectl -n quickfeed delete pod postgres-0
kubectl -n quickfeed get pods -w        # wait until both are Running again

# 3. Upgrade: deploy a new image tag, or
kubectl -n quickfeed rollout restart deploy/quickfeed

# 4. Node failure: move the database to another node
kubectl get pod -n quickfeed postgres-0 -o wide          # note the node
kubectl cordon <node> && kubectl -n quickfeed delete pod postgres-0
kubectl get pod -n quickfeed postgres-0 -o wide          # now on another node
kubectl uncordon <node>

# After each step, the data from step 1 is still there.
```

## Code changes

QuickFeed was written for SQLite. Hein expected switching to PostgreSQL to be
"mostly a one-line change". The driver change is small, but running QuickFeed's
full test suite against PostgreSQL showed several places where PostgreSQL is
stricter than SQLite:

| Problem found by the tests | Fix |
|---|---|
| Driver hard-coded to SQLite (`database/gormdb.go`) | `NewGormDB` chooses PostgreSQL when the source is a `postgres://` URL; `main.go` reads `QUICKFEED_DATABASE_URL` |
| Generated protobuf types use column type `datetime`, which PostgreSQL does not have | create `datetime` as a PostgreSQL domain over `timestamptz` (no change to generated code or to existing SQLite databases) |
| `group_id = 0` means "no group"; PostgreSQL enforces foreign keys, SQLite (as QuickFeed uses it) does not | do not create foreign key constraints on PostgreSQL, same behavior as before |
| `grades` has no primary key, and PostgreSQL needs a target for `ON CONFLICT DO UPDATE` | add the unique index (`submission_id, user_id`) as target |
| `ORDER BY 'order'` is a constant: SQLite ignores it, PostgreSQL rejects it | quote the column correctly (assignments are now really sorted by order) |
| Duplicate-group check compared SQLite's error text | `isUniqueViolation` recognizes both databases |
| Group update inserted each grade twice in one statement | fixed in `syncGroupGrades` |
| Rows inserted with a chosen ID: PostgreSQL's ID counter does not notice them | move the counter after such inserts |
| Go timestamps have nanoseconds, PostgreSQL stores microseconds | `database.Now()` rounds to microseconds |

The PostgreSQL-specific code is in `database/postgres.go`. SQLite still works
exactly as before and remains the default.

### Running the tests against PostgreSQL

The tests use a temporary SQLite file by default. With
`QUICKFEED_TEST_DATABASE_URL` set, every test gets its own new PostgreSQL
database instead:

```sh
docker run -d --name qf-pg -e POSTGRES_PASSWORD=test -p 5432:5432 postgres:17.6
QUICKFEED_TEST_DATABASE_URL='postgres://postgres:test@localhost:5432/postgres?sslmode=disable' \
  go test ./...
```

Results: all 21 test packages pass on SQLite and on PostgreSQL (tests that
need Docker or a GitHub token are skipped, as in any local run without them).
`TestCopyTo` copies the anonymized test database (`doc/dev/test.db`) and
checks every row of every table after the copy.

## Scaling (more than one QuickFeed pod)

PostgreSQL removes the database as the reason for running only one pod: many
pods can use one PostgreSQL server, but not one SQLite file. For now QuickFeed
still runs as one pod with `strategy: Recreate`, because:

- the `/data` volume (certificates, logs) is ReadWriteOnce;
- QuickFeed keeps some state in memory and runs the student tests itself
  (Docker), which would need to be shared or moved before running several
  copies.

These are not database questions and are left for later.
