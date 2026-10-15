# Lab 42 – Kubernetes Deployment and Runtime Validation

## 1. Environment

| Item | Value |
|---|---|
| OS | Ubuntu 26.04.1 LTS (2 vCPU, 3.3 GiB RAM) |
| k3s | `v1.36.5+k3s1` |
| Container runtime | `containerd://2.3.4-k3s1.36` |
| Node | `devops-vm` – `Ready` |
| Namespace | `quote-lab` (default context namespace) |

## 2. Image built with Docker and imported into k3s

k3s runs containers with containerd, whose image store is separate from Docker's. A Docker build alone is invisible to Kubernetes, so the image was exported and imported:

```
docker build -t quote-app:local -f docker/Dockerfile .
docker save quote-app:local -o ~/quote-app.tar
sudo k3s ctr images import ~/quote-app.tar
```

```
$ sudo k3s ctr images list | grep quote-app:local
docker.io/library/quote-app:local   ...   86.4 MiB   linux/amd64
```

## 3. Deployment and Service

Manifests: [`deployment.yaml`](deployment.yaml), [`service.yaml`](service.yaml).

```
$ kubectl rollout status deployment/quote-app
deployment "quote-app" successfully rolled out

$ kubectl get svc
NAME        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)
quote-app   ClusterIP   10.43.74.16   <none>        80/TCP

$ kubectl get endpointslices -l kubernetes.io/service-name=quote-app
NAME              ADDRESSTYPE   PORTS   ENDPOINTS
quote-app-wkkzv   IPv4          3000    10.42.0.9
```

How the manifests produce this behavior:

- The Deployment `selector.matchLabels` and the Pod template label are both `app: quote-app`, so the Deployment (through its ReplicaSet) owns those Pods.
- The Service selects the same label and forwards its `port: 80` to the container's `targetPort: 3000`, where the app listens (`PORT ?? 3000`, `HOST ?? 0.0.0.0` in `server.js`).
- `imagePullPolicy: IfNotPresent` makes the kubelet use the locally imported image instead of pulling from a registry.
- The readiness probe (`GET /health` on 3000) decides when a Pod becomes eligible for Service traffic.

## 4. Health requests

Port-forward (VM loopback → one selected Pod):

```
$ kubectl port-forward svc/quote-app 8080:80
$ curl --fail --show-error --max-time 5 http://127.0.0.1:8080/health
{"ok":true}
```

In-cluster, through cluster DNS and the Service path (temporary `busybox:1.37` client Pod):

```
$ kubectl exec service-check -- wget -qO- http://quote-app/health
{"ok":true}
```

The port-forward attaches to a single Pod and does not prove load balancing; the in-cluster request goes through DNS and the Service, like another workload in the cluster would.

## 5. Scaling to three replicas

`replicas: 1` was changed to `replicas: 3` in `deployment.yaml`, then applied, so the saved file and the cluster describe the same desired state:

```
$ kubectl get pods -l app=quote-app -o wide
NAME                         READY   STATUS    RESTARTS   IP
quote-app-69fbbcff49-h29kh   1/1     Running   0          10.42.0.12
quote-app-69fbbcff49-hk54p   1/1     Running   0          10.42.0.9
quote-app-69fbbcff49-s66r2   1/1     Running   0          10.42.0.11

$ kubectl get rs -l app=quote-app
NAME                   DESIRED   CURRENT   READY
quote-app-69fbbcff49   3         3         3
```

Changing only `replicas` does not change the Pod template, so the existing ReplicaSet scaled up and no new ReplicaSet was created. Scaling adds capacity and tolerance to the loss of one Pod; it does not create a database or validate quote operations, and three Pods on a single node do not survive the loss of the VM.

## 6. Pod replacement (reconciliation)

| | Pod name |
|---|---|
| Deleted | `quote-app-69fbbcff49-hk54p` |
| Replacement | `quote-app-69fbbcff49-4dhtl` |

```
$ kubectl get pods -l app=quote-app -w
quote-app-69fbbcff49-hk54p   1/1   Terminating         0   46m
quote-app-69fbbcff49-4dhtl   0/1   Pending             0   0s
quote-app-69fbbcff49-4dhtl   0/1   ContainerCreating   0   0s
quote-app-69fbbcff49-4dhtl   0/1   Running             0   1s
quote-app-69fbbcff49-4dhtl   1/1   Running             0   6s
```

