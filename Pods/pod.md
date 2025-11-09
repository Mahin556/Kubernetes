### References
- https://www.tutorialspoint.com/kubernetes/kubernetes_pod.htm
- https://www.geeksforgeeks.org/devops/kubernetes-creating-multiple-container-in-a-pod/
- https://spacelift.io/blog/kubernetes-cheat-sheet
- https://freedium-mirror.cfd/https://medium.com/@a-dem/a-simple-guide-to-kubectl-run-de8d67aac482
- https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/
- https://kubernetes.io/docs/reference/kubectl/ *
- https://devopscube.com/kubernetes-pod/
- https://spacelift.io/blog/kubernetes-imagepullpolicy
- https://devopscube.com/kubernetes-pod/
- https://devopscube.com/kubernetes-pod-lifecycle/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://www.geeksforgeeks.org/devops/kubernetes-pods/
- https://devopscube.com/kubernetes-pod/
- https://devopscube.com/kubernetes-pod/




---
* In modern Kubernetes versions (v1.18+), `kubectl run` creates a Pod. 
* For Deployments, use `kubectl create deployment` or YAML manifests.

```bash
kubectl run nginx --image=nginx

kubectl run nginx --image=nginx --dry-run=client -oyaml > pod.yaml

kubectl run nginx --image=nginx --dry-run=client -ojson > pod.json

# Start a hazelcast pod and let the container expose port 5701
kubectl run hazelcast --image=hazelcast/hazelcast --port=5701

# Start a hazelcast pod and set labels "app=hazelcast" and "env=prod" in the container
kubectl run hazelcast --image=hazelcast/hazelcast --labels="app=hazelcast,env=prod"

# Start a nginx pod, but overload the spec with a partial set of values parsed from JSON
kubectl run nginx --image=nginx --overrides='{ "apiVersion": "v1", "spec": { ... } }'

# Start a busybox pod and keep it in the foreground, don't restart it if it exits
kubectl run -i -t busybox --image=busybox --restart=Never

# Start the nginx pod using the default command, but use custom arguments (arg1 .. argN) for that command
kubectl run nginx --image=nginx -- <arg1> <arg2> ... <argN>

kubectl get -f ./pod.yaml

# Start the nginx pod using a different command and custom arguments
kubectl run nginx --image=nginx --command -- <cmd> <arg1> ... <argN>

kubectl get pod/example-pod1 replicationcontroller/example-rc1

kubectl get pod

kubectl get pod -owide

kubectl describe pod <name>

kubectl delete pod <name>

kubectl delete pod <pod_name> -n <namespace>

kubectl delete pod -l "app=myapp" -n <namespace> #Delete multiple pods by label

kubectl delete replicaset <name> -n <namespace> #Delete a ReplicaSet (if needed)

kubectl exec -it <pod> -- /bin/sh

kubectl top pod

kubectl port-forward <pod> 8080:80

kubectl label pod <pod> team=dev

kubectl annotate pod <pod> key=value

kubectl get po -l=app.kubernetes.io/name=hello

kubectl run -i --tty --rm debug-shell --image=busybox -- sh #Automatically remove the pod after you exit.

kubectl run my-app --image=my-image --env="DB_HOST=mydbhost" --env="DB_PORT=5432"

kubectl create -f pod.yaml

kubectl run mypod --image=nginx --restart=Never

kubectl get pods -A/--all-namespaces

kubectl delete -f pod.yaml

kubectl delete pod <pod-name>

kubectl delete pods --all -A

kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh

kubectl cp ./localfile <pod-name>:/app/remote-file
kubectl cp <pod-name>:/app/remote-file ./localfile

kubectl port-forward <pod-name> 8080:80

kubectl logs <pod-name>

kubectl logs <pod-name> -c <container-name>

kubectl logs -f <pod-name>

kubectl debug -it <pod-name> --image=busybox

kubectl get pods -w

kubectl get events --sort-by=.metadata.creationTimestamp

kubectl logs -f testpod1 -c c01

yq eval '.metadata.name="nginx1"' pod.yaml | kubectl apply -f -

kubectl get events

kubectl delete pod <pod-name>

kubectl delete pod <pod-name> --force --grace-period=0 #Force immediate deletion
```
```bash
# List all Succeeded or Failed pods in a namespace
kubectl get pods --namespace <ns> --field-selector=status.phase=Succeeded,status.phase=Failed

# Delete all Succeeded or Failed pods in a namespace
kubectl delete pods --namespace <ns> --field-selector=status.phase=Succeeded,status.phase=Failed

#Delete all Succeeded or Failed pods in a all namespaces
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded,status.phase=Failed
# Use `--dry-run=client -o yaml` to preview before deleting in bulk.
```

