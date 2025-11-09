
# **Multi-Container Pods in Kubernetes**

A Pod can contain **multiple containers** that work together as a single, cohesive unit. They **share:**

* The **same network namespace** (IP address, ports).
* The **same storage volumes** (for data sharing).

This allows for **tight coupling** of containers that must cooperate.

---

## **1. Init Containers**

* Run **before** the main app containers.
* Run **sequentially** (one after another).
* Must **exit successfully** before the next one (or the Pod fails).
* Perfect for **setup or preconditions**.

### ✅ Use Cases

* **Wait for dependency** (e.g., DB ready).
* **Data initialization** (load schema, pre-fill data).
* **Configuration injection** (clone repo into volume).
* **Service registration** (register pod with registry).

### 📝 Example Pod Manifest with Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z my-database 5432; do echo waiting for db; sleep 2; done;']
  containers:
    - name: webapp
      image: my-webapp:latest
      ports:
      - containerPort: 8080
```

🔎 Here:

* `wait-for-db` waits until port **5432** on `my-database` is open.
* Only then does the `webapp` container start.

---

## **2. Sidecar Containers**

* Run **alongside** the main app container.
* Start **with the app** and stop **when the pod stops**.
* Enhance functionality **without modifying app code**.

### ✅ Use Cases

* **Logging agent** (ship logs to ELK/Fluentd).
* **Metrics collector** (Prometheus exporter).
* **Service mesh proxy** (Envoy in Istio).
* **Data synchronizer** (sync from Git/S3).

### 📝 Example Pod Manifest with Sidecar

```yaml
apiVersion: v1
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
spec:
  containers:
    - name: webapp
      image: busybox
      command: 
        - sh
        - -c
        - |
            while true;do
                echo "$(date) : Log from main container" >> /var/log/app/app.log
                sleep 5 
            done
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: log-collector
      image: busybox
      command: ['sh', '-c', 'tail -n+1 -f /var/log/app/app.log']
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

🔎 Here:

* `webapp` writes logs to `/var/log/app/app.log`.
* `log-collector` sidecar **tails** those logs and can forward them.
* Both share an **emptyDir** volume.

---

## **Comparison: Init vs Sidecar**

| Feature               | Init Container 🏁        | Sidecar Container 🤝            |
| --------------------- | ------------------------ | ------------------------------- |
| Runs **before** main? | ✅ Yes                    | ❌ No                            |
| Runs **alongside**?   | ❌ No                     | ✅ Yes                           |
| Lifecycle             | One-shot, must finish    | Runs for entire pod lifetime    |
| Purpose               | Setup, preconditions     | Enhance/extend functionality    |
| Example               | Wait for DB, init schema | Log shipping, monitoring, proxy |

---

## **3. Combining Init + Sidecar**

You can use **both patterns** in a single Pod.
Example:
```yaml
# ---------------------------------------------------------
# 1. Web Application Pod with InitContainer and Log Sidecar
# ---------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
  labels:
    app: webapp
spec:
  # Init container waits until the database is available
  initContainers:
    - name: wait-for-db
      image: busybox
      command:
        - sh
        - -c
        - |
          echo "Checking database connectivity..."
          start_time=$SECONDS
          until nc -z my-database 5432; do 
              echo "Waiting for database to be ready..."
              sleep 2
          done
          end_time=$SECONDS
          elapsed_time=$((end_time - start_time))
          echo "Database became available after ${elapsed_time} seconds."

  containers:
    # Main application container that generates logs
    - name: webapp
      image: busybox
      command:
        - sh
        - -c
        - |
          mkdir -p /var/log/app
          echo "Starting WebApp container..."
          while true; do
              echo "$(date) : Log entry from main container" >> /var/log/app/app.log
              sleep 5
          done
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    # Sidecar container to collect and stream logs
    - name: log-collector
      image: busybox
      command:
        - sh
        - -c
        - |
          echo "Starting Log Collector..."
          tail -n+1 -f /var/log/app/app.log
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

  # Shared temporary storage for logs
  volumes:
    - name: logs
      emptyDir: {}

---

# ---------------------------------------------------------
# 2. Service for the Database
# ---------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: my-database
  labels:
    app: database
spec:
  selector:
    app: database
  ports:
    - name: db-port
      port: 5432
      targetPort: 5432
  type: ClusterIP

---

# ---------------------------------------------------------
# 3. Database Pod
# ---------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: database
spec:
  containers:
    - name: database
      image: postgres
      ports:
        - containerPort: 5432
      env:
        - name: POSTGRES_PASSWORD
          value: "examplepassword"
```
```bash
kubectl apply -f webapp-with-db.yaml
kubectl get pods
kubectl logs -f webapp-with-logging -c log-collector
kubectl logs -f webapp-with-logging -c wait-for-db
kubectl exec -it webapp-with-logging -c webapp -- nc -zv my-database 5432
```

### References
- https://www.geeksforgeeks.org/devops/kubernetes-pods/
