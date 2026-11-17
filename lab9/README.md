# Lab 9 · Package it with Helm

GitHub user: sruthysubash1 · Fallback image used: no · Fallback Mission 8 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password. -->

## 1 · The chart

```text
$ helm lint charts/course-app
==> Linting charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
  replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/sruthysubash1/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- 🔮Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted the API replicas line would change to 2, while the database replicas and image lines would remain unchanged, because `values.yaml` provides the default values and `templates/api-deployment.yaml` uses `.Values.api.replicas`. What happened: exactly as predicted; the API replicas changed to 2, while the database replicas remained 1 and the images stayed unchanged.
- In my own words, why the Namespace and the Secret are **not** in the chart: The Namespace is created separately to provide a place for the application. The Secret contains the database password and is created outside Helm so that Helm does not manage or delete it when the release is uninstalled.

## 2 · Install, upgrade, rollback

```text
$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Tue Nov 17 09:33:55 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Tue Nov 17 09:39:51 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-69d6cdc79b-88vnz   0/1     ErrImagePull   0          26s
api-6d564b47f7-b857g   1/1     Running        0          6m21s
api-6d564b47f7-t4dz2   1/1     Running        0          6m21s
api-6d564b47f7-t7ct9   1/1     Running        0          6m21s

$ helm history course-app -n course            (before the rollback)
1  superseded  Install complete
2  deployed    Upgrade complete

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ helm history course-app -n course            (after the rollback)
1  superseded  Install complete
2  superseded  Upgrade complete
3  deployed    Rollback to 1
```

- 🔮 Before the upgrade with a tag that doesn't exist: will `helm upgrade` say it failed? Will `GET /api/todos` still answer? I predicted Helm would report the upgrade as deployed because it accepts the configuration change without waiting for the new Pod to become ready, and the API would still answer because the old Pods would continue serving requests. What happened: Helm reported `STATUS: deployed`, one new Pod showed `ErrImagePull`, the three old Pods remained Running, and `GET /api/todos` returned the saved todo.
- 🔮 Before the rollback: which revision number will `helm rollback course-app 1` create? I predicted revision 3 because Helm keeps the existing history and creates a new revision for every rollback. What happened: revision 3 was created with status `deployed` and description `Rollback to 1`.
- 🔮 Before `helm uninstall` and a new `helm install`: will your todo still be there? I predicted no because uninstalling Helm also deletes the database PVC that stores the todo data. What happened: after reinstalling, `curl -s localhost:8081/api/todos` returned `[]`, confirming that the old todo was lost.

```text
$ curl -s localhost:8081/api/todos
[]
```

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: I expected the API image repository typo (`.Values.api.images.repository`) to cause the first Helm upgrade failure because the chart uses `image`, not `images`.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | Helm upgrade failed because the API image repository value used images instead of image. | helm upgrade course-app charts/course-app -n course → nil pointer evaluating interface {}.repository | Changed .Values.api.images.repository to .Values.api.image.repository (76f82da). |
| 2 | Helm could not import the existing course namespace because it was not managed by this Helm release. | helm upgrade course-app charts/course-app -n course → Namespace "course" ... exists and cannot be imported into the current release | Removed charts/course-app/templates/namespace.yaml (c7f41e4). |
| 3 | Helm upgrade succeeded, but the API had only one replica instead of three. | kubectl -n course get deployment api → api  1/1  1  1 | Changed replicaCount: 3 to replicas: 3 in values.yaml (5aa694c). |
