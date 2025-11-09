* Describe pod details and events
```bash
kubectl describe pod <pod-name>
```

* Check pod logs
```bash
kubectl logs <pod-name>
```

* Check previous container logs (CrashLoopBackOff)
```bash
kubectl logs <pod-name> --previous
```

* Check pod YAML
```bash
kubectl get pod <pod-name> -o yaml
```

* Test configuration with a minimal pod
```bash
kubectl run test --image=busybox -it --restart=Never -- sh
```

* Watch pod status live
```bash
kubectl get pods -w
```