### **1. What is `kubectl apply`?**

* `kubectl apply` is used to **create, update, or delete Kubernetes objects declaratively**.
* It is **smart**: it can create an object if it doesn’t exist, or update only the fields that have changed if it already exists.

**Basic usage:**

```bash
kubectl apply -f object-definition.yaml
```

---

### **2. How `kubectl apply` Works**

1. **It considers three things before making changes**:

   * **Local object configuration**: the YAML/JSON file you define.
   * **Live object configuration**: the current state of the object in the Kubernetes cluster.
   * **Last applied configuration**: stored metadata of the previous `kubectl apply`.

2. **Behavior:**

   * If the object **does not exist**, it is created.
   * If the object **already exists**, `kubectl apply` calculates the differences between the local file, the live object, and the last applied configuration, and updates only the necessary fields.

---

### **3. Example: Creating a Pod**

**Local object file (`local-object-definition.yml`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
   name: httpd
   labels:
      type: web-server
      app: my-app
spec:
   containers:
   - name: nginx-web-server
     image: nginx:1.17
```

**Command:**

```bash
kubectl apply -f local-object-definition.yml
```

**Live object configuration after creation:**

```yaml
apiVersion: v1
kind: Pod
metadata:
   name: httpd
   labels:
      type: web-server
      app: my-app
spec:
   containers:
   - name: nginx-web-server
     image: nginx:1.17
status:
  conditions:
  - lastProbeTime: null
    status: "True"
    type: Initialized
```

---

### **4. Last Applied Configuration**

* When `kubectl apply` creates an object, it stores a **JSON version of the local YAML** as the “last applied configuration”.
* This is used in **future updates** to track changes.

**Example:**

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "httpd",
    "labels": {
      "type": "web-server",
      "app": "my-app"
    }
  },
  "spec": {
    "containers": [
      {
        "name": "nginx-web-server",
        "image": "nginx:1.17"
      }
    ]
  }
}
```

---

### **5. Updating an Object**

1. Update your local YAML file:

```yaml
containers:
  - name: nginx-web-server
    image: nginx:1.18   # updated version
```

2. Apply the update:

```bash
kubectl apply -f local-object-definition.yml
```

**What happens internally:**

* Kubernetes compares:

  * Local configuration
  * Live object
  * Last applied configuration
* Only changes are applied (e.g., updates image version to `1.18`).
* Last applied configuration is updated with the new values.

---

### **6. Why Last Applied Configuration is Important**

* **Detects removed fields**:

  * If a field existed previously (in last applied) but is **missing in the local YAML**, it is **removed from the live object**.
* Fields **present in live but not in last applied or local YAML** are **left untouched**.
* Ensures declarative management: your local YAML truly reflects the desired state.

---

### **7. Where is Last Applied Configuration Stored?**

* Stored as an **annotation in the live object configuration**:

```yaml
apiVersion: v1
kind: Pod
metadata:
   name: httpd
   labels:
      type : web-server
      app: my-app
   annotations:
   kubectl.kubernetes.io/last-applied-configuration:
   {"apiVersion": "v1","kind": "Pod","metadata": {"name": "httpd", "labels": {"type": "web-server","app": "my-app"}},"spec": {"containers": [{"name": "nginx-web-server","image": "nginx:1.17"
}]}}
spec:
   containers:
   - name: nginx-web-server
     image: nginx:1.17
status:
  conditions:
  - lastProbeTime: null
    status: "True"
    type: Initialized
```

* **Note:** `kubectl create` or `kubectl replace` **do not store** last applied configuration.

---

### **8. Summary of Key Points**

* `kubectl apply` is **declarative** and smart in updating only what’s needed.
* Uses **three sources**: local YAML, live configuration, last applied configuration.
* Last applied configuration ensures:

  * Removed fields are deleted from live objects.
  * Updates are applied efficiently without overwriting unchanged fields.
* Stored as an annotation in the live object.
* Ideal for iterative updates and managing production workloads safely.


### References:
- https://technos.medium.com/how-kubectl-apply-command-works-d092121056d3