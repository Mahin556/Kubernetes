
## 🔑 What is `metadata`?

* It’s a block that stores **identifiers and organizational info** about the resource.
* Kubernetes itself and other tools use it to track, query, and manage resources.
* Metadata does **not** define how the app runs (that’s `spec`); it only describes the resource.

---

## 📌 Key Fields in Metadata

### 1. **name**

* Unique name of the object **within a namespace**.
* Used when referencing the object with `kubectl`.

✅ Example:

```yaml
metadata:
  name: nginx-deployment
```

---

### 2. **namespace**

* Logical grouping of resources in Kubernetes.
* Defaults to `default` if not specified.
* Common use cases: separating **dev/test/prod**, multi-tenancy.

✅ Example:

```yaml
metadata:
  namespace: dev-environment
```

---

### 3. **labels**

* Key-value pairs used for **grouping, filtering, and selecting** resources.
* Used heavily by **Services, Deployments, and Selectors**.
* Good for environment, app, version, or team tags.

✅ Example:

```yaml
metadata:
  labels:
    app: web
    env: prod
    version: v1.2
```

---

### 4. **annotations**

* Key-value pairs like labels, **but not used for selection**.
* Store **extra metadata**: links, configs, monitoring, build info, etc.
* Tools (Prometheus, Helm, Istio, ArgoCD) often depend on annotations.

✅ Example:

```yaml
metadata:
  annotations:
    monitoring: "true"
    maintainer: "team@company.com"
    git-commit: "a1b2c3d4"
```

---

### 5. **System-generated metadata (auto-added by Kubernetes)**

* Not written by you; Kubernetes creates them automatically:

  * `uid` → unique ID across cluster
  * `resourceVersion` → internal version for concurrency control
  * `creationTimestamp` → when the object was created
  * `managedFields` → tracks field ownership for server-side apply

✅ Example (automatically added when you run `kubectl get -o yaml`):

```yaml
metadata:
  uid: "9f8c15d2-cc9c-11e9-9f22-42010a8001cf"
  resourceVersion: "12345"
  creationTimestamp: "2025-09-24T12:34:56Z"
```

---

## 📌 Full Example with Metadata

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  namespace: prod
  labels:
    app: web
    platform: java
    release: "18.0"
  annotations:
    monitoring: "true"
    prod: "true"
    owner: "team-alpha"
```
### References
- https://devopscube.com/kubernetes-deployment-tutorial/