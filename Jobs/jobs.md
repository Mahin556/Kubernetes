
### What is a Job in Kubernetes?

* A **Job** is a Kubernetes workload that **runs a Pod (or multiple Pods) until the task is completed successfully**.
* Unlike Deployments (long-running apps), Jobs are meant for **short-lived, batch, or one-time tasks**.
* Once a Job finishes successfully, it will **not restart** unless you delete and re-create it.

---

### Key Features

* Ensures a task **runs to completion**.
* Can run **one Pod** or **multiple Pods in parallel**.
* Retries Pods if they fail (based on `restartPolicy`).
* Tracks how many Pods succeeded and stops once the desired count is reached.

---

### Common Use Cases

* Data processing tasks (ETL pipelines).
* Sending emails or notifications.
* Generating reports.
* Database migrations.
* One-time batch tasks.

---

### Job Lifecycle

1. You create a **Job**.
2. Kubernetes creates one or more Pods.
3. Each Pod runs until:

   * **Success** → Job records completion.
   * **Failure** → Job retries with a new Pod (if allowed).
4. Job finishes when the specified number of **successful completions** is reached.

---

### Important Fields

1. **completions**

   * Total number of Pods that must complete successfully.
   * Example: `completions: 3` → Job will finish after 3 successful Pods.

2. **parallelism**

   * Maximum number of Pods running at the same time.
   * Example: `parallelism: 2` → Run 2 Pods at once until completions are reached.

3. **backoffLimit**

   * Number of retries allowed for failed Pods. Default is 6.

4. **restartPolicy**

   * Usually `OnFailure` or `Never`.

---

### Example 1: Simple Job (runs once)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Hello from Kubernetes Job"]
      restartPolicy: Never
```

---

### Example 2: Job with Completions and Parallelism

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5       # Run 5 Pods successfully
  parallelism: 2       # Run 2 Pods at the same time
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Processing task $(date); sleep 5"]
      restartPolicy: OnFailure
```

* Kubernetes will run 2 Pods at once until 5 total successes are completed.

---

### Example 3: Job with Backoff Limit

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: fail
        image: busybox
        command: ["sh", "-c", "exit 1"]   # Always fails
      restartPolicy: OnFailure
```

* This Job will fail after retrying 3 times.

---

### Managing Jobs

* **Create**: `kubectl apply -f job.yaml`
* **List**: `kubectl get jobs`
* **Describe**: `kubectl describe job <name>`
* **Check Pods**: `kubectl get pods --selector=job-name=<job-name>`
* **Logs**: `kubectl logs <pod-name>`
* **Delete**: `kubectl delete job <name>`

---

### Job vs CronJob

* **Job** → One-time task.
* **CronJob** → Recurring scheduled task.

### References
- https://kubernetes.io/docs/concepts/workloads/controllers/job/