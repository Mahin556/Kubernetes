### **Event / Signal Handler**
* When a Pod is terminated (e.g., via `kubectl delete pod` or a rolling update), **Kubernetes doesn’t kill containers immediately**.
* Kubernetes starts the termination sequence.
* Kubernetes first sends a **`SIGTERM`** signal to the main process (`PID 1`) inside each container.
* This gives the application a **grace period** (default: 30 seconds, configurable with `terminationGracePeriodSeconds`) to:
    * Stop accepting new connections.
    * Close active network connections
    * Flush logs, buffered data, or transactions
    * Clean up temporary files or resources
    * Save in-memory state to disk


* The containerized application is expected to **handle this signal** using an **event or signal handler** — a function or routine that defines what to do before shutting down.

* If the container does **not exit voluntarily** within that grace period, Kubernetes then sends a **`SIGKILL`** to **force termination** immediately.
* All remaining connections are dropped instantly.

##### **Example — Handling SIGTERM in an App**

* **Python Example**

```python
import signal
import time
import sys

def handle_sigterm(signum, frame):
    print("Received SIGTERM. Cleaning up before exit...")
    time.sleep(5)
    print("Cleanup complete. Exiting now.")
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)

print("App running...")
while True:
    time.sleep(1)
```

* When this container receives a `SIGTERM`, it runs the `handle_sigterm()` function — simulating graceful shutdown logic (like saving state or closing files).

* **In Kubernetes**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: web
    image: myapp:latest
```

When deleted:

1. Kubernetes sends `SIGTERM` → app executes cleanup handler.
2. Waits up to 30 seconds.
3. If not exited → sends `SIGKILL` to force shutdown.

* PreStop Hook
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: nginx
    image: nginx:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit"]
```
* Here, `nginx -s` quit gracefully stops NGINX by finishing active connections before exiting.