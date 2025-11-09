```bash
kubectl get nodes #To list and retrieve the info about the nodes
```
```bash
NAME       STATUS   ROLES    AGE     VERSION
node-1     Ready    <none>   7d      v1.21.3
node-2     Ready    <none>   7d      v1.21.3
node-3     Ready    <none>   7d      v1.21.3
```
- **NAME**: The name of the node within the cluster.

- **STATUS**: The current status of the node, indicating whether it is ready, not - ready, or unreachable.

- **ROLES**: Any roles assigned to the node, such as master, worker, etcd.

- **AGE**: The duration since the node was created or added to the cluster.

- **VERSION**: The Kubernetes version is running on the node.

- **INTERNAL-IP**: The internal IP address assigned to the node.

- **EXTERNAL-IP**: The external IP address assigned to the node, if applicable.

* To get a wide information
```bash
kubectl get nodes -o wide
```
```bash
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
controlplane   Ready    control-plane   12d   v1.33.2   172.30.1.2    <none>        Ubuntu 24.04.3 LTS   6.8.0-51-generic   containerd://1.7.27
node01         Ready    <none>          12d   v1.33.2   172.30.2.2    <none>        Ubuntu 24.04.3 LTS   6.8.0-51-generic   containerd://1.7.27
```

```bash
kubectl describe node <name>
kubectl top node <name>
kubectl taint node <name> key=value:NoSchedule
kubectl describe node <node-name>
kubectl cordon <name>       # Unschedulable
kubectl uncordon <name>     # Schedulable
kubectl drain <name>        # Evict pods (maintenance)
kubectl label node node-1 environment=production disk=ssd
kubectl label node node-2 environment=staging disk=hdd
kubectl get nodes -l environment=production
kubectl get nodes -l "environment=production,disk=ssd"
kubectl get nodes --show-labels
kubectl annotate node <name> key=value
kubectl delete node <name>
kubectl get nodes -o yaml
kubectl get nodes -o json

# Advance selectors
kubectl get nodes --selector='environment in (production,staging)'
kubectl get nodes --selector='disk!=hdd'
```


```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory
```
```bash
systemctl status kubelet #Check kubelet service
journalctl -u kubelet
systemctl status containerd #Verify container runtime
systemctl status docker
#Ensure node can reach API server.
#Check firewall/DNS.
kubectl get pods -n kube-system #Look for CNI-related pods (Calico, Flannel, etc.).
```

### References
- https://spacelift.io/blog/kubernetes-cheat-sheet
- https://spacelift.io/blog/kubectl-get-nodes
