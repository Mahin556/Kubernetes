### References:
- https://devopscube.com/kubernetes-init-containers/
- https://devopscube.com/kubernetes-init-containers/
- https://medium.com/@manojkumar_41904/understanding-init-containers-and-sidecar-containers-in-kubernetes-ca94bec10a7b
- https://kubernetes.io/docs/concepts/workloads/pods/init-containers/ *


---

### What are Init Containers?

* **Init containers** are special containers in a Pod that **run before the main application containers start**.
* They always **run to completion** before the main containers are started.
* You can define **multiple init containers**, and they run sequentially(not in parallel) in the order they are defined.
* Each init container must complete successfully before the next one starts.
* Unlike app containers, init containers:
  * Always run **once**
  * Cannot be restarted unless the whole Pod restarts
  * Have their own image, filesystem, environment
* InitContainers are defined in a `spec.initContainers` field of a Pod’s manifest.
*  If an Init Container fails to execute successfully, the entire pod initialization fails, and the pod restarts until the Init Containers complete successfully.
* Single Responsibility: Each Init Container focuses on a specific task.
* Stages: start ---> run ---->complete,fail

---

### Why Init Containers are Useful(prepare)

Why Use Init Containers? (Real-world Use Cases)

* Dependency check, Environment setup, Secrets management, Database preparation, Cache warmup, Network wait, Security validation, Git clone / Config fetch, Separation of concerns
  - Fetching binaries, configs, or certificates before app starts. 
  - Running migrations or creating schema before app container connects.
  - DB availability
  - Cloning a Git repo
  - Retrieving secrets from Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager.
  - Pre-populating Redis, Memcached, or filesystem cache.
  - Wait until an external API, service or database is reachable before app container starts.
  - Running vulnerability scans, certificate checks, or policy validations.
  - Creating directories, setting permissions, or copying files.
  - Cloning repositories or fetching configuration data.
  - Keep the main container image lean while init containers handle setup logic.