---

### Containers
- Isolated environment
- Package an application code with it runtime, dependencies, libraries and run it in a self-contained, isolated environment.
- Container on host run/appear as a single process, but it can have multiple process running in it(but only one process will be main other will be child).
- Each container gets and ip, shared volume, CPU and memory resources.
- container is created using the linux kernel feature called namespaces and control groups(cgroup).  

### Pod
- In K8S Pods are the smallest deployable unit we does not create container directly instead we create a pod and pod have container within it.
- Abstraction over a containers
- Run actual workload.
- Pod can have 1 or more container within it, but only one container is main application container other container can be init or sidecar containers.
- Containers have shared network namespace(default) and pid name space(not default).
- In k8s pod gets a unique IP not containers, all containers within that Pod share that IP address and port space, meaning they can communicate with each other using localhost.
- 2 types of pod
  1. single container pod
  2. multi container pod
- Each Pod:
    - Contains 1 or more containers (usually 1).
    - Shares network namespace (IP, port space).
    - Can share storage volumes.
    - Same lifecycle (scheduled, started, stopped together)
- All containers within a single Pod are guaranteed to be co-located on the same worker node.
- Containers within a Pod can share storage volumes, allowing them to have a common filesystem and exchange data seamlessly.
- unmanaged pods(orphan pods):- not managed by any high level object
- managed pods:- managed by any high level object called replicasets.
- Most comman terms associated with pods.
  - rollout(to new version)
  - replicas
  - pod failover
  - pod replacement


### Pod Operating System
- A Pod in Kubernetes does not have its own OS; instead, it runs one or more containers, and each container’s OS is defined by the base image of the container.
- Example:
  - If the container image is based on Ubuntu, the pod runs processes inside Ubuntu userspace.
  - If based on Alpine, then Alpine is the OS inside the container.
- Pods rely on the Node’s kernel (Linux kernel of the worker node).
- That means containers share the kernel of the host machine but provide isolated userspaces depending on their image.
- Key takeaway:
  - Pod OS = container image’s userspace (Ubuntu, Alpine, Debian, CentOS, etc.).
  - Kernel = always the host node’s kernel.

### Controllers for Managing Pods
- ReplicationController (RC) :- Older object (legacy), Ensures a specified number of Pod replicas are running, Superseded by ReplicaSet.
- ReplicaSet (RS):- Ensures a fixed number of identical Pods are running, Handles scaling Pods up/down, Rarely used directly—usually managed by Deployments.
- Deployment:- Most common way to run applications, Manages ReplicaSets → which manage Pods, Provides(Rolling updates, Rollbacks, Scaling).
- DaemonSet:- Ensures one Pod per node (e.g., logging agent, monitoring agent, kube-proxy).

### Why Not Create Pods Directly?
- Direct Pods are not self-healing.
- If a Pod crashes or a Node fails → Pod disappears.
- controllers (higher-level objects) ensure Pods are automatically recreated and scaled.

