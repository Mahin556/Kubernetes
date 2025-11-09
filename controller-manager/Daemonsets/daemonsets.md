### References:- 
- https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- https://devopscube.com/kubernetes-daemonset/
- https://spacelift.io/blog/kubernetes-cheat-sheet
- https://youtu.be/kvITrySpy_k
- https://spacelift.io/blog/kubernetes-daemonset *


---
```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset
kubectl get daemonset -A
kubectl describe daemonset <daemonset_name>
kubectl delete daemonset <daemonset_name>
kubectl edit daemonset <daemonset_name>
kubectl describe ds <daemonset_name> -n <ns>
kubectl rollout restart daemonset/<daemonset_name>
kubectl rollout status daemonset/<daemonset_name>
```
---

* A **DaemonSet** is a Kubernetes workload object that ensures a copy of a specific Pod runs on **all eligible (or some) nodes** in a cluster.
* It is commonly used for **cluster-wide services** that need to run on every node, such as logging agents, monitoring agents, or networking daemons.
* We use it when we need to run specific application on all nodes.
* Native K8S Object
* Can't not scale only run one pod on a node.
* If the daemonset pod gets deleted from the node, the daemonset controller creates it again.
* If there are 500 worker nodes and you deploy a daemonset, the daemonset controller will run one pod per worker node by default. That is a total of 500 pods. However, using nodeSelector, nodeAffinity, Taints, and Tolerations, you can restrict the daemonset to run on specific nodes.
* For example, in a cluster of 100 worker nodes, one might have 20 worker nodes labeled GPU enabled to run batch workloads. And you should run a pod on those 20 worker nodes. In this case, you can deploy the pod as a Daemonset using a node selector.

