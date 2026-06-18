# The Ultimate Kubernetes Cheat Sheet
Kubernetes is an open-source platform for deploying, scaling, and managing containerized applications.

The smallest deployable unit is a Pod; in practice, you usually manage Pods through higher-level workload objects like Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs.

A Service gives a stable network endpoint to a changing set of Pods, and Ingress exposes HTTP/HTTPS into the cluster (though Kubernetes now recommends Gateway over Ingress for new work).

![Kubernetes Cluster Architecture](files/images/kubernetes-cluster-architecture.svg)

## 1) The 15 kubectl commands you'll use every day
```bash
# cluster / context
kubectl version
kubectl cluster-info
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>

# namespace safety
kubectl get ns
kubectl config set-context --current --namespace=<ns>

# read resources
kubectl get pods
kubectl get pods -A
kubectl get pods -o wide
kubectl get deploy,sts,ds -A
kubectl get svc -A
kubectl get events -A --sort-by-.metadata.creationTimestamp

# inspect / debug
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> --all-containers --previous
kubectl exec -it <pod> -n <ns> -- sh
kubectl port-forward pod/<pod> 8080:8080 -n <ns>
kubectl port-forward svc/<svc> 8080:80 -n <ns>

# apply / delete
kubectl apply -f app.yaml
kubectl delete -f app.yaml

# explain APIs
kubectl api-resources
kubectl explain deployment
kubectl explain deployment.spec.template.spec.containers
```

## 2) Safe context + namespace workflow
Best habit in Kubernetes: always check your current context and namespace before write operations. The kubeconfig controls which cluster kubectl talks to, and `kubectl config set-context --current --namespace=...` sets your default namespace.

```bash
kubectl config current-context
kubectl get ns
kubectl config set-context --current --namespace=dev
```

## 3) Core object types: when to use what
| Object | Use |
| ---- | ---- |
| Pod | the basic execution unit; usually not created directly for production apps. |
| Deployment | for stateless apps, rolling updates, rollbacks, and scaling. |
| StatefulSet | for apps needing stable identity and/or persistent storage (databases, brokers, clustered systems). |
| DaemonSet | run one Pod on every node (or selected nodes), often for logging/monitoring/network agents. |
| Job | run-to-completion task. |
| CronJob | scheduled job, like cron. |
| Service | stable endpoint in front of one or more pods. |
| Ingress | HTTP/HTTPS routing into the cluster; API is stable but frozen, and Gateway is the recommended newer direction. |

## 4) The YAML anatomy you must memorize
Every object has these top-level fields:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: my-app
    labels:
        app: my-app
spec:
    ...
```
This structure follows Kubernetes API object conventions: **apiVersion**, **kind**, **metadata**, and then an object-specific **spec**

## 5) Minimal app you can actually deploy
Deployment + Service (the 80% user case)

[Link to Example App](examples/app/app.yaml)

A Deployment manages a replicated set of Pods and supports controlled rollouts and rollbacks, while a Service exposes a stable endpoint to those Pods using label selectors.

Apply it:
```bash
kubectl apply -f app.yaml
kubectl get deploy,pods,svc
kubectl rollout status deploy/web
```
The rollout status and declarative apply workflows are standard Deployment management patterns in the official docs.

## 6) Labels + selectors = how Kubernetes wires things together
Labels are key/value metadata attached to objects.

Selectors match labels to decide which Pods a Deployment manags or which Pods a Service routes traffic to.

Use recommended labels like `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/managed-by` for consistency

Example:
```yaml
metadata:
    labels:
        app.kubernetes.io/name: payments
        app.kubernetes.io/instance: payments-prod
        app.kubernetes.io/version: "1.8.3"
        app.kubernetes.io/managed-by: helm
