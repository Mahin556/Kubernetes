# 🔑 What is `spec`?

* `spec` = **desired state** of the resource.
* Kubernetes **continuously reconciles** actual state → desired state.
* In a Deployment, it defines **how many Pods, how to select them, and what they should run**.

---

# 📌 Key Subfields of `spec`

### 1. **replicas**

* Number of **Pod copies** you want at all times.
* Deployment ensures that many Pods are running (self-healing).

✅ Example:

```yaml
spec:
  replicas: 3
```

---

### 2. **selector**

* Defines which Pods this Deployment manages.
* Must match the **labels in the Pod template**.
* Without this, Deployment won’t know which Pods belong to it.

✅ Example:

```yaml
spec:
  selector:
    matchLabels:
      app: nginx
```

---

### 3. **template**

* Describes the **Pod specification**.
* Includes **Pod metadata** (labels, annotations) and **Pod spec** (containers, volumes, etc.).
* Every Pod created by the Deployment is based on this template.

✅ Example:

```yaml
spec:
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

---

# 📌 Putting it Together (Full Example)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
### References
- https://devopscube.com/kubernetes-deployment-tutorial/