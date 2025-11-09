### References:
- https://spacelift.io/blog/kubectl-patch-command

---

* We create a resources in YAML deaclarative config.
* There a controller loops that monitors the desire state and actual state and make sure actual state is always equal to desire state.
* Resources include Pods, Deployment, Replica Sets, Config Maps, Secrets, Ingress, Persistent Volumes, Persistent Volume Claims, Services, Jobs/CronJobs, and more.
* To change in the config of existing resources  we uses `kubectl patch` command without disrupting the running services and preventing you from recreating your YAML file. 
* This process reduces the risk of accidentally overwriting or deleting other parts of the configuration because it is going to give an error if you try to update a wrong field
    ```bash
    controlplane:~$ kubectl patch pod nginx-deployment-5486bd6646-997qh --patch '{"spec":{"template":{"metadata":{"labels":{"environment":"dev"}}}}}'
    Warning: unknown field "spec.template"
    pod/nginx-deployment-5486bd6646-997qh patched (no change)
    ```
* With kubectl patch, you can quickly fix issues with updating the name, image, label, replicas, affinity/tolerations, environment variables, configaps, secrets, volumes, ports, etc.

* kubectl patch supports YAML and JSON formats. Using either format, you can drill into the specific field of the resource and change it to your desired value.


```bash
kubectl patch <resource-type> <resource-name> --patch '<patch-data>'
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    environment: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        environment: dev
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
```bash
kubectl patch deploy nginx-deployment --patch '{"metadata":{"labels":{"environment": "prod"}}}'
#metadata-->labels-->environment

kubectl patch pod nginx-deployment-5486bd6646-997qh --patch '{"spec":{"template":{"metadata":{"labels":{"environment":"dev"}}}}}'
#spec-->template-->metadata-->labels-->environment

kubectl patch deploy nginx-deployment --patch '{"spec":{"template":{"metadata":{"labels":{"environment":"dev","demo":"hello"}}}}}' #adding label

kubectl get pods --selector=app=nginx --show-labels
kubectl get pods --selector=environment=dev --show-labels
```

* Kubectl supports three different patching strategies: Patch types:
    * Strategic Merge Patch → default (smart merge, Kubernetes-aware)
    * JSON Merge Patch → simpler merge, overwrites lists
    * JSON Patch (RFC 6902) → granular, operation-based control

---

#### **Strategic Merge Patch (default)**
* Default patch type (`--type` not needed)
* Kubernetes-aware: understands structures like container lists, metadata.labels, etc.
* Merges only specified fields and keeps the rest intact.
* Best for modifying containers, labels, selectors, node selectors, etc.
* Strategic Merge Patch is applied:
    * Kubernetes intelligently merges only the specified parts of the resource.
    * The patch recognizes arrays and maps, especially in fields like containers, env, and ports.
    * It matches the container by its name ("apache-container") and modifies only that container’s fields.
* Existing fields are preserved:
    * Any other parts of the deployment (like annotations, sidecar containers, ports, etc.) remain untouched.
* New fields are added:
    * The resources section didn’t exist before — it’s now created.
* Pods are automatically updated:
    * The Deployment controller detects a change in the Pod template.
    * It performs a rolling update to recreate Pods with the new resource limits, ensuring zero downtime.

<br>

* **Update image container**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
    labels:
      app: apache
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: apache
      template:
        metadata:
          labels:
            app: apache
          annotations:
            team: devops
        spec:
          containers:
          - name: apache-container
            image: httpd:2.4
            ports:
            - containerPort: 80
          - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```
    This will require us to drill further into the list (the - is a list and will use [ ] to tap into it from the kubectl patch command) instead of a map (uses { }):

    ```bash
    kubectl patch deployment apache-deployment --patch '{"spec": {"template": {"spec": {"containers": [{"name": "apache-container", "image": "httpd:2.4.62"}]}}}}'
    ```
    ```bash
    kubectl get deployment apache-deployment -o yaml
    ```

    We get the following YAML file:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
      labels:
        app: apache
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: apache
      template:
        metadata:
          labels:
            app: apache
          annotations:
            team: devops
        spec:
          containers:
          - name: apache-container
            image: httpd:2.4.62 ##UPDATED
            ports:
            - containerPort: 80
          - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```

* **Update Pod label and container image**

    We can also perform multiple updates at the same time. For example, if we want to add a new label to the Pod for the environment: prod and update the image to httpd:2.4.62, we can run the following:

    ```bash
    kubectl patch deployment apache-deployment --patch '{"spec": {"template": {"metadata": {"labels": {"environment": "production"}}, "spec": {"containers": [{"name": "apache-container", "image": "httpd:2.4.62"}]}}}}'
    ```
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
      labels:
        app: apache
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: apache
      template:
        metadata:
          labels:
            app: apache
            environment: production ##ADDED
          annotations:
            team: devops
        spec:
          containers:
          - name: apache-container
            image: httpd:2.4.62 ##UPDATED
            ports:
            - containerPort: 80
          - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```

