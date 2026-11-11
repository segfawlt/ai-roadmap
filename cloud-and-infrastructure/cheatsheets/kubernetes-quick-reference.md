# Kubernetes Quick Reference

## Core primitives

| Resource | Purpose |
|----|----|
| **Pod** | Smallest deployable unit — one or more containers that share network and storage. Rarely created directly. |
| **Deployment** | Manages ReplicaSets of Pods. Handles rolling updates and rollbacks. Your default workload type. |
| **Service** | Stable virtual IP and DNS name for a set of Pods. Types: `ClusterIP`, `NodePort`, `LoadBalancer`. |
| **Ingress** | HTTP(S) routing from outside the cluster to Services. Requires an Ingress Controller (nginx-ingress, Traefik, AWS ALB, etc.). |
| **ConfigMap** | Non-secret key-value configuration injected into Pods as env vars or files. |
| **Secret** | Secret key-value data (base64-encoded, **not** encrypted by default). Use Sealed Secrets or external-secrets for real encryption. |
| **Namespace** | Logical isolation boundary. Use one per team or environment. |

## `kubectl` command cheatsheet

``` bash
# Context / cluster
kubectl config current-context
kubectl config get-contexts
kubectl config use-context prod-cluster

# Inspect
kubectl get pods -n my-namespace
kubectl get pods -o wide                    # with node + IP
kubectl get pods -l app=web                  # label selector
kubectl describe pod my-pod
kubectl logs my-pod -f                       # follow
kubectl logs my-pod -c sidecar --previous    # previous container
kubectl top pod                              # CPU / memory usage
kubectl get events --sort-by='.lastTimestamp'

# Apply / delete
kubectl apply -f manifest.yaml
kubectl apply -k ./overlays/prod             # kustomize
kubectl delete -f manifest.yaml
kubectl rollout status deployment/web
kubectl rollout undo deployment/web
kubectl rollout history deployment/web

# Exec / port-forward
kubectl exec -it my-pod -- sh
kubectl port-forward svc/web 8080:80
kubectl cp my-pod:/etc/config ./config

# Debugging
kubectl get pod my-pod -o yaml               # full manifest
kubectl explain deployment.spec.strategy     # built-in API reference
kubectl debug -it my-pod --image=busybox --target=my-container
```