### Examples
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```yaml
kind: Pod 
apiVersion: v1 # API version of pod
metadata:
  name: nginx1
  namespace: nginx
  labels: #we can specify any labels here to identify the pod
    env: dev
    type: frontend
spec:
  containers: #list of containers
  - name: nginx-pod
    image: nginx
    ports:
    - containerPort: 80 #list
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app.kubernetes.io/name: hello
spec:
  containers:
    - name: hello-container
      image: busybox
      command: ['sh', '-c', 'echo Hello from my container! && sleep 3600']
```
```yaml
# SINGLE CONTAINER POD
apiVersion: v1
kind: Pod
metadata:
  name: Tomcat
spec:
  containers:
  - name: Tomcat
    image: tomcat:8.0
    ports:
    - containerPort: 7500
    imagePullPolicy: Always #Always, IfNotPresent, Never
```
```yaml
# MULTI CONTAINER POD
apiVersion: v1
kind: Pod
metadata:
  name: Tomcat
spec:
  containers:
  - name: Tomcat
    image: tomcat: 8.0
    ports:
    - containerPort: 7500
    imagePullPolicy: Always
  - name: Database
    image: mongoDB
    ports:
    - containerPort: 7501
    imagePullPolicy: Always
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
  labels:
    app: web-server
    environment: production
  annotations:
    description: This pod runs the web server
spec:
  containers:
  - name: web-server
    image: nginx:latest
    ports:
    - containerPort: 80
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-with-db
  labels:
    app: my-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-webapp
  template:
    metadata:
      name: webapp-with-db
      labels:
        app: my-webapp
    spec:
      containers:
      - name: webapp
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app-srv
  labels:
    app: my-webapp-srv
spec:
  type: NodePort
  selector:
    app: my-webapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 31100   
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    ENV: dev
spec:
  containers:
    - name: nginx
      image: nginx
    - name: redis
      image: redis
    - name: nginx-second
      image: nginx:1.28.0-alpine-slim
```
```yaml
---
apiVersion: v1
kind: Pod
metadata: 
  name: web-server-pod
  labels:
    app: web-server
    environment: production
  annotations:
    description: This pod runs the web server
spec:
  containers:
  - name: web-server
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    command: ["sleep","30"]
```

### Ways to create a pod

You can create a pod in two ways:
- Imperative command(kubectl): Testing, learning, limitations.
- Declarative approach: YAML manifest, VCS-SCM

##### Imperative command
* troubleshooting, fast, simple, local deployment
```bash
kubectl run web-server-pod \
  --tty -i \
  --image=nginx:1.14.2 \
  --restart=Never \
  --port=80 \
  --labels=app=web-server,environment=production \
  --annotations description="This pod runs the web server" -- bash
```

##### Declarative YAML
* producation deployment, VCS, gitops, ci/cd
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
  labels:
    app: web-server
    environment: production
  annotations:
    description: This pod runs the web server
