### References:
- [Kubernetes Official Documentation: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Official Documentation: Container Images](https://kubernetes.io/docs/concepts/containers/images/)

---

Each container inside a Pod has its own status, including:
* `Waiting`: Container is waiting to start, usually due to image pull or resource constraints.
* `Running`: Container is actively running.
* `Terminated`: Container has finished execution (Succeeded or Failed).

**Example:**

```bash
kubectl get pod java-api-pod -o jsonpath='{.status.containerStatuses[*].state}'
```