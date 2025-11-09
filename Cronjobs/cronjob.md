
### What is a CronJob?

* A **CronJob** in Kubernetes is a workload object that **runs Jobs on a scheduled time** (like Linux `cron`).
* It is used for **periodic and recurring tasks**, such as:

  * Database backups
  * Sending reports
  * Cleanup tasks (log rotation, temporary file cleanup)
  * Automated notifications

---

### Key Features

* Runs **Jobs** at a scheduled time, based on a **cron expression**.
* Each execution creates a new **Job** object, which then creates one or more Pods to complete the task.
* Automatically manages:

  * **Missed schedules** (with limitations)
  * **Concurrency policies** (how overlapping jobs are handled)
* Works best for **short-lived tasks** (not long-running apps).

---

### CronJob vs Job

* **Job**: Runs once and ensures completion.
* **CronJob**: Runs Jobs repeatedly on a defined schedule.
* Example:

  * Job → Run a backup once.
  * CronJob → Run a backup every day at midnight.

---

### Cron Expression Format

The schedule uses standard cron syntax (in UTC by default):

```
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of the week (0-6, Sun=0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

Examples:

* `* * * * *` → Run every minute
* `0 0 * * *` → Run daily at midnight
* `0 */6 * * *` → Run every 6 hours
* `30 2 * * 1` → Run every Monday at 2:30 AM

---

### Example CronJob Manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 0 * * *"   # Run at midnight every day
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            imagePullPolicy: IfNotPresent
            args:
            - /bin/sh
            - -c
            - "echo Backup started at $(date) && sleep 10 && echo Backup finished at $(date)"
          restartPolicy: OnFailure
```

---

### Important Fields

1. **schedule**

   * Defines when the CronJob runs (cron syntax).

2. **jobTemplate**

   * Defines the Job spec (Pod template).

3. **restartPolicy**

   * Usually `OnFailure` or `Never`.

4. **concurrencyPolicy** (controls overlapping runs):

   * `Allow` → Jobs can run concurrently (default).
   * `Forbid` → Skip a new run if the previous is still running.
   * `Replace` → Cancel the currently running job and start a new one.

   ```yaml
   concurrencyPolicy: Forbid
   ```

5. **startingDeadlineSeconds**

   * If a job misses its schedule (e.g., controller was down), it will be started within this deadline.

6. **successfulJobsHistoryLimit** & **failedJobsHistoryLimit**

   * Controls how many old jobs are kept.

   ```yaml
   successfulJobsHistoryLimit: 3
   failedJobsHistoryLimit: 1
   ```

---

### Managing CronJobs

* **Create**: `kubectl apply -f cronjob.yaml`
* **List**: `kubectl get cronjobs`
* **Describe**: `kubectl describe cronjob <name>`
* **View Jobs**: `kubectl get jobs --watch`
* **View Pods**: `kubectl get pods --watch`
* **Delete**: `kubectl delete cronjob <name>`

---

### Example in Action

1. Apply CronJob:

   ```sh
   kubectl apply -f backup-cronjob.yaml
   ```
2. Wait for execution:

   ```sh
   kubectl get jobs
   kubectl get pods
   ```
3. View logs of a Pod:

   ```sh
   kubectl logs <pod-name>
   ```

---

### Best Practices

* Use **short-lived containers** (tasks should finish quickly).
* Use **concurrencyPolicy: Forbid** for tasks like backups to avoid overlap.
* Always set **successfulJobsHistoryLimit** and **failedJobsHistoryLimit** to avoid clutter.
* Keep **startingDeadlineSeconds** to catch up on missed runs if needed.
* Use **resource requests/limits** to prevent CronJobs from overloading the cluster.

### References
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
