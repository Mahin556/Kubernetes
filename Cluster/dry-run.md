## DRY-RUN
* `--dry-run` is a flag in Kubernetes CLI (kubectl) that simulates an operation without actually creating, updating, or deleting the resource.
It is widely used for:
    - Testing manifests
    - Generating YAML/JSON
    - CI/CD validation

#### There are two forms:
| Version          | Flag syntax                             | Behavior              |
| ---------------- | --------------------------------------- | --------------------- |
| Kubernetes ≤1.18 | `--dry-run=true/false`                  | Older boolean flag    |
| Kubernetes ≥1.18 | `--dry-run=client` / `--dry-run=server` | More granular control |

#### `--dry-run=client`
* Simulates the command locally.
* Does not contact the API server.
* Useful to generate manifests without applying them.
```bash
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml
```
* Output: YAML manifest of the deployment is printed, no deployment is created.
* Combine with `-o yaml` or `-o json` to save manifest files

#### `--dry-run=server`
* Simulates the command on the API server.
* Validates against the server-side admission controllers, validation rules, and RBAC.
* Requires server connectivity.
```bash
kubectl create deployment myapp --image=nginx --dry-run=server -o yaml
```
* Output: YAML manifest validated by the API server.
* If any admission controllers reject it (e.g., resource quotas, policies), the command fails.
* Validate manifests in a cluster with policies or constraints
* Check RBAC permissions without creating resources
* Ensure the resource would be accepted if applied


#### Differences Between client and server
| Feature    | `--dry-run=client` | `--dry-run=server`                |
| ---------- | ------------------ | --------------------------------- |
| Validation | Local validation   | Full server validation            |
| API call   | No API call        | Makes API call                    |
| Policies   | Ignored            | Checked (admission, quotas, RBAC) |
| Use case   | Generate manifest  | Validate before applying          |

#### Common Commands Using --dry-run
* Create Resource Without Applying
```bash
kubectl create pod mypod --image=nginx --dry-run=client -o yaml
kubectl create deployment mydep --image=nginx --dry-run=client -o yaml
kubectl create service clusterip mysvc --tcp=80:80 --dry-run=client -o yaml
```

* Apply Manifest With Dry Run
```bash
kubectl apply -f pod.yaml --dry-run=clientn #Check YAML correctness
kubectl apply -f pod.yaml --dry-run=server #Admission, RBAC, resource quotas
```
This checks whether the resource would be applied successfully without changing anything.

* Delete Resource With Dry Run
```bash
kubectl delete pod mypod --dry-run=client
kubectl delete deployment mydep --dry-run=server
```
Safety check before deletion


* Generate YAML/JSON Using Dry Run
```bash
kubectl create configmap myconfig --from-literal=key1=value1 --dry-run=client -o yaml > configmap.yaml
kubectl create secret generic mysecret --from-literal=password=1234 --dry-run=client -o json > secret.json
```
```bash
kubectl run mypod \
  --image=nginx \
  --restart=Never \
  --labels="app=nginx,tier=frontend" \
  --annotations="description='test pod'" \
  --dry-run=client -o yaml > pod.yaml
```

* Validating through API-server
```bash
kubectl apply -f deployment.yaml --dry-run=server
```
Checks RBAC, quotas, and admission policies before actual apply.

#### Tips & Best Practices
* Always use --dry-run=client + -o yaml to generate manifests for version control.
* Use --dry-run=server when you want cluster-level validation (admission controllers, policies).
* For CI/CD pipelines, combine with kubectl diff:
```bash
kubectl apply -f deployment.yaml --dry-run=server
kubectl diff -f deployment.yaml
```
* Use kubectl create with --dry-run for all resources: Pod, Deployment, Service, ConfigMap, Secret, Ingress.
