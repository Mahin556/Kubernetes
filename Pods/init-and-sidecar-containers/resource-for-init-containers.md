### References:
- https://devopscube.com/kubernetes-init-containers/


### 🔹 CPU and Memory for Init Containers

* **Why it matters:**
  Init containers may perform critical initialization tasks (like fetching secrets, creating configs, warming caches, etc.) before the main container starts. If they don’t have sufficient CPU/memory, they may fail or slow down Pod startup.

* **Resource Requests vs Limits:**

  * **Request:** Minimum guaranteed resource the init container gets.
  * **Limit:** Maximum resource the init container can use.

* **Effective Init Resources:**

  * When a Pod has multiple init containers, **Kubernetes calculates the effective resource request/limit** for the Pod as follows:

    * **CPU:** max CPU requested among all init containers.
    * **Memory:** max memory requested among all init containers.
  * This ensures that the Pod scheduling takes the **heaviest init container** into account.

---

### 🔹 Example YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  initContainers:
    - name: init-container-1
      image: busybox
      command: ["sh", "-c", "echo initializing...; sleep 5"]
      resources:
        requests:
          cpu: 50m       # Minimum CPU guaranteed
          memory: 64Mi   # Minimum Memory guaranteed
        limits:
          cpu: 100m      # Maximum CPU allowed
          memory: 128Mi  # Maximum Memory allowed
    - name: init-container-2
      image: busybox
      command: ["sh", "-c", "echo initializing again...; sleep 5"]
      resources:
        requests:
          cpu: 80m
          memory: 128Mi
        limits:
          cpu: 150m
          memory: 256Mi
  containers:
    - name: main-container
      image: nginx
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
```

---

### 🔹 Important Notes

1. **Effective request/limit for the Pod:**

   * CPU: max(50m, 80m) = 80m
   * Memory: max(64Mi, 128Mi) = 128Mi
   * Kubernetes scheduler uses these **effective init container resources** to schedule the Pod.

2. **Resource planning:**

   * Ensure that `initContainers + main containers` **do not exceed the node capacity**, otherwise the Pod will remain Pending.

3. **Monitoring:**

   * Use `kubectl top pod <pod-name>` or metrics server to monitor actual usage.
   * Adjust limits and requests based on observed resource consumption to optimize cluster utilization.


