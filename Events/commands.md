```
kubectl get events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector type=Warning
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=<node>
```

### References
- https://spacelift.io/blog/kubernetes-cheat-sheet
