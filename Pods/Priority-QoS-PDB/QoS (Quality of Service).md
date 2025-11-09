### References:-
- https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/

---

### **QoS (Quality of Service)**
  * Kubernetes uses **Quality of Service (QoS) classes** to **prioritize Pods** during **resource contention** (like CPU/memory pressure on a node).
  * QoS is **not manually configured** — it’s **inferred automatically**.
  * Each Pod is assigned a **QoS class automatically** based on how **CPU and memory resource requests and limits** are set.
  * This helps the kubelet decide **which Pods to evict first** when a node is under resource pressure.

  * When the node runs out of memory or CPU:
    * **BestEffort Pods** are killed **first**.
    * **Burstable Pods** may be killed **next**.
    * **Guaranteed Pods** are killed **last** (highest priority).
  * So QoS ensures **important workloads keep running** even under stress.

<br>

* You can check a Pod’s QoS with:
  ```bash
  kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
  ```

<br>

* **QoS Classes in Kubernetes**

    | QoS Class      | Definition                                                                                                      | Characteristics                                                       | Priority   |
    | -------------- | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ---------- |
    | **Guaranteed** | Every container in the Pod has **equal CPU & memory `requests` and `limits`**, and both are **set explicitly**. | Highest reliability. Never evicted unless node is out of resources.   | 🥇 Highest |
    | **Burstable**  | At least one container has a `request`, or not all containers have both set or request < limit.                          | Moderate reliability. Evicted after BestEffort but before Guaranteed. | 🥈 Medium  |
    | **BestEffort** | No `requests` or `limits` set for **any** container.                                                            | Lowest reliability. Evicted first during pressure.                    | 🥉 Lowest  |

<br>

* **Guaranteed QoS Example**
* Create a Pod where `requests` = `limits` for both CPU and memory.
    ```bash
    cat <<EOF > qos-guaranteed.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-guaranteed
    spec:
      containers:
      - name: guaranteed-container
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            memory: "200Mi"
            cpu: "200m"
          limits:
            memory: "200Mi"
            cpu: "200m"
    EOF

    kubectl apply -f qos-guaranteed.yaml
    kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}'
    kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}{"\n"}'
    kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}';echo
    ```

    ```
    Guaranteed
    ```
* For a Pod to be given a QoS class of Guaranteed:
    * Every Container in the Pod must have a memory limit and a memory request.
    * For every Container in the Pod, the memory limit must equal the memory request.
    * Every Container in the Pod must have a CPU limit and a CPU request.
    * For every Container in the Pod, the CPU limit must equal the CPU request.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "100m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "100m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "100m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
```

<br>

* **Burstable QoS Example**
* Create a Pod with `requests` less than `limits`.
    ```bash
    cat <<EOF > qos-burstable.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-burstable
    spec:
      containers:
      - name: burstable-container
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "200Mi"
            cpu: "400m"
    EOF

    kubectl apply -f qos-burstable.yaml
    kubectl get pod qos-burstable -o jsonpath='{.status.qosClass}'
    ```
    ```
    Burstable
    ```
* A Pod is given a QoS class of Burstable if:
    * The Pod does not meet the criteria for QoS class Guaranteed.
    * At least one Container in the Pod has a memory or CPU request or limit.

* At least one container has unequal requests and limits → **QoS = Burstable**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod-2
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "100m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "400m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
```

<br>

* **BestEffort QoS Example**
* No container in the Pod defines any requests or limits.
    ```bash
    cat <<EOF > qos-besteffort.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-besteffort
    spec:
      containers:
    - name: besteffort-container
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
    EOF

    kubectl apply -f qos-besteffort.yaml
    kubectl get pod qos-besteffort -o jsonpath='{.status.qosClass}';echo
    kubectl get pod qos-besteffort -o jsonpath='{.status.qosClass}{"\n"}'
    ```
    ```
    BestEffort
    ```
* No resource specs defined → **QoS = BestEffort**
* Even if limits are defined for ephemeral storage or GPUs only, but not CPU/memory, it’s still BestEffort because CPU/memory are not specified.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: burstable-container1
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
  - name: burstable-container2
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

<br>

* **How Kubernetes Uses QoS**
  * **During Scheduling:**
    * QoS affects **resource reservation** and **Pod placement**.
  * **During Eviction:**
    * If the node runs out of memory:
      * `BestEffort` → killed first
      * `Burstable` → next
      * `Guaranteed` → last
  * **During Resource Allocation:**
    * CPU `requests` define the **minimum guaranteed CPU time**.
    * Memory `limits` define the **maximum allowed memory** (beyond which container OOMs).


