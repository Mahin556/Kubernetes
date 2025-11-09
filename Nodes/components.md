# **Kubernetes Cluster Architecture**

A Kubernetes cluster has **two main parts**:

1. **Control Plane (Master Node(s))** – Brain of the cluster, responsible for decision-making.
2. **Worker Nodes** – Muscles of the cluster, responsible for running workloads (Pods/Containers).

---

## **1. Control Plane Components (Master Node)**

The **control plane** manages the **state, scheduling, and lifecycle** of the cluster.

* **API Server (`kube-apiserver`)**

  * Entry point of the cluster (acts like a front desk).
  * Exposes the **REST API** for all cluster operations.
  * All tools (`kubectl`, controllers, schedulers, etc.) interact with the API server.
  * Validates and processes requests, then updates the cluster state.

* **Scheduler (`kube-scheduler`)**

  * Decides **where Pods should run**.
  * Considers resource requirements, node health, constraints (taints/tolerations, affinity).
  * Example: If Pod needs SSD storage → Scheduler places it on the correct node.

* **Controller Manager (`kube-controller-manager`)**

  * Runs background **control loops** that keep the cluster in the desired state.
  * Types of controllers:

    * **Node Controller**: Detects failed nodes.
    * **Replication Controller**: Ensures the right number of Pods are running.
    * **Job Controller**: Manages one-time or batch jobs.
    * **Namespace Controller**: Creates default accounts, API tokens, etc.

* **etcd (Key-Value Store)**

  * Stores **all cluster data** (state, configuration, secrets, workloads).
  * Distributed and fault-tolerant.
  * Critical component → must be **secured & backed up**.
  * If etcd is lost → cluster state is lost.

* **Cloud Controller Manager (Optional)**

  * Integrates Kubernetes with **cloud providers** (AWS, GCP, Azure).
  * Manages cloud resources: load balancers, volumes, routes.
  * Separates cloud-specific logic from Kubernetes core.

---

## **2. Worker Node Components**

Worker nodes are the machines (VMs or physical) that run your **applications**.

* **Pods & Containers**

  * A **Pod** is the smallest unit in Kubernetes (wrapper for one or more containers).
  * Containers inside Pods run the actual application.

* **Kubelet**

  * An agent running on each worker node.
  * Talks to the **API server** to receive instructions.
  * Ensures containers in Pods are running as expected.
  * Reports back the node & Pod status to the control plane.

* **Kube-proxy**

  * Manages **networking** for Pods.
  * Handles **service discovery and load balancing**.
  * Ensures communication between Pods across nodes using IP tables or IPVS.

---

## **3. Interaction Between Components**

* User runs a command → `kubectl` talks to the **API Server**.
* **API Server** validates request → updates **etcd**.
* **Scheduler** decides which node should run the Pod.
* **Controller Manager** ensures desired state is maintained.
* On Worker node:

  * **Kubelet** receives instructions → starts container via **Container Runtime** (e.g., Docker, containerd, CRI-O).
  * **Kube-proxy** ensures networking rules are applied.

---

## **Simplified Diagram (Text Representation)**

```
                 +----------------------+
                 |    Control Plane     |
                 |----------------------|
  kubectl --->   |  API Server          |
                 |  Scheduler           |
                 |  Controller Manager  |
                 |  etcd (DB)           |
                 |  Cloud Controller    |
                 +----------------------+
                          |
          -----------------------------------------
          |                |                      |
   +---------------+ +---------------+   +---------------+
   |   Worker 1    | |   Worker 2    |   |   Worker 3    |
   |---------------| |---------------|   |---------------|
   | kubelet       | | kubelet       |   | kubelet       |
   | kube-proxy    | | kube-proxy    |   | kube-proxy    |
   | Pods/Containers| | Pods/Containers| | Pods/Containers|
   +---------------+ +---------------+   +---------------+
```

### References:
- https://spacelift.io/blog/kubernetes-cluster