## Minimal production Deployment

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: my-app
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: ghcr.io/acme/web:1.4.2
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: web-secrets
                  key: database_url
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
```

### What every one of those lines is actually doing for you

- **`replicas:`` ``3`** — at least 3 for quorum; survive one node loss.

- **`maxUnavailable:`` ``0`** — zero-downtime rolling updates.

- **`resources.requests`** — scheduler uses these to place the pod.

- **`resources.limits`** — kernel cgroup enforces these as hard caps.

- **`livenessProbe`** — restart container if it hangs.

- **`readinessProbe`** — remove from Service endpoints if it’s temporarily unhealthy. **These two are different.** A liveness probe that also handles readiness causes restart storms under load.

## Service + Ingress

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: my-app
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: web-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

## Gotchas

1.  **Default namespace** — never ship anything real to `default`. Always use a named namespace.

2.  **Secrets are base64, not encrypted** — enable encryption-at-rest at the etcd layer, or use Sealed Secrets / external-secrets.

3.  **`latest` tag** — never. Pin to an immutable tag. Use `imagePullPolicy:`` ``IfNotPresent`.

4.  **No resource requests** — the scheduler will pack pods anywhere, and noisy neighbors will crush you.

5.  **Missing PodDisruptionBudget** — cluster upgrades can evict all your replicas simultaneously. Always set a PDB for prod workloads.

## Practice

Use a local cluster (`kind`, `minikube`, or `k3d`) or a managed free tier. Stay in a dedicated namespace to avoid collisions.

### 1. Deploy the minimal web app

Use the Deployment + Service + Ingress YAML from the documentation page. Point the image at a public sample app like `nginx:1.27-alpine` and expose it under `localhost:8080` via `kubectl`` ``port-forward`. Confirm rolling updates work:

``` bash
kubectl set image deployment/web web=nginx:1.28-alpine
kubectl rollout status deployment/web
```

Then roll it back:

``` bash
kubectl rollout undo deployment/web
```

### 2. Readiness probe experiment

Modify the probe so that `/ready` returns `503` when a file `/tmp/not-ready` exists. Deploy. Then `kubectl`` ``exec` into the pod and `touch`` ``/tmp/not-ready`. Verify:

- The pod is removed from the Service endpoints (`kubectl`` ``get`` ``endpoints`` ``web`)

- The pod is **not** restarted (readiness failure is distinct from liveness failure)

- `rm`` ``/tmp/not-ready` returns it to the endpoint list

### 3. ConfigMap and Secret injection

Create a ConfigMap containing three key-value pairs and a Secret containing a fake API key. Mount the ConfigMap as a volume and the Secret as env vars. Verify inside the pod that both are readable.

### 4. PodDisruptionBudget

Add a PDB that guarantees at least 2 of 3 replicas stay up during a voluntary disruption. Drain a node (`kubectl`` ``drain`` ``<node>`` ``--ignore-daemonsets`). Verify the drain waits until rescheduling is possible before evicting the last pod.

### 5. Debug a failing pod

Deploy a pod intentionally broken (image tag that doesn’t exist, or a bad command). Walk through:

1.  `kubectl`` ``get`` ``pods` — see the `ImagePullBackOff` or `CrashLoopBackOff`

2.  `kubectl`` ``describe`` ``pod`` ``<name>` — read the events

3.  `kubectl`` ``logs`` ``<name>`` ``--previous` — see the last crash’s output

4.  Fix and redeploy

Being fluent at this cycle is 80% of real Kubernetes operations.

## Review Questions

1.  **What is the smallest deployable unit in Kubernetes?**

    - A. Container

    - B. Pod

    - C. Deployment

    - D. Node

2.  **Which Kubernetes resource manages rolling updates and rollbacks of Pods?**

    - A. Pod

    - B. Deployment

    - C. Service

    - D. Ingress

3.  **A liveness probe that fails causes Kubernetes to…**

    - A. Remove the pod from the Service endpoints only

    - B. Restart the container

    - C. Delete the pod permanently

    - D. Send an alert email

4.  **A readiness probe that fails causes Kubernetes to…**

    - A. Restart the container

    - B. Remove the pod from the Service endpoints (but leave the container running)

    - C. Scale the deployment

    - D. Delete the ConfigMap

5.  **Why are liveness and readiness probes different resources?**

    - A. Historical accident

    - B. They handle different failure modes: liveness catches hangs that need restart, readiness catches transient unhealthiness that should just remove traffic

    - C. Liveness is deprecated

    - D. Readiness is paid, liveness is free

6.  **How should you inject a database password into a Kubernetes Pod?**

    - A. Hardcode it in the Deployment YAML

    - B. Commit it to Git inside a ConfigMap

    - C. Put it in a Secret and reference via `secretKeyRef`

    - D. Embed it in the Docker image

7.  **Kubernetes Secrets are stored in etcd…**

    - A. Always encrypted

    - B. Base64-encoded by default (not encrypted). You must enable encryption-at-rest or use an external solution for real protection.

    - C. Hashed with SHA-256

    - D. Not stored at all — they exist only in memory

8.  **What does `kubectl`` ``rollout`` ``undo`` ``deployment/web` do?**

    - A. Deletes the deployment

    - B. Reverts to the previous revision of the Deployment

    - C. Deletes the namespace

    - D. Rolls the cluster back to the previous Kubernetes version

9.  **Why should you avoid the `latest` tag on container images in production?**

    - A. `latest` is slower to pull

    - B. `latest` is mutable — you can’t roll back or reason about what’s actually running

    - C. It’s banned by the Kubernetes API

    - D. It uses more disk space

10. **What does a PodDisruptionBudget protect you from?**

    - A. Noisy neighbors

    - B. Voluntary disruptions (node drains, cluster upgrades) evicting too many replicas at once

    - C. Attackers

    - D. OOMKills

**View Answer Key**

1.  **B** — Pod is the smallest deployable unit (containers are a level below but not directly scheduled).

2.  **B** — Deployment → ReplicaSet → Pods, with rollout/rollback support.

3.  **B** — Liveness failure → container restart.

4.  **B** — Readiness failure → remove from Service endpoints only.

5.  **B** — Different failure modes require different responses; conflating them causes restart storms.

6.  **C** — Secret + `secretKeyRef` is the correct injection pattern.

7.  **B** — Base64 is an encoding, not encryption. Enable etcd encryption-at-rest for real protection.

8.  **B** — Rolls the Deployment back to the previous revision stored in `revisionHistoryLimit`.

9.  **B** — `latest` is mutable; use immutable tags (or better, digests) for reproducibility.

10. **B** — PDBs ensure cluster maintenance doesn’t take down too many replicas simultaneously.
