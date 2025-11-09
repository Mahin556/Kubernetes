### References:-
- https://spacelift.io/blog/kubectl-delete-pod

---

* The `kubectl drain` command safely evicts pods from a node and reschedules them on other nodes.
    ```bash
    # Get nodes with details
    kubectl get nodes -o wide

    # Get pods with node assignments
    kubectl get pods -o wide

    kubectl drain <node-name>

    kubectl drain aks-node-001 --ignore-daemonsets --delete-emptydir-data
    ```

* DaemonSet-managed pods:
  ```bash
  --ignore-daemonsets
  ```

* Pods using local storage (emptyDir):
  ```bash
  --delete-emptydir-data
  ```