The ReplicaSet observed 2 Pods instead of the desired 3 and created a new Pod with a new name, Ready about 6 s later. The two other Ready Pods stayed eligible for Service traffic meanwhile; this demonstrates reconciliation, not a zero-downtime guarantee.

## 7. Deliberate readiness failure

Only the readiness probe path was changed, `/health` → `/does-not-exist`; image, port and `replicas: 3` were unchanged. The rollout stalled, as expected:

```
$ kubectl rollout status deployment/quote-app --timeout=60s
Waiting for deployment "quote-app" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition

$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS
quote-app-57b88c776b-hpf5n   0/1     Running   0
quote-app-69fbbcff49-4dhtl   1/1     Running   0
quote-app-69fbbcff49-h29kh   1/1     Running   0
quote-app-69fbbcff49-s66r2   1/1     Running   0

$ kubectl get rs -l app=quote-app
NAME                   DESIRED   CURRENT   READY
quote-app-57b88c776b   1         1         0
quote-app-69fbbcff49   3         3         3
```

**Probe failure event (the cause):**

```
$ kubectl describe pod quote-app-57b88c776b-hpf5n
    State:          Running
    Ready:          False
    Restart Count:  0
    Readiness:      http-get http://:3000/does-not-exist delay=3s timeout=1s period=5s #success=1 #failure=3
Events:
  Warning  Unhealthy  79s (x126 over 11m)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```

