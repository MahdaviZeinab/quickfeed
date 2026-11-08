# QuickFeed on Kubernetes

This folder deploys QuickFeed to the UiS Kubernetes cluster at
**https://qf-test.ux.uis.no**. Deployment is automatic: every merge to `master`
is tested, built into a container image, and rolled out by Argo CD without
anyone running `kubectl`. QuickFeed stores its data in a PostgreSQL database
that **is not stored in the pod**, so data survives pod restarts, crashes,
image upgrades and node failures.

| | |
|---|---|
| Environment | one test environment, `qf-test.ux.uis.no`, namespace `quickfeed` |
| Image | built by `Dockerfile`, pushed to `ghcr.io/mahdavizeinab/quickfeed:<commit SHA>` |
| CI | `.github/workflows/image.yml`: tests → build and push image → write the new tag into Git |
| CD | Argo CD (pull model) watches `deploy/k8s/overlays/test` and syncs the cluster |
| Rollback | `git revert`; Argo CD deploys the old version like any other change |
| HTTPS | Traefik with TLS passthrough; QuickFeed gets its own Let's Encrypt certificate |
| Database | PostgreSQL 17 (StatefulSet) with its own persistent volume |
| QuickFeed pods | 1 now; more is possible later (see "Scaling") |
| Upgrade | rolling update: the new pod is ready before the old one stops |
| Code changes | small, in `database/` (see "Code changes") |

## How a change reaches the cluster

```
 merge to master
       │
       ▼
 GitHub Actions (.github/workflows/image.yml)
   1. test-go, test-frontend      Go tests and frontend tests (in parallel)
   2. image                       build the image, push ghcr.io/mahdavizeinab/quickfeed:<SHA>
   3. update-tag                  write <SHA> into overlays/test/kustomization.yaml,
                                  commit "deploy(test): ... [skip ci]" to master
       │
       ▼
 Git (master)  ◄──── watched by ──── Argo CD (in the cluster, namespace argocd)
                                         │
                                         ▼
                              sync: apply overlays/test to namespace quickfeed
                                         │
                                         ▼
                              rolling update: new pod ready → old pod stops
```

