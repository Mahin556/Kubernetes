### References:-
- https://learn.kubernetes.anynines.com/advanced-kubernetes/pod-restart-policy/#:~:text=A%20Pod's%20restartPolicy%20determines%20what,all%20containers%20in%20the%20Pod.
- https://kubernetes.io/blog/2025/08/29/kubernetes-v1-34-per-container-restart-policy/
- https://decisivedevops.com/kubernetes-pod-policies-restartpolicy-8bd465effb30/
- https://kodekloud.com/community/t/container-restart-vs-pod-restart/42608/2
- https://www.gremlin.com/blog/how-to-ensure-your-kubernetes-pods-and-containers-can-restart-automatically

---

* `restartPolicy` define what happen when pod exits.
* Pod can exit with two conditions (completion,failed).
* 3 Types of restart policy
* Apply on pod level but work on containers.

![Alt text](/images/22b.png)

#### `Always`

  * Always restart not depend on what exit status/code is, restart the all container in pod when container stop.
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: example-pod
    spec:
      restartPolicy: OnFailure   # ✅ Can be Always | OnFailure | Never
      containers:
      - name: busybox
        image: busybox
        command: ["sh", "-c", "echo Hello && exit 1"]
    ```
  * Deployments, ReplicaSets, DaemonSets and StatefulSets allow only Always for the restartPolicy(.spec.template.spec.restartPolicy equal to Always is allowed). Refer this documentation https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pod-template
  * Controllers like Deployments and DaemonSets are designed for long-running applications. Their job is to ensure Pods are always running. If a container exits (success or failure), the kubelet restarts it immediately. The controller manages scaling and replacement — not one-shot jobs.
  * Create a Deployment with a Non-Allowed Policy
    ```bash
    cat <<EOF > invalid-restart.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: invalid-restart-demo
    spec:
        replicas: 1
        selector:
        matchLabels:
            app: restart-demo
        template:
        metadata:
            labels:
            app: restart-demo
        spec:
            restartPolicy: OnFailure    # ❌ Not allowed for Deployments
            containers:
            - name: busybox
            image: busybox
            command: ["sh", "-c", "echo 'Running...'; exit 1"]
    EOF

    kubectl apply -f invalid-restart.yaml
    ```
    You’ll get a warning or error similar to:
    ```bash
    The Deployment "invalid-restart-demo" is invalid: spec.template.spec.restartPolicy: Unsupported value: "OnFailure": supported values: "Always"
    ```
    Fix and Reapply
    ```bash
    # Change restartPolicy to Always
    sed -i 's/OnFailure/Always/' invalid-restart.yaml
    kubectl apply -f invalid-restart.yaml
    kubectl get pods -w
    ```
    ```bash
    kubectl describe pod -l app=restart-demo
    ```

#### `onFailure`
    * Restart when exit status/code it failed/Non-zore.
    * Used in **Jobs** or **CronJobs**, particularly for batch tasks or one-time processes that may fail and require retries to complete successfully.
    - **Example:**  
        A data processing job that processes input files might fail if a file is missing or corrupted. With `OnFailure`, the job will be retried to handle transient errors.
        ```yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
            name: data-processing-job
        spec:
            backoffLimit: 3       # Retry up to 3 times on failure
            template:
            spec:
                restartPolicy: OnFailure   # ✅ Restart only when the container exits with a non-zero code
                containers:
                - name: data-processor
                image: python:3.10
                command: ["python", "-c"]
                args:
                    - |
                    import os
                    import sys

                    filename = "/data/input.txt"
                    if not os.path.exists(filename):
                        print("❌ Input file missing!")
                        sys.exit(1)   # Non-zero exit triggers retry
                    print("✅ Processing complete.")
                    sys.exit(0)       # Zero exit = no retry
        ```

* `Never` --> Never restart
    * Used for one-time diagnostics, debugging, or exploratory pods where the goal is to observe the pod's behavior or logs post-exit.
    * The container is **not restarted**, regardless of the exit code (whether it succeeds or fails).
    * Used case:
        * Diagnostic Pods (e.g., DNS, API reachability, or storage checks).
        * Debugging ephemeral environments.
        * CI/CD pipelines where logs or exit codes are analyzed externally.
        * One-time analysis or data extraction tasks.

    * A simple Pod that checks cluster DNS resolution and then exits:
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: dns-check
        spec:
          restartPolicy: Never
          containers:
          - name: dns-checker
            image: busybox
            command: ["sh", "-c", "nslookup kubernetes.default.svc.cluster.local || echo 'DNS failed'"]
        ```
    ```bash
    kubectl run dns-check --image=busybox --restart=Never -- nslookup kubernetes.default
    kubectl logs dns-check
    ```

