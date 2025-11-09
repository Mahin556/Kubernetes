### References:-
- https://devopscube.com/pod-priorityclass-preemption/
- https://medium.com/@muppedaanvesh/a-hands-on-guide-to-kubernetes-qos-classes-%EF%B8%8F-571b5f8f7e58


---
```bash
controlplane:~$ kubectl api-resources | grep priorityclass
priorityclasses                     pc           scheduling.k8s.io/v1              false        PriorityClass

kubectl get pc
kubectl get priorityclass
```
```bash
$ kubectl get pods critical-app -oyaml | grep -E 'preemptionPolicy|priority|priorityClassName'
preemptionPolicy: PreemptLowerPriority
priority: 1000
priorityClassName: high-priority
```
---
### Pod Priority and PriorityClass

* **Pod Priority** determines **which Pod is more important** when resources (like CPU/memory) become scarce on a node or in the cluster.
* It’s a numeric value (integer).
* When the cluster runs out of resources:
  * **Higher-priority Pods** are scheduled first.
  * **Lower-priority Pods** may get **preempted (terminated)** to make room.
So, priority helps Kubernetes decide:
  * Which Pods to **schedule first**, and
  * Which Pods to **evict** during **resource pressure**.

* Priority Admission Controller

<br>

* Priority is **not** directly set in the Pod — instead, it is defined in a **PriorityClass object**.
* A `PriorityClass` is a **cluster-level resource**((non-namespaced)) that defines:
  * The **name** of the class.
  * The **numerical priority value**.
  * Whether it is the **default** for all Pods that don’t specify one.
  * An optional **description**.
* When the cluster lacks resources, lower-priority Pods can be preempted (deleted) to make room for higher-priority Pods.
* Priority is determined by the Priority Admission Controller, which reads the priorityClassName field in the Pod spec and assigns the corresponding priority value to the Pod.

* Example – PriorityClass Definition
  ```yaml
  apiVersion: scheduling.k8s.io/v1
  kind: PriorityClass
  metadata:
    name: high-priority
  value: 1000
  globalDefault: false
  description: "This class is for high priority workloads."
  ```

* **Explanation:**
  * `value`: The priority number (integer).
    * Higher number = higher priority.

  * `globalDefault`:
    * If `true`, Pods without a priority class get this one automatically.

  * `description`: Human-readable explanation.

<br>

* **Using PriorityClass in a Pod**
  * You assign the class name to a Pod via the `priorityClassName` field in its spec.
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: critical-app
    spec:
      priorityClassName: high-priority
      containers:
        - name: busy
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
    ```
* When created, the Pod automatically gets:
  * The **priority value** defined in the `PriorityClass` object.
  * The **scheduler** will use this value when making scheduling or eviction decisions.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: true
description: "This class is for high priority workloads."
---
apiVersion: v1
kind: Namespace
metadata:
  name: logic
---

apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  namespace: logic
spec:
  #priorityClassName: high-priority
  containers:
    - name: busy
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```
```bash
controlplane:~$ kubectl get pod critical-app -n logic -oyaml | grep -E "priority|priorityClassName|preemptionPolicy"
  preemptionPolicy: PreemptLowerPriority
  priority: 1000
  priorityClassName: high-priority
```


<br>

* **Priority Values Range**
  * Kubernetes priority is just an **integer value**, typically between:
    * **0 to 1000000000 (1 billion)**
  * There’s no fixed limit, but higher values mean higher priority.

    | Priority                 | Meaning                                 |
    | ------------------------ | --------------------------------------- |
    | Low (e.g., 0–100)        | Background or batch workloads           |
    | Medium (e.g., 500–1000)  | Normal applications                     |
    | High (e.g., 1000–100000) | Critical system or production workloads |

<br>

* **Pod Preemption – What Happens When Resources Are Full**
  * When a new Pod **cannot be scheduled** due to lack of resources:
      1. The scheduler looks for **nodes** where it *could* run if some lower-priority Pods were removed.
      2. If found:
        * The **lower-priority Pods are preempted** (killed).
        * The **higher-priority Pod is scheduled** in their place.
  * This process is called **Preemption**.
  * The pod preemption feature allows Kubernetes to preempt (evict) lower-priority pods from nodes when higher-priority pods are in the scheduling queue and no node resources are available.

<br>

* **Example of Preemption Scenario**
  * Node has limited CPU/memory.
  * Running Pods:
    * Pod A → Priority 100
    * Pod B → Priority 200
  * New Pod C → Priority 1000

