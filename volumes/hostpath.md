### References:
- [Day 27: Kubernetes Volumes | Persistent Storage | PV, PVC, StorageClass, hostPath DEMO](https://www.youtube.com/watch?v=C6fqoSnbrck&ab_channel=CloudWithVarJosh)
- [Kubernetes Documentation on hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath).
- Mounts a directory from the node directly into the pod.
- Primarily used for testing or simple workloads.
- In-tree only; there is no CSI implementation.
- **Caution:** Unsuitable for production due to security risks(malicious) and lack of scheduling guarantees.

---

* `hostPath` is a Kubernetes volume type that mounts a file or directory from the underlying node’s filesystem into a Pod.
* `hostPath` volumes are **node-specific**, meaning they are limited to the node where the Pod is scheduled. All Pods running on the same node can access and share data stored in a `hostPath` volume. However, since each node has its own independent storage, **Pods on different nodes cannot share the same `hostPath` volume**, making it unsuitable for distributed workloads requiring cross-node data sharing.
* Defined inside the Pod spec itself.
* The directory/file is created directly on the worker node’s filesystem.
* Tightly coupled → if the pod moves to another node, the data does not follow.
* This means the Pod gets direct access to the host’s actual files.
* **Where is hostPath used?**
  * Debugging or troubleshooting (mount /var/log or /etc)
  * Storing logs directly on the node
  * Device access in special workloads (GPU drivers, Kubelet plugins)
  * Injecting config files to the static pods because (static pods cannot use ConfigMaps)
  * Single-node clusters (Minikube, Docker Desktop)
  * Legacy applications needing local node paths
  * Path-based CSI driver development
  * Mount /var/log from the node → Pod can read host logs
  * Mount /etc/hosts → Pod can read system config
  * Mount /dev/null → Pod can access host device
  * Accessing host binaries (mount /usr/bin or /bin/docker)
  * Mount /var/run/docker.sock → Pod can control Docker daemon

  
* **Why is it Used?**  
  * It is used for scenarios such as debugging, accessing host-level files (logs, configuration files), or sharing specific host resources with containers.

* **Why Kubernetes Recommends Avoiding `hostPath`**  
  * Pod can access host system files(sensitive host files is `hostPath` is misconfigured) 
    * Access kubelet credentials
    * Access container runtime (Docker/Containerd)
    * Access host’s /etc, /var/lib/kubelet, /root, /proc
  * Since `hostPath` is tied to a node, it reduces flexibility in scheduling.  
  * Inject malicious files
  * Break out of container
  * Take control of the entire node
  * **Better Alternatives**: Kubernetes recommends using **PersistentVolumes (PV) with PersistentVolumeClaims (PVC)** or **local PersistentVolumes**, which offer better control and security.
  * Using hostPath for database storage.


> With `emptyDir`, all containers **within a single pod** can access the shared volume, but it is **not accessible to other pods**.  
> With `hostPath`, **any pod running on the same node** that mounts the same host directory path can access the same volume data, thus enabling **cross-pod sharing on the same node**.

---

### **Understanding `hostPath` in KIND: How Storage Works Under the Hood**

![Alt text](/images/27b.png)

When using KIND (Kubernetes IN Docker), it’s important to understand where your `hostPath` volumes are actually created — especially because Docker behaves differently across operating systems.

#### **1. On Ubuntu/Linux**

- On Linux distributions like **Ubuntu**, **Docker Engine runs natively** on the host OS.
- So when you define a `hostPath` volume, like `/tmp/hostfile`, it points directly to the **host’s actual filesystem** (i.e., your Ubuntu machine).
- The path `/tmp/hostfile` truly exists on the Ubuntu host, and the container will mount that exact path into the Pod.

#### **2. On macOS and Windows**

- Docker Engine **does not run natively** on macOS or Windows, since it requires a Linux kernel.
- Docker Desktop creates a **lightweight Linux VM** internally (via **HyperKit** on macOS or **WSL2** on Windows) to run the Docker Engine.
- KIND then runs each Kubernetes node (`control-plane`, `worker-1`, `worker-2`) as a **Docker container** inside that Linux VM.
- When you define a `hostPath` in a Pod spec (e.g., `/tmp/hostfile`), it **does not point to your macOS or Windows host filesystem**.
- Instead, it points to the **filesystem of the specific Docker container (the Kubernetes node) running that Pod**.
  
  > So, technically, the `hostPath` volume is **inside the container** representing your worker node — **not** on your macOS/Windows host, and **not even directly inside the lightweight Linux VM** used by Docker Desktop.

---

### **Key Takeaway**

| Platform      | Docker Engine Runs On | `hostPath` Points To                            |
|---------------|------------------------|-------------------------------------------------|
| Ubuntu/Linux  | Natively               | Host's actual filesystem (e.g., `/tmp/hostfile`) |
| macOS/Windows | Linux VM via Docker    | Filesystem **inside the Kubernetes node container** |

---

### **Why This Matters**

When testing `hostPath` on macOS/Windows using KIND, any file you write via the volume:
- Exists **only inside the worker node container**.
- Is **not visible** on your macOS/Windows host filesystem.
- Will be lost if the KIND cluster or node container is destroyed.

This is important when you're simulating persistent storage, as `hostPath` is not portable across nodes and shouldn't be used in production — but is often used for demos or local testing.

---

* **hostPath Volume Types**

| Type                | Meaning                    | Behavior                                  |
| ------------------- | -------------------------- | ----------------------------------------- |
| `""` (Empty string) | Default                    | No checks → Pod mounts whatever exists    |
| `DirectoryOrCreate` | Create dir if missing      | Creates directory (0755) owned by kubelet |
| `Directory`         | MUST already exist         | Pod fails if not a directory              |
| `FileOrCreate`      | Create file if missing     | Creates regular file (0644)               |
| `File`              | MUST already exist         | Pod fails if missing                      |
| `Socket`            | Must exist and be a socket | Used for docker.sock                      |
| `CharDevice`        | Must be character device   | e.g., `/dev/null`                         |
| `BlockDevice`       | Must be block device       | e.g., `/dev/sda`                          |


---

### Demo: hostPath

In this demonstration, we showcase how to use the `hostPath` volume type in Kubernetes to mount a file from the host node into a container.

We begin by creating the following Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
  - name: busybox-container
    image: busybox
    command: ["sh", "-c", "cat /data && sleep 3600"]
    # The container reads and prints the content of /data, then sleeps for 1 hour.
    volumeMounts:
    - mountPath: /data
      name: host-volume
      # Mounts the 'host-volume' to the /data path inside the container.
      # This gives the container access to the file on the host.
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/hostfile
      type: FileOrCreate
      # If /tmp/hostfile doesn't exist on the host, it is created as an empty file before being mounted.
```

> With this configuration, the file `/tmp/hostfile` on the host becomes accessible inside the container at `/data`.

Now, let’s populate the file with content using the command below:

```bash
kubectl exec -it hostpath-example -- sh -c 'echo "Hey you!!" > /data'
```

**Understanding `hostPath` Types**

Kubernetes supports several `hostPath` volume types that define how paths on the host are managed. For detailed information on supported types such as `Directory`, `File`, `Socket`, and `BlockDevice`, refer to the official documentation:

[Kubernetes Documentation - hostPath Volume Types](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath-volume-types)

---

**Verification Across Pods on the Same Node**
* Since `hostPath` uses the underlying node’s filesystem, the data can be shared between different pods **only if they are scheduled on the same node**.
* To determine where the initial pod was scheduled:

```bash
kubectl get pods -o wide
```
* Assuming the pod was created on a node named `my-second-cluster-worker2`, we can now schedule a new pod on the same node to validate data sharing:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-verify
spec:
  nodeName: my-second-cluster-worker2
  containers:
  - name: busybox-container
    image: busybox
    command: ["sh", "-c", "cat /data && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/hostfile
      type: FileOrCreate
```

To verify that the new pod can access the same file content:

```bash
kubectl exec hostpath-verify -- cat /data
```

You should see the output:
```
Hey you!!
```

This confirms that both pods are using the same file from the host node via `hostPath`.
- We can also check it bu manually scheduling it to other node and see new host path created.

---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: demo
    image: nginx
    volumeMounts:
    - name: host-storage
      mountPath: /usr/share/nginx/html

  volumes:
  - name: host-storage
    hostPath:
      path: /data/html
      type: Directory
```
`/data/html` must exits first othetwise:
```bash
kubectl describe pod hostpath-demo

Events:
  Type     Reason       Age                   From               Message
  ----     ------       ----                  ----               -------
  Normal   Scheduled    6m55s                 default-scheduler  Successfully assigned default/hostpath-demo to node01
  Warning  FailedMount  43s (x11 over 6m55s)  kubelet            MountVolume.SetUp failed for volume "host-storage" : hostPath type check failed: /data/html is not a directory
```
---
```yaml
volumes:
- name: host-storage
  hostPath:
    path: /data/myapp
    type: DirectoryOrCreate
```
- Kubernetes creates `/data/myapp`
- `0755` permissions
- owned by `root:root`
---
```yaml
hostPath:
  path: /etc/hosts
  type: File #hostPath for single files
```
---
```yaml
hostPath:
  path: /dev/sda
  type: BlockDevice #Mount block device into Pod
```
---
```yaml
hostPath:
  path: /dev/null
  type: CharDevice
```
---
```yaml
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket #hostPath for Unix socket (Docker socket example)
```
- This allows Pod to control the host Docker daemon.
- Used by:
  - Jenkins agents
  - Build tools
  - Docker-in-Docker alternatives
- With this, the Pod can:
  - Create containers on the host
  - Delete host containers
  - Break out of isolation
---
```bash
ssh user@worker-node

mkdir /data/test
echo "Hello from host" > /data/test/index.html
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: myvol
      mountPath: /usr/share/nginx/html
  volumes:
  - name: myvol
    hostPath:
      path: /data/test
      type: Directory
```
```bash
kubectl exec -it test-hostpath -- bash
cat /usr/share/nginx/html/index.html
exit
```
```bash
echo "Updated!" >> /data/test/index.html
cat /usr/share/nginx/html/index.html

kubectl exec -it test-hostpath -- bash
cat /usr/share/nginx/html/index.html
exit
```
---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example-linux
spec:
  os:
    name: linux          # Ensures POD is scheduled only on Linux
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: example-container
    image: registry.k8s.io/test-webserver
    volumeMounts:
    - mountPath: /foo
      name: example-volume
      readOnly: true      # Important for security
  volumes:
  - name: example-volume
    hostPath:
      path: /data/foo      # MUST exist
      type: Directory      # Optional but recommended
```
---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  os: { name: linux }
  nodeSelector:
    kubernetes.io/os: linux

  containers:
  - name: test-webserver
    image: registry.k8s.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa       # Directory mount
      name: mydir
    - mountPath: /var/local/aaa/1.txt # File mount
      name: myfile

  volumes:
  - name: mydir
    hostPath:
      path: /var/local/aaa
      type: DirectoryOrCreate

  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```
---
```bash
cat << EOF | kubectl apply -f -
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain #data is preserved even after PVC deletion
  storageClassName: hostpath-sc
  hostPath:
    path: /data/hostpath-demo
    type: DirectoryOrCreate
EOF
```
```bash
cat << EOF | kubectl apply -f -
# hostpath-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hostpath-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: hostpath-sc
EOF
```
```bash
cat << EOF | kubectl apply -f -
# hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  nodeName: controlplane
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: myvol

  volumes:
  - name: myvol
    persistentVolumeClaim:
      claimName: hostpath-pvc
EOF
```
```bash
ssh <node>

echo "Hello from the node!" > /data/hostpath-demo/index.html

kubectl exec -it hostpath-pod -- bash
cat /usr/share/nginx/html/index.html

curl localhost
```
---
### hostpath as PV
* Using hostPath inside a PersistentVolume (PV) allows you to treat a directory on a node as Kubernetes persistent storage.
* Not recommended for production clusters (reasons explained below).
  * Node failure = data loss
  * Node dependency
  * Security issues
* Used
  * Single-node clusters (minikube, k3s)
  * Labs and demos
  * Testing stateful apps
  * Bare-metal PoCs
  * Accessing node tools (logs, config files)
* PV created using hostpath ar backed by file/directory.
* PV bound to node ---> Only pods on that nodes can access the PV.
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain #data preserved even after PVC deletion
  storageClassName: hostpath-sc
  hostPath:
    path: /data/hostpath-demo
    type: DirectoryOrCreate
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hostpath-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: hostpath-sc
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-using-hostpath
spec:
  nodeSelector:
    kubernetes.io/hostname: node01   # <-- Must match your node name otherwide FailedScheduling: binding volumes: context deadline exceeded
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: hostpath-pvc
```
```bash
ssh node01
sudo mkdir -p /data/hostpath-demo
echo "Hello from the node!" > /data/hostpath-demo/index.html

kubectl exec -it app-using-hostpath -- bash
cat /usr/share/nginx/html/index.html

kubectl exec -it app-using-hostpath -- curl localhost
```
---
* `hostpath-sc` is just a name you give to a StorageClass for hostPath-based PersistentVolumes.
* hostPath PV does not require a StorageClass.
* If you manually create the PV, you can remove the StorageClass entirely:
* But both pv and pvc
  * SAME accessModes
  * SAME or larger capacity in PV
  * NO storageClassName in BOTH PV and PVC
  * PV must be Available and not bound
  * PVC must NOT define a storageClassName

* Create a PV manifest and include the claimRef field, specifying the name and namespace of the PVC it should bind to.
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: my-pv
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain # Or Delete, depending on your needs
    hostPath:
      path: "/mnt/data" # Replace with your actual storage path
    claimRef:
      name: my-pvc
      namespace: default # Replace with the namespace of your PVC
  ```
* Create a PVC manifest and include the volumeName field, specifying the name of the PV it should bind to.
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-pvc
    namespace: default # Replace with the namespace where the PVC will be created
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
    volumeName: my-pv
  ```
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: lvm-test
  spec:
    containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep infinity"]
      volumeMounts:
      - name: data
        mountPath: /data
    volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
  ```
* By setting both of these fields, you create a direct, one-to-one relationship between the PV and PVC, bypassing the need for a StorageClass-based matching process. Kubernetes will then bind these two objects together if their other specifications (like access modes and capacity) are compatible.

---
* what is `hostpath-sc`?
  * A custom name
  * Used only to match PV and PVC
  * Does NOT provide provisioning
  * Not required for manual PV
* You do not need a StorageClass for manual hostPath PV.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
---
#### Force a hostPath PV to a Specific Node
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: hostpath-sc
  nodeAffinity:   # <-- PIN THE PV TO A NODE
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
            - worker1  # <-- CHANGE THIS TO YOUR NODE NAME
  hostPath:
    path: /data/hostpath-demo
    type: DirectoryOrCreate
```