spec:
  containers:
  - name: web-server
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```
```bash
kubectl create -f nginx.yaml
```






### **1. Pod Update and Replacement in Kubernetes**

  ##### **a) Updating Pods**
  * Can **Update a Pod directly** (since Pods are immutable).
    ```bash
    controlplane:~$ kubectl run nginx --image=nginx
    pod/nginx created

    controlplane:~$ kubectl get pods
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          12s

    controlplane:~$ kubectl set image pod nginx nginx=nginx:1.21
    pod/nginx image updated

    controlplane:~$ kubectl get pods
    NAME    READY   STATUS    RESTARTS      AGE
    nginx   1/1     Running   1 (46s ago)   73s  #Container restarted after setting new image

    controlplane:~$ kubectl describe pod nginx
    Name:             nginx
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             node01/172.30.2.2
    Start Time:       Sun, 05 Oct 2025 19:32:51 +0000
    Labels:           run=nginx
    Annotations:      cni.projectcalico.org/containerID: ddd4034901156f5e6b7cd3c6d093649a84d9a0b9621badc5209310b02f2d73b0
                      cni.projectcalico.org/podIP: 192.168.1.4/32
                      cni.projectcalico.org/podIPs: 192.168.1.4/32
    Status:           Running
    IP:               192.168.1.4
    IPs:
      IP:  192.168.1.4
    Containers:
      nginx:
        Container ID:   containerd://bd84409ece6154c4010939d3e3aa7ff4f2d429f8df986a47575222f4e25e552e
        Image:          nginx:1.21
        Image ID:       docker.io/library/nginx@sha256:2bcabc23b45489fb0885d69a06ba1d648aeda973fae7bb981bafbb884165e514
        Port:           <none>
        Host Port:      <none>
        State:          Running
          Started:      Sun, 05 Oct 2025 19:33:26 +0000
        Last State:     Terminated
          Reason:       Completed
          Exit Code:    0
          Started:      Sun, 05 Oct 2025 19:33:00 +0000
          Finished:     Sun, 05 Oct 2025 19:33:18 +0000
        Ready:          True
        Restart Count:  1
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-p88lm (ro)
    Conditions:
      Type                        Status
      PodReadyToStartContainers   True 
      Initialized                 True 
      Ready                       True 
      ContainersReady             True 
      PodScheduled                True 
    Volumes:
      kube-api-access-p88lm:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type    Reason     Age                From               Message
      ----    ------     ----               ----               -------
      Normal  Scheduled  53s                default-scheduler  Successfully assigned default/nginx to node01
      Normal  Pulling    52s                kubelet            Pulling image "nginx"
      Normal  Pulled     45s                kubelet            Successfully pulled image "nginx" in 7.832s (7.832s including waiting). Image size: 72319283 bytes.
      Normal  Killing    26s                kubelet            Container nginx definition changed, will be restarted
      Normal  Pulling    25s                kubelet            Pulling image "nginx:1.21"
      Normal  Created    18s (x2 over 45s)  kubelet            Created container: nginx
      Normal  Started    18s (x2 over 44s)  kubelet            Started container nginx
      Normal  Pulled     18s                kubelet            Successfully pulled image "nginx:1.21" in 7.454s (7.454s including waiting). Image size: 56746739 bytes.

    controlplane:~$ kubectl set image pod nginx nginx=nginx:latest
    pod/nginx image updated

    controlplane:~$ kubectl get pods
    NAME    READY   STATUS    RESTARTS     AGE
    nginx   1/1     Running   2 (2s ago)   84s
    ```

  * Instead, you update the **controller** (Deployment/ReplicaSet/DaemonSet).
  * Kubernetes replaces old Pods with new ones in a **rolling update** fashion:
    * Creates new Pod (with updated config/image).
    * Gradually terminates old Pods.
    * Ensures no downtime.

  * Example: Update Nginx image in Deployment
    ```bash
    kubectl set image deployment/nginx-deploy nginx=nginx:1.25
    ```
  * This triggers a rolling update.


  ##### **b) Pod Replacement**
  * Pods are **ephemeral**. If:
    * A Pod **crashes**
    * A Node **fails**
    * Or you **delete the Pod**
  * The controller (like a Deployment or ReplicaSet) **automatically creates a replacement Pod** with the same spec but a new IP.
  * This gives **self-healing** capability.

  ##### **Car Analogy 🚗**
  * **Update (rolling upgrade):** like upgrading your engine software while the car is still running.
  * **Replacement (self-healing):** like swapping a flat tire with a spare while the car keeps moving.

---

### Pod Description

  ```bash
  kubectl describe pod web-server-pod
  ```
  ```bash
  Name:             web-server-pod
  Namespace:        default
  Priority:         0
  Service Account:  default
  Node:             node01/172.30.2.2
  Start Time:       Tue, 23 Sep 2025 08:06:18 +0000
  Labels:           app=web-server
                    environment=production
  Annotations:      cni.projectcalico.org/containerID: 2b0c82cc0e9b2c4f671a6e2c04f00fc57bc44f993cfae96e2d6f237deb186184
                    cni.projectcalico.org/podIP: 192.168.1.4/32
                    cni.projectcalico.org/podIPs: 192.168.1.4/32
                    description: This pod runs the web server
  Status:           Running
  IP:               192.168.1.4
  IPs:
    IP:  192.168.1.4
  Containers:
    web-server-pod:
      Container ID:   containerd://ddf086493793f41a8062e6f276af88c893a1def9870af1b144e8d438b70be1d0
      Image:          nginx:1.14.2
      Image ID:       docker.io/library/nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
      Port:           80/TCP
      Host Port:      0/TCP
      State:          Running
        Started:      Tue, 23 Sep 2025 08:06:26 +0000
      Ready:          True
      Restart Count:  0
      Environment:    <none>
      Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4hqgv (ro)
  Conditions:
    Type                        Status
    PodReadyToStartContainers   True 
    Initialized                 True 
    Ready                       True 
    ContainersReady             True 
    PodScheduled                True 
  Volumes:
    kube-api-access-4hqgv:
      Type:                    Projected (a volume that contains injected data from multiple sources)
      TokenExpirationSeconds:  3607
      ConfigMapName:           kube-root-ca.crt
      Optional:                false
      DownwardAPI:             true
  QoS Class:                   BestEffort
  Node-Selectors:              <none>
  Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
  Events:
    Type    Reason     Age   From               Message
    ----    ------     ----  ----               -------
    Normal  Scheduled  11s   default-scheduler  Successfully assigned default/web-server-pod to node01
    Normal  Pulling    10s   kubelet            Pulling image "nginx:1.14.2"
    Normal  Pulled     3s    kubelet            Successfully pulled image "nginx:1.14.2" in 7.501s (7.501s including waiting). Image size: 44710204 bytes.
    Normal  Created    3s    kubelet            Created container: web-server-pod
    Normal  Started    3s    kubelet            Started container web-server-pod
  ```

![Alt text](/images/pod--1-.png)

---

### Pod Features

##### **1. Resource Requests and Limits**
  * Control CPU and memory usage per container.
  * **Request:** Minimum resources guaranteed. The scheduler uses this to place Pods.
  * **Limit:** Maximum resources a container can consume. Prevents overuse.
  * Avoid noisy neighbor problems; ensure fair sharing of node resources.
    ```yaml
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    ```

##### **2. Labels**
  * Key-value metadata for grouping and identifying Pods.
  * **Use Case:**
    * Identify environment (`dev`, `prod`).
    * Select Pods for Services or Deployments.
    ```yaml
    metadata:
      labels:
        app: web-server
        environment: production
    ```

##### **3. Selectors**
  * Used by controllers and Services to **select Pods based on labels**.
  * **Use Case:** A Deployment with a selector ensures it manages only the intended Pods.
    ```yaml
    spec:
      selector:
        matchLabels:
          app: web-server
    ```

##### **4. Probes (Liveness, Readiness, Startup)**
  * Health checks for containers.
    * **Liveness Probe:** Detects dead containers → restarts them.
    * **Readiness Probe:** Determines if Pod is ready to serve traffic.
    * **Startup Probe:** Handles slow-starting containers without killing them.
    ```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 5

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    ```

##### **5. ConfigMaps**
  * Inject non-sensitive configuration into Pods.
  * **Use Case:** Update application config without rebuilding the container image.
    ```yaml
    volumes:
    - name: config
      configMap:
        name: app-config

    containers:
    - name: web
      image: nginx
      volumeMounts:
      - name: config
        mountPath: /etc/config
    ```

##### **6. Secrets**
  * Store sensitive data securely.
  * **Use Case:** Store DB passwords, API keys, TLS certificates.
    ```yaml
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    ```

##### **7. Volumes**
  * Provide persistent or shared storage to containers.
  * **Use Case:** Database storage, shared files between containers.
    ```yaml
    volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-1

    containers:
    - name: web
      volumeMounts:
      - mountPath: /data
        name: data
    ```

##### **8. Init Containers**
  * Run **before main containers** in sequence.
  * **Use Case:**
    * Preload data into volumes.
    * Initialize databases.
    * Run setup scripts.
    ```yaml
    initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'echo Preparing DB > /data/db.txt']
      volumeMounts:
      - name: data
        mountPath: /data
    ```

##### **9. Ephemeral Containers**
  * Temporary containers added to a running Pod for debugging.
  * **Use Case:** Debug a live Pod without restarting it.
    ```bash
    kubectl debug -it web-server-pod --image=busybox
    ```

##### **10. Service Account**
  * Control the Pod’s access to the Kubernetes API.
  * **Use Case:** Restrict which resources a Pod can read/write in the cluster.
    ```yaml
    spec:
      serviceAccountName: my-service-account
    ```

##### **11. SecurityContext**
  * Define permissions, capabilities, and host access for containers.
  * **Use Case:** Run as non-root, restrict privilege escalation, or set read-only filesystem.
    ```yaml
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      fsGroup: 2000
      allowPrivilegeEscalation: false
    ```

##### **12. Affinity & Anti-Affinity Rules**
  * Control Pod scheduling across nodes.
  * **Use Case:**
    * Co-locate Pods for performance.
    * Spread Pods across nodes for high availability.
    ```yaml
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web-server
          topologyKey: "kubernetes.io/hostname"
    ```

##### **13. Pod Preemption & Priority**
  * Decide which Pods are scheduled first or evicted during resource pressure.
  * **Use Case:** Ensure critical Pods run even if low-priority Pods are evicted.
    ```yaml
    priorityClassName: high-priority
    ```

##### **14. Pod Disruption Budget**
  * Ensure a minimum number of Pods remain available during voluntary disruptions.
  * **Use Case:** Node maintenance without affecting availability.
    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: pdb-web
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          app: web-server
    ```

