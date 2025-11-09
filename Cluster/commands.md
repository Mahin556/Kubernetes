```
kubectl cluster-info              # Show cluster info
kubectl version                   # Client/server versions
kubectl config view               # View kubeconfig
kubectl config current-context    # Current context
kubectl config get-contexts       # List contexts
kubectl config use-context <ctx>  # Switch context
kubectl api-resources             # List resources
kubectl api-versions              # List versions
kubectl get all -A                # All resources in all namespaces
kubectl get all
kubectl api-resources | grep -i TYPE
kubectl delete --all -A/--all-namespaces
```
### References
- https://spacelift.io/blog/kubernetes-cheat-sheet
