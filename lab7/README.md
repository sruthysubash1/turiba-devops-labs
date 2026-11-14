# Lab 7 · Run it on a cluster

GitHub user: sruthysubash1 · Fallback image used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. -->

## 1 · Deploy

- 🔮 Before my first `apply`: how many Pods, and what their names start with? I predicted 3 Pods with names starting with `api-` because the Deployment specifies `replicas: 3`. What happened: Kubernetes created 3 Pods, and all 3 became Running and Ready.

```text
$ kubectl -n course get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           8m1s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/api-9b5f7f68f   3         3         3       8m1s

NAME                      READY   STATUS    RESTARTS   AGE
pod/api-9b5f7f68f-cpgpz   1/1     Running   0          8m1s
pod/api-9b5f7f68f-fqzrk   1/1     Running   0          8m1s
pod/api-9b5f7f68f-vfrrb   1/1     Running   0          8m1s

$ curl -s localhost:8080/
{"name":"course-api","version":"0.1.0"}
```

## 2 · Heal and scale

- 🔮 After deleting one Pod, two seconds later I expected to see a replacement Pod because the Deployment maintains the desired number of replicas. What happened: Kubernetes created a new Pod, and all 3 Pods were Running and Ready.

```text
$ kubectl -n course get pods        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
api-9b5f7f68f-fqzrk   1/1     Running   0          18m
api-9b5f7f68f-sq5cn   1/1     Running   0          31s
api-9b5f7f68f-vfrrb   1/1     Running   0          18m

$ kubectl -n course get pods        (with replicas: 5)
NAME                  READY   STATUS    RESTARTS   AGE
api-9b5f7f68f-fqzrk   1/1     Running   0          18m
api-9b5f7f68f-j2tsv   1/1     Running   0          9s
api-9b5f7f68f-pd9qv   1/1     Running   0          9s
api-9b5f7f68f-sq5cn   1/1     Running   0          65s
api-9b5f7f68f-vfrrb   1/1     Running   0          18m
```

## 3 · Roll and roll back
### 3a. Roll forward to a newer image

The original image tag was `0.2.0`. I changed it to my latest successful CI image:

`ghcr.io/sruthysubash1/turiba-devops-labs/api:88314695cc1c5cdaf516711184d7849866d02309`

I created PR #13 with the commit message `chore: roll api to latest release`. The PR passed all four CI checks and was merged.

After applying the manifests, I checked the rollout:

`kubectl -n course rollout status deployment/api --timeout=180s`

Output: `deployment "api" successfully rolled out`

Three Pods were running and the API returned HTTP 200 OK.

### 3b. Test a broken image and roll back

I changed the image tag locally to `9.9.9` without committing it.

**Prediction:** I expected all three old Pods to keep running because the new image would not be available.

I applied the manifests and checked the rollout:

`kubectl -n course rollout status deployment/api --timeout=30s`

Output: `error: timed out waiting for the condition`

I checked the Pods. All three old Pods remained `1/1 Running`, while the new Pod was `0/1 ImagePullBackOff`.

I rolled back using:

`kubectl -n course rollout undo deployment/api`

Output: `deployment.apps/api rolled back`

The command warned that the Deployment had previously been managed with `kubectl apply` and that the rollback would not update the last-applied-configuration annotation.

**What I learned:** The rollback changes the running Kubernetes Deployment, but does not restore the Git manifest. I used `git restore k8s/` to restore the committed configuration, then ran `kubectl apply -f k8s/` to make the cluster match the Git files again.

The final rollout succeeded, and `git diff -- k8s/` produced no output.

The warning means that `kubectl rollout undo` changed the running Deployment but did not update the saved configuration annotation used by `kubectl apply`. I restored the original Git manifest using `git restore k8s/` and applied it again using `kubectl apply -f k8s/`. This made Git and the cluster agree again.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected the new API Pod to fail first because the image tag was changed to `0.2.O` (letter O instead of zero). I also expected the Service connection to fail because its selector and target port had been changed.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | New API Pod showed `ImagePullBackOff`. | `kubectl -n course describe pod api-77c4f7d58b-94jzz` — image `0.2.O` not found. | Corrected image tag to `0.2.0` (`8cd79e6`) |
| 2 | Service had no endpoints because its selector did not match the Pod labels. | `kubectl -n course get endpoints api` — `<none>`. Pods had label `app=api`. | Changed Service selector to `app: api` (`2b0a85f`) |
| 3 | Service target port was `8080`, but the API container listened on `5000`. | `kubectl -n course get endpoints api` — endpoints initially used port `8080`; Deployment showed `containerPort: 5000`. | Changed Service targetPort to `5000` (`09fb657`) |

After all three fixes, all three Pods were `1/1 Running`. The Service endpoints used port `5000`.

I tested the API using `curl -v --max-time 10 http://localhost:8080/`.

Output:
- HTTP status: `200 OK`
- Response: `{"name":"course-api","version":"0.1.0"}`

I committed each fix separately and pushed all three commits to PR #14. All four CI checks passed, and PR #14 was successfully merged into `main`.
