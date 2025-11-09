
# **1. How Do Kubernetes Pods Communicate With Each Other?**

### **a) Inside the Same Pod (Multi-container Pod)**

* Containers in the **same Pod** share:
  * **Network namespace** → same IP address and port space.
  * **Volumes** → shared storage.
* Communication is done via **`localhost`** (loopback).

  * Example: `container A` can call `http://localhost:8080` to talk to `container B`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: default
  labels:
    app: nginx
spec:
  initContainers:
    - name: init-con-1
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con1
      image: nginx
    - name: con2
      image: curlimages/curl:8.16.0
      command: ["/bin/sh","-c","sleep 3600"]
```
- https://hub.docker.com/r/curlimages/curl

---

### **b) Pod-to-Pod in the Same Cluster**

* Each Pod gets a **unique cluster-private IP** from Kubernetes.
* Pods can talk to each other **directly using these IPs** (no NAT inside cluster).
* Problem: Pod IPs are **ephemeral** (change if pod restarts).

👉 **Solution**: Use **Services** (ClusterIP, NodePort, LoadBalancer).

* A **Service** provides a **stable DNS name** (e.g., `myapp-svc.default.svc.cluster.local`) and load balances across Pods.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod1
  namespace: default
  labels:
    app: nginx
spec:
  initContainers:
    - name: init-con-1
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con1
      image: nginx

---

apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80


---

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod2
  namespace: default
  labels:
    app: curl
spec:
  initContainers:
    - name: init-con-2
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con2
      image: curlimages/curl:8.16.0
      command: ["/bin/sh","-c","sleep 3600"]
```

---

### **c) Pod-to-Pod Across Namespaces**

* Pods can reach others using **fully qualified DNS names**:

  ```
  <service-name>.<namespace>.svc.cluster.local
  ```
* Example: `webapp.frontend.svc.cluster.local`

```yaml
apiVersion: v1
kind: Namespace 
metadata:
  name: namespace1

---

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod1-namespace1
  namespace: namespace1
  labels:
    app: nginx
spec:
  initContainers:
    - name: init-con-1
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con1
      image: nginx

---

apiVersion: v1
kind: Service
metadata:
  name: myapp-namespace1
  namespace: namespace1
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod2-namespace1
  namespace: namespace1
  labels:
    app: curl
spec:
  initContainers:
    - name: init-con-2
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con2
      image: curlimages/curl:8.16.0
      command: ["/bin/sh","-c","sleep 3600"]

---

apiVersion: v1
kind: Namespace 
metadata:
  name: namespace2

---

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod1-namespace2
  namespace: namespace2
  labels:
    app: nginx
spec:
  initContainers:
    - name: init-con-1
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con1
      image: nginx

---

apiVersion: v1
kind: Service
metadata:
  name: myapp-namespace2
  namespace: namespace2
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod2-namespace2
  namespace: namespace2
  labels:
    app: curl
spec:
  initContainers:
    - name: init-con-2
      image: busybox
      command:
      - "/bin/sh"
      - "-c"
      - "sleep 20"
  containers:
    - name: con2
      image: curlimages/curl:8.16.0
      command: ["/bin/sh","-c","sleep 3600"]

...
```

---

### **d) Pod-to-External World**

* By default, Pods can **initiate outbound connections** (to the Internet) via the cluster’s NAT gateway.
* To allow **incoming traffic** from outside → use:

  * **NodePort Service**
  * **LoadBalancer Service**
  * **Ingress Controller** (for HTTP/HTTPS routing).

* Pod to External World (Outbound)
  ```yaml
  # pod.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: curl-pod
  spec:
    containers:
    - name: curl
      image: curlimages/curl
      command: ["sleep", "3600"]
  ```

* NodePort Service
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nodeport-svc
  spec:
    selector:
      app: nginx
    type: NodePort
    ports:
      - port: 80        # service port inside cluster
        targetPort: 80  # pod container port
        nodePort: 30080 # external port on node
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deploy
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
          ports:
          - containerPort: 80
  ```

* LoadBalancer Service (works in cloud providers like AWS, GCP, Azure, or Minikube with tunnel)
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-lb-svc
  spec:
    selector:
      app: nginx
    type: LoadBalancer
    ports:
      - port: 80
        targetPort: 80
  ```

* Ingress Controller
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
  spec:
    rules:
    - host: myapp.local
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: my-nodeport-svc
              port:
                number: 80
  ```

---

🔑 **Summary:**

* **Same Pod** → `localhost`
* **Same Cluster Pod** → Pod IP or Service
* **Different Namespace** → FQDN Service DNS
* **Outside World** → NodePort / LoadBalancer / Ingress

### References
- https://www.geeksforgeeks.org/devops/kubernetes-pods/
