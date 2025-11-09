### References:-
- https://www.youtube.com/watch?v=miZl-7QI3Pw&ab_channel=CloudWithVarJosh

---

![Alt text](/images/22c.png)

- Container image should be pulled from the container registry (like Docker Hub or a private registry).

- Kubectl apply -f pod.yaml ---> API-server ---> Node-Kubelet ---> CRI ---> PUll ---> REGISTRY(specified one, if not specefied then from default(docker registry)).

- The `imagePullPolicy` in Kubernetes specifies how the container runtime pulls container images for pods. 

- It governs whether Kubernetes should pull the image from a container registry or use a cached version already present on the node.

- This setting is vital for controlling how your deployments behave in development, testing, and production environments. 

- Requirement:-
  - Registry credential --> K8S Secrets ---> K8S use them to authenticate to the image registry and pull image(CRI use).
  - Image ---> registry/namespace/account/repository:tag

- Default `imagePullPolicy` depend on the image tag.

---

### **1. Always**
  - Always pull image from registry(not check local cache), Always pull latest image.
  - Pull on every pod start.
  - This ensures that the most recent version of the image is used.
  - Usefull in **development environments** to ensure you're always working with the latest image changes.
  - **Avoid in production**, since it causes unnecessary image pulls, bandwidth consumption, and inconsistent behavior.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: always-example
  spec:
    containers:
    - name: my-app
      image: myrepo/my-app:latest
      imagePullPolicy: Always
  ```

---

### **2. IfNotPresent**
  - The `IfNotPresent` policy directs Kubernetes to pull the container image from the registry **only if the image is not already available on the node/cache**.
  - If the image is present on the node's local cache, Kubernetes uses it without pulling a new version.
  - Use `IfNotPresent` in **production environments** to avoid unnecessary image pulls, which saves network bandwidth, reduces registry load and improves pod startup times.
  - Use when pull image with specific tag.
  - Stable, versioned images (e.g., `myapp:v1.2.3`).
   * **Risk:**
     If multiple developers push different images **with the same tag**, nodes might pull inconsistent versions — leading to mismatched application behavior across nodes.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: ifnotpresent-example
  spec:
    containers:
    - name: my-app
      image: myrepo/my-app:v1.2.3
      imagePullPolicy: IfNotPresent
  ```
  
---

### **3. Never**
* Kubernetes **never pulls the image** from a registry.
* The image **must already exist locally** on the node; otherwise, Pod creation fails.
* Useful in **air-gapped**  where images are preloaded on all nodes.
* **When to Use:**  
  - Use `Never` in **air-gapped environments or offline environments** or scenarios where nodes are preloaded with container images and external image pulls are not allowed.
  - Suitable for testing environments where you explicitly preload images manually.

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: never-example
spec:
  containers:
  - name: my-app
    image: myrepo/my-app:v1.2.3
    imagePullPolicy: Never
