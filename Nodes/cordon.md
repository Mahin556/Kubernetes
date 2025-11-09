### **Cordon the Node (Optional)**

Prevent new pods from scheduling on the node:

```bash
kubectl cordon <node-name>
```

### **Step 4: Uncordon the Node**

After maintenance, allow pods to be scheduled again:

```bash
kubectl uncordon <node-name>
```

Status changes from `SchedulingDisabled` → `Ready`.

### References:
- https://spacelift.io/blog/kubectl-delete-pod