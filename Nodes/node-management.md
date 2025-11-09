# 🌐 Complete Guide to Kubernetes Nodes & Node Controller

---

## 🔹 1. Kubernetes Node Name Uniqueness

* Each **Node in a Kubernetes cluster must have a unique name** (`metadata.name`).
* If two nodes share the same name:

  * The API server assumes they are the same object → leads to **state conflicts**.
  * Labels, taints, volumes may overlap incorrectly.
  * Troubleshooting becomes nearly impossible.
* Example risk: attaching a Persistent Volume to the wrong node if names clash.
* ✅ Always ensure **unique hostnames** or explicitly set node name when starting `kubelet` with:

  ```bash
  --hostname-override=<unique-node-name>
  ```

---

## 🔹 2. Node States

View with:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

States:

* **Ready** → Node is healthy, can schedule pods.
* **NotReady** → Node isn’t healthy (network issue, kubelet down, disk pressure).
* **Unknown** → API server lost contact (usually network partition or node down).

Example from `kubectl describe`:

```yaml
conditions:
- type: Ready
  status: "True"
  reason: KubeletReady
  message: kubelet is posting ready status
```

---

## 🔹 3. Node Registration

Kubelet handles registration with API server.

### Self-Registration (default):

* Enabled by `--register-node=true` (default).
* Kubelet auto-registers node with API server.
* Requires access to a **kubeconfig file** to authenticate.
* Scheduler can then place pods on this node.

### Manual Registration:

* Set `--register-node=false` in kubelet.
* Admin must manually create a `Node` object:

  ```bash
  kubectl create -f node.yaml
  ```
* Example `node.yaml`:

  ```yaml
  apiVersion: v1
  kind: Node
  metadata:
    name: worker1
    labels:
      role: custom
  spec:
    taints:
    - key: dedicated
      value: batch
      effect: NoSchedule
  ```

---

## 🔹 4. Node Controller

* Runs in the **kube-controller-manager** on the control plane.
* Responsible for:

  * **Monitoring node health** (via heartbeats from kubelet).
  * **Marking nodes NotReady/Unknown** if heartbeats fail (`--node-monitor-grace-period`).
  * **Evicting pods** from unhealthy nodes after timeout (`--pod-eviction-timeout`).
  * **Allocating resources** based on node reports.

Key Flags (in `/etc/kubernetes/manifests/kube-controller-manager.yaml`):

```yaml
- --node-monitor-period=5s
- --node-monitor-grace-period=40s
- --pod-eviction-timeout=5m
- --node-eviction-rate=0.1
```

---

## 🔹 5. Node Resource Capacity

When a node registers, kubelet reports:

* **CPU**
* **Memory**
* **Ephemeral storage**
* **Attachable volumes**

Scheduler uses this info to decide pod placement.
Check with:

```bash
kubectl describe node <node>
```

---

## 🔹 6. Node Topology (Scheduling Pods)

Nodes can be grouped by labels (zone, region, instance type).
You can control scheduling using:

* **nodeSelector** (simple key-value match)
* **nodeAffinity / antiAffinity** (more flexible rules)

Example:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
```

---

## 🔹 7. Graceful Node Shutdown

* Kubernetes ≥ 1.20 supports **graceful node shutdown**.
* Kubelet notifies pods before node shutdown.
* Benefits:

  * No data loss.
  * Stateful apps get time to save state.
  * Smooth failover.

If not graceful → pods are killed abruptly, may cause corruption.

---

## 🔹 8. Managing Kubernetes Nodes

Tasks include:

* **Provisioning**: add nodes to cluster (cloud API, kubeadm join).
* **Scaling**: add/remove nodes dynamically.
* **Updating**: OS patching, kubelet upgrades.
* **Maintaining**: drain nodes before maintenance:

  ```bash
  kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
  kubectl uncordon <node>
  ```

---

## 🔹 9. Optimizing Node Performance

* **Container Packing** → Efficient pod placement.
* **Resource Requests/Limits** → Prevent noisy neighbor problems.
* **Eviction Policies** → Ensure stability under pressure.
* **Monitoring** → Metrics via Prometheus, Metrics Server.
* **Scheduling Strategies** → Affinity/anti-affinity for HA.
* **Runtime Tuning** → Smaller images, updated runtimes.

---

## 🔹 10. Securing Nodes

* Harden OS: disable unused services.
* Limit access: SSH restrictions, IAM roles.
* Use Pod Security Standards or OPA/Gatekeeper.
* Keep kubelet & container runtime updated.
* Enable **TLS for kubelet**.

---

# ✅ Summary

* Nodes = worker machines in Kubernetes.
* Node Controller = ensures nodes are healthy, evicts pods if not.
* Unique node names = required for stability.
* Self-registration (default) vs manual registration.
* Key parameters can be tuned in kubelet and controller-manager.
* Use graceful shutdown, resource requests/limits, and affinity rules for reliability.


### References:
- https://www.geeksforgeeks.org/devops/kubernetes-node/
