- Kubernetes is a powerful container orchestration platform that automates the deployment, scaling, and management of containerized applications. At the core of Kubernetes are nodes — the worker machines that run containers.
- Cluster ---> Group of nodes(control and worker).
- For HA worker and control node can be multiple.
- A Kubernetes node is a physical machine, virtual machine, or cloud instance that runs workloads (pods/containers).
- Each node runs essential components:
  - Container Runtime → Runs containers (e.g., Docker, containerd, CRI-O).
  - Kubelet → Talks to the API server, manages pod lifecycle.
  - Kube-Proxy → Handles networking and service routing.
  - Monitoring & Logging tools → Provide health/performance insights.

- K8s cluster can run on
  - Cloud
  - on-premise(VMs, physical servers)
  - group of Raspberry Pis!
  - serverless engines(Amazon Fargate)

- Admins can perform actions like:
  - List nodes: kubectl get nodes
  - Inspect node details: kubectl describe node <node-name>
  - Add/remove labels: kubectl label
  - Cordon/uncordon nodes: Prevent or allow new pods
  - Drain nodes: Safely evict workloads
  - Scale nodes: Add/remove worker machines

What control nodes do:


### References
- https://labex.io/tutorials/kubernetes-how-to-assign-and-manage-custom-labels-on-kubernetes-nodes-415736
- https://spacelift.io/blog/kubernetes-cluster
