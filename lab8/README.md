# Lab 8 · Run the whole app

GitHub user: `sruthysubash1` · Fallback image used: no · Fallback Mission 7 files used: no

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted it would remain because PostgreSQL stores its data on a PersistentVolumeClaim, independently of the Pod. What happened: the replacement database Pod started successfully, and both todos were still available.

```text
$ kubectl -n course get pvc,pv
persistentvolumeclaim/db-data   Bound   pvc-e38aa773-39a7-4c2e-bc46-0a0cab61a5e6   1Gi   RWO   standard

persistentvolume/pvc-e38aa773-39a7-4c2e-bc46-0a0cab61a5e6   1Gi   RWO   Delete   Bound   course/db-data   standard

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"Lab 8 persistence verification"}' localhost:8080/api/todos
{"id":2,"title":"Lab 8 persistence verification","done":false,"created_at":"2026-11-14T09:56:14.910Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
db-7c669bb77d-dm5n2   1/1   Running   0   61m

$ kubectl -n course delete pod db-7c669bb77d-dm5n2
pod "db-7c669bb77d-dm5n2" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
db-7c669bb77d-hqzvc   1/1   Running   0   23s

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-14T08:45:13.335Z"},{"id":2,"title":"Lab 8 persistence verification","done":false,"created_at":"2026-11-14T09:56:14.910Z"}]
```

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
Found 3 pods, using pod/api-985965977-8hv2q
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
Found 3 pods, using pod/api-985965977-8hv2q
course-api 0.1.0 starting (Node.js v24.22.2)
API listening on port 5000
Database ready
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted it would not show the password as plain text because Kubernetes stores Secret data in base64-encoded form. What I saw, in words (don't paste the value): the Secret data is encoded, not displayed as the original plain-text password. Base64 is encoding, not encryption.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted the containers would be restarted because the liveness endpoint would return HTTP 404. What happened: the liveness probe failed, the API containers restarted repeatedly, and some Pods entered CrashLoopBackOff. The readiness probe also temporarily failed while containers were restarting.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
API Pods showed increasing restart counts; some entered CrashLoopBackOff.

$ kubectl -n course describe pod api-769cc9d449-27228   (only the Events)
Liveness probe failed: HTTP probe failed with statuscode: 404
Container api failed liveness probe, will be restarted
Back-off restarting failed container
```

- How I made Git and the cluster agree again afterwards: I ran `git restore k8s/` to discard the temporary probe change, then ran `kubectl apply -f k8s/`. I verified that all API Pods returned to `1/1 Running`, the database remained healthy, and `git status` showed a clean working tree.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected the readiness probe to fail first. However, the first fault I actually found was the API init container waiting for the database.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | API Pod stuck at `Init:0/1`. The init container could not reach the database because the hostname was incorrect. | `kubectl -n course logs api-5c5d5748bc-pdjh8 -c wait-for-db` → `database:5432 - no response` | Changed `DB_HOST` from `database` to `db`. Commit: `0df4524` |
| 2 | API Pod showed `CreateContainerConfigError` because the referenced Secret key did not exist. | `kubectl -n course describe pod` → `couldn't find key pasword in Secret course/db-secret` | Corrected the Secret key reference from `pasword` to `password`. Commit: `7cd55f4` |
| 3 | API Pod showed `0/1 Running` because the readiness probe used the wrong port. | `kubectl -n course describe pod` → `Readiness probe failed: Get "http://10.244.0.26:5001/healthz": dial tcp 10.244.0.26:5001: connect: connection refused` | Changed the readiness probe port from `5001` to `5000`. Commit: `a44122e` |