* **Add a Node Selector**

    The Node Selector will allow us to schedule these pods on Nodes that have the label zone:centralus, giving us more control over the placement of our resources:
    ```bash
    kubectl patch deployment apache-deployment --patch '{"spec": {"template": {"spec": {"nodeSelector": {"zone": "centralus"}}}}}'
    ```
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
      labels:
        app: apache
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: apache
      nodeSelector: ##ADDED
        zone: centralus
      template:
        metadata:
          labels:
            app: apache
          annotations:
            team: devops
        spec:
          containers:
          - name: apache-container
            image: httpd:2.4
            ports:
            - containerPort: 80
          - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```

* **Update resource requests and limits**
    ```bash
    kubectl patch deployment apache-deployment --patch \
    '{"spec": {"template": {"spec": {"containers": [{"name": "apache-container", "resources": {"requests": {"cpu": "500m", "memory": "256Mi"}, "limits": {"cpu": "1", "memory": "512Mi"}}}]}}}}'
    ```
    Breakdown of the JSON Patch
    ```json
    {
    "spec": {
        "template": {
        "spec": {
            "containers": [
            {
                "name": "apache-container",
                "resources": {
                "requests": {
                    "cpu": "500m",
                    "memory": "256Mi"
                },
                "limits": {
                    "cpu": "1",
                    "memory": "512Mi"
                }
                }
            }
            ]
        }
        }
    }
    }
    ```
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
      labels:
        app: apache
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: apache
      template:
        metadata:
          labels:
            app: apache
          annotations:
            team: devops
        spec:
          containers:
          - name: apache-container
            image: httpd:2.4
            resources:
              requests:
                cpu: 500m
                memory: 256Mi
              limits:
                cpu: 1
                memory: 512Mi
            ports:
            - containerPort: 80
          - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```