##### **15. Container Lifecycle Hooks**
  * Run commands when container starts or stops.
    ```yaml
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello World"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo Cleaning up"]
    ```

##### **16. DNS Settings**
  * **dnsPolicy:** Defines default DNS behavior (`ClusterFirst`, `Default`).
  * **dnsConfig:** Customize DNS servers, search domains, or options.
    ```yaml
    dnsPolicy: "ClusterFirst"
    dnsConfig:
      nameservers:
        - 8.8.8.8
      searches:
        - mydomain.local
    ```

### Pod Restart

##### **Why Restart a Pod?**
You may want to restart a pod in situations like:
* **Applying configuration changes:** ConfigMaps, Secrets, or environment variables have been updated.
* **Debugging:** Pod is misbehaving or in an error state.
* **Pod stuck in terminating state:** Needs a reset to resume normal operations.
* **OOM errors:** Pod terminated due to resource limits.
* **Force new image pull:** Update to the latest container image.
* **Resource contention:** Free resources consumed by a misbehaving pod.
> ⚠️ Manual restarts can cause downtime or disrupt applications if not managed carefully. Use declarative or rolling update strategies whenever possible.

##### **Methods to Restart Pods**

* **1. Rolling Restart (Recommended)**
  * Use `kubectl rollout restart` to refresh pods one at a time, ensuring no downtime.
    ```bash
    kubectl rollout restart deployment <deployment_name> -n <namespace>
    ```
  * Gradually replaces pods.
  * Preserves deployment configuration.
  * Triggers automatic pickup of updated ConfigMaps or Secrets.