* If no space for Pod C:
  * Kubernetes **evicts Pod A or B** (the lowest-priority one) to make room.
  * Pod C gets scheduled.

<br>


* **How Preemption Works (Internally)**
  1. **Scheduler** tries to schedule a Pod.
  2. If **no suitable node** is found:
   * It looks for **preemption opportunities**.
  3. Scheduler identifies:
     * Which **Pods can be removed** to free resources.
     * Which **node would fit** the pending Pod after preemption.
  4. Scheduler **marks victims** for preemption (lower-priority Pods).
  5. **Preemption occurs**:
     * Victim Pods receive `TerminationGracePeriodSeconds` to shut down.
    * The high-priority Pod is then scheduled.

<br>

* **Preemption and PriorityClass Settings**
  * Each `PriorityClass` can also affect how preemption is handled.
  * There’s a special field in Pod spec:
    ```yaml
    preemptionPolicy: Never
    ```
    * Error if used with globalDefault PriorityClass
  * By default, it is:
    ```yaml
    preemptionPolicy: PreemptLowerPriority
    ```
  **Options:**
    * `PreemptLowerPriority` → Default; allows preemption.
    * `Never` → Pod will **not preempt** lower-priority Pods even if it can’t be scheduled.

<br>

* **Example with `preemptionPolicy: Never`**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: no-preempt-pod
  spec:
    priorityClassName: high-priority
    preemptionPolicy: Never
    containers:
    - name: busy
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
  ```
  * This Pod will **not evict** other Pods to make room — it will simply stay pending until space is available.

<br>

* **System Reserved Priority Classes**
  * Kubernetes has **built-in PriorityClasses** for critical system components:
    * **system-node-critical**: This class has a value of 2000001000. Static pods Pods like etcd, kube-apiserver, kube-scheduler and Controller manager use this priority class.
    * **system-cluster-critical**: This class has a value of 2000000000. Addon Pods like coredns, calico controller, metrics server, etc use this Priority class.
  * These ensure critical system services **never get preempted** by user Pods.


#### **How Preemption Works in Kubernetes (Detailed Process)**

<br>

* **Step 1 — Pod Creation & Admission**
  * When a Pod is created with a `priorityClassName`, the **Priority Admission Controller**:
    * Looks up the `PriorityClass` object.
    * Adds its **numeric priority value** to the Pod spec as an internal field.
    * Example:
      ```yaml
      priorityClassName: high-priority
      # internally resolved as
      priority: 1000
      ```

<br>

* **Step 2 — Scheduling Queue Ordering**
  * The **Kubernetes Scheduler** maintains a **queue** of all pending Pods.
  * It always sorts this queue by **priority (descending order)** — meaning:
    * Higher-priority Pods are scheduled first.
    * Lower-priority Pods wait longer if resources are tight.
  * So if the cluster is under load:
  * The scheduler picks **high-priority Pods first** to place on nodes.

<br>

* **Step 3 — Resource Availability Check**
  * For the selected Pod (say, a high-priority Pod):
    * The scheduler checks **each node** to see if enough CPU, memory, and constraints (taints, affinity, etc.) are satisfied.
  * If **a node has enough resources →** Pod gets scheduled normally.
  * If **no nodes can fit →** preemption logic starts.

<br>

* **Step 4 — Preemption Triggered**
  * When no node can accommodate the Pod, the scheduler activates **preemption**:
    * It scans nodes again and checks:
    * "If I remove one or more **lower-priority Pods**, can I make space for this high-priority Pod?"

<br>

* **Step 5 — Victim Pod Selection**
  * The scheduler identifies potential **victim Pods**:
    * Only **lower-priority Pods** can be preempted.
    * Among them, it picks Pods that, if removed, will:
      * Free enough resources (CPU/memory).
      * Still meet the scheduling constraints of the high-priority Pod (affinity, taints, etc.).
    * Scheduler chooses **minimum necessary victims** to make room efficiently.

<br>

* **Step 6 — Preemption Decision**
  * Once victims are identified:
    * The scheduler **marks** those lower-priority Pods for deletion.
    * It then schedules the high-priority Pod to the selected node.
  * This process is not immediate killing — it follows graceful termination.

<br>

* **Step 7 — Victim Pod Termination**
  * Each **preempted Pod** receives a **grace period**:
    * Default: **30 seconds**
    * Custom: You can override it using
      ```yaml
      terminationGracePeriodSeconds: <seconds>
      ```
  * If a Pod defines **`preStop` lifecycle hooks**, they are run before termination.
  * After the grace period, Kubernetes **forcefully terminates** the Pod if it’s still running.

<br>

* **Step 8 — High-Priority Pod Scheduling**
  * As soon as the low-priority Pods are terminated and the resources free up:
    * The high-priority Pod is **scheduled on the node**.
    * Its containers are created and started normally.

<br>

* **Step 9 — If Preemption Fails**
  * If, even after removing lower-priority Pods, the scheduler still can’t satisfy:
    * Affinity / anti-affinity rules
    * Tolerations
    * Node selectors
    * Resource requests
  * Then:
    * The scheduler **abandons preemption for that attempt**.
    * It may move on and continue scheduling other (even lower-priority) Pods.
  * That’s why preemption is **not guaranteed** — it’s a “best effort” mechanism.

<br>

* **Important Internal Notes**
  * **Preemption is node-local**, not cluster-wide:
    * The scheduler only preempts Pods **on one node** where it wants to place the new Pod.
  * **DaemonSets, static Pods, and mirror Pods** are never preempted.
  * **PodDisruptionBudget (PDB)** can block preemption:
    * If evicting Pods would violate a PDB, the scheduler won’t preempt them.


* **Example Timeline**

| Step | Event                            | Explanation                                          |
| ---- | -------------------------------- | ---------------------------------------------------- |
| 1    | `high-pod` created               | Uses PriorityClass with value 1000                   |
| 2    | Scheduler picks `high-pod`       | Places it at the front of the queue                  |
| 3    | No node fits                     | Preemption triggered                                 |
| 4    | NodeX selected                   | Scheduler decides to remove `low-pod` (priority 100) |
| 5    | `low-pod` marked for termination | Given 30s to shut down gracefully                    |
| 6    | `low-pod` removed                | Resources freed                                      |
| 7    | `high-pod` scheduled             | Takes NodeX resources                                |
| 8    | Cluster stabilizes               | Scheduler continues normal operation                 |

<br>

* **Key YAML Fields Involved**
  * **Pod Spec:**
    ```yaml
    spec:
      priorityClassName: high-priority
      preemptionPolicy: PreemptLowerPriority  # or Never
      terminationGracePeriodSeconds: 10
    ```

  * **PriorityClass:**
    ```yaml
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: high-priority
    value: 1000
    globalDefault: false
    description: "For critical workloads"
    ```


| Concept                           | Description                                                              |
| --------------------------------- | ------------------------------------------------------------------------ |
| **Priority Admission Controller** | Assigns priority values to Pods based on `priorityClassName`             |
| **Scheduler Queue**               | Sorted by priority (high to low)                                         |
| **Preemption Trigger**            | Happens when no node can host a high-priority Pod                        |
| **Victim Pods**                   | Lower-priority Pods that are evicted to make room                        |
| **Grace Period**                  | 30 seconds by default (configurable)                                     |
| **PreemptionPolicy**              | Controls if a Pod can preempt others (`Never` or `PreemptLowerPriority`) |
| **Failure Case**                  | If constraints not met, scheduler skips preemption and moves on          |

![](/images/pod-priorityclass-1.png)


* To list all PriorityClasses:
  ```bash
  kubectl get priorityclasses
  ```

* To see Pod priority and preemption settings:
  ```bash
  kubectl get pod critical-app -n logic -oyaml | grep -E "priority|priorityClassName|preemptionPolicy"
  ```

* To check which Pods were preempted:
  ```bash
  kubectl describe pod <victim-pod-name>
  ```
  You’ll see messages like:
  ```
  Preempted by pod <high-priority-pod>
  ```

* **Example: Priority and Preemption in Action**
  * **Step 1 – Create PriorityClasses**
    ```bash
    cat <<EOF > priority-classes.yaml
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: high-priority
    value: 1000
    globalDefault: false
    description: "Used for critical workloads"

    ---
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: low-priority
    value: 100
    globalDefault: false
    description: "Used for non-critical workloads"
    EOF

    kubectl apply -f priority-classes.yaml
    kubectl get priorityclass

    Output Example:
    NAME             VALUE   GLOBAL-DEFAULT   AGE
    high-priority    1000    false            10s
    low-priority     100     false            10s
    ```

  * **Step 2 – Create Pods**
    ```bash
    cat <<EOF > pod-priority.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: high-priority-pod
    spec:
      priorityClassName: high-priority
      containers:
      - name: high
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "400m"
          limits:
            cpu: "400m"

    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: low-priority-pod
    spec:
      priorityClassName: low-priority
      containers:
      - name: low
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "400m"
          limits:
            cpu: "400m"
    EOF

    kubectl apply -f pod-priority.yaml
    kubectl get pods -o wide
    ```
    * Both Pods will run if the node has capacity.
    * If resources are limited, the high-priority Pod gets scheduled first.

<br>

  * **Step 3 – Observe Behavior**
    * Preemption happens when:
      * A high-priority Pod cannot be scheduled due to resource shortage.
      * The scheduler then removes lower-priority Pods to free up space `kubectl describe pod low-pod`-->`Preempted by Pod high-pod`.
    * This ensures that critical workloads always get the resources they need.

<br>

* **Best Practices**
  * Use **higher priority** for production and system-critical workloads.
  * Use **lower priority** for test, batch, or non-essential jobs.
  * Avoid setting **all Pods to high priority**, or preemption becomes meaningless.
  * Use `preemptionPolicy: Never` for sensitive Pods that must not be terminated automatically.


```bash
cat <<EOF > qos-priority-preemption.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000
globalDefault: false
description: "Critical application priority"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: normal-priority
value: 100
globalDefault: false
description: "Normal workload priority"

