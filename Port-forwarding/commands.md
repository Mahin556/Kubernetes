# **Accessing Applications in a Pod**

* Forward one or more local ports on your machine to a port on a **pod**, **deployment**, or **service** in Kubernetes.
* Useful for debugging, local testing, and securely accessing in-cluster services without exposing them publicly.

* We can expose our application in K8S in two main ways: **temporarily via port-forwarding** or **permanently via a Service**. Here we focus on **port-forwarding**.

---
## 📌 General Syntax

```bash
kubectl port-forward [TYPE/]NAME [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N] [options]
```

* `TYPE/NAME` → Resource type (`pod`, `deployment`, `service`, etc.) and name.
* `LOCAL_PORT` → Port on your local machine.
* `REMOTE_PORT` → Port inside the pod/service.

If only one port is given, local and remote ports are the same.
If `LOCAL_PORT:` is omitted, local port = remote port.


## **1. Using kubectl port-forward**

### Command:

```bash
kubectl port-forward pod/web-server-pod 8080:80
```
* Maps local port `8080` to Pod’s container port `80`.
* **kubectl** creates a **tunnel** from your local machine → cluster node → Pod.
* This allows you to access the Pod **without exposing it externally**.
* Open a browser and go to: `http://localhost:8080`
* To stop port forwarding, press **CTRL + C**.

---

## **2. What Happens Internally**
* API server forwards traffic → the **node** running the Pod → the Pod’s container port 80.
* All communication is **over a single HTTP/HTTPS connection** (tunnel).
---

# 📚 Options with Commands + Use Cases


## 1. `--address`

* List of addresses to bind on (default `localhost`).
Accepts IP addresses or `localhost`.

### 🔹 Command Example:

```bash
kubectl port-forward --address 0.0.0.0 pod/mypod 8888:5000
```
Listens on **all interfaces (0.0.0.0)**, so external clients can connect.

### 🔹 Use Case:

* When you want to expose a pod port to other machines (not just localhost).
* Example: Allow teammates to access a local-forwarded service from their browsers.

---

## 2. `--pod-running-timeout`

**Description**: How long to wait for a pod to be in `Running` state before failing. Default = `1m`.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --pod-running-timeout=2m
```

👉 Waits up to **2 minutes** for the pod to start.

### 🔹 Use Case:

* Useful in CI/CD pipelines where pods may take extra time to initialize before port-forwarding.

---

## 3. `-h, --help`

**Description**: Show help for `port-forward`.

### 🔹 Command Example:

```bash
kubectl port-forward --help
```

👉 Displays all available options.

### 🔹 Use Case:

* Quick reference for command flags and usage.

---

# 🏗 Parent Options (Inherited from `kubectl`)

---

## 4. `--as`

**Description**: Run command as another Kubernetes user.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --as admin
```

👉 Impersonates user `admin`.

### 🔹 Use Case:

* Testing RBAC permissions by simulating another user’s access.

---

## 5. `--as-group`

**Description**: Run as a specific group (can repeat).

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --as-group=dev-team
```

👉 Runs with group `dev-team`.

### 🔹 Use Case:

* Validate group-level permissions (e.g., dev-team vs ops-team).

---

## 6. `--as-uid`

**Description**: Run command as a specific UID.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --as-uid=1001
```

👉 Impersonates a UID directly.

### 🔹 Use Case:

* Advanced RBAC or debugging scenarios where UID matters.

---

## 7. `--cache-dir`

**Description**: Default kubeconfig cache directory.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --cache-dir=/tmp/kube-cache
```

### 🔹 Use Case:

* When using a temporary filesystem or in CI/CD environments where `$HOME` is not writable.

---

## 8. `--certificate-authority`

**Description**: Path to custom CA cert.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 --certificate-authority=/etc/k8s/ca.crt
```

### 🔹 Use Case:

* Accessing clusters with private CAs.

---

## 9. `--client-certificate` + `--client-key`

**Description**: TLS client authentication files.

### 🔹 Command Example:

```bash
kubectl port-forward pod/mypod 8080:80 \
  --client-certificate=/etc/k8s/client.crt \
  --client-key=/etc/k8s/client.key
```

### 🔹 Use Case:

* When API server requires **mutual TLS authentication**.

---

## 10. `--cluster`

**Description**: Specify cluster from kubeconfig.

```bash
kubectl port-forward pod/mypod 8080:80 --cluster=dev-cluster
```

👉 Targets the `dev-cluster`.

### 🔹 Use Case:

* If kubeconfig has multiple clusters.

---

## 11. `--context`

**Description**: Specify kubeconfig context.

```bash
kubectl port-forward pod/mypod 8080:80 --context=staging
```

👉 Uses `staging` context.

### 🔹 Use Case:

* Switching between clusters/namespaces quickly.

---

## 12. `--disable-compression`

**Description**: Disable server response compression.