```

* In this case, Kubernetes will attempt to use the `my-app:v1.2.3` image from the node’s local cache. If the image is unavailable, the pod fails with an error.

---

* **Default Behavior**

| Image Tag Type                | Default `imagePullPolicy` | Behavior                         |
| ----------------------------- | ------------------------- | -------------------------------- |
| `:latest`                     | `Always`                  | Always pull the latest version   |
| Custom Tag (e.g., `:v1.0.0`)  | `IfNotPresent`            | Pull only if not already present |
| No Tag (treated as `:latest`) | `Always`                  | Always pull the latest version   |

 
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
    imagePullPolicy: IfNotPresent #Always, Never and IfNotPresent
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

---

### **Key Considerations**
1. **Avoid Using `latest` Tag in Production:**  
   Using `Always` with `latest` in production can lead to unpredictable behavior if the image changes without notice. Instead, use specific version tags (`v1.2.3`) for stability.

2. **Preload Images for Faster Startup:**  
   In air-gapped environments or production, preload images on nodes to reduce reliance on the registry. Combine `imagePullPolicy: Never` with image preloading for tight control.

3. **Monitor Bandwidth Usage:**  
   Frequent image pulls with `Always` can increase bandwidth consumption and impact cluster performance. Use `IfNotPresent` or `Never` where appropriate to optimize.

---

#### The Risk — Tag Collisions and Inconsistent Deployments

* In Docker and Kubernetes, the **image tag** (like `v1.0.1` or `latest`) is **just a label** — not a unique identifier.
* When developers **push images with the same tag name** to the container registry (for example, Docker Hub, ECR, or GCR), the **old image is overwritten**.
* This leads to **different nodes running different image contents** — even though the tag appears identical.

<br>

* **Example of the Problem**

    1. Developer **Varun** builds an image:
        ```bash
        myapp:v1.0.1
        ```
    and pushes it to the registry.
        ```bash
        docker push myrepo/myapp:v1.0.1
        ```

    2. Later, developer **Kavita** also builds an updated image — but accidentally **uses the same tag**:
        ```bash
        docker push myrepo/myapp:v1.0.1
        ```
        This overwrites Varun’s image in the registry.
    3. Now, when your **Kubernetes cluster** pulls the image:
        * Node 1 (earlier deployment) already has **Varun’s version** cached.
        * Node 2 (new deployment) pulls **Kavita’s version** from the registry.
    * Both nodes **run different code**, but **the tag looks identical (`v1.0.1`)**.


* **Consequences**
    * **Inconsistent behavior across pods/nodes** → different users may experience different app versions.
    * **Hard to debug** → same tag, different code.
    * **Deployment rollback issues** → you can’t reliably roll back, because the tag doesn’t point to a known image anymore.
    * **Security risk** → overwritten tags could introduce unauthorized or unreviewed code into production.

<br>

* **The Solution — Enforce Unique, Immutable Image Versioning**
    * To prevent this, you must **guarantee that each image tag represents a unique build** and that **tags cannot be overwritten**.

1. **Use Immutable Tags**
    * Use **Semantic Versioning** (`Major.Minor.Patch`), e.g. `v1.0.0`, `v1.0.1`, `v1.1.0`.
    * Each build should produce a **new tag** — never reuse old ones.
    * Once pushed, that tag should **never be updated** or overwritten.


2. **Use Git-Based or Build-Based Tags**
    * Automate your CI/CD pipeline (e.g., Jenkins, GitHub Actions, GitLab CI) to **generate tags automatically**.
    * Common patterns:
    * Use **Git commit SHA**:
        ```bash
        myapp:<git-sha>
        ```
        Example:
        ```bash
        myapp:7a2b9c1
        ```
    * Use **build number** or **timestamp**:
        ```bash
        myapp:build-102
        myapp:2025-10-05_14-20
        ```
    This ensures each image corresponds exactly to a specific source code state.



3. **Enforce Registry-Level Protection**
    * Many registries (like **AWS ECR**, **Harbor**, **JFrog Artifactory**, **GitLab Container Registry**) support **immutable tags**.
    * You can **enable a policy** that prevents pushing an image with an existing tag.
    * Example (ECR):
        ```bash
        aws ecr put-image-tag-mutability --repository-name myapp --image-tag-mutability IMMUTABLE
        ```
    * Once set, trying to push an existing tag again will fail — protecting your production environment.

4. **Automate Tag Validation**
* Add a pre-push script or CI pipeline check:
  * Before pushing a new image, check if that tag already exists.
  * If it does, fail the build and alert the developer.
  * Example logic (pseudo-code):
    ```bash
    if docker manifest inspect myrepo/myapp:v1.0.1 >/dev/null 2>&1; then
        echo "❌ Tag v1.0.1 already exists. Choose a new version."
        exit 1
    fi
    ```

5. **Use SHA Digests for Absolute Uniqueness**
    * Even if tags are reused, you can always refer to an **image digest**, which uniquely identifies an image:
    ```bash
    myrepo/myapp@sha256:5d41402abc4b2a76b9719d911017c592
    ```
    * Digests are **immutable** — two images can never share the same digest.

6. **Kubernetes Best Practice**
    * In your Kubernetes YAML manifests:
    * Use **versioned tags** (not `latest`).
    * Optionally, use **image digests** for full consistency.
    * Example:
    ```yaml
    containers:
    - name: myapp
        image: myrepo/myapp:v1.0.3
        imagePullPolicy: IfNotPresent
    ```
    * or, even safer:
    ```yaml
    containers:
    - name: myapp
        image: myrepo/myapp@sha256:5d41402abc4b2a76b9719d911017c592
        imagePullPolicy: IfNotPresent
    ```

---

### **Version Tagging Best Practices**
* Use **semantic versioning**:
  `Major.Minor.Patch` → e.g., `1.2.3`
  * **Major:** Breaking changes
  * **Minor:** New features (backward-compatible)
  * **Patch:** Bug fixes
* Avoid reusing tags (like `latest`) in production — it causes unpredictable deployments.
* Automate CI/CD pipelines to **block overwriting existing tags** in the registry.

---

### Command issue 
- `ImagePullBackOff` BackOff ---> K8S keep attempting to pull the image, with an increasing “back-off“ delay.
  - invalid image name
  - Pulling from a private registry without an imagePullSecret or using an incorrect imagePullSecret, etc. 