* **Add Tolerations**
    Through Taints and Tolerations, we can also avoid placing our pods in specific Nodes. This allows us to reserve certain nodes for specific workloads or prevent pods from being scheduled on nodes. 
    Allow (tolerate) the Apache Deployment pods to be scheduled on nodes that have the taint app=nginx:NoSchedule.
    Result: Normally, a node tainted with app=nginx:NoSchedule would reject all Pods that don’t tolerate it.
    By adding this toleration, you’re allowing the Apache pods to run on those tainted nodes.
    ```bash
    kubectl patch deployment apache-deployment --patch \
    '{"spec": {"template": {"spec": {"tolerations": [{"key": "app", "operator": "Equal", "value": "nginx", "effect": "NoSchedule"}]}}}}'
    ```
    ```json
    "tolerations": [
    {
        "key": "app",
        "operator": "Equal",
        "value": "nginx",
        "effect": "NoSchedule"
    }
    ]
    ```
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: apache-deployment
    labels:
        app: apache
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: apache
    template:
        metadata:
        labels:
            app: apache
        annotations:
            team: devops
        spec:
        tolerations:   ## ADDED
        - key: app
            operator: Equal
            value: nginx
            effect: NoSchedule
        containers:
        - name: apache-container
            image: httpd:2.4
            ports:
            - containerPort: 80
        - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```
    ```bash
    kubectl taint nodes node1 app=nginx:NoSchedule
    ```
    Kubernetes understands the structure of the resource (tolerations is a list).

    It merges the patch intelligently, only updating the Pod template’s spec.

    It doesn’t replace the entire container section, annotations, or labels.

    The Deployment automatically handles updating the Pods safely.

    Deployement trigger a rolling update and update the pod one by one.

* Advantages of Using kubectl patch for Tolerations
    * No downtime: Rolling update ensures the app keeps running.
    * Minimal effort: One command, no need to edit YAML files manually.
    * Safe updates: Other configurations stay intact.
    * Instant application: Pods are re-created automatically with new specs.
    

Excellent — your entire explanation of the **JSON Merge Patch** method is fundamentally correct and well-written.
To give you the **full, detailed, and technically precise version**, I’ll expand upon what you’ve written — explaining not only *what happens*, but also *why* it happens internally inside Kubernetes and the API server.

---

#### **JSON Merge Patch — Full Detailed Explanation**
* **JSON Merge Patch** is a patching strategy defined by [RFC 7386](https://datatracker.ietf.org/doc/html/rfc7386).
* It’s a **simpler** patching method than the Strategic Merge Patch because it operates purely at the **JSON object level**, without understanding Kubernetes’ internal structures like container names, lists, or keys.
* When you apply a patch of type `--type=merge`, Kubernetes takes the **original resource object** (in JSON form) and merges your patch data on top of it, **replacing any fields** you specify.

**Key Behavior**

* It works by **replacing** the fields or objects you define.
* If you include a field in the patch → it **replaces** the current field.
* If you omit a field → it **remains unchanged**, *except in arrays/lists*.
* If you set a field to `null` → it **deletes** that field.
* For lists (like `containers`, `env`, or `ports`) → **you must re-specify all items** in the list. Otherwise, Kubernetes assumes you want to remove the missing ones.

This is the biggest difference compared to **Strategic Merge Patch**, which *intelligently merges* based on keys like `"name"` in containers.

* **Syntax**
    ```bash
    kubectl patch <resource-type> <resource-name> \
    --type=merge \
    --patch '<json-patch-data>'
    ```

<br>

* **Update Container Image**
    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"spec":{"template":{"spec":{"containers":[{"name":"apache-container","image":"httpd:2.4.62"}]}}}}'
    ```
    * Because JSON Merge Patch **replaces entire lists**, the `containers` list here contains **only one entry** — the `apache-container`.
    * Kubernetes will remove any containers that are not listed (e.g., `sidecar`).

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: apache-deployment
    spec:
    template:
        spec:
        containers:
        - name: apache-container
            image: httpd:2.4.62
            ports:
            - containerPort: 80
    ```
    **Note:** The `sidecar` container is gone because it was not included.

<br>

* **Add an Environment Variable**
If you want to **preserve** the sidecar container, you must explicitly include it again:

    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"spec":{"template":{"spec":{"containers":[{"name":"apache-container","image":"httpd:2.4","env":[{"name":"FOO_API_INSTANCE","value":"127.0.0.1:3245"}]},{"name":"sidecar","image":"fluent/fluent-bit:1.9.5"}]}}}}'
    ```
    The patch now replaces the container list with **two containers** — both `apache-container` and `sidecar`.
    The `apache-container` gains a new `env` variable.

    ```yaml
    spec:
    template:
        spec:
        containers:
        - name: apache-container
            image: httpd:2.4
            env:
            - name: FOO_API_INSTANCE
            value: 127.0.0.1:3245
        - name: sidecar
            image: fluent/fluent-bit:1.9.5
    ```
    Both containers remain because both were included.

* **Delete a Field**
If you want to **remove a field**, you set it to `null`.

    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"metadata":{"annotations":{"team":null}}}'
    ```

    * The `annotations.team` field is deleted from metadata.
    * Other annotations (if any) remain untouched.


    ```yaml
    metadata:
    annotations: {}
    ```

    Simpler and efficient for deleting specific fields.
    If you set an entire object to `null`, everything inside it gets removed.

<br>

* **Delete a List Item (Environment Variable)**
To remove an item from a list (like `env`), you **must redefine the entire list** without that item.

    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"spec":{"template":{"spec":{"containers":[{"name":"apache-container","image":"httpd:2.4"},{"name":"sidecar","image":"fluent/fluent-bit:1.9.5"}]}}}}'
    ```

* The new patch redefines the `containers` list, omitting the `env` field entirely.
* Kubernetes removes the environment variable section.

    ```yaml
    containers:
    - name: apache-container
    image: httpd:2.4
    - name: sidecar
    image: fluent/fluent-bit:1.9.5
    ```

    The environment variable is removed.
    The list is **fully replaced**.


