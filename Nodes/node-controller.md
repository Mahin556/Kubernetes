
---
### Node-Controller
* The **Node Controller** is a **component of the kube-controller-manager** in the control plane.
* Its primary job is to **monitor the health of nodes** and **update the cluster state accordingly**.
* It continuously watches all nodes in the cluster (based on their `metadata.name`), and:
    * Register & manage nodes in the cluster.
    * Marks nodes as **Ready/NotReady/Unknown** depending on their heartbeat.
    * Decides whether pods can be scheduled on them.
    * Takes action when nodes become unhealthy.
    * Evict pods if nodes go down.
    * Monitor node health (via heartbeats from kubelet).

---

#### How Node Controller Works

1. **Node Registration**
   * By default, when a kubelet starts, it registers itself(its node) with the API server (if `--register-node=true`).
   * Node Controller then starts monitoring it.
   * Node Controller stores node info in **etcd** via API server.
   * If `--register-node=false`, kubelets won’t auto-register; admin must manually add nodes using:
    ```bash
     kubectl apply -f node.yaml
    ```
    or via API request.

2. **Health Monitoring**
   * Each node (via kubelet) sends **status updates/heartbeats** to the API server at regular intervals (`--node-status-update-frequency`, default 10s).
   * Node Controller expects these updates.
   * Based on kubelet reports kupeapi-server updagte node statue:
     * `Ready`: node is healthy.
     * `NotReady`: node failed health checks.
     * `Unknown`: master hasn’t heard from node in a while.
   * If heartbeats stop, it assumes the node is unhealthy.

3. **Timeouts & Node States**
   * If no heartbeat received(no-update, power-down, network-cut)- node goes down/unreachable:
     * After **40s** (`--node-monitor-grace-period`) → Node marked `NotReady`.
     * After **5m** (`--pod-eviction-timeout`) → Node Controller starts evicting pods running on that node (default `pod-eviction-timeout`).
        * Evicted pods are rescheduled by the scheduler onto healthy nodes.
    * If node comes back, it rejoins automatically.

4. **Cloud Integration**(optional)
   * If running on a cloud (AWS, GCP, Azure), Node Controller talks to the **Cloud Controller Manager** to detect if a VM was deleted and updates node status.

---

#### Benefits of Node Controller

* **Self-healing cluster** (pods rescheduled if node fails).
* **Automation** (no need to babysit node health manually).
* **Flexibility** (manual mode possible with `register-node=false`).
* **Cloud awareness** (detects VM deletion in cloud environments).

---

#### Node States Managed by Node Controller

* `Ready` → Node is healthy & can host pods.
* `NotReady` → Node failed kubelet health check (e.g., network loss, disk issue).
* `Unknown` → Node hasn’t reported status in too long.
* `SchedulingDisabled` → Node was manually cordoned (`kubectl cordon`).

---

#### Node Controller Key Parameters

| Parameter                        | Default | Meaning                                                        |
| -------------------------------- | ------- | -------------------------------------------------------------- |
| `--register-node`                | true    | true --> Kubelet auto-registers with API server, false ---> admin must add nodes manually.                     |
| `--node-status-update-frequency` | 10s     | How often kubelet sends status                                 |
| `--node-monitor-grace-period`    | 40s     | How long Node Controller waits before marking node `NotReady`  |
| `--pod-eviction-timeout`         | 5m      | How long Node Controller waits before evicting pods from a dead node (default: 5m)                   |
| `--node-eviction-rate`           | 0.1     | Max nodes per second to evict pods from                        |
| `--secondary-node-eviction-rate` | 0.01    | Slower eviction rate in large failures                         |
| `--large-cluster-size-threshold` | 50      | Defines when cluster is considered large for eviction behavior |

---

#### Node Controller in Action
* **Example 1: Normal case**
    * Kubelet on `node-1` sends heartbeats every 10s.
    * Node Controller keeps it in `Ready`.
    * Scheduler assigns new pods to it.

* **Example 2: Node failure (network down)**
    * After 40s → Node marked `NotReady`.
    * After 5m → Pods on `node-1` evicted.
    * Scheduler reassigns them to `node-2` / `node-3`.

* **Example 3: Admin disables auto-registration**
    * Start kubelet with `--register-node=false`.
    * Admin must manually add node with:
      ```yaml
        apiVersion: v1
        kind: Node
        metadata:
            name: custom-node
        spec:
            taints: []
      ```
      ```bash
        kubectl apply -f node.yaml
      ```

---

#### How Node Controller Works with Other Components

* **API Server** → Node Controller reads/writes node objects.
* **etcd** → Stores node status.
* **Scheduler** → Reschedules pods when Node Controller evicts them.
* **Cloud Controller Manager** → Detects cloud VMs deletion & updates node list.

---

#### Best Practices for Node Controller
* **Tune timeouts** depending on environment:
  * For stable cloud → keep defaults (40s + 5m).
  * For flaky networks → increase `node-monitor-grace-period`.

* **Use taints/tolerations** so critical pods can survive node issues.
* **Monitor node status** using:
  ```bash
  kubectl get nodes
  kubectl describe node <node-name>
  ```

* **For production** → run at least **3 master/control-plane nodes** for HA Node Controller.

---

## 🔹 8. **Visualization of Node Lifecycle**

```
[Node Ready] → Heartbeats OK → Scheduler assigns pods
       ↓ (no heartbeat for 40s)
[Node NotReady] → Node Controller marks unhealthy
       ↓ (no heartbeat for 5m)
[Pod Eviction] → Evict pods → Scheduler reschedules
       ↓
[Node Unknown/Deleted] → Removed from cluster (if cloud detects VM gone)
```


### References:
- https://www.tutorialspoint.com/kubernetes/kubernetes_node.htm