---
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: critical-priority
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "300m"
        memory: "256Mi"
      limits:
        cpu: "300m"
        memory: "256Mi"

---
apiVersion: v1
kind: Pod
metadata:
  name: normal-app
spec:
  priorityClassName: normal-priority
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "yes > /dev/null"]
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
EOF

kubectl apply -f qos-priority-preemption.yaml
kubectl get pods -o wide
```

<br>

##### QoS (Quality of Service) vs Priority

| Feature       | QoS (Quality of Service)                             | Priority (Scheduling Importance)             |
| ------------- | ---------------------------------------------------- | -------------------------------------------- |
| Controlled by | `resources.requests` & `limits`                      | `PriorityClass`                              |
| Decides       | Pod eviction order under **resource pressure (OOM)** | Which Pod gets **scheduled first**           |
| Scope         | Per-Pod resource stability                           | Cluster-wide scheduling                      |
| Classes       | Guaranteed, Burstable, BestEffort                    | Custom integer-based (e.g., 100, 1000, etc.) |

<br>

**Example:**
A Guaranteed Pod with low priority may be preempted by a Burstable Pod with higher priority.
QoS affects eviction within the same priority, while priority affects scheduling order globally.



**Inspect Priority and QoS Together**
  ```bash
  kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priorityClassName,QOS:.status.qosClass
  ```

---


### DaemonSet Priority?
  * **DaemonSets** behave just like normal Pods when it comes to priority.
  * Every Pod created by a DaemonSet **inherits the PriorityClass** if you define one in its Pod template.
  * During resource shortages on a node, **low-priority DaemonSet Pods can also be evicted** — unless you explicitly give them a **higher PriorityClass**.

* For important DaemonSets like:
  * `node-exporter` (monitoring)
  * `fluentd` or `filebeat` (logging)
  * `kube-proxy`
  * `csi-node` plugins
    → assign a **high-priority class**, e.g. `system-node-critical` or a custom one.

  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: node-logger
  spec:
    template:
      spec:
        priorityClassName: high-priority
        containers:
          - name: logger
            image: fluentd
  ```