* **Clear Resource Limits**
Remove `limits` but keep `requests`.

    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"spec":{"template":{"spec":{"containers":[{"name":"apache-container","image":"httpd:2.4","resources":{"requests":{"cpu":"500m","memory":"256Mi"},"limits":null},"ports":[{"containerPort":80}]},{"name":"sidecar","image":"fluent/fluent-bit:1.9.5"}]}}}}'
    ```
* `limits` is explicitly set to `null` → field deleted.
* `requests` remain.
* Sidecar container remains because it’s included.

    ```yaml
    containers:
    - name: apache-container
    image: httpd:2.4
    resources:
        requests:
        cpu: 500m
        memory: 256Mi
    - name: sidecar
    image: fluent/fluent-bit:1.9.5
    ```
    Resource limits cleared successfully.

* Summary of JSON Merge Patch Characteristics

| Feature               | Description                                              |
| --------------------- | -------------------------------------------------------- |
| **Patch type flag**   | `--type=merge`                                           |
| **RFC reference**     | RFC 7386                                                 |
| **Merging behavior**  | Shallow merge; replaces objects directly                 |
| **Array handling**    | Replaces entire arrays; must specify all items           |
| **Field deletion**    | Set field to `null`                                      |
| **When to use**       | Full overwrites, quick deletions, simple updates         |
| **When *not* to use** | Partial list updates (use Strategic Merge Patch instead) |

---

* **Best Practices**
    * Always **backup or export** (`kubectl get -o yaml`) your resource before applying a merge patch.
    * When patching lists, always **include all items** to avoid accidental data loss.
    * For small, atomic changes → **Strategic Merge Patch** is safer.
    * For complete overwrites or field deletions → **JSON Merge Patch** is ideal.
    * For advanced JSON-level edits (like moving or adding specific list items), use **JSON Patch (`--type=json`)**, which uses the RFC 6902 operation model (`add`, `replace`, `remove`).


---

#### **JSON Patch (RFC 6902) — Deep Explanation**

* JSON Patch is a **precise, operation-based patching method** standardized in [RFC 6902].
* It operates on the **exact JSON structure** of a Kubernetes object, not YAML (though YAML is just JSON under the hood).
* You must specify `--type=json` for Kubernetes to interpret your patch operations correctly.

Each patch is a **list of operation objects**, each having:

```json
{
  "op": "operation",
  "path": "/json/pointer",
  "value": "new_value"
}
```

or sometimes `"from": "/source/path"` for `copy` and `move`.

* Supported Operations (and how K8s interprets them)

| Operation   | Description                                                   | Example                                         |
| ----------- | ------------------------------------------------------------- | ----------------------------------------------- |
| **add**     | Adds a new key/value or list item.                            | Add new container port, env var, or annotation. |
| **remove**  | Deletes a field or list element at the given path.            | Remove an annotation or port.                   |
| **replace** | Replaces an existing value. Fails if the path does not exist. | Update an image tag.                            |
| **move**    | Removes a value from `from` path and adds it to `path`.       | Move a port from one container to another.      |
| **copy**    | Copies a value from one path to another.                      | Duplicate env vars or ports between containers. |
| **test**    | Ensures a field matches a given value before proceeding.      | Conditional patching to avoid errors.           |



* Key Technical Behavior
* JSON Patch **operates at exact JSON path levels** (e.g., `/spec/template/spec/containers/0/image`).
  If that path doesn’t exist, the operation fails — unlike Strategic Merge Patch, which can create missing fields.
* Indexes in lists are **zero-based** (`/containers/0` = first container).
* When adding to the end of a list, use `"-"` as the index (e.g., `/ports/-`).

<br>

* **Realistic Examples Recap**

    * **Replace container image**

        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"httpd:2.4.62"}]'
        ```
        → Safely updates image without touching other fields.

    * **Add container port**
        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"add","path":"/spec/template/spec/containers/0/ports/-","value":{"containerPort":443}}]'
        ```
        → Adds new port entry at the end of ports list.

    * **Remove container port**
        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"remove","path":"/spec/template/spec/containers/0/ports/1"}]'
        ```
       → Removes the port at index 1 (the second port).
    
    * **Test + Replace annotation**
        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"test","path":"/metadata/annotations/team","value":"devops"},{"op":"replace","path":"/metadata/annotations/team","value":"platform"}]'
        ```
       → Replaces only if existing annotation equals `"devops"` — otherwise fails.

    * **Move field**
        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"move","from":"/spec/template/spec/containers/0/ports/1","path":"/spec/template/spec/containers/1/ports"}]'
        ```
       → Moves port 443 from apache-container to sidecar.

    * **Copy field**
        ```bash
        kubectl patch deployment apache-deployment \
        --type=json \
        --patch '[{"op":"copy","from":"/spec/template/spec/containers/0/ports","path":"/spec/template/spec/containers/1/ports"}]'
        ```
       → Duplicates ports list to sidecar container.