```bash
kubectl port-forward pod/mypod 8080:80 --disable-compression
```

👉 Turns off response gzip.

### 🔹 Use Case:

* Debugging network issues where compression interferes.

---

## 13. `--insecure-skip-tls-verify`

**Description**: Skip TLS certificate verification.

```bash
kubectl port-forward pod/mypod 8080:80 --insecure-skip-tls-verify
```

👉 Ignores invalid/self-signed certs.

### 🔹 Use Case:

* Quick debugging in dev environments.

---

## 14. `--kubeconfig`

**Description**: Path to a kubeconfig file.

```bash
kubectl port-forward pod/mypod 8080:80 --kubeconfig=/tmp/custom-kubeconfig
```

👉 Uses `/tmp/custom-kubeconfig`.

### 🔹 Use Case:

* Running with alternate kubeconfig files.

---

## 15. `--kuberc`

**Description**: Path to kuberc preferences file.

```bash
kubectl port-forward pod/mypod 8080:80 --kuberc=/etc/kube/kuberc
```

### 🔹 Use Case:

* Advanced configs; can disable with `KUBECTL_KUBERC=false`.

---

## 16. `--match-server-version`

**Description**: Require server/client version match.

```bash
kubectl port-forward pod/mypod 8080:80 --match-server-version
```

### 🔹 Use Case:

* Avoids API incompatibility issues.

---

## 17. `-n, --namespace`

**Description**: Set namespace.

```bash
kubectl port-forward -n mynamespace pod/mypod 8080:80
```

👉 Forwards from a pod in namespace `mynamespace`.

### 🔹 Use Case:

* When pod is not in `default`.

---

## 18. `--password` / `--username`

**Description**: Basic authentication credentials.

```bash
kubectl port-forward pod/mypod 8080:80 --username=user --password=pass
```

### 🔹 Use Case:

* Legacy clusters still using basic auth.

---

## 19. `--profile` + `--profile-output`

**Description**: Performance profiling.

```bash
kubectl port-forward pod/mypod 8080:80 --profile=cpu --profile-output=pf-profile.pprof
```

### 🔹 Use Case:

* Debugging kubectl performance.

---

## 20. `--request-timeout`

**Description**: Timeout for API requests.

```bash
kubectl port-forward pod/mypod 8080:80 --request-timeout=30s
```

### 🔹 Use Case:

* Prevent long hangs if API server is slow.

---

## 21. `-s, --server`

**Description**: API server address.

```bash
kubectl port-forward pod/mypod 8080:80 -s https://1.2.3.4:6443
```

### 🔹 Use Case:

* Direct connect without kubeconfig.

---

## 22. Storage Driver Options

* `--storage-driver-buffer-duration`
* `--storage-driver-db`
* `--storage-driver-host`
* `--storage-driver-password`
* `--storage-driver-secure`
* `--storage-driver-table`
* `--storage-driver-user`

👉 Rarely used directly with `kubectl port-forward`.
These are for **metrics collection (cadvisor)**.

Example:

```bash
kubectl port-forward pod/mypod 8080:80 \
  --storage-driver-host=localhost:8086 \
  --storage-driver-db=metricsdb \
  --storage-driver-user=admin --storage-driver-password=secret
```

---

## 23. `--tls-server-name`

**Description**: Override TLS SNI.

```bash
kubectl port-forward pod/mypod 8080:80 --tls-server-name=mycluster.local
```

### 🔹 Use Case:

* When server cert CN doesn’t match hostname.

---

## 24. `--token`

**Description**: Bearer token authentication.

```bash
kubectl port-forward pod/mypod 8080:80 --token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

### 🔹 Use Case:

* Service account or automation scripts.

---

## 25. `--user`

**Description**: Specify kubeconfig user.

```bash
kubectl port-forward pod/mypod 8080:80 --user=devuser
```

### 🔹 Use Case:

* When kubeconfig has multiple users.

---

## 26. `--version`

**Description**: Print version or set compatibility.

```bash
kubectl port-forward --version
kubectl port-forward --version=raw
```

### 🔹 Use Case:

* Debugging kubectl-client mismatch.

---

## 27. `--warnings-as-errors`

**Description**: Treat warnings as errors.

```bash
kubectl port-forward pod/mypod 8080:80 --warnings-as-errors
```

### 🔹 Use Case:

* Strict CI/CD pipelines where warnings must fail.

---

# ✅ Example Scenarios

1. **Debugging a database** inside cluster:

```bash
kubectl port-forward pod/mongo-0 27017:27017
```

2. **Access web app in pod via localhost**:

```bash
kubectl port-forward deployment/myapp 8080:80
```

3. **Access HTTPS service by name**:

```bash
kubectl port-forward service/myservice 8443:https
```

4. **Expose to teammates**:

```bash
kubectl port-forward --address 0.0.0.0 pod/myapi 9000:8080
```

### References:
- https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/