# **1. Core (Built-in) Object Types**

### **a) Pods**

* Smallest deployable unit.
* Encapsulates one or more containers, storage, and networking.
* Example: `nginx-pod.yaml`

### **b) Services**

* Exposes Pods to the network.
* Types:

  * **ClusterIP** – internal only
  * **NodePort** – external access via node port
  * **LoadBalancer** – cloud LB
  * **ExternalName** – DNS alias
* Example: `nginx-service.yaml`

### **c) ConfigMaps**

* Stores non-confidential configuration data.
* Can be mounted as **environment variables** or **volumes**.

### **d) Secrets**

* Stores sensitive data (passwords, tokens, keys).
* Similar usage as ConfigMaps but **base64-encoded**.

### **e) Volumes**

* Abstracts persistent storage.
* Types: `emptyDir`, `hostPath`, `persistentVolumeClaim` etc.

---

# **2. Controllers (Workload Object Types)**

Controllers manage Pods and ensure the **desired state** matches the **actual state**.

### **a) ReplicaSet**

* Ensures a specified number of **replicas** of a Pod are running.
* Example: `replicas: 3` for nginx Pods.

### **b) Deployment**

* Declarative updates for Pods + ReplicaSets.
* Supports **rolling updates**, **rollback**, and **scaling**.

### **c) StatefulSet**

* For **stateful applications** (databases, queues).
* Maintains **stable Pod identity** and **persistent storage**.

### **d) DaemonSet**

* Runs one Pod on **every node** (or subset via labels).
* Common for logging, monitoring agents.

### **e) Job**

* Runs **Pods to completion** (batch jobs).
* Ensures Pods complete **successfully once or N times**.

### **f) CronJob**

* Runs Jobs on a **scheduled basis** (like cron in Linux).

---

# **3. Namespace & Policy Objects**

### **a) Namespace**

* Logical cluster partitioning.
* Example: `dev`, `qa`, `prod` namespaces.

### **b) ResourceQuota**

* Limits resources in a namespace (CPU, memory, number of Pods).

### **c) LimitRange**

* Sets **min/max resource limits** for Pods/containers in a namespace.

---

# **4. Networking & Ingress Objects**

### **a) NetworkPolicy**

* Controls **Pod-to-Pod traffic** using ingress/egress rules.

### **b) Ingress**

* Exposes HTTP/HTTPS routes to services from outside the cluster.

---

# **5. Storage Objects**

### **a) PersistentVolume (PV)**

* Cluster-wide storage resource (physical or cloud).

### **b) PersistentVolumeClaim (PVC)**

* Request for storage by a Pod.
* PVCs bind to available PVs.

---

# **6. Custom Objects (CRDs)**

### **CustomResourceDefinition (CRD)**

* Allows you to **create your own object types**.
* Example: `Prometheus`, `Istio VirtualService`
* Managed like built-in objects via controllers.

---

# **7. Quick Reference Table**

| Object Type              | Purpose                   | Example Usage  |
| ------------------------ | ------------------------- | -------------- |
| Pod                      | Run containers            | nginx-pod      |
| Service                  | Expose Pods               | nginx-svc      |
| ConfigMap                | Non-confidential config   | app-config     |
| Secret                   | Sensitive data            | db-password    |
| Deployment               | Manage Pods declaratively | nginx-deploy   |
| ReplicaSet               | Ensure N replicas         | nginx-rs       |
| StatefulSet              | Stateful apps             | mysql-stateful |
| DaemonSet                | One Pod per node          | node-exporter  |
| Job                      | Run pods to completion    | backup-job     |
| CronJob                  | Scheduled Jobs            | cleanup-cron   |
| PersistentVolume         | Storage resource          | pv1            |
| PersistentVolumeClaim    | Request storage           | pvc1           |
| NetworkPolicy            | Control traffic           | allow-frontend |
| Ingress                  | HTTP routing              | app-ingress    |
| CustomResourceDefinition | Create custom objects     | prometheus-cr  |