---

### ⚙️ Core Difference — `kubectl patch` vs `kubectl apply`

| Feature           | `kubectl patch`                                             | `kubectl apply`                                        |
| ----------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
| **Purpose**       | Make *direct, specific* changes to an existing live object  | Maintain a *declarative desired state* from a manifest |
| **Patch Type**    | Uses JSON Merge Patch, Strategic Merge Patch, or JSON Patch | Uses 3-way merge with last-applied-configuration       |
| **Change Scope**  | Updates only the fields you specify                         | Reconciles *entire object* with the manifest           |
| **Best Use Case** | Quick fixes, automation, debugging                          | GitOps, CI/CD, Infrastructure-as-Code                  |
| **Persistence**   | Does **not** update your YAML file                          | Updates reflect in version control (manifest)          |
| **Risk**          | Manual patches can be overwritten by `apply`                | Can revert patches not in the manifest                 |

---

### 🧩 Technical Explanation

#### `kubectl patch`

* It directly modifies the **live resource** stored in etcd via the Kubernetes API server.
* You can choose your patch type:

  * **Strategic Merge Patch (default)** – merges fields intelligently (e.g., merges `containers` by name).
  * **JSON Merge Patch** – replaces entire sections of JSON where changes are made.
  * **JSON Patch (RFC 6902)** – performs precise, operation-based updates (add/replace/remove/test/move/copy).
* Changes **take effect immediately** and cause a rollout (if the change affects the Pod template).
* However, **these changes are not reflected in your YAML file**, so the next time you use `kubectl apply`, those changes can be lost.

---

#### `kubectl apply`

* It compares three versions of the resource:

  1. The **live** object currently in the cluster.
  2. The **last-applied-configuration** annotation (previous manifest state).
  3. The **new manifest file** you are applying.
* It then performs a **three-way merge** to bring the live object into the desired state defined by your manifest.
* It is ideal for **declarative management** — your YAML file becomes the *single source of truth*.
* Any field missing in the manifest gets **reset or removed** from the live resource.

---

### 🧠 Example Deep Dive

You gave the perfect example with `apache-deployment` 👇

#### Initial manifest (`httpd:2.4`)

```yaml
containers:
- name: apache-container
  image: httpd:2.4
```

Then you run:

```bash
kubectl patch deployment apache-deployment \
--type=json \
--patch '[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "httpd:2.4.62"}]'
```

Now your live object in the cluster has:

```yaml
image: httpd:2.4.62
```

But your YAML file still says:

```yaml
image: httpd:2.4
```

So later, if someone runs:

```bash
kubectl apply -f apache-deployment.yaml
```

Kubernetes sees that the manifest “desired state” says `2.4`, so it rolls back your live resource — removing your `patch` changes — because `apply` reconciles the live state to match the manifest.

That’s why we say:

> 🟠 “`kubectl apply` enforces declarative state.”
> 🔵 “`kubectl patch` performs imperative ad-hoc modifications.”

---

* **In CI/CD and GitOps Pipelines**

* **Best practice:**

  * Update manifests → commit to Git → deploy via `kubectl apply` (or via ArgoCD, Flux, etc.).
  * This ensures all environments stay in sync and you can roll back by reverting commits.
* **When to use `kubectl patch`:**

  * Temporary or automated tweaks (e.g., bumping image tags during rollout testing).
  * Scripting in maintenance tasks, CI/CD hooks, or debugging live clusters.
  * Emergency fixes — when you can’t immediately update the manifest repository.

---

* **Common Mistake**

Many beginners do:

```bash
kubectl patch ...   # change live object
kubectl apply -f manifest.yaml   # later reapply manifest
```

→ Result: **The live patch is overwritten.**

Because `apply` doesn’t “merge” your manual patch unless it’s also defined in the YAML manifest.

---

