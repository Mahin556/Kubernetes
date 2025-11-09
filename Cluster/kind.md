
## 🔑 What is `kind` in Kubernetes?

* `kind` tells Kubernetes **what type of resource** the YAML manifest represents.
* Example:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  ```

  → This manifest is creating a **Deployment** object.

---

## 📌 Commonly Used `kind` Values in Kubernetes

Here’s a structured list of the major resource types you mentioned (with short explanations):

| **Kind**                           | **Description**                                                                                 |
| ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| **componentstatuses**              | Shows the status of cluster components (deprecated, replaced by `kubectl get componentstatus`). |
| **configmaps**                     | Stores configuration data (key-value pairs, files) for use inside Pods.                         |
| **daemonsets**                     | Ensures a Pod runs on **every node** (or selected nodes).                                       |
| **deployments**                    | Manages stateless applications with Pods, scaling, and rolling updates.                         |
| **events**                         | Records events about objects in the cluster (like Pod scheduling failures).                     |
| **endpoints**                      | Internal objects that link Services to Pods (automatically managed).                            |
| **horizontalpodautoscalers (HPA)** | Automatically scales Pods based on CPU/memory or custom metrics.                                |
| **ingress**                        | Manages external HTTP/HTTPS access to services, with rules for routing.                         |
| **jobs**                           | Runs Pods that complete a **task** and then exit.                                               |
| **limitranges**                    | Enforces resource usage limits (CPU/memory requests and limits) per namespace.                  |
| **namespaces**                     | Provides logical separation of cluster resources.                                               |
| **nodes**                          | Represents a worker node in the cluster.                                                        |
| **pods**                           | The smallest deployable unit in Kubernetes (containers grouped together).                       |
| **persistentvolumes (PV)**         | Defines storage in the cluster that can be used by workloads.                                   |
| **persistentvolumeclaims (PVC)**   | A request for storage by a Pod (binds to a PV).                                                 |
| **resourcequotas**                 | Sets resource usage limits per namespace (like max Pods, CPU, memory).                          |
| **replicasets**                    | Ensures a specified number of identical Pods are always running.                                |
| **replicationcontrollers**         | Legacy object (older than ReplicaSets) that ensures Pod replicas.                               |
| **serviceaccounts**                | Provides an identity for Pods to interact with the API server.                                  |
| **services**                       | Exposes a set of Pods to other Pods or external clients (ClusterIP, NodePort, LoadBalancer).    |

---

## 📌 Example Manifests

### 1. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: localhost
  DB_PORT: "5432"
```

### 2. DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```

### 3. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
### References
- https://devopscube.com/kubernetes-deployment-tutorial/