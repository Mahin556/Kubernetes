### References:
- [Kubernetes Official Documentation: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Official Documentation: Container Images](https://kubernetes.io/docs/concepts/containers/images/)

---

![Alt text](/images/22d.png)

* A Pod goes through a series of **phases** from creation to termination. Understanding these phases is crucial for troubleshooting, monitoring, and designing applications.
* The status we see in `kubectl get pod` is the status of worst container
if 3 out of 4 container run fine and 1 give `errImagePull` that this is the also status of pod.


```bash
kubectl get pod java-api-pod
kubectl describe pod java-api-pod
```

<br>

#### **Pending**
  - **Scheduling:** Assigning the Pod to a node based on available resources, constraints, and scheduling policies.
  - **Image Pulling:** Fetching container images from the registry if they are not already present on the node.
  - Init container hasn’t completed.
  - **Issues:**  
    - Insufficient cluster resources (e.g., CPU, memory, storage).
    - Unsatisfied scheduling constraints (like node affinity, taints, etc.).
    - Image download delays or errors, such as `ErrImagePull` or `ImagePullBackOff`.
  - **Debug**
    - `kubectl describe pod <pod-name>` --> `Events` section:
      - "FailedScheduling" due to resource constraints.
      - "Failed to pull image" due to incorrect image names or registry authentication issues.
  * **Create a Pod (Pending → ContainerCreating)**
    ```bash
    # Create a simple Pod definition
    cat <<EOF > pod-lifecycle.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: lifecycle-demo
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Pod is Running"; sleep 3600']
    EOF

    # Apply the Pod
    kubectl apply -f pod-lifecycle.yaml

    # Watch Pod status change
    kubectl get pods -w
    ```
    Output:
      ```bash
      lifecycle-demo   Pending             0/1   0s
      lifecycle-demo   ContainerCreating   0/1   0s
      lifecycle-demo   Running             1/1   3s
      ```
    * Initially, the Pod is Pending until the scheduler assigns it to a node. Once started, it enters the ContainerCreating phase.

<br>

#### **ContainerCreating**  
  - Once the Pod is scheduled, Kubernetes enters the `ContainerCreating` subphase where:
  - The container runtime (e.g., Docker, containerd) sets up the container environment.
  - Storage volumes, networking, and environment variables are initialized for the container.
  - Pulling the container image (if not cached).
  - Setting up storage volumes and network configurations.
  - Initializing any environment variables or secrets.

<br>

#### **Running**
  - The Pod has been successfully scheduled to a node, and at least one container is in the `Running` state.
  - However, the overall Pod status in `kubectl get pods` reflects the **worst container state**. For example:
  - If one container is `Running` but another is in `ErrImagePull`, the Pod status will show `ErrImagePull`.
  
<br>

#### **Succeeded**
  * The Pod has completed its lifecycle, and all containers within the Pod have terminated successfully (exit code `0`).
  * No restarts will occur, as this state is terminal.
  * Batch jobs, short-lived Pods, cronjobs where containers run once and exit cleanly.
  * When the container finishes execution with exit 0, Kubernetes marks the Pod phase as Succeeded.
    ```bash
    cat <<EOF > pod-success.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: lifecycle-success
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Task complete"; exit 0']
    EOF

    kubectl apply -f pod-success.yaml
    kubectl get pods -w
    ```
  * Expected Output:
    ```bash
    NAME                READY   STATUS    RESTARTS   AGE
    lifecycle-success   0/1     Pending   0          0s
    lifecycle-success   0/1     ContainerCreating   0          1s
    lifecycle-success   0/1     Completed           0          3s
    lifecycle-success   0/1     Completed           1 (3s ago)   5s
    lifecycle-success   0/1     CrashLoopBackOff    1 (2s ago)   6s
    lifecycle-success   1/1     Running             2 (18s ago)   22s
    lifecycle-success   0/1     Completed           2 (19s ago)   23s
    lifecycle-success   0/1     CrashLoopBackOff    2 (13s ago)   35s
    lifecycle-success   0/1     Completed           3 (28s ago)   50s
    ```
  * RestartPolicy=Never/OnFailure
    * You’ll clearly see how Kubernetes restart decisions differ for each policy.
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: lifecycle-success
    spec:
      restartPolicy: OnFailure
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Task complete"; exit 0']
    ```
    ```bash
    controlplane:~$ kubectl get pods -w
    NAME                READY   STATUS              RESTARTS   AGE
    lifecycle-success   0/1     ContainerCreating   0          2s
    lifecycle-success   0/1     Completed           0          2s
    lifecycle-success   0/1     Completed           0          4s
    lifecycle-success   0/1     Completed           0          4s
    ```
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: lifecycle-success
    spec:
      restartPolicy: Never
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Task complete"; exit 0']
    ```
    ```bash
    controlplane:~$ kubectl get pods -w
    NAME                READY   STATUS      RESTARTS   AGE
    lifecycle-success   0/1     Completed   0          7s
    ```
    Change restartPolicy to:
      `Always` → Pod restarts automatically (default for Deployments)
      `OnFailure` → Pod restarts only when container exits with non-zero
      `Never` → Pod stays in Error or Completed state

<br>

#### **Failed**
  * The Pod lifecycle has ended, but at least one or more containers have failed (exited with **non-zero** exit codes).
  * Init container crashes.
  * Node failure or eviction.
  * ActiveDeadlineSeconds exceeded (Jobs).
  * This phase is also terminal, meaning the Pod won't restart (restartPolicy=Never) unless recreated by a controller(in case `crashLoopBackOff`).
  * **Trigger a Failed Phase**
    * When a any one container exits with exit 1, Kubernetes marks the Pod as Failed.
      ```bash
      cat <<EOF > pod-fail.yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: lifecycle-fail
      spec:
        containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'echo "Something went wrong"; exit 1']
      EOF

      kubectl apply -f pod-fail.yaml
      kubectl get pods -w
      ```
      ```bash
      NAME                READY   STATUS              RESTARTS   AGE
      lifecycle-fail      0/1     ContainerCreating   0          0s
      lifecycle-success   0/1     Completed           0          3m18s
      lifecycle-fail      0/1     ContainerCreating   0          1s
      lifecycle-fail      0/1     Error               0          3s
      lifecycle-fail      0/1     Error               1 (8s ago)   10s
      lifecycle-fail      0/1     CrashLoopBackOff    1 (2s ago)   11s
      lifecycle-fail      1/1     Running             2 (17s ago)   26s
      lifecycle-fail      0/1     Error               2 (18s ago)   27s
      lifecycle-fail      0/1     CrashLoopBackOff    2 (15s ago)   41s
      ```

<br>

  * **Observe Pod Failure (CrashLoopBackOff)**
    * The Pod repeatedly restarts and shows CrashLoopBackOff — Kubernetes marks it as Failed internally, but attempts restarts depending on its policy.
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: crashpod
      spec:
        containers:
        - name: crash-container
          image: busybox
          command: ["sh", "-c", "echo 'Starting...'; sleep 3; exit 1"]
          # This container exits with error code 1
      ```
      ```bash
      kubectl apply -f crashpod.yaml
      kubectl get pods -w
      ```
      ```bash
      NAME       READY   STATUS   RESTARTS   AGE
      crashpod   0/1     Error    0          9s
      crashpod   1/1     Running   1 (2s ago)   10s
      crashpod   0/1     Error     1 (6s ago)   14s
      crashpod   0/1     CrashLoopBackOff   1 (16s ago)   28s
      crashpod   1/1     Running            2 (17s ago)   29s
      ```
    * You’ll see statuses change: ContainerCreating → Running → Error → CrashLoopBackOff.
    * Inspect the logs and describe output:
      ```bash
      kubectl logs crashpod
      kubectl describe pod crashpod
      ```

<br>

  * **Liveness Probe Failure**
    * Simulate a failed health check to see how Kubernetes marks a container unhealthy.
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: liveness-demo
      spec:
        containers:
        - name: liveness-container
          image: busybox
          args:
          - /bin/sh
          - -c
          - >
            touch /tmp/healthy;
            sleep 5;
            rm -f /tmp/healthy;
            sleep 600
          livenessProbe:
            exec:
              command:
              - cat
              - /tmp/healthy
            initialDelaySeconds: 5
            periodSeconds: 5
      ```
      ```bash
      kubectl apply -f liveness-fail.yaml
      kubectl get pods -w
      ```
    * Watch it restart automatically when /tmp/healthy file is removed.
      ```bash
      kubectl describe pod liveness-demo
      ```
    * You’ll see repeated restarts and liveness probe failures (`Liveness probe failed: cat: can't open '/tmp/healthy'`).
      ```bash
      controlplane:~$ kubectl get pods -ww
      NAME            READY   STATUS              RESTARTS   AGE
      liveness-demo   0/1     ContainerCreating   0          1s
      liveness-demo   1/1     Running             0          2s
      liveness-demo   1/1     Running             1 (2s ago)   52s
      
      Events:
      Type     Reason     Age                From               Message
      ----     ------     ----               ----               -------
      Normal   Scheduled  40s                default-scheduler  Successfully assigned default/liveness-demo to node01
      Normal   Pulling    40s                kubelet            Pulling image "busybox"
      Normal   Pulled     39s                kubelet            Successfully pulled image "busybox" in 590ms (590ms including waiting). Image size: 2224358 bytes.
      Normal   Created    39s                kubelet            Created container: liveness-container
      Normal   Started    39s                kubelet            Started container liveness-container
      Warning  Unhealthy  20s (x3 over 30s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
      Normal   Killing    20s                kubelet            Container liveness-container failed liveness probe, will be restarted
      ```

<br>

  * **Combine Crash and Liveness**
    * Observe how Kubernetes differentiates between a crash failure and a liveness failure.
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: combo-fail
      spec:
        containers:
        - name: combo-container
          image: busybox
          command: ["sh", "-c", "sleep 10; exit 1"]
          livenessProbe:
            exec:
              command: ["false"]
            initialDelaySeconds: 3
            periodSeconds: 5
      ```
      ```bash
      kubectl apply -f combo-fail.yaml
      kubectl describe pod combo-fail
      ```
    * Expected Outcome:
      You’ll see that:
        * Liveness probe fails first → Pod restarted.
        * Then the process itself exits with code 1 → Pod enters crash loop.
        * Both show up as events under kubectl describe.

<br>

  * **Observe Failure with a Deployment**
    * See how higher-level controllers (Deployment) handle failed Pods automatically.
    * The Deployment continuously tries to maintain replicas=2, recreating failed Pods indefinitely — a real-world example of self-healing.
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: fail-deploy
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: fail-demo
        template:
          metadata:
            labels:
              app: fail-demo
          spec:
            containers:
            - name: fail-app
              image: busybox
              command: ["sh", "-c", "echo 'Failing...'; exit 1"]
      ```
      ```bash
      kubectl apply -f fail-deploy.yaml
      kubectl get pods -w
      ```

<br>

#### **Unknown**
  * The Pod status cannot be obtained by the API server, usually due to communication issues with the node.
  * This phase is rare and typically indicates infrastructure issues.

<br>

With RestartPolicy: Always, Kubernetes continuously restarts the container, so you'll typically see statuses like Running, Pending, or CrashLoopBackOff instead of Completed or Failed. While the pod may briefly show Failed or Completed, it will quickly transition as the kubelet restarts it. We'll observe this behavior in the demo.

---

### **Pod Status in Multi-Container Pods**
When you run `kubectl get pods`, the Pod status shown represents the **worst container's state**. Kubernetes aggregates container statuses within the Pod and prioritizes showing errors over successes. 

- **Example:**  
  If a Pod has 4 containers and:
  - 3 containers are `Running`
  - 1 container is in `ErrImagePull`
  - The overall Pod status will be `ErrImagePull`.

This behavior ensures that issues within a Pod are highlighted immediately.

---

### **Differentiation Between `Succeeded` and `Completed`**

- **`Succeeded`:**  
  - This is the **official lifecycle phase** of the Pod in Kubernetes, as described internally in the Pod's status object.
  - You’ll see `Succeeded` when running `kubectl describe pod <pod-name>`. Example output:
    ```
    Status: Succeeded
    ```

- **`Completed`:**  
  - This is the **human-readable status** displayed in the `STATUS` column when you run `kubectl get pods`.
  - Kubernetes uses "Completed" to summarize the `Succeeded` lifecycle phase for easier interpretation.

**Key Difference:**  
- `"Succeeded"` is the lifecycle phase tracked internally by Kubernetes.
- `"Completed"` is the user-friendly representation shown in the `kubectl get pods` output.


* **Demo 1: Observing Short-Lived Pod Lifecycle**
   ```bash
   kubectl run ubuntu-2 --image=ubuntu --restart=Never -- ls
   ```
   * Pending ---> ContainerCreating ---> Completed
   * The Pod did not enter the visible `Running` phase because the task (`ls` command) completed so quickly that Kubernetes transitioned the Pod directly to `Completed`.

* **Demo 2: Observing Longer Pod Lifecycle**
   ```bash
   kubectl run ubuntu-2 --image=ubuntu --restart=Never -- sleep 5
   ```
   * Pending ---> ContainerCreating ---> Running ---> Completed
   * pod enter a running state for 5 seconds.