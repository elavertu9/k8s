# The Ultimate Kubernetes Cheat Sheet
Kubernetes is an open-source platform for deploying, scaling, and managing containerized applications.

The smallest deployable unit is a Pod; in practice, you usually manage Pods through higher-level workload objects like Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs.

A Service gives a stable network endpoint to a changing set of Pods, and Ingress exposes HTTP/HTTPS into the cluster (though Kubernetes now recommends Gateway over Ingress for new work).

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