* A **Pod Disruption Budget (PDB)** limits the number of **simultaneous voluntary disruptions** that can happen to your application.
* It ensures **high availability** by specifying **how many Pods must remain available** at any given time.

<br>

* **What Are "Voluntary Disruptions"?**
    * Voluntary disruptions are **intentional actions** by administrators or the system, such as:
    * Draining a node for maintenance:
        ```bash
        kubectl drain <node-name> --ignore-daemonsets
        ```
    * Upgrading cluster nodes (e.g., via `kops`, GKE, EKS upgrades)
    * Deleting a deployment or scaling it down
    * Evicting Pods manually

* **Involuntary disruptions** (which PDB does *not* prevent) include:
    * Node crashes
    * Hardware failures
    * Network partitioning
    * Out-of-memory (OOM) kills

<br>

* **How Does PDB Work?**
    * When a voluntary disruption is requested (like draining a node), Kubernetes checks all affected Pods:
    * If a **PodDisruptionBudget** applies to that Pod, Kubernetes ensures that **disruptions do not violate the PDB’s availability requirements**.
    * If the PDB requirement cannot be met, the disruption **is blocked** until enough replicas are running elsewhere.


* **PDB Configuration Fields**
  * You can define a PDB in two ways:
  * **Option 1: Using `minAvailable`**
    * Ensures that at least **this many Pods remain running**.
      ```yaml
      minAvailable: 2
      ```
  * **Option 2: Using `maxUnavailable`**
    * Ensures that at most **this many Pods are allowed to be unavailable**.
    ```yaml
    maxUnavailable: 1
    ```
  * You can specify **an absolute number** or a **percentage**:
    ```yaml
    minAvailable: 60%   # at least 60% of replicas must remain available
    maxUnavailable: 25% # at most 25% of replicas can be down
    ```
  * Zero Disruptions
    ```yaml
    spec:
      minAvailable: 100%
    ```
    No voluntary disruption is allowed until all pods are healthy.
  * Custom Label Selector
    ```yaml
    spec:
      selector:
        matchLabels:
        app: web
        tier: frontend
    ```
    PDB applies only to pods matching both labels.

* **Example: Simple PDB for a Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```


```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: web
```

**Meaning:**

* At least **4 Pods** must remain available.
* Only **1 Pod** can be disrupted voluntarily at a time.


Alternatively:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

This means:
* At most **1 Pod** can be unavailable.
* Equivalent to keeping **4 out of 5 Pods** available.

---

* How PDBs Interact with Node Drains
```bash
kubectl drain node1 --ignore-daemonsets
```
Kubernetes will:
1. Check which Pods are affected.
2. Check if any of them are controlled by a PDB.
3. Calculate whether evicting them violates the PDB rules.
4. If yes, it blocks the drain and shows:
   ```
   error: unable to drain node "node1" due to PDB violation: cannot evict pod "web-app-xyz" as it would violate the PDB "web-pdb"
   ```
You must wait until replacement Pods come up on other nodes before proceeding.

* Check PDB Status
```bash
kubectl get pdb
```
Example output:
```
NAME       MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
web-pdb    4               N/A               1                     10m
```
To get detailed info:

```bash
kubectl describe pdb web-pdb
```
You’ll see fields like:

* `Expected Pods`
* `Disrupted Pods`
* `Allowed Disruptions`
* `Current Healthy`
* `Desired Healthy`

* When You Need PDBs

* **High availability apps** (like web servers, API gateways, or databases)
* **Rolling upgrades** (ensures one Pod updates at a time)
* **Cluster autoscaling** (prevents autoscaler from evicting too many Pods)
* **Node maintenance** (graceful upgrades without downtime)


* PDBs Don’t Guarantee 100% Uptime

* **Only limit voluntary disruptions**, not all failures.
* **Do not reschedule Pods themselves** — that’s handled by Deployments/ReplicaSets.
* **Do not apply** to static Pods or DaemonSets.