* This ensures your logging DaemonSet Pods **stay stable** during node resource pressure.

---

### **What is the Significance of Pod Priority?**

Pod Priority lets you control **which workloads are protected** and **which can be sacrificed** when resources are tight.

Example use cases:

* 🟢 **Critical system workloads:** `metrics-server`, `coredns`, `kube-proxy`
* 🟢 **Important business apps:** `payment-service`, `api-gateway`, `logging agents`
* 🔵 **Normal workloads:** frontend, test services
* ⚪ **Low-priority/batch jobs:** reports, analytics, etc.

During cluster overload:

* Kubelet **evicts lower-priority Pods**.
* Scheduler ensures **critical workloads** always get scheduled first.

✅ **You can design a hierarchy like:**

| Tier             | Example Apps                    | Priority Value |
| ---------------- | ------------------------------- | -------------- |
| Mission-critical | Payment API, logging DaemonSets | 1000+          |
| Core services    | Frontend, user API              | 500            |
| Batch/analytics  | Reports, test apps              | 100            |

This ensures **important apps stay alive**, even under heavy load or resource competition.

---


### **Why Should DevOps Engineers Understand Pod Priority?**

Because it’s crucial for:

* Designing **resilient production clusters**
* Managing **resource starvation scenarios**
* Ensuring **critical services stay available**
* Optimizing **multi-tenant clusters**

