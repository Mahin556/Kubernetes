
## 🔑 What is `apiVersion`?

* Every Kubernetes object (Deployment, Pod, Service, etc.) belongs to an **API group** and a **version**.
* The `apiVersion` field tells Kubernetes **which API group + version** to use when interpreting your object.
* The format is:

```
apiVersion: <group>/<version>
```

* If the resource belongs to the **core group** (like Pods, Services, ConfigMaps), only the version is shown:

```
apiVersion: v1
```

---

## 📌 API Version Types in Kubernetes

1. **Alpha (`v1alpha1`)**

   * Experimental, may contain bugs.
   * Disabled by default in many clusters.
   * **Example**:

     ```yaml
     apiVersion: scalingpolicy.kope.io/v1alpha1
     ```

2. **Beta (`v1beta1`)**

   * Tested, but not yet guaranteed to be stable.
   * Still subject to changes.
   * Enabled by default in most clusters.
   * **Example**:

     ```yaml
     apiVersion: batch/v1beta1
     ```

3. **Stable (`v1`)**

   * Fully supported for production use.
   * Backward compatibility guaranteed.
   * **Example**:

     ```yaml
     apiVersion: apps/v1
     ```

---

## 📌 API Groups

Kubernetes organizes resources into **API groups**.
Some examples:

| API Group                 | Example `apiVersion`           | Common Resources                      |
| ------------------------- | ------------------------------ | ------------------------------------- |
| (core)                    | `v1`                           | Pods, Services, ConfigMaps            |
| apps                      | `apps/v1`                      | Deployments, StatefulSets, DaemonSets |
| batch                     | `batch/v1`                     | Jobs, CronJobs                        |
| autoscaling               | `autoscaling/v1`               | HorizontalPodAutoscaler               |
| networking.k8s.io         | `networking.k8s.io/v1`         | Ingress, NetworkPolicy                |
| rbac.authorization.k8s.io | `rbac.authorization.k8s.io/v1` | Roles, RoleBindings                   |
| policy                    | `policy/v1beta1` (deprecated)  | PodSecurityPolicy                     |

---

## 📌 How to Find Available API Versions in Your Cluster

You can **list all API groups and versions** with:

```bash
kubectl api-versions
```

Or get detailed info about resources:

```bash
kubectl api-resources
```

If you start a **local proxy**:

```bash
kubectl proxy
```

Then visit [http://localhost:8001/apis](http://localhost:8001/apis) → you’ll see a JSON list like the one you pasted.

---

## 📌 Example — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

Here:

* `apiVersion: apps/v1` → Deployment belongs to **apps API group**, stable version.
* `kind: Deployment` → resource type.
* `metadata`, `spec` → object details.
### References
- https://devopscube.com/kubernetes-deployment-tutorial/