GitHub never connects to the cluster and holds no cluster credentials. The
cluster pulls from Git instead (GitOps, pull model). This is why we chose Argo
CD over a self-hosted GitHub runner on the cluster, which GitHub does not
recommend for public repositories (see issue #37).

## Files

```
Dockerfile                          multi-stage build of the QuickFeed image
.dockerignore                       keeps databases, secrets and node_modules out of the build
.github/workflows/image.yml         CI: tests, image, update-tag
deploy/
├── argocd/
│   └── quickfeed-test.yaml         Argo CD Application: watches overlays/test, automatic sync
└── k8s/
    ├── overlays/
    │   └── test/kustomization.yaml     qf-test.ux.uis.no: namespace quickfeed + image tag (set by image.yml)
    ├── base/
    │   ├── kustomization.yaml          shared parts; no namespace, no tag, no Secrets
    │   ├── data-pvc.yaml               volume for Let's Encrypt certificates and logs (never deleted by Argo CD)
    │   ├── deployment.yaml             QuickFeed: 1 replica, RollingUpdate, probes, waits for PostgreSQL
    │   ├── service.yaml                ports 443 (https) and 80 (http)
    │   ├── ingressroute.yaml           Traefik routes for qf-test.ux.uis.no (TLS passthrough on 443, HTTP on 80)
    │   ├── postgres-statefulset.yaml   PostgreSQL 17 with its own volume
    │   ├── postgres-service.yaml       address postgres:5432, inside the cluster only
    │   └── postgres.env.example        database name, user, password (copy to postgres.env)
    └── jobs/
        └── migrate-sqlite-to-postgres.yaml  one-time copy of a SQLite database (qf.db) into PostgreSQL
```

`base` holds everything that is the same in every environment. An overlay adds
the namespace and the image tag. A production environment would be a second
overlay (for example `overlays/prod`) with its own host name and namespace,
without copying the base. We have only `test`: a second environment would need
another public IP, and the scope agreed with our mentor was to keep the setup
simple and focus on CI/CD.

## The container image

`Dockerfile` has two stages:

1. **build** (`golang:1.27.2-trixie` + Node.js + `protoc`): installs the Go and
   npm dependencies (cached as long as `go.mod`, `go.sum` and
   `package-lock.json` do not change), builds the frontend with `make ui`,
   and builds two programs: `quickfeed` and `qf-dbcopy` (used by the
   migration Job). The build fails if no frontend bundle was produced.
2. **runtime** (`debian:trixie-slim`): only the two programs, the frontend
   files and CA certificates. Go, Node.js and the source code are not in the
   final image.

Security:

- QuickFeed runs as the non-root user `10001` (`USER 10001:10001`, no login
  shell). It can still open ports 443 and 80, because the binary gets only
  that one extra permission (`setcap cap_net_bind_service`).
- The image contains no database and no secrets: `.dockerignore` keeps out
  `*.db`, `.env`, `*.pem` and `*.key`, and `/data` is empty until the volume is
  mounted.
- All configuration comes from environment variables at run time; `/app/.env`
  is empty.

## CI: `.github/workflows/image.yml`

| Job | Runs when | What it does |
|---|---|---|
| `test-go` | push and pull request to `master` | `go test` for all packages (same as upstream `go.yml`) |
| `test-frontend` | push and pull request to `master` | `npm ci` and `npm run test:ci` (same as upstream `vitest.yml`) |
| `image` | after both test jobs pass | builds the image; **pushes** to GHCR only on push to `master`, not on pull requests |
| `update-tag` | after `image`, only on push to `master` | `kustomize edit set image` in `overlays/test`, commits as `github-actions[bot]` with `[skip ci]`, pushes to `master` |

- A pull request is tested and its image is built, but nothing is pushed or
  deployed until it is merged.
- `[skip ci]` stops the tag commit from starting the workflow again.
- `concurrency: update-tag` makes two quick merges update the tag one after
  the other, never at the same time; `git pull --rebase` before the push
  avoids conflicts.
- `master` must allow GitHub Actions to push (repository settings, issue #32).

## CD: Argo CD

Argo CD is a tool that runs inside the cluster and keeps it the same as Git.
It watches the folder `deploy/k8s/overlays/test` on `master`.
When something there changes, usually the new image tag from CI, Argo CD
updates QuickFeed on the cluster by itself.

Our settings (`deploy/argocd/quickfeed-test.yaml`):

- **Automatic sync:** every change in Git is deployed without anyone running
  `kubectl`.
- **prune:** if a file is removed from Git, Argo CD removes it from the
  cluster too.
- **selfHeal:** if someone changes the cluster by hand, Argo CD changes it
  back to what Git says.
- **Protected volume:** Argo CD never deletes `quickfeed-data` (certificates
  and logs).

### Monitoring in Argo CD

The Argo CD UI shows QuickFeed live:

- **Sync status:** is the cluster the same as Git?
- **Health of each resource:** every pod, Service and volume is shown as
  Healthy, Progressing or Degraded.
- **Pod logs and events:** read directly in the UI, without `kubectl`.
- **History:** every deployed version, and which commit it came from.

During a deploy you can watch the new pod start and the old pod stop.

Argo CD is installed once by hand; then the Application is applied once:

```sh
kubectl apply -f deploy/argocd/quickfeed-test.yaml
```

### Rollback

Every deployed version is a commit, so a rollback is a revert:

```sh
git revert <commit of the bad change>   # or "Revert" on the pull request in GitHub
git push                                # CI builds the old code again; Argo CD deploys it
```

We tested a deploy and a rollback with a visible change: the page title was
changed to "QuickFeed (test)" (PR #43) and reverted (PR #44); both reached
qf-test.ux.uis.no by themselves (issue #36).

## Before you start (one-time setup)

1. `kubectl get nodes` works against the UiS cluster.
2. There is a StorageClass for Cinder:

   ```sh
   kubectl get storageclass
   ```

   The UiS cluster has no default StorageClass, so `base/data-pvc.yaml` names
   it: `csi-cinder-sc-retain` (the Cinder volume is kept even if the PVC is
   deleted). Use the same name for `storageClassName` in
   `base/postgres-statefulset.yaml` if the PostgreSQL volume stays `Pending`.
3. The three Secrets are **not in Git** and are created by hand in the
   namespace `quickfeed`:

   ```sh
   kubectl create namespace quickfeed

   # QuickFeed configuration: GitHub App, DOMAIN, ... (see .env-template)
   kubectl -n quickfeed create secret generic quickfeed-env --from-env-file=.env

   # GitHub App private key, mounted as /secrets/app-key.pem
   kubectl -n quickfeed create secret generic quickfeed-app-key --from-file=app-key.pem=<path to the .pem file>

   # PostgreSQL user, password and database name
   cp deploy/k8s/base/postgres.env.example deploy/k8s/base/postgres.env
   # edit postgres.env: set POSTGRES_PASSWORD (letters and digits only), e.g. openssl rand -hex 24
   kubectl -n quickfeed create secret generic postgres --from-env-file=deploy/k8s/base/postgres.env
   ```

4. The GHCR package `quickfeed` is public. If it is private, the cluster also
   needs an image pull secret.
5. Argo CD is installed, and the Application is applied (see "CD: Argo CD").
   From then on, merges to `master` deploy by themselves.

## HTTPS with Traefik (TLS passthrough)

QuickFeed is exposed through Traefik by `base/ingressroute.yaml`. QuickFeed
gets its own Let's Encrypt certificate (cached on the data volume), so Traefik
must not end TLS itself:

- `IngressRouteTCP` on the entry point `websecure` (443) with
  `tls.passthrough: true`: Traefik routes the encrypted connection by its host
  name (SNI) straight to QuickFeed.
- `IngressRoute` on the entry point `web` (80): plain HTTP goes to QuickFeed's
  redirect server, which answers Let's Encrypt HTTP-01 challenges and redirects
  everything else to HTTPS. This is an HTTP route, because a TCP route on port
  80 cannot match a host name and would take port 80 traffic for every host.

The host name there must match `DOMAIN` in `quickfeed-env`. This requires
Traefik v2.10 or newer with the Kubernetes CRD provider enabled, and entry
points named `websecure` (port 443) and `web` (port 80). Check that the CRD
exists with `kubectl get crd ingressroutetcps.traefik.io`.
After deploying, verify passthrough: the certificate issuer must be Let's
Encrypt, not Traefik's default certificate:

```sh
openssl s_client -connect qf-test.ux.uis.no:443 -servername qf-test.ux.uis.no </dev/null | grep issuer
```

## Health and resources

| | QuickFeed | PostgreSQL |
|---|---|---|
| Readiness probe | TCP port 443 open, every 10 s | `pg_isready`, every 10 s |
| Liveness probe | TCP port 443 open, after 30 s, every 20 s | `pg_isready` |
| Requests / limits | 100m CPU, 256Mi memory / 1Gi memory | 100m CPU, 256Mi memory / 1Gi memory |

- Two init containers run before QuickFeed: `prepare-data` creates the log
  folder on the volume, and `wait-for-postgres` waits until PostgreSQL accepts
  connections (`pg_isready -U postgres`; `-U` is needed because the pod runs as
  uid 10001, issue #9).
- The pod runs with `runAsNonRoot: true` and `runAsUser: 10001`;
  `fsGroup: 10001` makes the volume writable for that user.
- The probes tell Kubernetes and Argo CD whether each pod is healthy.
  Logs are in the Argo CD UI, or with
  `kubectl -n quickfeed logs deploy/quickfeed`.

## Deploy by hand (without Argo CD)

Normally Argo CD does this. To deploy by hand, for example on a test cluster:

```sh
kubectl apply -k deploy/k8s/overlays/test
kubectl -n quickfeed get pods
```

QuickFeed waits until PostgreSQL is ready (init container `wait-for-postgres`),
then logs `Using PostgreSQL database from QUICKFEED_DATABASE_URL` and creates
its tables. The volume at `/data` stays, for the Let's Encrypt certificates
and the logs.

## The database

QuickFeed uses only PostgreSQL. The one-time Job in `jobs/` copies an existing
SQLite database (`qf.db`) into PostgreSQL (see "Moving from SQLite to PostgreSQL").

### Why the data survives

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

### Moving from SQLite to PostgreSQL (keeping the data)

If there is SQLite data to keep (`/data/qf.db` on the volume `quickfeed-data`), copy it once.
First set the same image tag as in `overlays/test/kustomization.yaml` in
`jobs/kustomization.yaml` (it says `CHANGE-ME-TO-COMMIT-SHA`).

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
With Argo CD's `selfHeal` on, Argo CD scales QuickFeed back to 1 replica;
pause automatic sync in Argo CD during the copy.

### Checking that the data survives (for the demo / report)

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

### Code changes

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
still runs as one pod, because:

- the `/data` volume (certificates, logs) is ReadWriteOnce: it can be attached
  to only one node at a time;
- QuickFeed keeps some state in memory and runs the student tests itself
  (Docker), which would need to be shared or moved before running several
  copies.

These are not database questions and are left for later.

### Deploying without downtime

The Deployment uses `strategy: RollingUpdate` with `maxSurge: 1` and
`maxUnavailable: 0`: Kubernetes starts the new pod first, waits until its
readiness probe passes, and only then stops the old pod. The old pod keeps
serving while the new one pulls the image and waits for PostgreSQL (about
3.5 minutes on the UiS cluster). With the earlier `strategy: Recreate`,
QuickFeed was down for that time on every deploy.

For a moment there are two QuickFeed pods. Because the `/data` volume is
ReadWriteOnce, both must run on the same node; otherwise the new pod waits for
the volume and the rollout stops (the old pod keeps serving). So far both pods
have always started on `qf-worker-1`; a rule that guarantees this is still
being decided (issue #46).

## Known limitations

- **One environment:** only `test`; no production overlay (agreed scope).
- **Manual one-time steps:** the three Secrets and the Argo CD installation are
  created by hand, not from Git.
- **Simple probe:** the QuickFeed probes check that port 443 is open.
  An HTTP check on a QuickFeed page would make the health shown in Argo CD
  more exact.
- **Possible extension:** notifications (for example Argo CD Notifications to
  Slack or email) and metrics over time (Prometheus and Grafana).
- **Same-node rule:** no affinity rule yet keeps both pods on one node during a
  rollout (issue #46).
- **No database backup:** the nightly backup CronJob was removed (issue #14).
