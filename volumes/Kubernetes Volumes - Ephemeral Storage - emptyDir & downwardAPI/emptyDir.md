### References:-
- [Day 26: Kubernetes Volumes | Ephemeral Storage | emptyDir & downwardAPI DEMO](https://www.youtube.com/watch?v=Zyublb8bSbU&ab_channel=CloudWithVarJosh)
- https://www.devopsschool.com/blog/kubernetes-volume-emptydir-explained-with-examples/
- - [Kubernetes Volumes - emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)

---

- Ephemeral storage refers to storage that is temporary, meaning that any data written to it only lasts for the duration of the Pod’s lifetime. When the Pod is deleted, the data is also lost.
- Enable containers within a Pod to share temporary storage(Scratch space within a Pod).

![Alt text](/images/26a.png)

**What is EmptyDir?**  
  - `emptyDir` is an empty directory created when a Pod is assigned to a node. The directory exists for the lifetime of the Pod.
  - If you define an `emptyDir` volume in a Deployment, each Pod created by the Deployment will have its **own unique** `emptyDir` volume. Since `emptyDir` exists **only as long as the Pod is running**, any data stored in it is **lost when the Pod is terminated or deleted**.  
  - It is shared by each and every container in the pod a long as each container mount that volume.
  - Defining an `emptyDir` in a Deployment ensures that **each Pod replica gets its own isolated temporary storage**, independent of other replicas. This is particularly useful for workloads where each Pod requires **scratch storage** that doesn’t need to persist beyond the Pod’s lifecycle.
  - It is fast because it is internal to the node. 
  - Using `emptyDir` enables inter-container file sharing within a Pod.
  - An emptyDir volume is first created when a Pod is assigned to a Node and initially its empty
  - If a container in a Pod crashes the emptyDir content is unaffected.
  - All containers in a Pod share use of the emptyDir volume .
  - Each container can independently mount the emptyDir at the same / or different path.
  - When a Pod is removed from a node for any reason, the data in the emptyDir is deleted forever along with the container.
  - A Container crashing does NOT remove a Pod from a node, so the data in an emptyDir volume is safe across Container crashes.
  - By default, emptyDir volumes are stored on whatever medium is backing the node – that might be disk or SSD or network storage.

**Why is it Used?**  
  It is ideal for scratch space (Temporary workspace for processing.), checkpointing a long computation for recovery from crashes , caching, or temporary computations where data persistence isn’t required.
  It is on the host so it is fast accessible then the external storage.

**Writable vs Read-Only Layers in Containers**
    If a Pod has **three containers** that all use the **same container image**, they **share the read-only image layer** to save space and optimize resource usage. However, **each container still gets its own writable layer**. So, any file changes or modifications done by one container remain **isolated** and are **not visible to the others**.
    But what if you want them to share data?
    That’s where **volumes** come into play—specifically, an `emptyDir` volume. If you mount the **same `emptyDir` volume** into all three containers, they now have access to a **shared writable space**. This allows them to **collaborate and share data** during the Pod’s lifetime, effectively **creating a shared writable layer** across containers.

**When Not to Use `emptyDir`**
* You need data to survive Pod restart, eviction, or rescheduling
    * `emptyDir` is deleted when the Pod is deleted or rescheduled.
    * If you need durable data → use a PVC (PersistentVolumeClaim).
    * Use instead:
      * PVC + storage class (EBS, NFS, Ceph, CSI drivers)

<br>
    
* Data is too large to fit on node’s local disk or RAM
  * emptyDir uses:
    * Node’s disk OR
    * Node’s memory (medium: "Memory")
  * If data may exceed node storage → use external storage.
  * Use instead:
    * S3 / GCS / Azure Blob
    * NFS
    * Ceph
    * Longhorn
    * CSI-backed volumes

<br>

* Workloads that must preserve data across deployments
  * StatefulSets, databases, message queues, ML models that need persistence
  * `emptyDir` is not appropriate
  * Use instead:
    * PVC (ReadWriteOnce / ReadWriteMany)

<br>

* Multi-pod shared storage
  * `emptyDir` cannot be shared between Pods
  * Only containers inside the same Pod can share it
  * Use instead:
    * PVC with RWX storage
    * NFS / CephFS / EFS

<br>

* Sensitive data requiring encryption at rest
  * `emptyDir` relies on node filesystem → no guaranteed encryption
  * Prefer PVCs backed by cloud-native encrypted storage

---

| Use Case                  | Storage Type | Persistence | Best Medium |
| ------------------------- | ------------ | ----------- | ----------- |
| Temporary scratch space   | Disk         | No          | `""`        |
| Shared between containers | Disk         | No          | `""`        |
| Log sharing               | Disk         | No          | `""`        |
| Caching                   | RAM          | No          | `"Memory"`  |
| Build workspace           | Disk         | No          | `""`        |
| Shared memory (IPC)       | RAM          | No          | `"Memory"`  |
| Staging uploads           | Disk         | No          | `""`        |

---

* **Where is emptyDir stored?**
  * Stored on node’s filesystem under:
    ```bash
    /var/lib/kubelet/pods/<pod-id>/volumes/kubernetes.io~empty-dir/<volname>/
    ```
    ```yaml
    emptyDir: {}
    ```
    * The {} at the end means we do not supply any further requirements for the emptyDir .
    ```yaml
    emptyDir:
      medium: ""
    ```
  * In RAM (tmpfs)
    * Uses node memory — extremely fast.
    ```yaml
    emptyDir:
      medium: "Memory"
      sizeLimit: "1Gi"
    ```

---
* **When to Use `emptyDir` (12 Real Use Cases)**
  * **Temporary scratch space**
    * ETL jobs
    * ML pipelines
    * Image/video processing

* **Shared storage between containers (sidecar pattern)**
    * Logger sidecar
    * Uploader sidecar
    * Proxy container

* **Buffering between app and sidecar**
    * Nginx writes logs → sidecar uploads to S3
    * App logs → Fluentd/Vector reads

* **Caching**
    * Package manager cache
    * ML intermediate cache
    * API response cache

* **Huge speed with RAM**
    * tmpfs
    * `/dev/shm`
    * computational scratch

* **Download/extraction workspace**
    * CI builds
    * Image processing
    * Data ingestion

* **CI/CD build workspace**
    * Tekton
    * Jenkins agents
    * Argo Workflows

* **Backup staging**
    * App writes → emptyDir
    * Sidecar uploads → PVC/S3

* **Decoupling temp data from PVC**
    * Don’t pollute persistent storage
    * Store only clean final output in PVC

* **CrashLoopBackOff recovery (container restarts but pod stays)**
    * Use emptyDir for:
      * lock files
      * partial progress

* **High-speed shared memory (`/dev/shm`)**
    * Chrome
    * PostgreSQL
    * TensorFlow/PyTorch

* **Serving temporary content**
    * Cache generated HTML files
    * Short-lived web artifacts

---

  ```yaml
  apiVersion: v1  # Specifies the API version used to create the Pod.
  kind: Pod       # Declares the resource type as a Pod.
  metadata:
    name: emptydir-example  # Name of the Pod.
  spec:
    containers:
    - name: busybox-container  # Name of the container inside the Pod.
      image: busybox           # Using the lightweight BusyBox image.
      command: ["/bin/sh", "-c", "sleep 3600"]  # Overrides the default command. Keeps the container running for 1 hour (3600 seconds).
      volumeMounts:
      - mountPath: /data       # Mount point inside the container where the volume will be accessible.
        name: temp-storage     # Refers to the volume defined in the `volumes` section below.
    - name: busybox-container-2  # Name of the container inside the Pod.
      image: busybox           # Using the lightweight BusyBox image.
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeMounts:
      - mountPath: /data
        name: temp-storage
    volumes:
    - name: temp-storage       # Name of the volume, must match the name in `volumeMounts`.
      emptyDir: {}             # Creates a temporary directory that lives as long as the Pod exists.
                              # Useful for storing transient data that doesn't need to persist.

  ```
  *Here, `/data` is an emptyDir volume that will be removed along with the Pod.*

  ```bash
  kubectl apply -f emptydir-example.yaml
  ```
  ```bash
  kubectl exec emptydir-example -c busybox-container -- sh -c 'echo "What a file!" > /data/myfile.txt'
  ```
  ```bash
  kubectl exec emptydir-example -c busybox-container-2 -- cat /data/myfile.txt
  ```
  ```
  What a file!
  ```
  ```bash
  ssh node01

  node01:~$ crictl ps
    CONTAINER           IMAGE               CREATED             STATE               NAME                  ATTEMPT             POD ID              POD                        NAMESPACE
    e630950cd27ed       08ef35a1c3f05       50 seconds ago      Running             busybox-container-2   0                   8e11541f662f7       emptydir-example           default
    cd3ac827e0f3e       08ef35a1c3f05       51 seconds ago      Running             busybox-container     0                   8e11541f662f7       emptydir-example           default
    af0da552dd4d4       52546a367cc9e       10 minutes ago      Running             coredns               1                   6ee72e9892472       coredns-76bb9b6fb5-4z2mz   kube-system
    d2af2863f6b20       52546a367cc9e       10 minutes ago      Running             coredns               1                   1887b08aafb0b       coredns-76bb9b6fb5-7vqcc   kube-system
    97b6dc83ec59a       e6ea68648f0cd       10 minutes ago      Running             kube-flannel          1                   01155d711b5f4       canal-kc8m5                kube-system
    4d99e6daea095       75392e3500e36       10 minutes ago      Running             calico-node           1                   01155d711b5f4       canal-kc8m5                kube-system
    b1001781f396e       fc25172553d79       11 minutes ago      Running             kube-proxy            1                   8b504e477aaeb       kube-proxy-kgl6s           kube-system

  node01:~$ ls /var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/
    myfile.txt

  node01:~$ cat /var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/myfile.txt 
    What a file!
  ```
  ```bash
  controlplane:~$ kubectl delete pod emptydir-example
    pod "emptydir-example" deleted from default namespace

  node01:~$ ls /var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/
    ls: cannot access '/var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/': No such file or directory
  
  node01:~$ cat /var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/myfile.txt 
    cat: /var/lib/kubelet/pods/74d5370f-076d-4829-b1e2-a99d4093d8a7/volumes/kubernetes.io~empty-dir/temp-storage/myfile.txt: No such file or directory
  ```

* **Types of `emptyDir`**
  ```yaml
  emptyDir:
    medium: ""        # default → stored on node disk
  ```
  or
  ```yaml
  emptyDir:
    medium: "Memory"  # stored in RAM (tmpfs)
  ```
  * `medium: ""` → Disk-based storage (default)
  * `medium: "Memory"` → RAM-based (faster, volatile)

---
* `tmpfs` is a temporary file system that stores data in volatile memory (RAM) instead of a disk, making it much faster. It appears as a normal file system but is temporary, with all data lost after a reboot or unmount. It is used for temporary files in places like /tmp, /run, and /dev/shm, which can improve performance and is also used for temporary storage in containers. 
* Common uses
  * `/tmp`: A common location for storing temporary files created by users and applications, as seen on Super User.
  * `/run`: Stores runtime data, such as process IDs, for programs that are running, which is only relevant while the system is active.
  * `/dev/shm`: Used for POSIX shared memory, a way for different processes to share data.
  * `Containers`: Provides ephemeral storage for containers, as shown in AWS Builder Center and Docker Docs. 

* since K8s `v1.22` you can set sizeLimit (feature SizeMemoryBackedVolumes) to get a predictable tmpfs size. Otherwise use container memory limits or a privileged tmpfs mount as a fallback.
* Default tmpfs size historically ≈ 50% of node RAM (may vary).
* Kubernetes v1.22+: sizeLimit for memory-backed emptyDir is supported (feature enabled by default).
    * `sizeLimit` on memory-backed emptyDir -> hard limit enforced by kernel (writes fail with ENOSPC).
    * `sizeLimit` on disk-backed emptyDir -> soft (eviction-based), not a strict quota.

---
* Memory-backed emptyDir with explicit sizeLimit (K8s ≥1.22)
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: tmpfs-size-demo
  spec:
    containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
      - name: ram-cache
        mountPath: /cache
    volumes:
    - name: ram-cache
      emptyDir:
        medium: "Memory"
        sizeLimit: "1Gi"        # hard 1Gi tmpfs cap (kernel-enforced)
  ```
* Use container memory limit to bound tmpfs (works without sizeLimit)
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: tmpfs-by-container-limit
  spec:
    containers:
    - name: app
      image: busybox
      resources:
        limits:
          memory: 2Gi           # tmpfs will not exceed this for the container
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
      - name: ram-cache
        mountPath: /cache
    volumes:
    - name: ram-cache
      emptyDir:
        medium: "Memory"
  ```
* Fallback (privileged tmpfs mount inside container)
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: tmpfs-privileged
  spec:
    containers:
    - name: app
      image: quay.io/buildah/stable:v1.23.1
      command: ["/bin/sh","-c"]
      args:
        - mkdir -p /var/lib/containers &&
          mount -t tmpfs -o size=1G tmpfs /var/lib/containers &&
          sleep infinity
      securityContext:
        privileged: true
  ```
  ```bash
  kubectl exec -it <pod> -- df -h /cache

  #Test ENOSPC (if limit small):
  kubectl exec -it <pod> -- sh -c "dd if=/dev/zero of=/cache/bigfile bs=1M count=1024" 
  #If above exceeds tmpfs, you’ll get dd: write error: No space left on device.

  kubectl exec -it <pod> -- sh -c "free -h && df -h /cache"
  ```
---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```
---

<br>

* **Multiple containers in the same Pod can **share files** via a common `emptyDir`**
    ```yaml
    containers:
    - name: producer
      image: busybox
      command: ["/bin/sh", "-c", "echo 'data' > /data/file"]
      volumeMounts:
      - name: shared
        mountPath: /data
    - name: consumer
      image: busybox
      command: ["/bin/sh", "-c", "cat /data/file"]
      volumeMounts:
      - name: shared
        mountPath: /data
    volumes:
    - name: shared
      emptyDir: {}
    ```
* **Example: Nginx + Log-Uploader Sidecar Using EmptyDir**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-with-sidecar
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
          # Nginx will write access.log and error.log here

      - name: log-uploader
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
        - |
            echo "Starting log uploader...";
            while true; do
            if [ -f /logs/access.log ]; then
                echo "Uploading logs...";  
                # Example upload command (replace as needed)
                cat /logs/access.log;
                # aws s3 cp /logs/access.log s3://mybucket/logs/;
                > /logs/access.log;
            fi
            sleep 10;
            done
        volumeMounts:
        - name: shared-logs
          mountPath: /logs
          # Sidecar reads logs from same shared volume

      volumes:
      - name: shared-logs
        emptyDir: {}
    ```
    * Real-World Variations You Can Use
      * Replace busybox sidecar with:
        * `fluentd`
        * `vector`
        * `logstash`
        * `custom Python uploader`
      * Sync logs to:
        * `S3`
        * `ElasticSearch`
        * `Splunk`
        * `Kafka`

* **Example: App Using In-Memory Cache (emptyDir.medium: Memory)**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: app-with-cache
  spec:
    containers:
      - name: my-app
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Using in-memory cache...";
            echo "Caching data..." > /cache/data.txt;
            sleep 3600;   # simulate long-running app
        volumeMounts:
          - name: cache-volume
            mountPath: /cache
            # app writes & reads cached data here

    volumes:
      - name: cache-volume
        emptyDir:
          medium: "Memory"     # Store data in RAM (tmpfs)
          sizeLimit: "1Gi"     # Optional limit
  ```
* Common scenarios:
  * Caching frequently accessed files
  * Temporary computation results
  * ML model intermediate outputs
  * Build or compilation caches
  * Web server or API response cache
  * Sorting, indexing, or buffering workloads
* `emptyDir.medium: Memory:` stored in RAM — very fast, like tmpfs
  * Best for high-performance workloads
  * Data disappears if Pod is deleted or node reboots

* **Example: App Stages Data in emptyDir → Sidecar Uploads to S3**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: backup-staging-pod
  spec:
    containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Creating backup files...";
            echo "Backup content at $(date)" > /staging/backup-$(date +%s).txt;
            echo "Backup done. Waiting..."
            sleep 3600;
        volumeMounts:
          - name: staging-area
            mountPath: /staging
        # App writes data to /staging (emptyDir)

      - name: uploader
        image: amazon/aws-cli
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Uploader started...";
            while true; do
              for file in /data/*; do
                if [ -f "$file" ]; then
                  echo "Uploading $file to S3...";
                  # Replace bucket name with your actual bucket
                  aws s3 cp "$file" s3://my-bucket/backups/;
                  echo "Uploaded. Removing local file.";
                  rm -f "$file";
                fi
              done
              sleep 10;
            done
        volumeMounts:
          - name: staging-area
            mountPath: /data
        # Sidecar reads from same emptyDir and uploads to S3

    volumes:
      - name: staging-area
        emptyDir: {}   # Temporary staging area
  ```

* **Example: App Writes Intermediate Files to emptyDir → Final Output Saved to PVC**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: app-with-scratchpad
  spec:
    containers:
      - name: processor
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Generating temporary files...";
            echo "temp-data" > /scratch/tmp1.txt;

            echo "Processing data...";
            cat /scratch/tmp1.txt | tr a-z A-Z > /output/final-result.txt;

            echo "Done! Final output saved to PVC.";
            sleep 3600;
        volumeMounts:
          - name: scratchpad
            mountPath: /scratch       # temporary intermediate data
          - name: final-storage
            mountPath: /output        # persistent storage (PVC)

    volumes:
      - name: scratchpad
        emptyDir: {}                 # ephemeral scratch volume

      - name: final-storage
        persistentVolumeClaim:
          claimName: output-pvc      # persistent output storage
  ```
* App writes logs → sidecar rotates them in emptyDir → only rotated/compressed logs stored to PVC/S3

* **Example: CI/CD Build Pod Using emptyDir Workspace**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: build-pod
  spec:
    containers:
      - name: builder #Builder container->Clones code,Builds it,Produces artifacts
        image: alpine/git
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Cloning repository...";
            git clone https://github.com/example/app.git /workspace/app;

            echo "Building app...";
            cd /workspace/app;
            touch build-output.txt;  # simulate app build

            echo "Build complete. Workspace is temporary.";
            sleep 3600;
        volumeMounts:
          - name: workspace
            mountPath: /workspace

      - name: packager #Packager container (optional sidecar)-->Reads compiled artifacts,Could push them to:(OCI registry,S3,artifact storage)
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Packaging artifacts...";
            ls /workspace/app;
            sleep 3600;
        volumeMounts:
          - name: workspace
            mountPath: /workspace
        # Sidecar container sees the same workspace

    volumes:
      - name: workspace
        emptyDir: {}   # Temporary build workspace
  ```

* Example: App Using emptyDir to Preserve State Across Container Restarts
  * Scenario:
    * Container keeps crashing and restarting
    * Pod stays alive
    * emptyDir volume retains temporary files (lock files, progress markers)
    * When container comes back up, it can resume or skip processed work

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-recovery-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Starting app...";
          
          # Simulate reading previous state
          if [ -f /state/progress.txt ]; then
            echo "Previous progress: $(cat /state/progress.txt)"
          else
            echo "No previous progress found. Starting fresh."
          fi

          # Simulate processing
          echo "Saving progress..."
          date > /state/progress.txt

          echo "Crashing the container intentionally..."
          exit 1   # This simulates a crash (CrashLoopBackOff)

      volumeMounts:
        - name: recover-state
          mountPath: /state
  volumes:
    - name: recover-state
      emptyDir: {}    # Persists across container restarts
```
* Container restarts
  * The container crashes (exit 1)
  * Kubernetes restarts ONLY the container
  * `emptyDir` survives, so /state/progress.txt is still there
  * The next time the container starts, it sees the previous progress
* Pod restarts
  * If the Pod gets rescheduled or recreated:
  * `emptyDir` is wiped
  * State is lost (by design)


* **Use emptyDir as /dev/shm shared memory for applications that require fast inter-process communication (IPC), such as:**
  * PostgreSQL
  * Chrome / Chromium
  * Machine learning frameworks (PyTorch / TensorFlow)
  * Video processing
  * High-performance apps that need large shared memory
  * **Example: Mount emptyDir (RAM) as /dev/shm**
    * By default, Kubernetes gives a very small /dev/shm (64MB).
    * Using an emptyDir with medium: Memory gives you large, fast shared memory.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: shm-enabled-app
  spec:
    containers:
      - name: app
        image: postgres:14
        # PostgreSQL benefits from a larger /dev/shm
        volumeMounts:
          - name: dshm
            mountPath: /dev/shm

    volumes:
      - name: dshm
        emptyDir:
          medium: "Memory"
          sizeLimit: "1Gi"   # optional limit
  ```

* **Soft Limit vs Hard limit**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: emptydir-size-limit-demo
  spec:
    containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Writing files...";
            dd if=/dev/zero of=/disk-cache/bigfile1 bs=1M count=6000;   # write 6GB
            dd if=/dev/zero of=/ram-cache/bigfile2 bs=1M count=1200;   # write 1.2GB
            sleep 3600

        ## Mount both volumes into the container
        volumeMounts:
          - name: disk-cache
            mountPath: /disk-cache
          - name: ram-cache
            mountPath: /ram-cache

    volumes:

      # --------------------------------------------------------
      # 1️⃣ emptyDir using HOST DISK (medium: "")
      # --------------------------------------------------------
      # ✔ sizeLimit works but is NOT strictly enforced
      # ✔ Kubernetes will TRY to prevent usage above 5Gi
      # ✘ If node runs out of disk → Pod gets EVICTED
      # ✔ Good for temporary files, logs, builds
      #
      # Behavior:
      # - Directory can grow up to ~5Gi (soft limit)
      # - If write exceeds available node storage → Pod eviction
      #
      # Not a hard quota!
      #
      - name: disk-cache
        emptyDir:
          medium: ""          # stored on node filesystem
          sizeLimit: "5Gi"    # soft limit (best-effort)

      # --------------------------------------------------------
      # 2️⃣ emptyDir using RAM (medium: "Memory")
      # --------------------------------------------------------
      # ✔ Uses tmpfs stored in RAM
      # ✔ sizeLimit is STRICTLY enforced by Linux kernel
      # ✔ Writes fail with ENOSPC when exceeding 1Gi
      # ✔ High-speed I/O, great for caches & `/dev/shm`
      #
      # Behavior:
      # - EXACT 1Gi cap enforced
      # - Cannot exceed this limit
      # - Using more RAM may cause OOMKill (container killed)
      #
      # Hard quota!
      #
      - name: ram-cache
        emptyDir:
          medium: "Memory"    # stored in RAM (tmpfs)
          sizeLimit: "1Gi"    # strict hard limit
  ```
* Linux has no per-directory quota
* tmpfs supports _exact_ size limits


---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - name: myvolumes-container-1
    image: alpine
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo "The Bench Container 1 is Running" ; sleep 3600']
    volumeMounts:
    - name: demo-volume
      mountPath: /demo1   # This container sees the shared volume here

  - name: myvolumes-container-2
    image: alpine
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo "The Bench Container 2 is Running" ; sleep 3600']
    volumeMounts:
    - name: demo-volume
      mountPath: /demo2   # This container sees the same shared volume here

  - name: myvolumes-container-3
    image: alpine
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo "The Bench Container 3 is Running" ; sleep 3600']
    volumeMounts:
    - name: demo-volume
      mountPath: /demo3   # Same shared volume in another path

  volumes:
  - name: demo-volume
    emptyDir: {}   # Shared ephemeral storage for all containers in this Pod
```
---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container

    command: ['sh', '-c', 'echo Container 1 is Running ; sleep 3600']

    volumeMounts:
    - mountPath: /demo
      name: demo-volume
  volumes:
  - name: demo-volume
    emptyDir:
      medium: Memory     
```
```bash
kubectl exec myvolumes-pod -i -t -- /bin/sh

/ # df -h
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                   932.3M         0    932.3M   0% /sys/fs/cgroup
tmpfs                   932.3M         0    932.3M   0% /demo
```
* Default behavior:
  * RAM-based emptyDir uses half of node’s RAM
  * Example:
    * Node RAM = 1.8 GB
    * emptyDir tmpfs = ~900 MB

```bash
dd if=/dev/urandom of=/demo/largefile bs=10M count=10 #Write 100MB:

df -h /demo
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   932.3M    100.0M   832.3M   11% /demo

dd if=/dev/urandom of=/demo/largefile bs=10M count=20 #Write another 100MB:
tmpfs                   932.3M    200.0M   732.3M   21% /demo
```
---
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: disk-size-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: ram-cache
      mountPath: /cache
  volumes:
  - name: ram-cache
    emptyDir:
      medium: ""
      sizeLimit: "100Mi"   
```
```bash
controlplane:~$ kubectl exec -it disk-size-demo -c app -- sh
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  18.3G      6.0G     12.3G  33% /
tmpfs                    64.0M         0     64.0M   0% /dev
/dev/vda1                18.3G      6.0G     12.3G  33% /cache
/dev/vda1                18.3G      6.0G     12.3G  33% /etc/hosts
/dev/vda1                18.3G      6.0G     12.3G  33% /dev/termination-log
/dev/vda1                18.3G      6.0G     12.3G  33% /etc/hostname
/dev/vda1                18.3G      6.0G     12.3G  33% /etc/resolv.conf
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                     1.8G     12.0K      1.8G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                   951.7M         0    951.7M   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/interrupts
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/latency_stats
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                   951.7M         0    951.7M   0% /proc/scsi
tmpfs                   951.7M         0    951.7M   0% /sys/firmware
```
```bash
$ dd if=/dev/random of=/cache/demo.txt bs=10M count=100
100+0 records in
100+0 records out
1048576000 bytes (1000.0MB) copied, 2.778655 seconds, 359.9MB/s
```
```bash
$ df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  18.3G      7.0G     11.4G  38% /
tmpfs                    64.0M         0     64.0M   0% /dev
/dev/vda1                18.3G      7.0G     11.4G  38% /cache
/dev/vda1                18.3G      7.0G     11.4G  38% /etc/hosts
/dev/vda1                18.3G      7.0G     11.4G  38% /dev/termination-log
/dev/vda1                18.3G      7.0G     11.4G  38% /etc/hostname
/dev/vda1                18.3G      7.0G     11.4G  38% /etc/resolv.conf
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                     1.8G     12.0K      1.8G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                   951.7M         0    951.7M   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/interrupts
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/latency_stats
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                   951.7M         0    951.7M   0% /proc/scsi
tmpfs                   951.7M         0    951.7M   0% /sys/firmware
```
```bash
$ dd if=/dev/random of=/cache/demo.txt bs=10M count=1000
1000+0 records in
1000+0 records out
10485760000 bytes (9.8GB) copied, 29.092761 seconds, 343.7MB/s
/ # command terminated with exit code 137
```
```bash
controlplane:~$ kubectl exec -it disk-size-demo -c app -- sh
error: cannot exec into a container in a completed pod; current phase is Failed
```
```bash
controlplane:~$ kubectl get pods
NAME             READY   STATUS   RESTARTS   AGE
disk-size-demo   0/1     Error    0          5m27s
```
```bash
controlplane:~$ kubectl describe pod disk-size-demo -- sh
```
```bash
Events:
  Type     Reason     Age    From               Message
  ----     ------     ----   ----               -------
  Normal   Scheduled  5m45s  default-scheduler  Successfully assigned default/disk-size-demo to node01
  Normal   Pulling    5m45s  kubelet            Pulling image "busybox"
  Normal   Pulled     5m44s  kubelet            Successfully pulled image "busybox" in 692ms (692ms including waiting). Image size: 2224358 bytes.
  Normal   Created    5m44s  kubelet            Created container: app
  Normal   Started    5m44s  kubelet            Started container app
  Warning  Evicted    40s    kubelet            Usage of EmptyDir volume "ram-cache" exceeds the limit "100Mi".
  Normal   Killing    40s    kubelet            Stopping container app
```