Application log excerpt (one of the kubelet's probe requests, every 5 s):

```
"url": "/does-not-exist", "remoteAddress": "10.42.0.1"
Route GET:/does-not-exist not found
res: { "statusCode": 404 }
```

**Service endpoints during the failure:**

| Pod | IP | `conditions.ready` |
|---|---|---|
| `quote-app-69fbbcff49-s66r2` | 10.42.0.11 | `true` |
| `quote-app-69fbbcff49-h29kh` | 10.42.0.12 | `true` |
| `quote-app-69fbbcff49-4dhtl` | 10.42.0.13 | `true` |
| `quote-app-57b88c776b-hpf5n` | 10.42.0.14 | `false` |

```
$ kubectl exec service-check -- wget -qO- http://quote-app/health
{"ok":true}
```

The new template created a new ReplicaSet (`57b88c776b`). Its Pod ran but never became Ready, so the endpoint was marked `ready: false` and excluded from the Service. With the default rolling update (3 replicas → maxSurge 1, maxUnavailable 0), no old Pod was removed: the rollout was stuck, but the three old healthy Pods kept serving, so there was no outage. `Restart Count: 0` shows a failing readiness probe does not restart the container.

Diagnostic signals compared: the status (`0/1`) said *what* was wrong, the logs showed a healthy app answering 404 to an unknown route, and the **event** (`Readiness probe failed … 404`) explained the cause most directly.

## 8. Recovery

The probe path was restored to `/health` and applied:

```
$ kubectl rollout status deployment/quote-app --timeout=120s
deployment "quote-app" successfully rolled out

$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS    RESTARTS
quote-app-69fbbcff49-4dhtl   1/1     Running   0
quote-app-69fbbcff49-h29kh   1/1     Running   0
quote-app-69fbbcff49-s66r2   1/1     Running   0

$ kubectl get rs -l app=quote-app
NAME                   DESIRED   CURRENT   READY
quote-app-57b88c776b   0         0         0
quote-app-69fbbcff49   3         3         3

$ kubectl get endpointslices -l kubernetes.io/service-name=quote-app
NAME              ADDRESSTYPE   PORTS   ENDPOINTS
quote-app-wkkzv   IPv4          3000    10.42.0.11,10.42.0.12,10.42.0.13

$ kubectl exec service-check -- wget -qO- http://quote-app/health
{"ok":true}
```

The restored template matched the original one, so the Deployment scaled the original ReplicaSet `69fbbcff49` back as the current revision and scaled `57b88c776b` to 0; the failing Pod was removed and the same three healthy Pods remained.

## 9. Symptom / Cause / Fix

- **Symptom:** after applying the change, a new Pod stayed `0/1` Ready while `Running` with 0 restarts, and `kubectl rollout status` timed out.
- **Cause:** the readiness probe path was changed to `/does-not-exist`; the app has no such route and answers HTTP 404, so every probe failed (`Readiness probe failed: HTTP probe failed with statuscode: 404`).
- **Fix:** restored the readiness probe path to `/health`, a route that exists and returns `{"ok":true}` with HTTP 200. The probe succeeded, the Pods became Ready and the rollout completed. The probe was kept, not removed, so it still protects Service traffic.

## 10. Extension – Lane B: liveness probe

**Change:** a `livenessProbe` was added to the container, copied from the readiness probe (`GET /health` on 3000, `initialDelaySeconds: 3`, `periodSeconds: 5`, default `failureThreshold: 3`). The final committed manifest keeps both probes on `/health`.

Healthy configuration:

```
$ kubectl describe pod quote-app-574875b8c-...
    Restart Count:  0
    Liveness:       http-get http://:3000/health delay=3s timeout=1s period=5s #success=1 #failure=3
    Readiness:      http-get http://:3000/health delay=3s timeout=1s period=5s #success=1 #failure=3
```

**Deliberate liveness failure:** only the liveness path was changed to `/does-not-exist`; readiness stayed on `/health`.

```
$ kubectl get pods -l app=quote-app
NAME                         READY   STATUS             RESTARTS
quote-app-6b69879984-dpbsn   1/1     Running            5 (11s ago)
quote-app-6b69879984-r79lz   0/1     CrashLoopBackOff   5 (11s ago)
quote-app-6b69879984-rf546   0/1     CrashLoopBackOff   5 (4s ago)

$ kubectl describe pod quote-app-6b69879984-dpbsn
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      09:31:36
      Finished:     09:31:51
    Restart Count:  5
    Liveness:       http-get http://:3000/does-not-exist delay=3s timeout=1s period=5s #success=1 #failure=3
    Readiness:      http-get http://:3000/health delay=3s timeout=1s period=5s #success=1 #failure=3
Events:
  Warning  Unhealthy  3s (x18 over 113s)  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    3s (x6 over 103s)   kubelet  Container quote-app failed liveness probe, will be restarted
  Warning  Unhealthy  2s (x3 over 102s)   kubelet  Readiness probe failed: ... connect: connection refused
  Warning  BackOff    2s (x4 over 57s)    kubelet  Back-off restarting failed container quote-app ...
```

Each container lived about 15 s (3 failed checks × 5 s) before the kubelet killed and restarted it: 18 liveness failures ÷ 3 = 6 `Killing` events. `READY` alternated between `1/1` and `0/1`, `RESTARTS` kept increasing, and the kubelet then delayed restarts with `CrashLoopBackOff`. While the app restarted, readiness also failed with `connection refused`.

**Restored healthy probe:**

```
$ kubectl rollout status deployment/quote-app --timeout=180s
deployment "quote-app" successfully rolled out

$ kubectl get pods -l app=quote-app      # ~50 s later, about 10 liveness checks
NAME                        READY   STATUS    RESTARTS
quote-app-574875b8c-2pbwr   1/1     Running   0
quote-app-574875b8c-lgjhv   1/1     Running   0
quote-app-574875b8c-wfwp8   1/1     Running   0
```

**What this demonstrates:**

- **Readiness vs liveness:** readiness answers "may this Pod receive traffic?" and its failure only removes the Pod from the Service endpoints. Liveness answers "is this container still alive?" and its failure makes the kubelet kill and restart the container. That is why the readiness failure in section 7 kept `Restart Count: 0`, while the liveness failure raised it to 5.
- **Rollout behavior:** the Deployment only waits for *readiness* before replacing old Pods. With a broken readiness probe the rollout stalled and the old Pods kept serving. With a broken liveness probe, the new Pods briefly became Ready, so the rollout **replaced all three healthy Pods** and then every Pod entered a restart loop, degrading the service. A misconfigured liveness probe is therefore more dangerous than a misconfigured readiness probe; in production it should stay simple and tolerant (longer `initialDelaySeconds`, higher `failureThreshold`).
- The bad URL models a failing check; it is not a real application deadlock.

## 11. What this evidence leaves untested

- `/health` returns `{"ok": true}` without querying PostgreSQL, so these probes prove the Node.js process answers HTTP, not that the database, quote retrieval or quote insertion work (planned in Lab 52).
- All Pods run on a single node: replicas tolerate the loss of a Pod, not the loss of the VM.
- The port-forward test reaches one Pod only; Service load balancing was not measured.
- The app does not handle `SIGTERM` (terminating containers ended in `Error` / exit code 1), so graceful shutdown of in-flight requests was not validated.
- No resource requests/limits, autoscaling or load were tested.
