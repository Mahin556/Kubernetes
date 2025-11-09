### References:
- [Kubernetes Official Documentation: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Official Documentation: Container Images](https://kubernetes.io/docs/concepts/containers/images/)

---

* When a container fails immediately after start, Kubernetes doesn’t restart it infinitely fast.
* Instead, it applies an exponential backoff algorithm to prevent CPU overload and system thrashing.
* Backoff sequence (approximate):
   `10s → 20s → 40s → 80s → 160s (≈5 minutes max)`
* Each failure doubles the delay before the next restart attempt.
* After reaching 5 minutes, Kubernetes stabilizes the delay — restarts every 5 minutes until success.

* **What “CrashLoopBackOff” Means**
    * `CrashLoop` = Container repeatedly crashes (exit != 0).
    * `BackOff` = Kubernetes delaying restart per backoff policy.
* The status is CrashLoopBackOff until the container starts successfully.

* **How the Backoff Resets**
    * If the container starts successfully and stays running for 10 minutes,
    * Kubernetes resets the backoff timer to its initial value (10 seconds).
    * If it crashes before 10 minutes — the timer continues increasing.


| Restart Attempt | Time Delay       | Description                       |
| --------------- | ---------------- | --------------------------------- |
| 1st failure     | 10 sec           | Immediate retry after short delay |
| 2nd failure     | 20 sec           | Doubled delay                     |
| 3rd failure     | 40 sec           | Doubled again                     |
| 4th failure     | 80 sec           | Increasing delay                  |
| 5th+ failure    | 160 sec (≈5 min) | Max delay maintained              |