* **2. Scale Deployment**
  * Scaling a deployment down to 0 and back up forces pods to terminate and recreate.
    ```bash
    # Scale down
    kubectl scale deployment <deployment_name> -n <namespace> --replicas=0

    # Scale back up
    kubectl scale deployment <deployment_name> -n <namespace> --replicas=3

    # Check pod status
    kubectl get pods -n <namespace>
    ```
  * Introduces downtime; use only if application can tolerate it.


* **3. Force Replace a Pod**
  * Useful for manually created pods not managed by higher-level objects:
    ```bash
    kubectl get pod <pod_name> -n <namespace> -o yaml | kubectl replace --force -f -
    ```
  * Only use if pod is not controlled by Deployment/StatefulSet; otherwise, it may cause duplicate pods.

* **4. Update Environment Variables**
  * Changing an environment variable triggers a rolling restart:
    ```bash
    kubectl set env deployment <deployment_name> -n <namespace> DEPLOY_DATE="$(date)"
    ```
  * Safe method, no downtime.
  * Useful for picking up new ConfigMaps, Secrets, or refreshing application state.
  * Works well in automation and CI/CD pipelines.

---

### Best Practices for Restarting Pods

* **Use readiness and liveness probes** to ensure traffic only goes to healthy pods.
* **Avoid manual deletion** when possible; prefer rolling restarts.
* **Implement rolling updates** for zero downtime.
* **Configure resource limits** to prevent OOM kills.
* **Monitor CrashLoopBackOff** and investigate root causes before restarting.
* **Avoid frequent restarts**; focus on fixing underlying issues.
* **Log and monitor restarts** to track patterns and prevent recurring failures.

