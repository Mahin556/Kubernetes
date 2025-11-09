### **Endpoints**

* An **Endpoint** is the actual **Pod IP + Port** where the Service traffic is redirected.
* Kubernetes automatically maintains these Endpoints:

  * When a new Pod matching the Service selector is created → its IP is added as an Endpoint.
  * When a Pod is deleted or becomes unavailable → its IP is removed.
* This makes Services dynamic and ensures load balancing across all available Pods.

---

### **How It Works**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx    
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

1. **Service YAML** (example):

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: cluster-svc
     labels:
       env: demo
   spec:
     type: ClusterIP
     ports:
       - port: 80
         targetPort: 80
     selector:
       env: demo
   ```

   * `type: ClusterIP` → creates an internal Service IP.
   * `port` → Service port exposed.
   * `targetPort` → Actual Pod container port.
   * `selector` → Matches Pods with label `env=demo`.

2. **Service created**:

   ```bash
   kubectl get svc
   ```

   Example output:

   ```
   NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    
   cluster-svc   ClusterIP   10.96.166.236   <none>        80/TCP
   kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP
   ```

3. **Service details**:

   ```bash
   kubectl describe svc cluster-svc
   ```

   Key fields:

   * **ClusterIP** → Virtual service IP (e.g., `10.96.166.236`).
   * **Ports** → Service port and target port mapping.
   * **Endpoints** → Actual Pod IPs (`10.244.x.x:80`).

4. **Endpoints object**:

   ```bash
   kubectl get ep
   ```

   Example output:

   ```
   NAME          ENDPOINTS                               AGE
   cluster-svc   10.244.1.3:80,10.244.2.2:80,10.244.2.3:80   117s
   ```

   * Shows Pod IPs where Service traffic is forwarded.
   * Keeps updating automatically with Pod lifecycle changes.