![](/images/image-4-44.png)
![image](https://github.com/piyushsachdeva/CKA-2024/assets/40286378/bb803dc2-f9ab-4fe3-a0bb-0eacdfcf3ce0)

* Control plane component kube-proxy and CNI is run as daemonset.
* DaemonSets automatically add a Pod to new nodes and remove it from deleted nodes.
* Scheduling is done by the DaemonSet controller, not the default Kubernetes scheduler.
* Node addition/removal:
  - New nodes → Pod automatically added.
  - Removed nodes → Pod automatically deleted.
* Node labels updated → Pods are added/removed based on label matching.
* Updating a DaemonSet
  - You can modify the Pod template in the DaemonSet.
  - Limitations:
      - Some fields in existing Pods cannot be updated.
      - The DaemonSet controller applies the original template when new nodes are added.

* Deleting a DaemonSet:
  - If a DaemonSet is deleted, the Pods it created are automatically cleaned up.
  - `--cascade=orphan` → Leaves Pods running on nodes.
  - Re-creating a DaemonSet with the same selector can adopt existing Pods.

```yaml
apiVersion: apps/v1
kind:  DaemonSet
metadata:
  name: nginx-ds
  labels:
    env: demo
spec:
  template:
    metadata:
      labels:
        env: demo
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
  selector:
    matchLabels:
      env: demo
```

#### Common Use Cases

Why use DaemonSets?
Useful when you need a Pod running on every node. Common use cases:
  - Logging agents: Run on every node to collect logs for central logging system Eg: Splunk, Humio, Fluentd, logstash, fluentbit etc.
  - Monitoring agents: Deploy monitoring agents, such as Prometheus Node Exporter, on every node in the cluster to collect and expose node-level metrics. This way prometheus gets all the required worker node metrics.
  - Kubernetes system components: kube-proxy.
  - Network Management: Running a network plugin or firewall on every node to ensure consistent network policy enforcement. For example, Flannel, Calico, or other CNI plugins runs as a Daemonset on all the nodes.
  - Security and Compliance: Running CIS Benchmarks on every node using tools like kube-bench. Also deploy security agents, such as intrusion detection systems or vulnerability scanners, on specific nodes that require additional security measures. For example, nodes that handle PCI, and PII-compliant data.
  - Storage Provisioning: Running a storage plugin on every node to provide a shared storage system to the entire cluster.
  - Run GPU drivers only on nodes with GPUs.
Guarantees coverage across all nodes, which ReplicaSets cannot guarantee.


#### Example DaemonSet Manifest

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: log-agent
  template:
    metadata:
      labels:
        name: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
        resources:
          limits:
            memory: "200Mi"
            cpu: "200m"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
* In this example, `fluentd` is deployed on every node to collect logs from `/var/log`.

#### Updating a DaemonSet
* By default, Pods are replaced **one at a time** (rolling update).
* DaemonSets support **two update strategies** under `.spec.updateStrategy`:
  * **RollingUpdate** (default): Replace Pods gradually.
  * **OnDelete**: New Pods are created only after you manually delete the old ones.

* **OnDelete**
* DaemonSet controller does **not** automatically replace Pods after a template change.
* You must **manually delete Pods** → only then the controller creates new ones.
* Best for **critical system components** (e.g., networking plugins) where you want **full control** over restarts.
```bash
kubectl create namespace logging
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-ondelete
  namespace: logging
spec:
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 200m
            memory: 200Mi
```
* Get image
  ```bash
  kubectl describe daemonset fluentd-ondelete -n logging | grep -i image
  Image:      quay.io/fluentd_elasticsearch/fluentd:v2.5.2

  kubectl describe pods fluentd-ondelete-b2j5l -n logging | grep -m 1 Image
  Image:          quay.io/fluentd_elasticsearch/fluentd:v2.5.2
  ```

* update image
  ```bash
  kubectl set image daemonset fluentd-ondelete fluentd=ewok/fluentd:v2.5.1 -n logging

  kubectl describe daemonset fluentd-ondelete -n logging | grep -i image
  Image:      ewok/fluentd:v2.5.1

  kubectl describe pods fluentd-ondelete-b2j5l -n logging | grep -m 1 Image
  Image:          quay.io/fluentd_elasticsearch/fluentd:v2.5.2
  ```

* If you change the image version here, Pods **stay old** until you delete them:
  ```bash
  kubectl delete pod -l app=fluentd -n logging

  kubectl describe pods fluentd-ondelete-p5pqz -n logging | grep -m 1 Image
  Image:          ewok/fluentd:v2.5.1
  ```
* DaemonSet controller then replaces them with new Pods.

---

* **RollingUpdate Strategy (default)**
* When you update the DaemonSet Pod spec, old Pods are automatically killed and replaced with new ones.
* Controlled by `rollingUpdate.maxUnavailable`:
    * `maxUnavailable: 1` → ensures that only 1 Pod is unavailable at a time.
    * You can use absolute numbers (e.g., `2`) or percentages (e.g., `20%`).
* Ensures **gradual replacement** across nodes.
```bash
kubectl create namespace logging
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-rolling
  namespace: logging
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # can be "1" or "20%"
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            memory: 200Mi
```
```bash
kubectl set image daemonset fluentd-rolling fluentd=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n logging

kubectl rollout status daemonset fluentd-rolling -n logging

kubectl describe pod fluentd-rolling-lb926 -n logging | grep -m 1 Image
Image:          quay.io/fluentd_elasticsearch/fluentd:v2.6.0

kubectl rollout undo daemonset fluentd-rolling -n logging #If something goes wrong with an update

kubectl describe pod fluentd-rolling-wz576 -n logging | grep -m 1 Image
Image:          quay.io/fluentd_elasticsearch/fluentd:v2.5.2

kubectl rollout undo daemonset fluentd-rolling -n logging --to-revision=2

kubectl delete daemonset fluentd-rolling -n logging

kubectl delete daemonset fluentd-rolling -n logging --cascade=false #Keep Pods running but remove DaemonSet controller (orphan Pods)
#This is useful if you want Pods to keep running independently after deleting the DaemonSet.

kubectl delete daemonset fluentd-rolling -n logging --cascade=false 
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
daemonset.apps "fluentd-rolling" deleted from logging namespace
```

* **Different Ways to Trigger a RollingUpdate in DaemonSets**
Rolling updates happen **whenever the Pod template changes**.
Here are the common ways:

1. **Update the image**
```bash
kubectl set image daemonset fluentd-rolling fluentd=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n logging
```
This replaces Pods gradually (following `maxUnavailable`).

2. **Patch the DaemonSet**
Apply a patch to change spec fields (like image, env, args):
```bash
kubectl patch daemonset fluentd-rolling -n logging \
  -p '{"spec": {"template": {"spec": {"containers": [{"name": "fluentd","image":"quay.io/fluentd_elasticsearch/fluentd:v2.6.1"}]}}}}'
```

3. **Edit the DaemonSet YAML**
```bash
kubectl edit daemonset fluentd-rolling -n logging
```
Modify the Pod spec (image, env, resource limits). The controller starts rolling out updates automatically.

4. **Apply a new manifest**
If you have a YAML with changes:
```bash
kubectl apply -f fluentd-rolling.yaml
```

5. **Force a rolling restart (without changing spec)**
Sometimes you want to restart Pods to pick up ConfigMap/Secret changes without changing the container image. You can do this with an **annotation trick**:
```bash
kubectl patch daemonset fluentd-rolling -n logging \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"kubectl.kubernetes.io/restartedAt\":\"$(date +%Y-%m-%dT%H:%M:%S%z)\"}}}}}"

kubectl rollout restart fluentd-rolling -n logging
```
This updates the Pod template’s annotation → Kubernetes treats it as a spec change → triggers a rolling restart.


* Use **RollingUpdate** for most DaemonSets (monitoring, logging).
* Use **OnDelete** for sensitive system-level DaemonSets (CNI plugins).
* Always check rollout progress (`kubectl rollout status`).
* Keep **resource limits** defined to avoid node pressure during rollout.
* Use `maxUnavailable=0` for **zero downtime upgrades** (but slower rollout).

#### Taint-toleration
```bash
kubectl taint nodes k8s-worker-2 app=fluentd-logging:NoExecute
kubectl describe node k8s-worker-2 | grep Taints
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "fluentd-logging"
        effect: "NoExecute"
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
```
```bash
kubectl apply -f fluentd-daemonset.yaml
kubectl get pods -o wide -n kube-system -l app=fluentd
```
* This allows **Fluentd pods** to still run on `k8s-worker-2` despite the taint.
* If you **remove this toleration**, the DaemonSet pod running on `k8s-worker-2` will be evicted (as you noticed).

#### nodeSelector
* Normally, a DaemonSet schedules one Pod per node across the entire cluster. But sometimes, you don’t want it on *every node*, only on a subset of nodes (e.g., logging agents only on worker nodes). That’s where **nodeSelector** (or affinities/taints) comes in.
```bash
kubectl label node k8s-worker-1 type=platform-tools
kubectl label node k8s-worker-2 type=platform-tools
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-monitoring-agent
  labels:
    app: logging
spec:
  selector:
    matchLabels:
      app: logging
  template:
    metadata:
      labels:
        app: logging
    spec:
      nodeSelector:
        type: platform-tools  # Ensures it runs on specific nodes with the label "platform-tools"
      containers:
        - name: splunk-monitoring-agent
          image: splunk:latest
          ports:
            - containerPort: 8088  # Assuming the agent exposes a port
          volumeMounts:
            - name: splunk-config
              mountPath: /etc/splunk  # Specify where the config should be mounted in the container
      volumes:
        - name: splunk-config
          configMap:
            name: splunk-config-map  # Ensure you have a ConfigMap named splunk-config-map
```
* The controller looks at all nodes.
* Only nodes that match the selector get a Pod.
* Nodes without that label are skipped (DaemonSet won’t schedule Pods there).
* If you add a new node with that label, DaemonSet will automatically create a Pod on it.

#### Affinity
* `nodeSelector`: Simple equality-based matching (`key=value`). Very limited.
* `nodeAffinity`: More expressive, supports operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`)

* `requiredDuringSchedulingIgnoredDuringExecution`: **Hard rule** – if no node matches, the Pod won’t be scheduled.
* `preferredDuringSchedulingIgnoredDuringExecution`: **Soft rule** – scheduler tries to place Pods there, but falls back if unavailable.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: type
          operator: In
          values:
          - platform-tools
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: instance-type
          operator: In
          values:
          - t2.large
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values:
                - platform-tools
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: instance-type
                operator: In
                values:
                - t2.large
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
* You can combine **affinity** + **tolerations** in DaemonSets.


### DaemonSet vs Deployment
* **Deployment**: Scales Pods arbitrarily across nodes, focuses on stateless apps.
* **DaemonSet**: Ensures **one Pod per node**, ideal for infrastructure-level services.
* **Deployment + HPA**: Used for scalable workloads (e.g., web apps).
* **DaemonSet**: Used for node-level background tasks.



#### Multiple DaemonSets for One Daemon
  * Sometimes, you need the *same kind daemon* (e.g., Fluentd or a monitoring agent) but with **different configurations** depending on hardware, node pool, or workload type.
  * Instead of trying to overload a single DaemonSet with complex logic, you can define **separate DaemonSets**, each scoped to the right nodes.

* **Use Cases**
  1. **Different Resource Requirements**
     * Example: Nodes with GPUs may need higher memory/CPU allocations for monitoring agents.
     * Solution: One DaemonSet with `resources.requests` tailored for GPU nodes, another with lighter requests for standard nodes.

  2. **Different Configuration Flags**
     * Example: A logging agent needs to parse logs differently on database nodes vs application nodes.
     * Solution: Deploy two DaemonSets, each mounting different config files via ConfigMap.

  3. **Different Hardware Types**
     * Example: Bare-metal nodes vs virtual machines may need different system daemons or driver-related flags.
     * Solution: Use multiple DaemonSets with `nodeSelector` / `nodeAffinity` targeting the correct hardware labels.

  4. **Mixed Operating Systems / Architectures**
     * Example: A cluster with both Linux and Windows nodes.
     * Solution: One DaemonSet runs the daemon container built for Linux (`nodeSelector: kubernetes.io/os=linux`), another for Windows.

* **Implementation**
  * Use **labels and selectors** to scope each DaemonSet:
    ```yaml
    spec:
      template:
        spec:
          nodeSelector:
            hardwareType: gpu
    ```
  * Alternatively, use **affinity rules** or **tolerations** for tainted node pools.


#### DaemonSet Best Practices
* **Restart Policy**
  * Set to `Always` or leave unspecified.
  * Ensures DaemonSet pods **automatically restart** if they fail.

* **Namespace Isolation**
  * Deploy each DaemonSet in its **own namespace**.
  * Helps in **resource management** and avoids conflicts with other DaemonSets.

* **Scheduling Preferences**
  * Prefer `preferredDuringSchedulingIgnoredDuringExecution` over `requiredDuringSchedulingIgnoredDuringExecution`.
  * Reason: If required nodes are unavailable, pods won’t start at all. Preferred scheduling is **flexible**.

* **DaemonSet Priority**
  * Set `priorityClassName` to **high priority** (≥ 10000).
  * Prevents critical DaemonSet pods from being **evicted** under resource pressure.

* **Pod Selector**
  * Must match `.spec.template.labels`.
  * Ensures **correct pods are managed** by the DaemonSet controller.

* **minReadySeconds**
  * Defines how long Kubernetes should wait before creating the next pod.
  * Guarantees that **existing pods are ready** before new pods are rolled out during updates.

* **Node Deployment**
  * Use proper **labels and node selectors** to deploy pods only on intended nodes.
  * Useful for **specialized workloads**, e.g., monitoring agents or network plugins.

* **Resource Requests/Limits**
  * Keep CPU and memory requests **minimal** because pods run on every node.
  * Prevents unnecessary resource consumption cluster-wide.

* **PodDisruptionBudgets (PDB)**
  * Define PDB to control **eviction during maintenance or upgrades**.
  * Ensures that **enough DaemonSet pods remain running** to maintain functionality.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-monitoring-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: logging
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: logging
    spec:
      restartPolicy: Always
      priorityClassName: high-priority
      nodeSelector:
        type: platform-tools
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: type
                operator: In
                values:
                - platform-tools
      containers:
        - name: splunk-monitoring-agent
          image: splunk:latest
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: splunk-agent-pdb
  namespace: monitoring
spec:
  minAvailable: 80%
  selector:
    matchLabels:
      app: logging
```


#### Purpose of Pod Priority
  * Determines the **importance of a pod** relative to other pods.
  * Higher priority pods are **less likely to be preempted** when resources are scarce.
  * Useful for critical system components.
  * Critical system components (like `kube-proxy` or CNI pods) often use high-priority classes.
  * Ensures **critical DaemonSet pods remain running**, even under node resource pressure.
  * Useful for components like network plugins, logging agents, or monitoring daemons that must always be active.

* **PriorityClass Object**
  * Defines the priority value of a pod.
  * Can be any **32-bit integer ≤ 1 billion**.
  * Higher value → higher priority.

* **Example PriorityClass Creation**
  ```yaml
  apiVersion: scheduling.k8s.io/v1
  kind: PriorityClass
  metadata:
    name: high-priority
  value: 100000
  globalDefault: false
  description: "daemonset priority class"
  ```

  * Check existing priority classes:
    ```bash
    kubectl get priorityclass
    ```

* **Using PriorityClass in DaemonSet**
  ```yaml
  spec:
    priorityClassName: high-priority
    containers:
      ...
    terminationGracePeriodSeconds: 30
    volumes:
      ...
  ```

* **Built-in Critical Priority Classes**
  * `system-node-critical` → highest priority for node-level essential pods; not evicted under any circumstances.
  * `system-cluster-critical` → high priority for cluster-critical pods.

#### Some Troubleshooting
* **When is a DaemonSet Unhealthy?**
  * Any pod in the DaemonSet is **not running on a node**.
  * Common pod statuses causing issues:
    * `CrashLoopBackOff` → pod repeatedly crashing.
    * `Pending` → pod cannot be scheduled.
    * `Error` → pod failed to start or encountered runtime issues.

* **Initial Troubleshooting Step**
  * Check pod logs to identify the issue:
    ```bash
    kubectl -n <NAMESPACE> logs <POD NAME> -f
    ```

* **Common Fixes**
  * **Resource issues:**
    * Pod might be running out of CPU or memory.
    * Reduce **resource requests/limits** in the DaemonSet spec.
  * **Node pressure:**
    * Move some pods off affected nodes.
    * Use **taints and tolerations** to control pod placement.
  * **Cluster capacity:**
    * Scale up the cluster by adding more nodes.
  * **Other pod troubleshooting:**
    * Check events: `kubectl describe pod <POD NAME> -n <NAMESPACE>`
    * Ensure container images exist and are accessible.
    * Validate network connectivity if required by the pod.

* **Key Tip**
  * Since DaemonSets run **on all (or selected) nodes**, even a single node issue can make the DaemonSet appear unhealthy. Start troubleshooting **node by node**.


### **DaemonSet Scaling Concept**
  * **DaemonSets** automatically ensure **one Pod per node** (or per matching node).
  * You **cannot use** `kubectl scale` like you do with Deployments.
  * Scaling occurs **indirectly** by:
    * Adding nodes → **scales up** (creates new pods)
    * Removing nodes → **scales down** (deletes pods)
```bash
kubectl label node node01 type=demo
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: splunk-monitoring-agent
  labels:
    app: demo
spec:
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      nodeSelector:
        type: demo  # Ensures it runs on specific nodes with the label "platform-tools"
      containers:
        - name: splunk-monitoring-agent
          image: splunk:latest
          ports:
            - containerPort: 8088  # Assuming the agent exposes a port
          volumeMounts:
            - name: splunk-config
              mountPath: /etc/splunk  # Specify where the config should be mounted in the container
      volumes:
        - name: splunk-config
          configMap:
            name: splunk-config-map  # Ensure you have a ConfigMap named splunk-config-map
```

* **Manually Scaling a DaemonSet to Zero**
  * If you want to *temporarily remove all DaemonSet pods*, you can use a **non-matching node selector** to stop the DaemonSet from scheduling on any node.
  ```bash
  kubectl patch daemonset <daemonset-name> -n <namespace> \
    -p '{"spec": {"template": {"spec": {"nodeSelector": {"none": "match"}}}}}'
  ```
  ```bash
  kubectl patch daemonset splunk-monitoring-agent -n kube-system \
    -p '{"spec": {"template": {"spec": {"nodeSelector": {"dummy-nodeselector": "foobar"}}}}}'
  ```
  * This applies a fake label (`none=match`) that doesn’t exist on any node.
  * Result → All existing DaemonSet pods are **terminated**, and **no new pods** will be scheduled.

* **Scale Back to Normal**
  * To restore it, simply remove the dummy selector or reapply the original configuration:

* Option 1: Remove the selector completely
  ```bash
  kubectl patch daemonset splunk-monitoring-agent -n kube-system \
    -p '{"spec": {"template": {"spec": {"nodeSelector": null}}}}'
  ```

* Option 2: Restore the real selector
  ```bash
  kubectl patch daemonset splunk-monitoring-agent -n kube-system \
    -p '{"spec": {"template": {"spec": {"nodeSelector": {"type": "platform-tools"}}}}}'
  ```

* **Verification**
  * Check DaemonSet pods before and after:
  ```bash
  kubectl get pods -n kube-system -l app=logging -o wide
  ```
  * When scaled down → **No pods should appear**.
  * When restored → Pods will **reappear on all matching nodes**.