---

<br>

### `dnsPolicy` – DNS Behavior Policy

`dnsPolicy` defines **how the Pod’s DNS settings are configured**. It determines whether the Pod uses:

* The **cluster’s DNS service (CoreDNS)**
* The **node’s default DNS configuration**
* Or **custom settings**

<br>

There are **4 main types** of `dnsPolicy`:

<br>

**a) ClusterFirst (default for most Pods)**

* This is the **default policy** when Pods are running inside a Kubernetes cluster.
* DNS queries for **Services inside the cluster** (like `my-service.default.svc.cluster.local`) are automatically resolved by **CoreDNS**.
* For **external domains** (like `google.com`), the request is forwarded to the upstream DNS servers.

**Example:**

```yaml
dnsPolicy: ClusterFirst
```

**When used:**
✅ For almost all Pods running inside the cluster that need to access both internal and external services.

<br>

#### **b) Default**

* The Pod inherits the **DNS configuration from the node’s /etc/resolv.conf** (the host machine).
* No automatic integration with cluster DNS.

**Example:**

```yaml
dnsPolicy: Default
```

**When used:**
✅ When Pods need **exactly the same DNS behavior as the host**.
⚠️ Not recommended for normal Pods, as it bypasses CoreDNS and cluster service discovery.

<br>

#### **c) None**

* This disables all default DNS configuration.
* You must specify **dnsConfig** manually to define custom nameservers, searches, and options.
* Gives you full control over DNS behavior.

**Example:**

```yaml
dnsPolicy: None
dnsConfig:
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
  searches:
    - mycorp.local
    - example.com
  options:
    - name: ndots
      value: "2"
```

**When used:**
✅ When you want to fully control DNS settings (for testing, custom network setups, or isolated environments).


#### **d) ClusterFirstWithHostNet**

* Used when Pods run with **hostNetwork: true**, meaning they share the node’s network namespace.
* Ensures DNS queries for cluster services are still sent to CoreDNS (not host DNS).

**Example:**

```yaml
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet
```

**When used:**
✅ For Pods using host networking but still requiring access to cluster DNS.

<br>

### `dnsConfig` – Custom DNS Settings

`dnsConfig` allows you to **add or override specific DNS settings** within the Pod, regardless of the policy (in some cases).
It’s used for **customizing name servers, search domains, and resolver options**.

**Fields of `dnsConfig`:**

**a) `nameservers`**

* List of custom DNS server IPs.
* Maximum of 3 IPs allowed.
* Example:

  ```yaml
  nameservers:
    - 8.8.8.8
    - 1.1.1.1
  ```

<br>

**b) `searches`**

* Specifies search domains for DNS lookups.
* When you run `ping myservice`, Kubernetes will try:

  ```
  myservice.<namespace>.svc.cluster.local
  myservice.svc.cluster.local
  myservice.cluster.local
  myservice.mydomain.local
  myservice
  ```

  Example:

  ```yaml
  searches:
    - mydomain.local
    - example.com
  ```

<br>

**c) `options`**

* Adds resolver options from `/etc/resolv.conf`.
* Common examples:

  * `ndots`: How many dots a name must have before trying absolute resolution.
    Example: `ndots: 2`
  * `timeout`: How long to wait for a DNS query response.
  * `attempts`: Number of retries before giving up.

  Example:

  ```yaml
  options:
    - name: ndots
      value: "2"
    - name: timeout
      value: "3"
  ```

#### **Example – Full DNS Customization**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-example
spec:
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "cat /etc/resolv.conf; sleep 3600"]
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 1.1.1.1
    searches:
      - mydomain.local
      - example.com
    options:
      - name: ndots
        value: "2"
      - name: timeout
        value: "3"
```

If you run into this Pod and check:

```bash
kubectl exec -it dns-example -- cat /etc/resolv.conf
```

You’ll see:

```
nameserver 8.8.8.8
nameserver 1.1.1.1
search mydomain.local example.com
options ndots:2 timeout:3
```

<br>

---

<br>

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