---
### Exercise

* **Step 1 — Create Guaranteed Pod**
    File: `guaranteed-pod.yaml`
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: guaranteed-pod
    spec:
      containers:
      - name: nginx-container
        image: nginx
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "100m"
    ```
    Apply it:
    ```bash
    kubectl apply -f guaranteed-pod.yaml
    ```
    Check status:
    ```bash
    kubectl get pods -o wide
    ```
    Check QoS class:
    ```bash
    kubectl get pod guaranteed-pod -o jsonpath='{.status.qosClass}'
    ```
    **Expected output:**
    ```
    Guaranteed
    ```

* **Step 2 — Create Burstable Pod**
    File: `burstable-pod.yaml`
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: burstable-pod
    spec:
      containers:
      - name: nginx-container
        image: nginx
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
    ```
    Apply it:
    ```bash
    kubectl apply -f burstable-pod.yaml
    ```
    Check QoS:
    ```bash
    kubectl get pod burstable-pod -o jsonpath='{.status.qosClass}'
    ```
    **Expected output:**
    ```
    Burstable
    ```

* **Step 3 — Create BestEffort Pod**
    File: `besteffort-pod.yaml`
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: besteffort-pod
    spec:
      containers:
      - name: nginx-container
        image: nginx
    ```
    Apply it:
    ```bash
    kubectl apply -f besteffort-pod.yaml
    ```
    Check QoS:
    ```bash
    kubectl get pod besteffort-pod -o jsonpath='{.status.qosClass}'
    ```
    **Expected output:**
    ```
    BestEffort
    ```

* **Step 4 — Confirm All Pods Are Running**
    ```bash
    kubectl get pods
    ```
    **Expected output:**
    ```
    NAME              READY   STATUS    RESTARTS   AGE
    guaranteed-pod    1/1     Running   0          1m
    burstable-pod     1/1     Running   0          1m
    besteffort-pod    1/1     Running   0          1m
    ```

* **View All QoS Classes Together**
    ```bash
    kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
    ```
    ```
    NAME              QOS
    qos-guaranteed    Guaranteed
    qos-burstable     Burstable
    qos-besteffort    BestEffort
    ```
  
  <br>

* **Step 5 — Simulate Resource Pressure**
    
  * **Option A** — Run stress manually on the node
    If you can SSH to the worker node:
    ```bash
    sudo apt-get update && sudo apt-get install -y stress-ng
    sudo stress-ng --vm 1 --vm-bytes 90% --timeout 120s
    ```
    This will cause **memory pressure**, and kubelet will start evicting Pods.

  * **Option B** — Create a CPU/MEM stress Deployment (safer for labs)
    File: `high-cpu-utilization.yaml`
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: high-cpu-utilization-deployment
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: cpu-utilization-app
    template:
        metadata:
        labels:
            app: cpu-utilization-app
        spec:
        containers:
        - name: cpu-utilization-container
            image: ubuntu
            command: ["/bin/sh", "-c", "apt-get update && apt-get install -y stress-ng && while true; do stress-ng --vm 1 --vm-bytes 80% --timeout 600; done"]
            resources:
            requests:
                cpu: "1"
            limits:
                cpu: "2"
    ```
    Apply it:
    ```bash
    kubectl apply -f high-cpu-utilization.yaml
    ```

* **Step 6 — Watch Pods for Eviction Events**
    Keep watching Pods in another terminal:
    ```bash
    kubectl get pods -w
    ```
    **Expected sequence:**
    ```
    NAME              READY   STATUS    RESTARTS   AGE
    besteffort-pod    0/1     Evicted   0          1m
    burstable-pod     1/1     Running   0          1m
    guaranteed-pod    1/1     Running   0          1m
    ```
    After a while (if stress continues):
    ```
    burstable-pod     0/1     Evicted   0          2m
    guaranteed-pod    1/1     Running   0          2m
    ```

* **Step 7 — Observe Eviction Events**
    You can see which Pods got evicted and why:
    ```bash
    kubectl get events --sort-by=.metadata.creationTimestamp
    ```
    Example output:
    ```
    Normal  Evicted  pod/besteffort-pod  The node had memory pressure.
    Normal  Evicted  pod/burstable-pod   The node had memory pressure.
    ```


---