![](https://github.com/Mahin556/K8S-artifects/blob/main/images/image-71-7.png)

---

### How Init Containers Work (Step by Step Lifecycle)
* Pod creation → Kubelet sees the Pod spec has init containers.
* Pending phase → Pod does not move to "Running" until all init containers finish.
* Execution order → Init containers run sequentially in the order listed.
  - Only one runs at a time.
  - If one fails, Kubernetes retries according to the Pod’s restartPolicy.
* Completion → Once all init containers finish, the main application containers start.
* Restart behavior → If the Pod restarts, init containers run again before app containers.
⚠️ Important differences:
  - Init containers do not support probes (livenessProbe, readinessProbe, startupProbe).
  - They share the same networking as main containers but can have different images/tools.
  - They can mount the same volumes to share files/data with the main app container.

![](https://github.com/Mahin556/K8S-artifects/blob/main/images/init-container-2.gif)

---

### Key Characteristics
* Sequential Execution: Init containers run one by one, and the next init container starts only if the previous one succeeds.
* Failure Handling: If an init container fails, Kubernetes keeps restarting((with backoff delay) the pod until it succeeds.
* No Restart After Success: Once an init container completes successfully, it never runs again (unless the pod is restarted).
* Shared Pod Volumes & Network: Init containers can write data/config into shared volumes that the main containers can later use. They also share the same networking namespace, so they can communicate via localhost.
* Each init container can have its own:
  * Image
  * Command
  * Resource limits
  * Volumes

---

### Example Use Cases
* **Fetching secrets**
  - Init container pulls secrets from a secure vault → writes to a shared volume → main app reads them.

* **Waiting for dependencies**
  - Init container pings a database or API until it’s available → only then the main app starts.

* **Data preparation**
  - Init container downloads required files or configurations → main container consumes them.

---

### Example: Basic Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'echo Waiting for service... && sleep 10']
  containers:
  - name: myapp-container
    image: nginx
    ports:
    - containerPort: 80
```

Flow:
  1. Init container runs → prints message + sleeps for 10s.
  2. After success → main `nginx` container starts.

---

### Example: Init Container for Dependency Check

* Imagine you’re deploying a microservice that depends on a PostgreSQL database:
* You don’t want the app container to start until the database is ready.
* You can use an init container like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z my-database 5432; do echo waiting for db; sleep 5; done;']
  containers:
  - name: app-container
    image: my-app:latest
    ports:
    - containerPort: 8080
```

Here:
* Init container waits for DB (`my-database:5432`) to become available.
* Once DB is ready → app container starts.

---

### Example: Sharing Data Between Init and App Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-shared-data
spec:
  volumes:
  - name: workdir
    emptyDir: {}
  initContainers:
  - name: init-download
    image: busybox
    command: ['sh', '-c', 'echo "Config from init container" > /work-dir/config.txt']
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  containers:
  - name: main-app
    image: busybox
    command: ['sh', '-c', 'cat /app/config.txt && sleep 3600']
    volumeMounts:
    - name: workdir
      mountPath: /app
```

Here:
* Init container writes a config file into a **shared volume** (`emptyDir`).
* Main app container reads the same file.

---

### Real-World Use Cases

* ✅ Wait for a **database or API** before app starts.
* ✅ Set up **config files** or environment dynamically.
* ✅ Run **database migrations** before app.
* ✅ Fetch secrets from a vault and mount them into volumes.
* ✅ Perform **filesystem permissions** setup.

---

### Monitoring Init Containers

Check pod status:

```bash
kubectl get pod myapp-pod
```

Describe pod:

```bash
kubectl describe pod myapp-pod
```

You’ll see:

* Init containers listed separately.
* Their logs accessible with:

```bash
kubectl logs myapp-pod -c <init-container-name>
```

---

### Differences: Init Containers vs App Containers

| Feature            | Init Container                   | App Container                         |
| ------------------ | -------------------------------- | ------------------------------------- |
| **Purpose**        | Setup / pre-checks               | Run the main app                      |
| **Execution**      | Sequential, must finish first    | Run in parallel (after init finishes) |
| **Restart Policy** | Pod restarts if init fails       | Can restart individually              |
| **Lifetime**       | Short-lived (one-time execution) | Long-running                          |

---

### Native SideCar containers

* In **Kubernetes 1.28+**, native sidecar support was introduced using **init containers with `restartPolicy: Always`**.
* Traditionally, init containers run **to completion** before main containers start.
* **Native sidecars** extend this concept: they **start before the main container(with init container) and continue running** for the **entire Pod lifecycle**, behaving like a persistent sidecar.

<br>

* **How to Convert an Init Container into a Native Sidecar**
  * Use the `restartPolicy: Always` field in the init container spec.
  * Without this field, the init container behaves as usual (runs once and exits).
  * This allows the container to continuously run alongside main containers.

<br>

* **Scenario:**
  * Nginx main container writes logs to a shared volume `/var/log/nginx`.
  * Fluentd logging agent needs to read these logs in real-time.

<br>

* **Pod YAML Example**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: webserver-pod
  spec:
    initContainers:
    - name: logging-agent #logging-agent sidecar container
      image: fluentd:latest
      restartPolicy: Always   # Makes this init container behave as a native sidecar
      volumeMounts:
      - name: nginx-logs
        mountPath: /var/log/nginx
    containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
      - containerPort: 80
      volumeMounts:
      - name: nginx-logs
        mountPath: /var/log/nginx
    volumes:
    - name: nginx-logs
      emptyDir: {}
  ```
  ```bash
  kubectl apply -f sidecar.yaml
  ```
  ```bash
  kubectl get pods -w
  ```
  ```bash
  controlplane:~$ kubectl get pods -w
  NAME            READY   STATUS     RESTARTS   AGE
  webserver-pod   0/2     Init:0/1   0          8s
  webserver-pod   1/2     PodInitializing   0          13s
  webserver-pod   2/2     Running           0          20s
  ```
  * You will see `2/2 containers running`:
    * `logging-agent` (native sidecar)
    * `nginx` (main container)

<br>

* **Key Properties of Native Sidecars**
  1. **Dedicated Lifecycle**
      * Runs independently of the main container.
      * Continues running throughout the Pod lifecycle.

  2. **Non-blocking Termination**
      * Native sidecars don’t block Pod termination like regular sidecars.

  3. **Lifecycle Handlers & Probes**
      * You can attach `PostStart`, `PreStop`, and probes (`readiness`, `liveness`, `startup`) for sidecar readiness.

  4. **Shared Volume Usage**
      * Works well for tasks like logging, metrics collection, or configuration watching.

---

