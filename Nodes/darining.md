### **Drain the Node**

The `kubectl drain` command safely evicts pods from a node and reschedules them on other nodes.

```bash
kubectl drain <node-name>
```

#### **Handling Errors**

* DaemonSet-managed pods:

```bash
--ignore-daemonsets
```

* Pods using local storage (emptyDir):

```bash
--delete-emptydir-data
```

**Example:**

```bash
kubectl drain aks-node-001 --ignore-daemonsets --delete-emptydir-data
```

### References:
- https://spacelift.io/blog/kubectl-delete-pod