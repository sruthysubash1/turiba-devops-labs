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

- 3a: I rolled from the tag ____ to the tag ____.

```text
$ kubectl -n course rollout status deployment/api
____
```

- 3b: 🔮 With the tag 9.9.9, I predicted that ____ old Pods keep running, because ____. What happened: ____

```text
$ kubectl -n course get pods        (during the stall)
____

$ kubectl -n course rollout undo deployment/api
____
```

- What the `rollout undo` warning means, in my own words, and what I did so that Git and the cluster agree again: ____

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: ____

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | ____ | ____ | ____ |
| 2 | ____ | ____ | ____ |
| 3 | ____ | ____ | ____ |