* **Practical Summary**

| Use Case                                | Recommended Command                     |
| --------------------------------------- | --------------------------------------- |
| Quick one-time fix (e.g., label, image) | `kubectl patch`                         |
| Long-term state management              | `kubectl apply`                         |
| CI/CD / GitOps                          | `kubectl apply` (manifest in Git)       |
| Testing a change before committing      | `kubectl patch`, then update YAML later |
| Automation scripts                      | `kubectl patch` (fast, no files needed) |

---

#### **Core Difference — `kubectl edit` vs `kubectl patch`**

    | Feature            | `kubectl edit`                                                | `kubectl patch`                                                     |
    | ------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------- |
    | **Purpose**        | Interactive, manual editing of the entire live resource       | Programmatic, targeted modification of specific fields              |
    | **Scope**          | Entire resource — you see all fields                          | Only the fields you specify in the patch                            |
    | **Validation**     | Relies on user to input correct YAML/JSON                     | Validates paths and structure automatically depending on patch type |
    | **Best Use Case**  | Exploratory changes, debugging, multiple simultaneous updates | Safe, granular, repeatable updates; scripting/automation            |
    | **Rollout Effect** | Changes trigger rollout if Pod template is modified           | Changes trigger rollout if Pod template is modified                 |
    | **Risk**           | Higher risk of accidental misconfiguration                    | Lower risk — only specified fields are affected                     |
    | **Ease of Use**    | Easy for interactive, multi-field updates                     | Requires knowledge of JSON/YAML paths for precise updates           |


* **How They Work**

* **`kubectl edit`**
    * Opens the resource in your default editor (vi/nano/emacs, depending on your system configuration).
    * You can modify any part of the resource.
    * After saving:
        * Kubernetes validates basic syntax.
        * Any changes to `spec.template` trigger a rolling update of Pods.
    * Useful for:
        * Adding multiple labels or annotations at once.
        * Updating several container fields simultaneously.
        * Quick debugging or exploratory changes.

* **`kubectl patch`**
    * Sends a **specific change** directly to the Kubernetes API server.
    * Supports:
        * **Strategic Merge Patch (default)** — merges intelligently based on keys.
        * **JSON Merge Patch** — replaces entire fields.
        * **JSON Patch (RFC 6902)** — fine-grained operations like add/remove/replace/test.
    * Only affects the fields specified in the patch.
    * Safer for automation or scripts because it reduces human error.

* **Example Comparison**

    **Original YAML**
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: apache-deployment
    labels:
        app: apache
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: apache
    template:
        metadata:
        labels:
            app: apache
        annotations:
            team: devops
        spec:
        containers:
        - name: apache-container
            image: httpd:2.4
    ```

* **Using `kubectl edit`**
    Change image version and add an annotation:
    ```yaml
    containers:
    - name: apache-container
    image: httpd:2.4.62
    metadata:
    annotations:
        team: devops
        owner: team-a
    ```
    * Pros:
      * Easy to do multiple edits at once.
      * Full visibility of the resource.
    * Cons:
      * Manual errors can break the resource.
      * Not ideal for automation.

<br>

* **Using `kubectl patch`**
Same changes using JSON Merge Patch:

    ```bash
    kubectl patch deployment apache-deployment \
    --type=merge \
    --patch '{"spec": {"template": {"spec": {"containers": [{"name": "apache-container", "image": "httpd:2.4.62"}]}}}, "metadata": {"annotations": {"owner": "team-a"}}}'
    ```

    * Pros:
        * Programmatic, can be used in scripts or CI/CD.
        * Only changes specified fields, reducing accidental mistakes.
    * Cons:
        * Requires precise knowledge of the paths to target.
        * Updating multiple fields can get verbose.

<br>

* **Key Takeaways**

* Use **`kubectl edit`** when:

  * You want **interactive, multiple changes**.
  * Doing quick exploratory modifications or debugging live resources.
  * You are comfortable manually editing YAML and validating changes.

* Use **`kubectl patch`** when:

  * You want **precise, repeatable, ad-hoc changes**.
  * Integrating changes into automation, scripts, or CI/CD pipelines.
  * Minimizing risk of accidentally modifying unrelated fields.

* **Rollout behavior**:

  * Both trigger a new pod rollout if the `spec.template` of the Deployment changes (e.g., container image, resources, or environment variables).