🧠 **In Kubernetes certifications (CKA/CKS/CKAD)**:

* You must understand how `PriorityClass`, `preemptionPolicy`, and `PodDisruptionBudget` interact.
* Preemption behavior and YAML creation questions are frequently asked.

---

### **6. Summary — Key Takeaways**

| Concept                | Description                          | Notes                             |
| ---------------------- | ------------------------------------ | --------------------------------- |
| **PriorityClass**      | Defines numerical priority           | Cluster-scoped object             |
| **priorityClassName**  | Used in Pod spec                     | Refers to PriorityClass           |
| **Preemption**         | Evicts low-priority Pods             | Triggered by Scheduler            |
| **QoS Eviction**       | Node-level eviction                  | Triggered by Kubelet              |
| **DaemonSet Priority** | Set explicitly                       | Important for logging, monitoring |
| **PDB Effect**         | Scheduler respects, but can override | For very high-priority Pods       |

---

### ✅ **Example: Protecting Critical Workloads**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-services
value: 1000
description: "Priority for core infrastructure and monitoring pods"

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector:
    matchLabels:
      app: logger
  template:
    metadata:
      labels:
        app: logger
    spec:
      priorityClassName: critical-services
      containers:
        - name: fluentd
          image: fluentd
```

This ensures that your logging DaemonSet Pods **are never preempted or evicted** before other lower-priority Pods.


### Preemption Policy
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-apps
value: 1000000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Mission Critical apps."
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: dev
spec:
  containers:
  - name: web
    image: nginx:latest
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority-apps
```
* Preemption means evicting lower-priority Pods to make room for higher-priority Pods when cluster resources (CPU, memory, etc.) are scarce.
* It evicts them (gracefully), freeing up resources for the important one.
* Part of K8S scheduling process
* Default `preemptionPolicy` == `PreemptLowerPriority`

* **`preemptionPolicy==PreemptLowerPriority`
  * Pods with this priority can preempt (evict) other Pods with lower priority values.
  * The scheduler tries to minimize disruption, but it will remove lower-priority Pods if necessary.

  ```yaml
  apiVersion: scheduling.k8s.io/v1
  kind: PriorityClass
  metadata:
    name: high-priority
  value: 1000
  globalDefault: false
  description: "This class preempts lower-priority pods."
  preemptionPolicy: PreemptLowerPriority
  ```
  * Here, if a Pod with this class (priorityClassName: high-priority) cannot find resources, it may evict other lower-priority Pods (like one with value 500 or no priority).

* **`PreemptionPolicy: Never`**
  * Disables preemption for that priority class.
  * Pods using this PriorityClass cannot evict (preempt) other pods, even if their priority value is higher.
  * They will not remove or replace lower-priority pods that are already running.
  ```yaml
  apiVersion: scheduling.k8s.io/v1
  kind: PriorityClass
  metadata:
    name: safe-priority
  value: 1000
  globalDefault: false
  preemptionPolicy: Never
  description: "High priority class that will not preempt other pods."
  ```
  * If the cluster doesn’t have enough resources, these pods will remain in the “Pending” state until resources are freed naturally (for example, if another pod is deleted or finishes).
  * However, other higher-priority pods (without PreemptionPolicy: Never) can evict this pod if they have a higher priority and need resources.
  * `PreemptionPolicy: Never` protects other pods from being preempted by this one, but it does not protect this pod from being preempted by higher-priority pods.
  * We use PreemptionPolicy: Never when we want a pod to be high-priority for scheduling order but non-evicting — it waits for resources instead of forcing them.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: aggressive-priority
value: 1000
preemptionPolicy: PreemptLowerPriority
description: "Can evict lower priority pods if needed."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gentle-priority
value: 900
preemptionPolicy: Never
description: "Will not evict any pods."
```
```yaml
#low-priority Pods to fill up the node
apiVersion: v1
kind: Pod
metadata:
  name: low-pod
spec:
  priorityClassName: ""
  containers:
  - name: low
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "200m"
        memory: "100Mi"
---
#high-priority Pod with preemption enabled
apiVersion: v1
kind: Pod
metadata:
  name: preempting-pod
spec:
  priorityClassName: aggressive-priority
  containers:
  - name: high
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "500m"
        memory: "200Mi"
---
#high-priority Pod with preemption disabled
apiVersion: v1
kind: Pod
metadata:
  name: non-preempting-pod
spec:
  priorityClassName: gentle-priority
  containers:
  - name: gentle
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "500m"
        memory: "200Mi"
```