- **Deployments**: Ensures pods are always running as per the desired replicas.  
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: example-deployment
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          restartPolicy: Always   # ✅ Must be Always (default)
          containers:
          - name: nginx
            image: nginx:latest
    ```

- **ReplicaSets**: Provides fault tolerance and ensures the specified number of replicas.  
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: example-rs
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          restartPolicy: Always   # ✅ Must be Always
          containers:
          - name: nginx
            image: nginx:latest
    ```

- **ReplicationController**: Provides fault tolerance and ensures the specified number of replicas.  
    ```yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: example-rc
    spec:
      replicas: 2
      selector:
        app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          restartPolicy: Always   # ✅ Must be Always
          containers:
          - name: nginx
            image: nginx:latest
    ```
- **StatefulSets**: Manages stateful applications with persistence and ordered updates.  
    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: example-statefulset
    spec:
      serviceName: "web"
      replicas: 2
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          restartPolicy: Always   # ✅ Must be Always
          containers:
          - name: nginx
            image: nginx:latest
    ```


- **DaemonSets**: Ensures one pod per node is always running for background tasks.
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: example-daemonset
    spec:
      selector:
        matchLabels:
          app: node-monitor
      template:
        metadata:
          labels:
            app: node-monitor
        spec:
          restartPolicy: Always     # ✅ Must be Always
          containers:
          - name: node-exporter
            image: prom/node-exporter:latest
    ```

* **Job** — only OnFailure or Never
    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: example-job
    spec:
      template:
        spec:
          restartPolicy: OnFailure    # ✅ Only OnFailure or Never
          containers:
          - name: job-container
            image: busybox
            command: ["sh", "-c", "echo Processing && exit 0"]
        ```
* **CronJob** - Schedule Jobs to run periodically (e.g., every night or every hour).
    ```yaml
    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: cleanup-cronjob
    spec:
      schedule: "0 2 * * *"   # Every day at 2 AM
      jobTemplate:
        spec:
          backoffLimit: 3
          template:
            spec:
              restartPolicy: OnFailure
              containers:
              - name: cleanup
                image: alpine
                command: ["sh", "-c", "rm -rf /tmp/*"]
    ```

* **

* In a typical web application or API server where uptime is critical, using `Always` ensures the pod remains perpetually running. Kubernetes automatically replaces failed containers to minimize downtime.
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-server-deployment
      labels:
        app: web-server
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: web-server
      template:
        metadata:
          labels:
            app: web-server
        spec:
        # For Deployments, this must be Always (default)
          restartPolicy: Always
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80

        # 🩺 Liveness Probe
        # Checks if the app is still healthy and responsive.
            livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3

        # 🚦 Readiness Probe
        # Ensures traffic is only sent to ready pods.
            readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3

        # ⚙️ Resource Requests & Limits
            resources:
              requests:
                cpu: "100m"
                memory: "128Mi"
              limits:
                cpu: "250m"
                memory: "256Mi"
    ```

```bash
kubectl run -i --tty busybox --image=busybox --restart=Never -- sh #Rune once then never restart

kubectl run -i --tty busybox --image=busybox --restart=Always -- sh #Container run and restart multiple times
```
* In man Linux/Unix systems a non-zero exit values sugggests that the process has failed. Based on this, Kubernetes interprets container processes existing with non-zero values to be failed, too.
```bash
kubectl run -i --tty busybox --image=busybox --restart=OnFailure -- sh -c "exit 1" #Restart when fail
```



### **Summary Table:**

| **Restart Policy** | **Use Case**                                     | **Default Kubernetes Objects**                        | **Details**                                                                                  |
|--------------------|-------------------------------------------------|-----------------------------------------------------|----------------------------------------------------------------------------------------------|
| **Always**         | **Continuous availability and scalability**          | **Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Standalone Pods (Manual)** | - Ensures Pods are **restarted indefinitely**, regardless of exit codes.<br>- Suitable for **long-running workloads** such as web servers or background tasks.<br>- **Default** for objects designed to maintain **high availability and fault tolerance**.<br>- **Example**: Deployments use ReplicaSets, which always manage Pods with `restartPolicy: Always`. |
| **OnFailure**      | **Batch jobs or retrying failed tasks**              | **Jobs, CronJobs, Standalone Pods**                     | - Restarts containers **only if they fail** (non-zero exit code).<br>- Stops if the container exits successfully.<br>- **Default** for finite workloads such as batch processing and scheduled jobs.<br>- **Example**: Jobs are designed to retry failed tasks but exit cleanly on success. |
| **Never**          | **One-off diagnostics or exploratory tasks**         | **Not default for any object**                         | - Containers are **not restarted**, regardless of the exit code.<br>- Must be **explicitly set**.<br>- Suitable for **one-off tasks** or debugging scenarios where restarts are undesirable.<br>- **Example**: Running diagnostic commands in standalone Pods without restarts. |