```bash
alias k=kubectl
alias kdr='kubectl --dry-run=client -o yaml'
```
```bash
kdr run web --image=nginx:latest > nginx.yaml
```

### References:
- https://devopscube.com/create-kubernetes-yaml/