```

The `kubernetes.io` and `k8s.io` label namespaces are reserved for Kubernetes, and the `app.kubernetes.io/*` set is the recommended application labeling convention.

## 8) Probes: one of the most important reliability features
Kubernetes supports startup, readiness, and liveness probes. 

A readiness probe removes a Pod from Service endpoints until it is ready.

A liveness probe restarts a stuck container.

A startup probe protects slow-starting apps by delaying liveness/readiness until startup succeeds.

```yaml
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
    periodSeconds: 5

startupProbe:
    httpGet:
        path: /startup
        port: 8080
    periodSeconds: 5
    failureThreshold: 24
```

Use readiness for "can receive traffic".

Use liveness for "must be restarted".

Use startup for "boot isn't done yet".

## 9) Resource requests vs limits (critical for stability)
Requests tell the scheduler how much CPU/memory a container needs for placement.

Limits are enforced at runtime by the kubelet/kernal. CPU overages are throttled; memory overages can trigger OOM kills.

If you set a limit but no request, Kubernetes may copy the limit into the request.

```yaml
resources:
    requests:
        cpu: "250m"
        memory: "256Mi"
    limits:
        cpu: "500m"
        memory: "512Mi"
```

Rule of thumb: set realistic requests, be deliberate with memory limits, and avoid guessing values blindly. The official docs explicitly describe the scheduler/request relationship and the different runtime behavior of CPU throttling vs memory OOM.

[Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

## 10) ConfigMaps vs Secrets
A ConfigMap stores non-confidential config as key/value data. Pods can consume it via environment variables, command arguments, or mounted files.

A ConfigMap is not secret storage; use a Secret for confidential data.

### ConfigMap example
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: app-config
data:
    LOG_LEVEL: "info"
    FEATURE_FLAG_X: "true"
```

Mount as env:
```yaml
envFrom:
    - configMapRef:
        name: app-config
```

ConfigMaps decouple environment-specifc configuration from container images, and Pods must reference ConfigMaps in the same namespace.

### Secret example
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: db-secret
type: Opaque
stringData:
    username: appuser
    password: supersecret
```

Use Secrets for sensitive material, not ConfigMaps.

## 11) Services: what type should I use?
ClusterIP - internal-only stable service endpoint; default type.

NodePort - exposes a port on every node.

LoadBalancer - asks your cloud provider / implementation for an external load balancer.

Ingress (or preferably Gateway) - application-layer HTTP/HTTPS routing into the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Services abstract over ephemeral Pod IPs by presenting a stable endpoint for clients.

## 12) Ingress quick reference
Ingress provides HTTP/HTTPS routing by hostname/path rules, but the API is frozen and the Kubernetes project recommends Gateway for newer designs. Also, creating an Ingress resource alone does nothing unless an Ingress controlelr is installed.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: web
spec:
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

## 13) NetworkPolicy: default-open unless you lock it down
NetworkPolicies let you control traffic at Layer 3/4 (IP/port/protocol) between Pods and between Pods and the outside world, but only if your network plugin supports NetworkPolicy enforcement. By default, Pods are generally non-isolated until a matching policy applies.

Default deny ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
    name: default-deny-ingress
spec:
    podSelector: {}
    policyTypes:
        - Ingress
```

Allow traffic from same namespace to Pods labeled app=api
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
    name: allow-api
spec:
    podSelector:
        matchLabels:
            app: api
    policyTypes:
        - Ingress
    ingress:
        - from:
            - podSelector: {}
```
Policies are application-centric and use selectors to define allowed traffic.

## 14) Storage: PV, PVC, and StatefulSets
A PersistentVolume (PV) is cluster storage, provisioned statically or dynamically.

A PersistentVolumeClaim (PVC) is a user request for storage, similar to how a Pod requests compute.

Use StatefulSets when the app needs stable identity and persistent storage; Kubernetes notes that if you do not need stable identifiers or ordered deployment, a Deployment is usually a better fit.

PVC example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: data
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 10Gi
```

Mount into a Pod/Deployment
```yaml
volumeMounts:
    - name: data
      mountPath: /data
volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

## 15) DaemonSets, Jobs, and CronJobs
#### DaemonSet
Use when a Pod must run on every node (logging agents, monitoring agents, CNI helpers).

```bash
kubectl get ds -A
kubectl describe ds <name> -n <ns>
```

#### Job
Use for one-off tasks that must run to completion, with retries when needed.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
    name: migrate-db
spec:
    template:
        spec:
            restartPolicy: Never
            containers:
                - name: migrate
                  image: my-app:latest
                  command: ["./migrate.sh"]
    backoffLimit: 4
```

#### CronJob
Use for scheduled tasks like backups or reports. CronJobs create Jobs on a schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
    name: nightly-backup
spec:
    schedule: "0 2 * * *"
    jobTemplate:
        spec:
            template:
                spec:
                    restartPolicy: OnFailure
                    containers:
                        - name: backup
                          image: my-backup:latest
```